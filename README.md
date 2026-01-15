# Voting System - Docker Deployment

This repository contains the complete voting system deployment configuration with backend API, frontend, and database services orchestrated via Docker Compose.

## Project Structure

```
voting-deployment/
├── docker-compose.yml          # Main orchestration file
├── .env.example                # Environment template
├── .gitmodules                 # Git submodules configuration
├── voting-backend/             # NestJS backend API (submodule)
│   ├── Dockerfile
│   ├── src/
│   └── prisma/
└── voting-frontend/            # Ionic frontend (submodule)
    └── src/
```

## Quick Start

### Prerequisites
- Docker 20.10+
- Docker Compose 2.0+
- Git

### Clone the Repository

**Clone with submodules (recommended):**
```bash
git clone --recursive https://github.com/d4sda123/voting-deployment.git
cd voting-deployment
```

**Or if already cloned:**
```bash
git clone https://github.com/d4sda123/voting-deployment.git
cd voting-deployment
git submodule update --init --recursive
```

### Setup

1. **Copy environment template:**
```bash
cp .env.example .env
```

2. **Edit `.env` with your configuration:**
   - Set secure `POSTGRES_PASSWORD`
   - Set secure `JWT_SECRET`
   - Configure `CORS_ORIGINS` with your frontend URLs
   - Update `PASSWORD_RESET_URL`
   - Configure SMTP (optional)
   - Customize seed data

3. **Start all services:**
```bash
docker compose up -d --build
```

This will start:
- **voting-db**: PostgreSQL 17.6 (Supabase) on port 5432
- **voting-adminer**: Adminer (Database UI) on port 8080
- **voting-backend**: NestJS API on port 3000

4. **Initialize the database:**
```bash
# Run migrations
docker compose exec backend npx prisma migrate deploy

# Seed initial data (roles, permissions, admin user)
docker compose exec backend npx prisma db seed
```

5. **Verify services:**
- API: http://localhost:3000
- Swagger Docs: http://localhost:3000/docs
- Health Check: http://localhost:3000/health
- Adminer (Database UI): http://localhost:8080

## Services

### Backend (NestJS)
- **Container**: `voting-backend`
- **Port**: 3000 (configurable via `APP_PORT`)
- **Health Check**: `/health` endpoint
- **Documentation**: `/docs` (Swagger)

### Database (PostgreSQL)
- **Container**: `voting-db`
- **Port**: 5432
- **Image**: Supabase PostgreSQL 17.6
- **Volume**: `postgres_data` (persistent)

### Adminer (Database UI)
- **Container**: `voting-adminer`
- **Port**: 8080 (configurable via `ADMINER_PORT`)
- **Purpose**: Visual database management
- **Login**: Use credentials from `.env` file

## Common Commands

### View Logs
```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f backend
docker compose logs -f db
docker compose logs -f adminer
```

### Restart Services
```bash
# All services
docker compose restart

# Specific service
docker compose restart backend
```

### Stop Services
```bash
docker compose down
```

### Stop and Remove Data (⚠️ Destructive)
```bash
docker compose down -v
```

### Access Containers
```bash
# Backend shell
docker compose exec backend sh

# Database CLI
docker compose exec db psql -U postgres -d voting
```

### Database Operations
```bash
# Run migrations
docker compose exec backend npx prisma migrate deploy

# Seed database
docker compose exec backend npx prisma db seed

# Reset database (⚠️ destructive)
docker compose exec backend npx prisma migrate reset

# View database schema
docker compose exec backend npx prisma db pull
```

## Updating the Application

### Update Submodules
```bash
# Update to latest commits
git submodule update --remote

# Commit the updates
git add voting-backend voting-frontend
git commit -m "Update submodules"
```

### Rebuild and Restart
```bash
# Rebuild backend
docker compose build backend

# Run migrations if schema changed
docker compose exec backend npx prisma migrate deploy

# Restart services
docker compose up -d
```

## Database Backup & Restore

### Create Backup
```bash
docker compose exec db pg_dump -U postgres voting > backup_$(date +%Y%m%d_%H%M%S).sql
```

### Restore Backup
```bash
cat backup_file.sql | docker compose exec -T db psql -U postgres -d voting
```

## Troubleshooting

### Backend can't connect to database
- Check database health: `docker compose ps`
- View database logs: `docker compose logs db`
- Verify `DATABASE_URL` in `.env`

### Port already in use
- Change `APP_PORT` or `ADMINER_PORT` in `.env`
- Or stop the conflicting service

### Prisma errors
- Check migrations: `docker compose exec backend npx prisma migrate status`
- Reset if needed: `docker compose exec backend npx prisma migrate reset`

### Submodules not initialized
```bash
git submodule update --init --recursive
```

### Clean slate restart
```bash
docker compose down -v
docker compose up -d --build
docker compose exec backend npx prisma migrate deploy
docker compose exec backend npx prisma db seed
```

## Environment Variables

See `.env.example` for all required and optional variables.

### Required
- `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`
- `APP_PORT`, `ADMINER_PORT`
- `JWT_SECRET`, `JWT_EXPIRES_IN`
- `CORS_ORIGINS`
- Seed data variables (admin user, organization structure)

### Optional
- SMTP configuration (for email notifications)

## Development

For backend development details, see [voting-backend/README.md](voting-backend/README.md)

## Production Deployment

This Docker Compose setup is production-ready with:
- ✅ Health checks on all services
- ✅ Automatic restart policies
- ✅ Persistent data volumes
- ✅ Security best practices (non-root users)
- ✅ Optimized multi-stage builds

For production, consider:
- Using Docker secrets for sensitive data
- Setting up SSL/TLS termination (Nginx/Traefik)
- Implementing automated backups
- Adding monitoring (Prometheus/Grafana)
- Using a managed database service for high availability
