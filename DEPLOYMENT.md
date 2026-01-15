# VPS Deployment Guide

Complete guide for deploying the Voting System to a production VPS server.

## Prerequisites

### VPS Requirements
- **OS**: Ubuntu 22.04 LTS or later
- **RAM**: Minimum 2GB (4GB+ recommended)
- **Storage**: 20GB+ available
- **CPU**: 2+ cores recommended
- **Network**: Public IP address with ports 80, 443 accessible

### Domain Setup
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
# Download Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# Make executable
sudo chmod +x /usr/local/bin/docker-compose

# Verify installation
docker-compose --version
```

---

## Step 3: Install Git & Clone Repository

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

# PostgreSQL Configuration
POSTGRES_HOST_AUTH_METHOD=scram-sha-256
POSTGRES_INITDB_ARGS=--auth-host=scram-sha-256

# Application
APP_PORT=3000
NODE_ENV=production

# JWT (Generate secure secret)
JWT_SECRET=YOUR_VERY_SECURE_JWT_SECRET_MINIMUM_32_CHARS
JWT_EXPIRES_IN=7d

# Password Reset
PASSWORD_RESET_EXPIRES_IN=1h
PASSWORD_RESET_URL=https://voting.yourdomain.com/reset-password

# CORS (Your frontend domain)
CORS_ORIGINS=https://voting.yourdomain.com,https://www.voting.yourdomain.com

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

## Step 5: Setup Nginx Reverse Proxy

### Install Nginx
```bash
sudo apt install -y nginx
```

### Create Nginx Configuration
```bash
sudo nano /etc/nginx/sites-available/voting
```

Add this configuration:
```nginx
# Redirect HTTP to HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name voting.yourdomain.com;
    
    # Certbot challenge
    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }
    
    location / {
        return 301 https://$server_name$request_uri;
    }
}

# HTTPS Configuration
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name voting.yourdomain.com;

    # SSL certificates (will be added by Certbot)
    # ssl_certificate /etc/letsencrypt/live/voting.yourdomain.com/fullchain.pem;
    # ssl_certificate_key /etc/letsencrypt/live/voting.yourdomain.com/privkey.pem;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Backend API
    location /api {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    # Swagger docs
    location /docs {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Health check
    location /health {
        proxy_pass http://localhost:3000;
        access_log off;
    }

    # Frontend (if serving from same domain)
    location / {
        root /var/www/voting-frontend;
        try_files $uri $uri/ /index.html;
    }
}

# Adminer (Database UI) - Restrict access
server {
    listen 8080;
    server_name voting.yourdomain.com;
    
    # Restrict to specific IPs (optional)
    # allow YOUR_IP_ADDRESS;
    # deny all;
    
    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### Enable Site
```bash
# Create symlink
sudo ln -s /etc/nginx/sites-available/voting /etc/nginx/sites-enabled/

# Test configuration
sudo nginx -t

# Reload Nginx
sudo systemctl reload nginx
```

---

## Step 6: Setup SSL with Let's Encrypt

### Install Certbot
```bash
sudo apt install -y certbot python3-certbot-nginx
```

### Obtain SSL Certificate
```bash
# Get certificate
sudo certbot --nginx -d voting.yourdomain.com

# Follow prompts:
# - Enter email
# - Agree to terms
# - Choose redirect option (2)
```

### Auto-Renewal
```bash
# Test renewal
sudo certbot renew --dry-run

# Certbot auto-renewal is enabled by default
# Verify timer
sudo systemctl status certbot.timer
```

---

## Step 7: Deploy Application

### Build and Start Services
```bash
cd ~/apps/voting-deployment

# Build and start
docker compose up -d --build

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
curl http://localhost:3000/health

# Should return: {"status":"ok"}
```

---

## Step 8: Setup Firewall

### Configure UFW
```bash
# Enable UFW
sudo ufw enable

# Allow SSH
sudo ufw allow OpenSSH

# Allow HTTP/HTTPS
sudo ufw allow 'Nginx Full'

# Check status
sudo ufw status
```

---

## Step 9: Setup Monitoring & Logs

### View Application Logs
```bash
# Real-time logs
docker compose logs -f

# Specific service
docker compose logs -f backend
docker compose logs -f db

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

## Step 10: Backup Strategy

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

# Update submodules
git submodule update --remote

# Rebuild and restart
docker compose down
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
# Open: https://voting.yourdomain.com:8080

# Via CLI
docker compose exec db psql -U postgres -d voting
```

---

## Troubleshooting

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
sudo certbot certificates

# Renew manually
sudo certbot renew

# Check Nginx config
sudo nginx -t
```

### Out of Disk Space
```bash
# Check disk usage
df -h

# Clean Docker
docker system prune -a

# Clean old logs
sudo journalctl --vacuum-time=7d
```

---

## Security Checklist

- [ ] Strong passwords for database and JWT
- [ ] Firewall configured (UFW)
- [ ] SSL/TLS enabled
- [ ] Regular backups scheduled
- [ ] Adminer access restricted
- [ ] SSH key authentication enabled
- [ ] Root login disabled
- [ ] Fail2ban installed (optional)
- [ ] Regular security updates
- [ ] Environment variables secured

---

## Production URLs

After deployment, your services will be available at:

- **API**: https://voting.yourdomain.com/api
- **Swagger**: https://voting.yourdomain.com/docs
- **Health**: https://voting.yourdomain.com/health
- **Adminer**: https://voting.yourdomain.com:8080 (restrict access!)

---

## Support & Monitoring

### Health Monitoring
Setup external monitoring (e.g., UptimeRobot) to check:
- https://voting.yourdomain.com/health

### Email Alerts
Configure SMTP to receive alerts for critical issues.

### Regular Maintenance
- Weekly: Check logs and disk space
- Monthly: Review backups and test restore
- Quarterly: Security updates and dependency updates

---

## Next Steps

1. **Frontend Deployment**: Deploy frontend application
2. **CI/CD**: Setup GitHub Actions for automated deployments
3. **Monitoring**: Add Prometheus + Grafana
4. **CDN**: Setup Cloudflare for DDoS protection
5. **Scaling**: Consider load balancing for high traffic
