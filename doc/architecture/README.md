# Mesh Router Architecture

This document describes the overall architecture of the Mesh Router system.

## Request Flow

```
                                                External Request
                                                      │
                                                      ▼
                                            ┌─────────────────┐
                                            │   Cloudflare    │
                                            │  (DNS + CDN)    │
                                            └────────┬────────┘
                                                     │
                                                     ▼
                                            ┌─────────────────┐
                                            │  NSL Server     │
                                            │  (Gateway)      │
                                            └────────┬────────┘
                                                     │
                         ┌───────────────────────────┴───────────────────────────┐
                         │                                                       │
                         ▼                                                       ▼
              ┌──────────────────────┐                             ┌──────────────────────┐
              │    Route 1: Direct   │                             │    Route 2: Tunnel   │
              │    (priority 1)      │                             │    (priority 2)      │
              │   via public IPv6    │                             │   via WireGuard VPN  │
              └──────────┬───────────┘                             └──────────┬───────────┘
                         │                                                    │
                         │                                         ┌──────────┴──────────┐
                         │                                         │  mesh-router-tunnel │
                         │                                         │     (Provider)      │
                         │                                         └──────────┬──────────┘
                         │                                                    │
                         └────────────────────────┬───────────────────────────┘
                                                  │
                                                  ▼
                                      ┌──────────────────────┐
                                      │    User's PCS        │
                                      │  (Caddy → Services)  │
                                      └──────────────────────┘
```

## Route Selection

1. **Direct Route (Priority 1)**: Used when the PCS has a public IP registered via mesh-router-agent
2. **Tunnel Route (Priority 2)**: Used when the PCS is behind NAT/CGNAT and connects via WireGuard tunnel

The gateway selects the highest-priority available route. If the direct route is unavailable or unhealthy, traffic falls back to the tunnel route.

## Components

| Component | Purpose | Technology |
|-----------|---------|------------|
| **mesh-router-gateway** | HTTP reverse proxy, SSL termination | OpenResty (Nginx + Lua) |
| **mesh-router-backend** | Domain/route registry, API | Express.js, Firebase, Redis |
| **mesh-router-tunnel** | WireGuard VPN for NAT traversal | TypeScript, WireGuard |
| **mesh-router-agent** | Direct IP registration | TypeScript, minimal |
| **mesh-dashboard** | User management UI | Next.js 14, React Admin |
| **Cloudflare** | DNS, CDN, DDoS protection | External service |

## Data Flows

### Registration Flow

```
mesh-dashboard ─── Firebase Auth ───► Firebase
       │
       │ (user registers domain)
       ▼
mesh-router-backend ───► Firestore (domain data)
                    ───► Redis (route cache)
```

### Route Registration Flow

```
mesh-router-agent ─── POST /routes/:userid/:sig ───► mesh-router-backend
       │                                                     │
       │ (direct IP, priority 1)                             │
       │                                                     ▼
       │                                                  Redis
       │
mesh-router-tunnel ─── POST /routes/:userid/:sig ───► mesh-router-backend
       │                                                     │
       │ (tunnel IP, priority 2)                             │
       │                                                     ▼
       │                                                  Redis
```

### Request Resolution Flow

```
mesh-router-gateway ─── GET /resolve/v2/:domain ───► mesh-router-backend
       │                                                     │
       │                                                     ▼
       │                                          Redis (route lookup)
       │                                                     │
       │◄────────── {routes: [{ip, port, priority}]} ────────┘
       │
       ▼
   Proxy to best route
```

## Domain Examples

- `casaos-username.nsl.sh` - CasaOS container management
- `immich-username.nsl.sh` - Immich photo management
- `nextcloud-username.nsl.sh` - Nextcloud file sync
- Custom: `*.mydomain.com` (with custom domain setup)

## Security

- **Authentication**: Firebase Auth for user registration, Ed25519 signatures for route registration
- **TLS**: Let's Encrypt certificates managed by gateway/Caddy
- **DDoS Protection**: Cloudflare in front of public endpoints
- **VPN**: WireGuard for encrypted tunnel traffic

## Diagram Files

Architecture diagrams are available as Excalidraw files:

| File | Description |
|------|-------------|
| `mesh-router/mesh-router-arch-v1.exdraw` | Legacy architecture diagram |
| `mesh-router/mesh-router-arch-v2.exdraw` | Current architecture diagram |

Open `.exdraw` files with [Excalidraw](https://excalidraw.com/) for interactive viewing and editing.
