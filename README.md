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
nano .env

# Start the stack
docker-compose up -d
```

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
| `ENABLE_LOCAL_SIGN_UP` | Allow local user registration | `true` |
| `AI_POWERED_RECEIPTS` | Enable AI features | `false` |

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

Default credentials (if local signup is enabled):
- Create an account on first access

### Configuration Issues

The application reads configuration from:
1. Environment variables (recommended)
2. `/app/receipt-wrangler-api/config/config.json` (optional)

Environment variables take precedence over the config file.

## Related Issues

- Issue #21: Docker deployment configuration guide

## License

See the main Receipt Wrangler repository for license information.
