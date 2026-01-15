# Nginx Configuration

This directory contains Nginx configuration files for the voting system.

## Files

- **`nginx.conf`**: Main Nginx configuration
- **`conf.d/default.conf`**: HTTP-only configuration (default)
- **`conf.d/ssl.conf.template`**: HTTPS configuration template

## Quick Start (HTTP Only)

The default setup uses HTTP without SSL:

```bash
docker compose up -d
```

Access the API at: http://localhost/api

## Enable SSL/HTTPS (Optional)

### Prerequisites
- Domain name pointing to your server
- Ports 80 and 443 open

### Steps

1. **Update SSL template with your domain:**
```bash
# Edit ssl.conf.template
# Replace 'yourdomain.com' with your actual domain
```

2. **Obtain SSL certificate:**
```bash
# Run certbot
docker compose --profile ssl run --rm certbot certonly --webroot \
  -w /var/www/certbot \
  -d yourdomain.com \
  -d www.yourdomain.com \
  --email your-email@example.com \
  --agree-tos \
  --no-eff-email
```

3. **Enable HTTPS configuration:**
```bash
# Backup current config
mv nginx/conf.d/default.conf nginx/conf.d/default.conf.bak

# Copy SSL template
cp nginx/conf.d/ssl.conf.template nginx/conf.d/default.conf

# Update domain in the new config
nano nginx/conf.d/default.conf
```

4. **Restart Nginx:**
```bash
docker compose restart nginx
```

5. **Enable auto-renewal:**
```bash
# Start certbot service
docker compose --profile ssl up -d certbot
```

## SSL Certificate Renewal

Certificates auto-renew when certbot service is running:

```bash
# Check renewal status
docker compose --profile ssl run --rm certbot renew --dry-run

# Manual renewal
docker compose --profile ssl run --rm certbot renew
docker compose restart nginx
```

## Troubleshooting

### Check Nginx configuration
```bash
docker compose exec nginx nginx -t
```

### View Nginx logs
```bash
docker compose logs nginx
```

### Test SSL
```bash
curl -I https://yourdomain.com/health
```

## Configuration Details

### Default Locations

- `/api` → Backend API
- `/docs` → Swagger documentation
- `/health` → Health check endpoint
- `/` → Root (proxied to backend)

### Security Headers

- X-Frame-Options
- X-Content-Type-Options
- X-XSS-Protection
- Strict-Transport-Security (HTTPS only)

### Timeouts

- Connection: 60s
- Send: 60s
- Read: 60s
