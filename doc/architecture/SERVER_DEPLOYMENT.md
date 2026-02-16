# Server-Side Deployment Guide

This guide covers deploying the mesh-router server components (backend, gateway, tunnel, dashboard) on a production or staging server.

## Prerequisites

- Docker and Docker Compose v2+
- Traefik reverse proxy (for SSL termination and routing)
- A domain with wildcard DNS pointing to your server

## Network Architecture

The mesh-router services communicate via two Docker networks:

```
                        Internet
                            │
                            ▼
                   ┌─────────────────┐
                   │     Traefik     │  (webtraefik network)
                   │   SSL + Routing │
                   └────────┬────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│   Dashboard  │   │   Backend    │   │   Gateway    │
│  (port 4321) │   │  (port 8192) │   │  (port 80)   │
└──────────────┘   └──────┬───────┘   └──────┬───────┘
                          │                   │
                          │  mesh-internal    │
                          │  (172.30.0.0/16)  │
                          │                   │
                   ┌──────┴───────────────────┴──────┐
                   │                                  │
                   │            Tunnel                │
                   │         (172.30.0.5)             │
                   │                                  │
                   └──────────────────────────────────┘
                                  │
                                  ▼
                          WireGuard VPN
                         (10.77.0.0/16)
```

## Network Setup

### 1. Create External Networks

Before starting any services, create the required external networks:

```bash
# Traefik network (for public-facing services)
docker network create webtraefik

# Internal mesh network (for service-to-service communication)
docker network create mesh-internal --subnet 172.30.0.0/16
```

### 2. Verify Networks

```bash
docker network ls | grep -E "webtraefik|mesh-internal"
```

## The mesh-internal Network

The `mesh-internal` network enables direct communication between mesh-router services without going through Traefik. This is critical for:

| Communication Path | Purpose |
|-------------------|---------|
| Gateway → Backend | Domain resolution API (`/resolve/v2/:domain`) |
| Tunnel → Backend | Route registration API (`/routes/:userid/:sig`) |
| Gateway → Tunnel | VPN traffic forwarding (via PROVIDER_ROUTE_IP) |

### IP Assignments

Services use static IPs on the mesh-internal network:

| Service | IP Address | Purpose |
|---------|------------|---------|
| Backend | dynamic | Accessed by hostname `mesh-router-backend-{env}` |
| Gateway | 172.30.0.4 | Fixed IP for Tunnel's PROVIDER_ROUTE_IP |
| Tunnel | 172.30.0.5 | Fixed IP for Gateway's routing target |

### Why Static IPs?

The tunnel needs to know where to forward incoming VPN traffic. By assigning a static IP to the gateway (172.30.0.4), the tunnel's `PROVIDER_ROUTE_IP` environment variable reliably routes traffic:

```yaml
# In tunnel docker-compose.yml
environment:
  - PROVIDER_ROUTE_IP=172.30.0.4
  - PROVIDER_ROUTE_PORT=80
```

## Service Configuration

### Backend

```yaml
services:
  mesh-router-backend-{env}:
    image: ghcr.io/yundera/mesh-router-backend:2.0.7
    networks:
      - webtraefik      # Traefik access for public API
      - mesh-internal   # Internal access from gateway/tunnel
    environment:
      - SERVER_DOMAIN={your-domain}
      - REDIS_URL=redis://mesh-router-backend-redis-{env}:6379
```

### Gateway

```yaml
services:
  mesh-router-gateway-{env}:
    image: ghcr.io/yundera/mesh-router-gateway:1.2.2
    networks:
      webtraefik:       # Traefik access for public traffic
      mesh-internal:
        ipv4_address: 172.30.0.4  # Static IP for tunnel routing
    environment:
      - SERVER_DOMAIN={your-domain}
      - BACKEND_URL=http://mesh-router-backend-{env}:8192
```

### Tunnel

```yaml
services:
  mesh-router-tunnel-{env}:
    image: ghcr.io/yundera/mesh-router-tunnel:1.2.2
    networks:
      webtraefik:       # Traefik access for API
      mesh-vpn:         # WireGuard VPN network
      mesh-internal:
        ipv4_address: 172.30.0.5  # Static IP
    environment:
      - PROVIDER_ROUTE_IP=172.30.0.4   # Gateway's static IP
      - PROVIDER_ROUTE_PORT=80
      - AUTH_API_URL=https://{your-domain}/router/api/verify
```

## DNS Configuration

### Gateway nginx.conf

The gateway's nginx.conf should only use Docker's internal DNS resolver:

```nginx
# Correct - Docker DNS only
resolver 127.0.0.11 valid=30s ipv6=off;

# WRONG - Public DNS causes 502 errors
# resolver 127.0.0.11 8.8.8.8 1.1.1.1 valid=30s ipv6=off;
```

**Why?** Public DNS servers (8.8.8.8, 1.1.1.1) return NXDOMAIN for Docker hostnames faster than Docker's DNS responds, causing intermittent 502 errors when resolving service names like `mesh-router-backend-{env}`.

## Troubleshooting

### Service Can't Reach Backend

```bash
# Test from gateway container
docker exec mesh-router-gateway-{env} curl -s http://mesh-router-backend-{env}:8192/health

# Check network connectivity
docker exec mesh-router-gateway-{env} ping -c 1 mesh-router-backend-{env}
```

### 502 Errors on Gateway

1. Check nginx.conf doesn't have public DNS resolvers
2. Verify backend is healthy: `docker logs mesh-router-backend-{env}`
3. Check mesh-internal network exists: `docker network inspect mesh-internal`

### Tunnel Traffic Not Reaching Gateway

1. Verify static IPs: `docker inspect mesh-router-gateway-{env} | grep 172.30`
2. Check PROVIDER_ROUTE_IP matches gateway's IP
3. Test connectivity: `docker exec mesh-router-tunnel-{env} curl http://172.30.0.4/_health`

## Complete Example

Directory structure for a deployment (e.g., `nsl.sh`):

```
/workspace/config/mesh-router-nsl/
├── backend/
│   ├── docker-compose.yml
│   └── serviceAccount.json
├── gateway/
│   ├── docker-compose.yml
│   ├── nginx.conf
│   └── gateway.conf
├── tunnel/
│   ├── docker-compose.yml
│   ├── config.lua
│   └── wg/
│       └── wg0.conf
└── dashboard/
    ├── docker-compose.yml
    ├── serviceAccount.json
    ├── firebaseConfig.json
    └── core.env.json
```

### Startup Order

```bash
# 1. Create networks (one-time)
docker network create webtraefik
docker network create mesh-internal --subnet 172.30.0.0/16

# 2. Start services in dependency order
cd /workspace/config/mesh-router-nsl
docker compose -f backend/docker-compose.yml up -d
docker compose -f gateway/docker-compose.yml up -d
docker compose -f tunnel/docker-compose.yml up -d
docker compose -f dashboard/docker-compose.yml up -d
```

## Related Documentation

- [Architecture Overview](./README.md)
- [Gateway README](../../mesh-router-gateway/README.md)
- [Tunnel README](../../mesh-router-tunnel/README.md)
- [Backend README](../../mesh-router-backend/backend-dev/README.md)
