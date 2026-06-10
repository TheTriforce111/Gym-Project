# 🚀 Phase 13 — Deployment Guide

Complete production deployment for the GymOS system across all components.

---

## 📁 What's in This Package

```
gym-deployment-phase13/
├── docker-compose.prod.yml        ← Full production stack (all 6 services)
├── docker-compose.monitoring.yml  ← Optional: Prometheus + Grafana
├── .env.prod.example              ← Copy this → .env.prod and fill in secrets
│
├── docker/
│   └── Dockerfile.api             ← Multi-stage production Docker image
│
├── nginx/
│   ├── nginx.conf                 ← Main nginx configuration (production-hardened)
│   └── sites/
│       └── api.conf               ← API + admin dashboard + SSL configuration
│
├── scripts/
│   ├── server-setup.sh            ← Run ONCE on fresh Ubuntu server
│   ├── deploy.sh                  ← Zero-downtime deploy script
│   ├── backup.sh                  ← Automated PostgreSQL backup + S3 upload
│   └── db-init.sql                ← Database extensions + performance indexes
│
├── ci-cd/
│   └── github-actions.yml         ← Full CI/CD pipeline (test → build → deploy)
│
└── terraform/
    └── aws/
        └── main.tf                ← AWS infrastructure as code (EC2, RDS, Redis, ALB)
```

---

## 🏗️ Architecture

```
Internet
    │
    ▼
┌─────────────────────────────────────────────┐
│  AWS / VPS                                  │
│                                             │
│   Nginx (80/443)                            │
│     ├── /api/*         → FastAPI × 2        │
│     ├── /admin/*       → React Dashboard    │
│     └── /flower/*      → Celery Monitor     │
│                                             │
│   FastAPI × 2          (Gunicorn workers)   │
│   Celery Worker        (background tasks)   │
│   Celery Beat          (cron scheduler)     │
│     │                                       │
│     ├── PostgreSQL     (persistent data)    │
│     └── Redis          (tokens, queues)     │
└─────────────────────────────────────────────┘
```

---

## 📋 Deployment Options

### Option A: Single VPS (Simplest — recommended for start)

**Best for:** < 500 members, budget-conscious, getting started

**Requirements:** Ubuntu 22.04, 2 vCPU, 4GB RAM, 40GB SSD

**Providers:** DigitalOcean ($24/mo), Hetzner ($8/mo), Linode ($18/mo)

```bash
# 1. SSH into your new server
ssh root@YOUR_SERVER_IP

# 2. Run the setup script
curl -fsSL https://raw.githubusercontent.com/your-org/gym-backend/main/scripts/server-setup.sh | bash

# 3. Clone and configure
git clone https://github.com/your-org/gym-backend.git /opt/gymapp
cd /opt/gymapp
cp .env.prod.example .env.prod
nano .env.prod   # Fill in all values

# 4. Get SSL certificate
certbot certonly --standalone -d api.yourgym.com
# Update nginx/sites/api.conf with your domain

# 5. Start everything
docker compose -f docker-compose.prod.yml up -d

# 6. Verify
curl https://api.yourgym.com/health
```

---

### Option B: AWS (Scalable — recommended for growth)

**Best for:** > 500 members, auto-scaling, managed databases

**Cost:** ~$75/month (EC2 + RDS + ElastiCache + ALB)

```bash
# Prerequisites
# - AWS account
# - Terraform installed: https://developer.hashicorp.com/terraform/install
# - AWS CLI configured: aws configure

# 1. Configure variables
cd terraform/aws
cp terraform.tfvars.example terraform.tfvars
nano terraform.tfvars   # Set domain, passwords, key pair

# 2. Initialize Terraform
terraform init

# 3. Preview what will be created
terraform plan

# 4. Create infrastructure
terraform apply
# Takes ~10 minutes to provision RDS and ElastiCache

# 5. SSH into the EC2 instance
ssh -i your-key.pem ubuntu@$(terraform output -raw ec2_public_ip)

# 6. Complete setup on the server
cd /opt/gymapp
cp .env.prod.example .env.prod
nano .env.prod
# Use the RDS endpoint from terraform output for DATABASE_URL
# Use the ElastiCache endpoint for REDIS_URL

# 7. Deploy
docker compose -f docker-compose.prod.yml up -d
```

---

## 🔐 SSL Certificate Setup

```bash
# Install certbot on your server
apt-get install certbot

# Get certificate (server must be reachable on port 80)
certbot certonly --standalone \
    -d api.yourgym.com \
    -d admin.yourgym.com \
    --email admin@yourgym.com \
    --agree-tos \
    --non-interactive

# Certificate files will be at:
# /etc/letsencrypt/live/api.yourgym.com/fullchain.pem
# /etc/letsencrypt/live/api.yourgym.com/privkey.pem

# Auto-renewal is set up by server-setup.sh (runs twice daily)
# Test renewal:
certbot renew --dry-run
```

---

## 🔄 CI/CD Pipeline Setup

### GitHub Actions

1. Copy `ci-cd/github-actions.yml` to your repo as `.github/workflows/deploy.yml`

2. Add these GitHub Secrets (Settings → Secrets and variables → Actions):

   | Secret | Value |
   |--------|-------|
   | `DEPLOY_HOST` | Your server's IP address |
   | `DEPLOY_USER` | SSH username (e.g. `ubuntu`) |
   | `DEPLOY_SSH_KEY` | Private SSH key content |
   | `GHCR_TOKEN` | GitHub token with `packages:write` |

3. Generate SSH key pair for deployments:
   ```bash
   ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/deploy_key
   # Add deploy_key.pub to server: ~/.ssh/authorized_keys
   # Add deploy_key content to GitHub Secret: DEPLOY_SSH_KEY
   ```

4. Push to `main` branch → pipeline triggers automatically

### Pipeline Flow

```
Push to main
    │
    ▼
Run pytest (with PostgreSQL + Redis services)
    │
    ▼
Build Docker image → Push to GitHub Container Registry
    │
    ▼
SSH to server → Pull new image → Run migrations → Rolling restart
    │
    ▼
Health check → ✅ Done  or  ❌ Rollback
```

---

## 🗄️ Database Migrations

```bash
# Run migrations manually
docker compose -f docker-compose.prod.yml run --rm api alembic upgrade head

# Check current migration version
docker compose -f docker-compose.prod.yml run --rm api alembic current

# Rollback one migration
docker compose -f docker-compose.prod.yml run --rm api alembic downgrade -1

# Create a new migration after changing models
docker compose -f docker-compose.prod.yml run --rm api \
    alembic revision --autogenerate -m "add_new_column"
```

---

## 💾 Backup & Restore

```bash
# Manual backup now
./scripts/backup.sh

# List available backups
ls -la /var/backups/gymapp/

# Restore from backup (WARNING: replaces all data!)
./scripts/restore.sh gymapp_backup_20250210_020000.sql.gz

# View backup cron schedule
cat /etc/cron.d/gymapp-backup
```

Backups run automatically at **2am every day** and are:
- Stored locally for 30 days
- Optionally uploaded to S3 (set `S3_BACKUP_BUCKET` in `.env.prod`)
- Verified for integrity after creation

---

## 📊 Monitoring

```bash
# Start monitoring stack
docker compose -f docker-compose.prod.yml \
               -f docker-compose.monitoring.yml up -d

# Access Grafana dashboard
# http://your-server:3001 (admin / your GRAFANA_PASSWORD)
```

Key metrics to watch:
- **API P95 latency** — should be < 200ms
- **Error rate** — should be < 0.1%
- **PostgreSQL connections** — should be < 150 (of 200 max)
- **Redis memory** — should be < 80% of limit
- **Celery queue depth** — should be near 0 (tasks processed quickly)

---

## 🔧 Common Operations

```bash
# View all running services
docker compose -f docker-compose.prod.yml ps

# View API logs (live)
docker compose -f docker-compose.prod.yml logs -f api

# View worker logs
docker compose -f docker-compose.prod.yml logs -f worker

# Restart just the API (no downtime — rolling)
docker compose -f docker-compose.prod.yml up -d --no-deps api

# Run a management command
docker compose -f docker-compose.prod.yml run --rm api python -c "..."

# Connect to PostgreSQL
docker exec -it gym_db psql -U gymapp -d gymdb

# Connect to Redis
docker exec -it gym_redis redis-cli -a YOUR_REDIS_PASSWORD

# Check certificate expiry
certbot certificates

# Force certificate renewal
certbot renew --force-renewal

# Scale API instances up/down
docker compose -f docker-compose.prod.yml up -d --scale api=3
```

---

## 🆘 Troubleshooting

| Problem | Check | Fix |
|---------|-------|-----|
| API returns 502 | `docker logs gym_nginx` | Restart: `docker compose restart api` |
| Database connection error | `docker logs gym_db` | Check `DATABASE_URL` in `.env.prod` |
| Redis connection error | `docker logs gym_redis` | Check `REDIS_PASSWORD` matches |
| Stripe webhooks failing | Check Stripe dashboard | Verify `STRIPE_WEBHOOK_SECRET` |
| Celery tasks stuck | `docker logs gym_worker` | Restart worker: `docker compose restart worker` |
| Certificate expired | `certbot certificates` | `certbot renew --force-renewal` |
| Out of disk space | `df -h` | `docker system prune -f` |

---

## 🔒 Production Security Checklist

- [ ] `.env.prod` has `chmod 600` (not readable by other users)
- [ ] `SECRET_KEY` is 64+ random characters (not the example value)
- [ ] `DEBUG=False` in `.env.prod`
- [ ] Stripe LIVE keys configured (not test keys)
- [ ] SSH root login disabled (`/etc/ssh/sshd_config`: `PermitRootLogin no`)
- [ ] UFW firewall active (`ufw status`)
- [ ] fail2ban active (`systemctl status fail2ban`)
- [ ] SSL certificate valid (`certbot certificates`)
- [ ] Database backups running (`ls /var/backups/gymapp/`)
- [ ] Security headers pass: https://securityheaders.com/?q=api.yourgym.com
