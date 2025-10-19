# Receipt Wrangler Monolith

This repository contains the monolithic Docker image for Receipt Wrangler, which packages both the frontend (Angular) and backend (Go API) into a single container served by Nginx.

## Quick Start with Docker

### Option 1: Using Docker Run

```bash
# Generate secrets
export SECRET_KEY=$(openssl rand -base64 32)
export ENCRYPTION_KEY=$(openssl rand -base64 32)

# Run the container
docker run -d \
  --name receipt-wrangler \
  --restart unless-stopped \
  -p 8080:80 \
  -e SECRET_KEY="${SECRET_KEY}" \
  -e ENCRYPTION_KEY="${ENCRYPTION_KEY}" \
  -e DB_ENGINE=sqlite \
  -e DB_FILENAME=wrangler.sqlite \
  -e REDIS_HOST=redis \
  -e REDIS_PORT=6379 \
  -v receipt-wrangler-data:/app/receipt-wrangler-api/data \
  -v receipt-wrangler-sqlite:/app/receipt-wrangler-api/sqlite \
  -v receipt-wrangler-logs:/app/receipt-wrangler-api/logs \
  noah231515/receipt-wrangler:latest
```

### Option 2: Using Docker Compose (Recommended)

See [docker-compose.yml](docker-compose.yml) for a complete setup including Redis.

```bash
# Copy the example environment file
cp .env.example .env

# Edit .env with your configuration
# Generate secrets:
#   SECRET_KEY=$(openssl rand -base64 32)
#   ENCRYPTION_KEY=$(openssl rand -base64 32)
nano .env

# Start the stack
docker-compose up -d
```

**Important:** When using docker-compose, set `REDIS_HOST=redis` in your `.env` file (not an IP address). Docker Compose networking uses service names for container-to-container communication.

## Configuration

Receipt Wrangler can be configured using environment variables. Create a `.env` file or pass them directly to Docker.

### Required Environment Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `SECRET_KEY` | JWT token signing key | Generate with `openssl rand -base64 32` |
| `ENCRYPTION_KEY` | Data encryption key | Generate with `openssl rand -base64 32` |
| `DB_ENGINE` | Database engine | `sqlite` or `postgresql` |
| `DB_FILENAME` | SQLite database filename | `wrangler.sqlite` |
| `REDIS_HOST` | Redis hostname | `redis` or `localhost` |
| `REDIS_PORT` | Redis port | `6379` |

### Optional Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `PORT` | Port to expose the application on | `8080` |
| `ENABLE_LOCAL_SIGN_UP` | Allow local user registration | `true` |
| `AI_POWERED_RECEIPTS` | Enable AI features (requires AI-enabled image) | `false` |

**Note on AI Features:** The production Docker image (`noah231515/receipt-wrangler:latest`) does not include AI dependencies. AI features require a Python virtual environment with machine learning libraries. Set `AI_POWERED_RECEIPTS=false` when using the standard production image.

### PostgreSQL Configuration

If using PostgreSQL instead of SQLite:

```bash
DB_ENGINE=postgresql
DB_HOST=postgres
DB_PORT=5432
DB_NAME=receiptwrangler
DB_USER=receiptwrangler
DB_PASSWORD=your_secure_password
```

## Deployment with Cloudflare Tunnel

For secure public access without port forwarding, you can deploy Receipt Wrangler behind a Cloudflare Tunnel.

### Prerequisites

- Cloudflare account with a domain
- `cloudflared` installed on your server or router
- Cloudflare Tunnel created and authenticated

### Tunnel Configuration

Add Receipt Wrangler to your tunnel's ingress rules in `~/.cloudflared/config.yml`:

```yaml
tunnel: YOUR_TUNNEL_ID
credentials-file: /path/to/credentials.json

ingress:
  - hostname: receipts.yourdomain.com
    service: http://localhost:8085
  # ... other services
  - service: http_status:404
```

**Important:** Place the Receipt Wrangler rule before the catch-all `http_status:404` rule.

### DNS Configuration

The DNS record should already exist if you used `cloudflared tunnel route dns`. If not, create an A or CNAME record in Cloudflare Dashboard pointing to your tunnel.

### Restart the Tunnel

After updating the configuration, restart the tunnel:

```bash
# If using systemd
sudo systemctl restart cloudflared

# If using init.d (OpenWrt, etc.)
/etc/init.d/cloudflared restart

# Manual restart
pkill cloudflared
cloudflared --config ~/.cloudflared/config.yml tunnel run
```

**Critical:** Always use the proper restart method (init script or systemctl). Manually killing and restarting cloudflared can result in multiple processes running simultaneously, which causes routing conflicts and 404 errors.

### Verify the Configuration

Test that the ingress rule is correctly configured:

```bash
cloudflared tunnel ingress rule https://receipts.yourdomain.com
```

Expected output:
```
Using rules from /path/to/config.yml
Matched rule #X
	hostname: receipts.yourdomain.com
	service: http://localhost:8085
```

### Troubleshooting

**404 Errors After Configuration**

If you're getting 404 errors:

1. **Check for multiple cloudflared processes:**
   ```bash
   ps aux | grep cloudflared
   ```
   You should see only ONE process. If multiple exist, restart using the init script.

2. **Verify local service is running:**
   ```bash
   curl http://localhost:8085/
   ```

3. **Check tunnel logs:**
   ```bash
   # Systemd
   journalctl -u cloudflared -n 50

   # OpenWrt/init.d
   logread | grep cloudflared

   # Manual
   cat /var/log/cloudflared.log
   ```

4. **Clear Cloudflare cache:**
   - Go to Cloudflare Dashboard
   - Navigate to Caching → Configuration
   - Click "Purge Everything"

## Building from Source

### Production Build

```bash
docker build -t receipt-wrangler-custom .
```

### Development Build

The development Dockerfile includes SSH access for debugging:

```bash
cd dev
docker build -t receipt-wrangler-dev .

# Run development container
docker run -d \
  --name receipt-wrangler-dev \
  -p 8080:80 \
  -p 2222:22 \
  receipt-wrangler-dev

# SSH into container
ssh root@localhost -p 2222
# Password: development
```

## Architecture

This monolithic image contains:

- **Frontend**: Angular application served by Nginx
- **Backend**: Go API server on port 8081 (internal)
- **Nginx**: Reverse proxy routing API requests to backend
- **Asynq**: Background job processing using Redis

### Port Mapping

- Port 80: Nginx (frontend + API proxy)
- Port 8081: Go API (internal only)

## Volumes

Three volumes are used for persistent data:

- `/app/receipt-wrangler-api/data` - Uploaded receipts and attachments
- `/app/receipt-wrangler-api/sqlite` - SQLite database (if using SQLite)
- `/app/receipt-wrangler-api/logs` - Application logs

## Troubleshooting

### Application Won't Start

Check the logs:
```bash
docker logs receipt-wrangler
```

Common issues:
- **Missing SECRET_KEY or ENCRYPTION_KEY**: Ensure both are set
- **Redis connection error**: Ensure Redis is accessible at the configured host/port
- **Database errors**: Check database configuration and permissions

### Accessing the Application

Once running, access Receipt Wrangler at:
- http://localhost:8080 (or your configured port)

**Default credentials:**
- Username: `admin`
- Password: `admin`

**IMPORTANT:** After first login, immediately change the default password:
1. Click on your user avatar in the top right
2. Navigate to user management
3. Update your password to something secure

### Configuration Issues

The application reads configuration from:
1. Environment variables (recommended)
2. `/app/receipt-wrangler-api/config/config.json` (optional)

Environment variables take precedence over the config file.

## Related Issues

- Issue #21: Docker deployment configuration guide

## License

See the main Receipt Wrangler repository for license information.
