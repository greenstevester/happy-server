# Deploying Happy Server with Cloudflare Tunnels

Self-hosting guide for exposing Happy Server via Cloudflare Tunnels.

## Prerequisites

- Docker and Docker Compose installed
- Cloudflare account with a domain
- `cloudflared` installed and authenticated

## Quick Start

```bash
# 1. Generate a secret
openssl rand -hex 32

# 2. Configure environment
cp .env.docker.example .env.docker
# Edit .env.docker with your secret and URLs

# 3. Start all services
docker compose up -d

# 4. Check logs
docker compose logs -f happy-server
```

## Services and Ports

All ports bind to localhost only - use Cloudflare Tunnels for public access.

| Service | Port | Purpose |
|---------|------|---------|
| Happy Server | 127.0.0.1:13005 | API + WebSocket |
| MinIO S3 | 127.0.0.1:19000 | File storage API |
| MinIO Console | 127.0.0.1:19001 | Storage web UI |
| PostgreSQL | 127.0.0.1:15432 | Database |
| Redis | 127.0.0.1:16379 | Cache/pubsub |

## Cloudflare Tunnel Setup

You need **2 tunnels** for full functionality:

| Hostname | Points to | Purpose |
|----------|-----------|---------|
| `happy.yourdomain.com` | `http://localhost:13005` | API + WebSocket |
| `happy-files.yourdomain.com` | `http://localhost:19000` | File downloads (avatars) |

### Example cloudflared config

In `~/.cloudflared/config.yml`:

```yaml
tunnel: your-tunnel-id
credentials-file: /path/to/credentials.json

ingress:
  - hostname: happy.yourdomain.com
    service: http://localhost:13005
  - hostname: happy-files.yourdomain.com
    service: http://localhost:19000
  - service: http_status:404
```

WebSockets are supported automatically by Cloudflare Tunnels.

## Environment Configuration

Edit `.env.docker`:

```bash
# Required: Generate with `openssl rand -hex 32`
HANDY_MASTER_SECRET=your-64-char-hex-string

# Required for public access: Your file storage URL
S3_PUBLIC_URL=https://happy-files.yourdomain.com/happy
```

## Deployment Commands

```bash
# Start all services
docker compose up -d

# Rebuild after code changes
docker compose up -d --build

# View logs
docker compose logs -f happy-server

# Stop everything
docker compose down

# Stop and remove volumes (WARNING: deletes all data)
docker compose down -v
```

## Verify Deployment

```bash
# Test API
curl https://happy.yourdomain.com/
# Expected: "Welcome to Happy Server!"

# Test file storage
curl https://happy-files.yourdomain.com/minio/health/live
# Expected: healthy response
```

## Client Configuration

Point your Claude Code clients to:
```
https://happy.yourdomain.com
```

## Security Notes

- All internal services (PostgreSQL, Redis, MinIO) bind to localhost only
- Only the Happy Server API and MinIO file access are exposed via tunnels
- All data is end-to-end encrypted on the client - the server stores only encrypted blobs
- The `HANDY_MASTER_SECRET` is used for server-side encryption of OAuth tokens only
