# Mesh Router Development Stack

This folder contains everything needed to run the complete Mesh Router stack locally for development and testing.

## Architecture

```
                         Internet
                            │
              ┌─────────────┴─────────────┐
              │                           │
              ▼                           ▼
    ┌──────────────────┐       ┌──────────────────┐
    │  VPN Tunnel      │       │  Direct IP       │
    │  (mesh-router-   │       │  (mesh-router-   │
    │   tunnel)        │       │   agent)         │
    └────────┬─────────┘       └────────┬─────────┘
             │                          │
             └──────────┬───────────────┘
                        │
                        ▼
               ┌──────────────────┐
               │      Caddy       │
               │  (reverse proxy) │
               └────────┬─────────┘
                        │
                        ▼
               ┌──────────────────┐
               │     CasaOS       │
               │  (or your app)   │
               └──────────────────┘
```

## Prerequisites

- Docker and Docker Compose
- A registered domain on [NSL.sh](https://nsl.sh) or [Yundera](https://yundera.com)
- Your provider credentials (user ID and signature)

## Quick Start

### 1. Configure Environment

```bash
# Copy the example environment file
cp .env.example .env

# Edit .env with your credentials
nano .env  # or use your preferred editor
```

Required values:
- `PROVIDER`: Your connection string from NSL.sh/Yundera (`<url>,<userid>,<signature>`)
- `PCS_DOMAIN`: Your assigned domain (e.g., `myname.nsl.sh`)
- `PUBLIC_IP`: Your public IP (for direct routing via mesh-router-agent)

Recommended (for backup):
- `PRIVATE_KEY`: Your Ed25519 private key used to generate the signature

> **Note**: The `PRIVATE_KEY` is not used at runtime but is recommended to keep in your `.env` file for reference. You'll need it to regenerate the signature if you switch providers or need to rotate keys.

### 2. Start the Stack

```bash
docker compose up -d
```

### 3. View Logs

```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f mesh-router-tunnel
```

### 4. Access Your Services

Once running, your services will be accessible at:
- **Via VPN tunnel**: `https://<yourname>.nsl.sh`
- **Direct IP** (if configured): `https://<your-public-ip>:10443`

## Services

| Service | Description | Port |
|---------|-------------|------|
| `mesh-router-tunnel` | WireGuard VPN tunnel to provider | - |
| `mesh-router-agent` | Direct IP registration agent | - |
| `caddy` | Reverse proxy | 10443 (external) |
| `casaos` | Container management UI | 8080 (internal) |

## Configuration Files

| File | Description |
|------|-------------|
| `.env` | Environment variables (create from `.env.example`) |
| `docker-compose.yml` | Service definitions |
| `Caddyfile` | Caddy reverse proxy configuration |

## Customization

### Using Your Own Application

Replace the `casaos` service with your own application:

```yaml
services:
  myapp:
    image: your-app:latest
    networks:
      - pcs
    # ... your config
```

Update the `Caddyfile` to point to your app:

```
:80 {
    reverse_proxy myapp:3000
}
```

### VPN-Only Mode

If you don't need direct IP routing, remove the `mesh-router-agent` service from `docker-compose.yml`.

### Direct IP Only Mode

If your server has a public IP and doesn't need VPN tunneling, remove the `mesh-router-tunnel` service and keep only `mesh-router-agent`.

## Troubleshooting

### VPN Connection Issues

```bash
# Check mesh-router-tunnel logs
docker compose logs mesh-router-tunnel

# Verify WireGuard interface
docker compose exec mesh-router-tunnel wg show
```

### Caddy Not Routing

```bash
# Check Caddy logs
docker compose logs caddy

# Verify Caddyfile syntax
docker compose exec caddy caddy validate --config /etc/caddy/Caddyfile
```

### Agent Not Registering IP

```bash
# Check agent logs
docker compose logs mesh-router-agent

# Verify your PUBLIC_IP is correct
curl -4 ifconfig.me  # IPv4
curl -6 ifconfig.me  # IPv6
```

## Cleanup

```bash
# Stop all services
docker compose down

# Stop and remove volumes
docker compose down -v
```

## Related Documentation

- [mesh-router-tunnel README](../mesh-router-tunnel/README.md)
- [mesh-router-agent README](../mesh-router-agent/README.md)
- [mesh-router-backend README](../mesh-router-backend/README.md)
