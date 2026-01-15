# VPS Deployment Guide

Complete guide for deploying the Voting System to a production VPS server with Docker and integrated Nginx.

## Prerequisites

### VPS Requirements
- **OS**: Ubuntu 22.04 LTS or later
- **RAM**: Minimum 2GB (4GB+ recommended)
- **Storage**: 20GB+ available
- **CPU**: 2+ cores recommended
- **Network**: Public IP with ports 80, 443 accessible

### Domain Setup (Optional for SSL)
- Domain name pointed to your VPS IP
- DNS A record configured (e.g., `voting.yourdomain.com`)

---

## Step 1: Initial Server Setup

### Connect to VPS
```bash
ssh root@your-vps-ip
```

### Update System
```bash
apt update && apt upgrade -y
```

### Create Non-Root User
```bash
# Create user
adduser deploy
usermod -aG sudo deploy

# Switch to new user
su - deploy
```

### Setup SSH Key Authentication (Recommended)
```bash
# On your local machine
ssh-copy-id deploy@your-vps-ip

# Test connection
ssh deploy@your-vps-ip
```

---

## Step 2: Install Docker & Docker Compose

### Install Docker
```bash
# Install dependencies
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common

# Add Docker GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io

# Add user to docker group
sudo usermod -aG docker $USER

# Apply group changes (logout and login again)
newgrp docker

# Verify installation
docker --version
```

### Install Docker Compose
```bash
# Docker Compose is included with Docker Desktop
# For Linux, install the plugin
sudo apt install -y docker-compose-plugin

# Verify installation
docker compose version
```

---

## Step 3: Clone Repository

### Install Git
```bash
sudo apt install -y git
```

### Setup SSH Key for GitHub (Recommended)
```bash
# Generate SSH key
ssh-keygen -t ed25519 -C "your-email@example.com"

# Display public key
cat ~/.ssh/id_ed25519.pub

# Copy the output and add to GitHub:
# GitHub → Settings → SSH and GPG keys → New SSH key
```

### Clone Repository with Submodules
```bash
# Create project directory
mkdir -p ~/apps
cd ~/apps

# Clone with submodules
git clone --recursive git@github.com:d4sda123/voting-deployment.git
cd voting-deployment

# Or if using HTTPS
git clone --recursive https://github.com/d4sda123/voting-deployment.git
cd voting-deployment
```

---

## Step 4: Configure Environment

### Create Production Environment File
```bash
cp .env.example .env
nano .env
```

### Essential Production Settings
```env
# Database
POSTGRES_USER=postgres
POSTGRES_PASSWORD=YOUR_VERY_SECURE_PASSWORD_HERE
POSTGRES_DB=voting

# Application
APP_PORT=3000
NODE_ENV=production

# JWT (Generate secure secret)
JWT_SECRET=YOUR_VERY_SECURE_JWT_SECRET_MINIMUM_32_CHARS
JWT_EXPIRES_IN=7d

# Password Reset
PASSWORD_RESET_EXPIRES_IN=1h
PASSWORD_RESET_URL=https://voting.yourdomain.com/reset-password

# CORS (Your domain)
CORS_ORIGINS=https://voting.yourdomain.com,http://your-vps-ip

# Nginx Ports
HTTP_PORT=80
HTTPS_PORT=443

# Adminer
ADMINER_PORT=8080

# SMTP (Optional - for email notifications)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASS=your-app-password
SMTP_FROM=noreply@yourdomain.com

# Seed Data (Initial Setup)
ASSOCIATION_NAME=Your Association Name
BRANCH_NAME=Main Branch
SPECIALTY_NAME=General
CHAPTER_NAME=Main Chapter
ADMIN_DNI=00000001
ADMIN_PASSWORD=ChangeThisPassword123!
ADMIN_EMAIL=admin@yourdomain.com
ADMIN_PHONE=+1234567890
ADMIN_FIRST_NAME=Admin
ADMIN_LAST_NAME=User
```

### Generate Secure Secrets
```bash
# Generate JWT secret
openssl rand -base64 32

# Generate strong password
openssl rand -base64 24
```

---

## Step 5: Deploy Application

### Build Frontend
```bash
cd ~/apps/voting-deployment

# Build frontend (first time)
docker compose --profile build up -d --build frontend

# Wait for build to complete (check logs)
docker compose --profile build logs -f frontend
# Press Ctrl+C when you see "Build complete"
```

### Start All Services
```bash
# Start services
docker compose up -d

# Check status
docker compose ps
```

### Initialize Database
```bash
# Run migrations
docker compose exec backend npx prisma migrate deploy

# Seed initial data
docker compose exec backend npx prisma db seed
```

### Verify Deployment
```bash
# Check logs
docker compose logs -f backend

# Test health endpoint
curl http://localhost/health
# Should return: {"status":"ok"}

# Test API
curl http://localhost/api/auth/login
```

---

## Step 6: Setup Firewall

### Configure UFW
```bash
# Enable UFW
sudo ufw enable

# Allow SSH
sudo ufw allow OpenSSH

# Allow HTTP/HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Check status
sudo ufw status
```

---

## Step 7: Setup SSL with Let's Encrypt (Optional)

### Option A: Using Docker Certbot (Recommended)

```bash
cd ~/apps/voting-deployment

# 1. Update nginx/conf.d/ssl.conf.template with your domain
nano nginx/conf.d/ssl.conf.template
# Replace 'yourdomain.com' with your actual domain

# 2. Get SSL certificate
docker compose --profile ssl run --rm certbot certonly --webroot \
  -w /var/www/certbot \
  -d voting.yourdomain.com \
  --email your-email@example.com \
  --agree-tos \
  --no-eff-email

# 3. Enable HTTPS configuration
mv nginx/conf.d/default.conf nginx/conf.d/default.conf.bak
cp nginx/conf.d/ssl.conf.template nginx/conf.d/default.conf

# 4. Update domain in config
nano nginx/conf.d/default.conf
# Replace 'yourdomain.com' with your actual domain

# 5. Restart nginx
docker compose restart nginx

# 6. Enable auto-renewal
docker compose --profile ssl up -d certbot
```

### Option B: Manual Certbot (Alternative)

```bash
# Install Certbot
sudo apt install -y certbot

# Get certificate
sudo certbot certonly --standalone -d voting.yourdomain.com

# Copy certificates to project
sudo cp /etc/letsencrypt/live/voting.yourdomain.com/fullchain.pem ~/apps/voting-deployment/certbot/conf/live/voting.yourdomain.com/
sudo cp /etc/letsencrypt/live/voting.yourdomain.com/privkey.pem ~/apps/voting-deployment/certbot/conf/live/voting.yourdomain.com/

# Follow steps 3-5 from Option A
```

### Test SSL
```bash
# Test HTTPS
curl -I https://voting.yourdomain.com/health

# Check certificate
openssl s_client -connect voting.yourdomain.com:443 -servername voting.yourdomain.com
```

---

## Step 8: Setup Monitoring & Logs

### View Application Logs
```bash
# Real-time logs
docker compose logs -f

# Specific service
docker compose logs -f backend
docker compose logs -f nginx

# Last 100 lines
docker compose logs --tail=100 backend
```

### Setup Log Rotation
```bash
# Create logrotate config
sudo nano /etc/logrotate.d/docker-compose
```

Add:
```
/var/lib/docker/containers/*/*.log {
    rotate 7
    daily
    compress
    missingok
    delaycompress
    copytruncate
}
```

---

## Step 9: Backup Strategy

### Database Backup Script
```bash
# Create backup directory
mkdir -p ~/backups

# Create backup script
nano ~/backup-db.sh
```

Add:
```bash
#!/bin/bash
BACKUP_DIR=~/backups
DATE=$(date +%Y%m%d_%H%M%S)
cd ~/apps/voting-deployment

docker compose exec -T db pg_dump -U postgres voting > $BACKUP_DIR/voting_$DATE.sql
gzip $BACKUP_DIR/voting_$DATE.sql

# Keep only last 7 days
find $BACKUP_DIR -name "voting_*.sql.gz" -mtime +7 -delete

echo "Backup completed: voting_$DATE.sql.gz"
```

Make executable:
```bash
chmod +x ~/backup-db.sh
```

### Setup Cron Job
```bash
crontab -e
```

Add daily backup at 2 AM:
```
0 2 * * * /home/deploy/backup-db.sh >> /home/deploy/backup.log 2>&1
```

---

## Maintenance Commands

### Update Application
```bash
cd ~/apps/voting-deployment

# Pull latest changes
git pull origin main
git submodule update --remote

# Rebuild frontend
docker compose --profile build up -d --build frontend

# Rebuild and restart services
docker compose up -d --build

# Run new migrations
docker compose exec backend npx prisma migrate deploy
```

### Restart Services
```bash
# Restart all
docker compose restart

# Restart specific service
docker compose restart backend
docker compose restart nginx
```

### View Resource Usage
```bash
# Docker stats
docker stats

# System resources
htop
```

### Database Access
```bash
# Via Adminer
# Open: http://your-vps-ip/adminer

# Via CLI
docker compose exec db psql -U postgres -d voting
```

---

## Troubleshooting

### Frontend Not Loading
```bash
# Check if frontend was built
docker compose --profile build exec frontend ls -la /app/www

# Check nginx has files
docker compose exec nginx ls -la /usr/share/nginx/html

# Rebuild frontend
docker compose --profile build up -d --build frontend
docker compose restart nginx
```

### Backend Not Starting
```bash
# Check logs
docker compose logs backend

# Check environment
docker compose exec backend env | grep DATABASE_URL

# Restart
docker compose restart backend
```

### Database Connection Issues
```bash
# Check database health
docker compose ps db

# Check database logs
docker compose logs db

# Test connection
docker compose exec db psql -U postgres -d voting -c "SELECT version();"
```

### SSL Certificate Issues
```bash
# Check certificate
docker compose --profile ssl run --rm certbot certificates

# Renew manually
docker compose --profile ssl run --rm certbot renew
docker compose restart nginx

# Check Nginx config
docker compose exec nginx nginx -t
```

### Nginx 404 Errors
```bash
# Check nginx config
docker compose exec nginx nginx -t

# Check nginx logs
docker compose logs nginx

# Verify routing
docker compose exec nginx cat /etc/nginx/conf.d/default.conf
```

### Out of Disk Space
```bash
# Check disk usage
df -h

# Clean Docker
docker system prune -a

# Clean old logs
sudo journalctl --vacuum-time=7d

# Clean old backups
find ~/backups -name "voting_*.sql.gz" -mtime +30 -delete
```

---

## Security Checklist

- [ ] Strong passwords for database and JWT
- [ ] Firewall configured (UFW)
- [ ] SSL/TLS enabled (for production)
- [ ] Regular backups scheduled
- [ ] Adminer access restricted (IP whitelist or remove in production)
- [ ] SSH key authentication enabled
- [ ] Root login disabled
- [ ] Fail2ban installed (optional)
- [ ] Regular security updates
- [ ] Environment variables secured (.env not committed)
- [ ] Change default admin password immediately

---

## Production URLs

After deployment, your services will be available at:

**Without SSL (HTTP):**
- **Frontend**: http://your-vps-ip
- **API**: http://your-vps-ip/api
- **Swagger**: http://your-vps-ip/docs
- **Health**: http://your-vps-ip/health
- **Adminer**: http://your-vps-ip/adminer

**With SSL (HTTPS):**
- **Frontend**: https://voting.yourdomain.com
- **API**: https://voting.yourdomain.com/api
- **Swagger**: https://voting.yourdomain.com/docs
- **Health**: https://voting.yourdomain.com/health
- **Adminer**: https://voting.yourdomain.com/adminer

---

## Support & Monitoring

### Health Monitoring
Setup external monitoring (e.g., UptimeRobot) to check:
- https://voting.yourdomain.com/health

### Email Alerts
Configure SMTP to receive alerts for critical issues.

### Regular Maintenance
- **Daily**: Automated backups
- **Weekly**: Check logs and disk space
- **Monthly**: Review backups and test restore
- **Quarterly**: Security updates and dependency updates

---

## Next Steps

1. **Change Admin Password**: Login and change default password immediately
2. **Configure SMTP**: Setup email notifications
3. **Setup Monitoring**: Add UptimeRobot or similar
4. **CI/CD**: Setup GitHub Actions for automated deployments
5. **CDN**: Consider Cloudflare for DDoS protection
6. **Scaling**: Add load balancing for high traffic

---

## Architecture Overview

```
Internet
    ↓
┌─────────────────────────────────────┐
│    Nginx Reverse Proxy (Port 80/443)│
│  ┌─────────────────────────────────┐│
│  │ /          → Frontend (static)  ││
│  │ /api       → Backend API        ││
│  │ /docs      → Swagger            ││
│  │ /health    → Health check       ││
│  │ /adminer   → Database UI        ││
│  └─────────────────────────────────┘│
└─────────────────────────────────────┘
    ↓           ↓           ↓
┌──────────┐ ┌──────────┐ ┌──────────┐
│ Frontend │ │ Backend  │ │ Adminer  │
│ (Alpine) │ │ (Node20) │ │          │
└──────────┘ └──────────┘ └──────────┘
                  ↓
            ┌──────────┐
            │PostgreSQL│
            │  (v17)   │
            └──────────┘
```

All services run in Docker containers, orchestrated by Docker Compose, with Nginx handling SSL termination and routing.
