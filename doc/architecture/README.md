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
                                            │  CF Worker      │
                                            │  (Edge Routing) │
                                            └────────┬────────┘
                                                     │
                         ┌───────────────────────────┴───────────────────────────┐
                         │                                                       │
                         ▼                                                       ▼
              ┌──────────────────────┐                             ┌──────────────────────┐
              │  Domain Routes       │                             │  Gateway Fallback    │
              │  (sslip.io, nip.io)  │                             │  (OpenResty)         │
              │  priority 2-3        │                             │                      │
              └──────────┬───────────┘                             └──────────┬───────────┘
                         │                                                    │
                         │                                         ┌──────────┴──────────┐
                         │                                         │  IP Routes + Tunnel │
                         │                                         │  (priority 1, 10)   │
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

## Route Types

Routes are categorized by type and selected by priority (lower number = higher priority):

| Type | Priority | Source | Example | Used By |
|------|----------|--------|---------|---------|
| `ip` | 1 | agent | `88.187.147.189:443` | Gateway (OpenResty) |
| `domain` | 2 | agent | `88-187-147-189.sslip.io` | CF Worker |
| `domain` | 3 | agent | `88-187-147-189.nip.io` | CF Worker |
| `ip` | 10 | tunnel | `172.30.0.3:80` | Gateway (via WireGuard) |

### Why Two Route Types?

- **CF Workers cannot fetch IP addresses directly** (Cloudflare error 1003)
- **Domain routes** (`sslip.io`, `nip.io`) are DNS services that resolve to embedded IPs
- **IP routes** are used by the OpenResty gateway which can connect to IPs directly

### Route Selection by Component

| Component | Route Types Used | Selection Logic |
|-----------|------------------|-----------------|
| **CF Worker** | `domain` only | Pick lowest priority healthy domain route |
| **OpenResty Gateway** | `ip` only | Pick lowest priority healthy IP route |

## Route Registration & Validation

Routes are validated at registration time by the backend:

```
Agent                           Backend                         Redis
  │                                │                               │
  ├─ POST /routes                  │                               │
  │  { routes: [                   │                               │
  │    {type: "ip", ip: "..."},    │                               │
  │    {type: "domain", domain: "...sslip.io"},                    │
  │    {type: "domain", domain: "...nip.io"}                       │
  │  ]}                            │                               │
  │ ───────────────────────────────>                               │
  │                                │                               │
  │                    For each route:                             │
  │                    ├─ Test connectivity (5s timeout)           │
  │                    ├─ Verify SSL if HTTPS                      │
  │                    └─ Only store if healthy                    │
  │                                │                               │
  │                                ├───────────────────────────────>
  │                                │  Store healthy routes only    │
  │<───────────────────────────────┤                               │
  │  { accepted: [...], rejected: [...] }                          │
```

**Key Benefit**: Bad routes are never stored, eliminating timeout delays in the request path.

## Components

| Component | Purpose | Technology |
|-----------|---------|------------|
| **mesh-router-gateway-cf** | Edge routing via domain routes | Cloudflare Worker |
| **mesh-router-gateway** | Fallback proxy, IP route handling | OpenResty (Nginx + Lua) |
| **mesh-router-backend** | Domain/route registry, route validation | Express.js, Firebase, Redis |
| **mesh-router-tunnel** | WireGuard VPN for NAT traversal | TypeScript, WireGuard |
| **mesh-router-agent** | IP + domain route registration | TypeScript |
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
       │ Routes submitted:                                   │ Validation:
       │ ├─ {type: ip, ip: 88.x.x.x, priority: 1}           │ ├─ Test connectivity
       │ ├─ {type: domain, domain: *.sslip.io, priority: 2} │ ├─ Verify SSL
       │ └─ {type: domain, domain: *.nip.io, priority: 3}   │ └─ Store only healthy
       │                                                     ▼
       │                                                  Redis
       │                                               (healthy routes)
       │
mesh-router-tunnel ─── POST /routes/:userid/:sig ───► mesh-router-backend
       │                                                     │
       │ Routes submitted:                                   │
       │ └─ {type: ip, ip: 172.30.x.x, priority: 10}        │
       │                                                     ▼
       │                                                  Redis
```

### Request Resolution Flow

**CF Worker (primary path):**
```
CF Worker ─── GET /resolve/v2/:domain ───► mesh-router-backend
    │                                              │
    │                                              ▼
    │                                   Redis (route lookup)
    │                                              │
    │◄── {routes: [{type, ip/domain, port, priority}]} ──┘
    │
    ├─ Filter: type=domain only
    ├─ Sort by priority (lowest first)
    ├─ Skip unhealthy routes
    │
    ▼
 Proxy to: https://{appPrefix}-{route.domain}:{port}/
 Example:  https://admin-88-187-147-189.sslip.io:443/
```

**OpenResty Gateway (fallback path):**
```
OpenResty ─── GET /resolve/v2/:domain ───► mesh-router-backend
    │                                              │
    │                                              ▼
    │                                   Redis (route lookup)
    │                                              │
    │◄── {routes: [{type, ip/domain, port, priority}]} ──┘
    │
    ├─ Filter: type=ip only
    ├─ Sort by priority (lowest first)
    ├─ Check health status
    │
    ▼
 Proxy to: https://[route.ip]:{port}/
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

## Deployment

For server-side deployment instructions including network configuration:

- [Server Deployment Guide](./SERVER_DEPLOYMENT.md) - Backend, gateway, tunnel, and dashboard deployment with mesh-internal network setup

## Diagram Files

Architecture diagrams are available as Excalidraw files:

| File | Description |
|------|-------------|
| `mesh-router/mesh-router-arch-v1.exdraw` | Legacy architecture diagram |
| `mesh-router/mesh-router-arch-v2.exdraw` | Current architecture diagram |

Open `.exdraw` files with [Excalidraw](https://excalidraw.com/) for interactive viewing and editing.
