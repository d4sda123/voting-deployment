# Voting System - Production Deployment

Complete Docker-based voting system with NestJS backend, Ionic/Angular frontend, PostgreSQL database, and Nginx reverse proxy.

## 📋 Project Structure

```
voting-deployment/
├── docker-compose.yml       # Main orchestration
├── .env.example             # Environment template
├── nginx/                   # Nginx reverse proxy configs
│   ├── nginx.conf          # Main nginx config
│   ├── conf.d/
│   │   ├── default.conf    # HTTP configuration
│   │   └── ssl.conf.template # HTTPS template
│   └── README.md           # SSL setup guide
├── voting-backend/          # NestJS API (submodule)
│   ├── Dockerfile
│   ├── src/
│   └── prisma/
└── voting-frontend/         # Ionic frontend (submodule)
    ├── Dockerfile
    └── src/
```

## 🚀 Quick Start

### Prerequisites
- Docker 20.10+
- Docker Compose 2.0+
- Git

### 1. Clone with Submodules

```bash
git clone --recursive https://github.com/d4sda123/voting-deployment.git
cd voting-deployment
```

**If already cloned:**
```bash
git submodule update --init --recursive
```

### 2. Configure Environment

```bash
cp .env.example .env
nano .env
```

**Essential settings:**
```env
# Database
POSTGRES_PASSWORD=your_secure_password_here

# JWT
JWT_SECRET=your_very_secure_jwt_secret_minimum_32_chars

# Admin User
ADMIN_DNI=00000001
ADMIN_PASSWORD=Admin123!
ADMIN_EMAIL=admin@example.com
```

**Generate secure secrets:**
```bash
# JWT secret
openssl rand -base64 32

# Strong password
openssl rand -base64 24
```

### 3. Build Frontend

```bash
# Build frontend (first time)
docker compose --profile build up -d --build frontend

# Wait for build to complete
docker compose --profile build logs -f frontend
```

### 4. Start Services

```bash
# Start all services
docker compose up -d

# Check status
docker compose ps
```

### 5. Initialize Database

```bash
# Run migrations
docker compose exec backend npx prisma migrate deploy

# Seed initial data (admin user, roles, permissions)
docker compose exec backend npx prisma db seed
```

### 6. Access Application

- **Frontend**: http://localhost
- **API**: http://localhost/api
- **Swagger Docs**: http://localhost/docs
- **Health Check**: http://localhost/health
- **Adminer (DB UI)**: http://localhost/adminer

**Default admin credentials:**
- DNI: `00000001`
- Password: `Admin123!`

## 🏗️ Architecture

```
┌─────────────────────────────────────────────┐
│           Nginx Reverse Proxy               │
│         (Port 80/443 - Single Entry)        │
│                                             │
│  Routes:                                    │
│  /          → Frontend (static files)       │
│  /api       → Backend API                   │
│  /docs      → Swagger                       │
│  /health    → Health check                  │
│  /adminer   → Database UI                   │
└─────────────────────────────────────────────┘
         ↓              ↓              ↓
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│   Frontend   │ │   Backend    │ │   Adminer    │
│  (Alpine)    │ │  (Node 20)   │ │              │
│  Static      │ │  NestJS API  │ │  DB Manager  │
│  Files       │ │  + Prisma    │ │              │
└──────────────┘ └──────────────┘ └──────────────┘
                        ↓
                 ┌──────────────┐
                 │  PostgreSQL  │
                 │   (v17)      │
                 └──────────────┘
```

## 🔄 Development Workflow

### Rebuild Frontend After Changes

```bash
# Rebuild frontend
docker compose --profile build up -d --build frontend

# Restart nginx to serve new files
docker compose restart nginx
```

### Update Backend

```bash
# Rebuild backend
docker compose up -d --build backend

# Run new migrations
docker compose exec backend npx prisma migrate deploy
```

### View Logs

```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f backend
docker compose logs -f nginx
```

## 📦 Services

| Service | Container | Port | Description |
|---------|-----------|------|-------------|
| **nginx** | voting-nginx | 80, 443 | Reverse proxy + frontend |
| **backend** | voting-backend | 3000 | NestJS API |
| **frontend** | voting-frontend | - | Build container (profile: build) |
| **db** | voting-db | 5432 | PostgreSQL database |
| **adminer** | voting-adminer | 8080 | Database UI |

## 🔒 SSL Setup (Optional)

See [`nginx/README.md`](nginx/README.md) for detailed SSL configuration with Let's Encrypt.

**Quick SSL setup:**
```bash
# 1. Get certificate
docker compose --profile ssl run --rm certbot certonly --webroot \
  -w /var/www/certbot -d yourdomain.com

# 2. Enable HTTPS config
mv nginx/conf.d/default.conf nginx/conf.d/default.conf.bak
cp nginx/conf.d/ssl.conf.template nginx/conf.d/default.conf

# 3. Update domain in config
nano nginx/conf.d/default.conf

# 4. Restart nginx
docker compose restart nginx

# 5. Enable auto-renewal
docker compose --profile ssl up -d certbot
```

## 🛠️ Maintenance

### Update Application

```bash
# Pull latest changes
git pull origin main
git submodule update --remote

# Rebuild and restart
docker compose down
docker compose --profile build up -d --build frontend
docker compose up -d --build

# Run migrations
docker compose exec backend npx prisma migrate deploy
```

### Database Backup

```bash
# Manual backup
docker compose exec -T db pg_dump -U postgres voting > backup_$(date +%Y%m%d).sql

# Restore
docker compose exec -T db psql -U postgres voting < backup_20260115.sql
```

### Clean Up

```bash
# Stop all services
docker compose down

# Remove volumes (WARNING: deletes data)
docker compose down -v

# Clean Docker system
docker system prune -a
```

## 🐛 Troubleshooting

### Frontend not loading
```bash
# Check if frontend was built
docker compose --profile build exec frontend ls -la /app/www

# Check nginx has files
docker compose exec nginx ls -la /usr/share/nginx/html

# Rebuild frontend
docker compose --profile build up -d --build frontend
docker compose restart nginx
```

### Backend 404 errors
```bash
# Check backend logs
docker compose logs backend

# Verify backend is healthy
curl http://localhost/health

# Check nginx routing
docker compose exec nginx nginx -t
```

### Database connection issues
```bash
# Check database health
docker compose ps db

# Test connection
docker compose exec db psql -U postgres -d voting -c "SELECT version();"

# Check backend environment
docker compose exec backend env | grep DATABASE_URL
```

## 📚 Documentation

- **[DEPLOYMENT.md](DEPLOYMENT.md)** - VPS deployment guide
- **[nginx/README.md](nginx/README.md)** - SSL and Nginx configuration
- **[Backend README](voting-backend/README.md)** - Backend API docs
- **[Frontend README](voting-frontend/README.md)** - Frontend app docs

## 🔐 Security Notes

- ✅ Change default admin password immediately
- ✅ Use strong, unique passwords for database and JWT
- ✅ Restrict Adminer access in production
- ✅ Enable SSL/TLS for production deployments
- ✅ Keep secrets in `.env` (never commit)
- ✅ Regular backups and updates

## 📄 License

MIT License - See LICENSE file for details
