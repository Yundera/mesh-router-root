# Data Flow

This document provides detailed diagrams showing how data flows through each gateway mode in Mesh Router.

---

## Request Flow Overview

All gateway modes follow a similar high-level pattern:

```
Client Request
      │
      ▼
┌─────────────┐
│  DNS        │  Resolves *.user.domain to gateway/CF
│  Resolution │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Gateway    │  Routes request to correct PCS
│  Layer      │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  PCS        │  Serves the application
│  Instance   │
└─────────────┘
```

---

## Mode 1: OpenResty Gateway Flow

### Complete Request Lifecycle

```
 CLIENT                    CLOUDFLARE              OPENRESTY GATEWAY                    BACKEND API                PCS
    │                          │                          │                                  │                      │
    │  1. HTTPS Request        │                          │                                  │                      │
    │  app.alice.nsl.sh/api    │                          │                                  │                      │
    │─────────────────────────▶│                          │                                  │                      │
    │                          │                          │                                  │                      │
    │                          │  2. Forward to origin    │                                  │                      │
    │                          │─────────────────────────▶│                                  │                      │
    │                          │                          │                                  │                      │
    │                          │                          │  3. Lua: Extract subdomain       │                      │
    │                          │                          │     "alice" from Host header     │                      │
    │                          │                          │                                  │                      │
    │                          │                          │  4. Check local cache            │                      │
    │                          │                          │     (10MB shared dict)           │                      │
    │                          │                          │                                  │                      │
    │                          │                          │  5. Cache MISS                   │                      │
    │                          │                          │─────────────────────────────────▶│                      │
    │                          │                          │  GET /resolve/v2/alice           │                      │
    │                          │                          │                                  │                      │
    │                          │                          │  6. Return routes                │                      │
    │                          │                          │◀─────────────────────────────────│                      │
    │                          │                          │  {routes: [{ip, port, priority}]}│                      │
    │                          │                          │                                  │                      │
    │                          │                          │  7. Select best route            │                      │
    │                          │                          │     (lowest priority, healthy)   │                      │
    │                          │                          │                                  │                      │
    │                          │                          │  8. NEW TLS Handshake            │                      │
    │                          │                          │─────────────────────────────────────────────────────────▶│
    │                          │                          │  (100-120ms overhead)            │                      │
    │                          │                          │                                  │                      │
    │                          │                          │  9. Proxy request                │                      │
    │                          │                          │─────────────────────────────────────────────────────────▶│
    │                          │                          │  Headers: Host, X-Real-IP,       │                      │
    │                          │                          │  X-Forwarded-For/Proto           │                      │
    │                          │                          │                                  │                      │
    │                          │                          │  10. Response from app           │                      │
    │                          │                          │◀─────────────────────────────────────────────────────────│
    │                          │                          │                                  │                      │
    │                          │  11. Return response     │                                  │                      │
    │                          │◀─────────────────────────│                                  │                      │
    │                          │                          │                                  │                      │
    │  12. Response            │                          │                                  │                      │
    │◀─────────────────────────│                          │                                  │                      │
    │                          │                          │                                  │                      │

Total latency: ~283ms (p50)
├── Client → CF: ~30ms
├── CF → Gateway: ~20ms
├── Resolver lookup: <1ms (cache) or ~50ms (API call)
├── Gateway → PCS TLS: ~100-120ms  ◀── MAIN BOTTLENECK (no session reuse)
├── PCS processing: ~30ms
└── Return path: ~50ms
```

### Lua Resolver Logic

```
┌─────────────────────────────────────────────────────────────┐
│                    Lua Resolver (access phase)              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Parse Host Header                                       │
│     ┌─────────────────────────────────────────────────┐     │
│     │ app.alice.nsl.sh → subdomain = "alice"         │     │
│     │ alice.nsl.sh     → subdomain = "alice"         │     │
│     │ nsl.sh           → subdomain = nil (default)   │     │
│     └─────────────────────────────────────────────────┘     │
│                           │                                 │
│                           ▼                                 │
│  2. Check Cache                                             │
│     ┌─────────────────────────────────────────────────┐     │
│     │ shared_dict "resolver_cache" (10MB)            │     │
│     │ Key: "alice"                                   │     │
│     │ TTL: 60-300 seconds                            │     │
│     └─────────────────────────────────────────────────┘     │
│                           │                                 │
│              ┌────────────┴────────────┐                    │
│              │                         │                    │
│           HIT                        MISS                   │
│              │                         │                    │
│              ▼                         ▼                    │
│     Use cached routes          Query Backend API            │
│                                GET /resolve/v2/alice        │
│                                        │                    │
│                                        ▼                    │
│                               Store in cache                │
│                                        │                    │
│              └────────────┬────────────┘                    │
│                           │                                 │
│                           ▼                                 │
│  3. Route Selection                                         │
│     ┌─────────────────────────────────────────────────┐     │
│     │ Routes sorted by priority (1 = best)           │     │
│     │                                                 │     │
│     │ For each route:                                │     │
│     │   ├── No healthCheck? → Use immediately       │     │
│     │   └── Has healthCheck?                        │     │
│     │       ├── Check health cache (5 min TTL)      │     │
│     │       ├── If expired → HTTP HEAD probe        │     │
│     │       └── If healthy → Use this route         │     │
│     │                                                 │     │
│     │ All unhealthy? → Use first route as fallback  │     │
│     └─────────────────────────────────────────────────┘     │
│                           │                                 │
│                           ▼                                 │
│  4. Set Backend Variable                                    │
│     ngx.var.backend = "https://[ip]:port"                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Mode 2: Cloudflare Worker Flow

### Complete Request Lifecycle

```
 CLIENT                    CLOUDFLARE EDGE           CF WORKER                            BACKEND API                PCS
    │                          │                          │                                  │                      │
    │  1. HTTPS Request        │                          │                                  │                      │
    │  app.alice.nsl.sh/api    │                          │                                  │                      │
    │─────────────────────────▶│                          │                                  │                      │
    │                          │                          │                                  │                      │
    │                          │  2. Route to Worker      │                                  │                      │
    │                          │─────────────────────────▶│                                  │                      │
    │                          │  (edge, <10ms)           │                                  │                      │
    │                          │                          │                                  │                      │
    │                          │                          │  3. Extract subdomain            │                      │
    │                          │                          │     "alice" from Host header     │                      │
    │                          │                          │                                  │                      │
    │                          │                          │  4. Query resolve API            │                      │
    │                          │                          │─────────────────────────────────▶│                      │
    │                          │                          │  GET /resolve/v2/alice           │                      │
    │                          │                          │  (cached 60s at CF edge)         │                      │
    │                          │                          │                                  │                      │
    │                          │                          │  5. Return routes                │                      │
    │                          │                          │◀─────────────────────────────────│                      │
    │                          │                          │  {routes: [{ip, port}]}          │                      │
    │                          │                          │                                  │                      │
    │                          │                          │  6. Convert IP to nip.io         │                      │
    │                          │                          │     203.0.113.5 →                │                      │
    │                          │                          │     203.0.113.5.nip.io           │                      │
    │                          │                          │                                  │                      │
    │                          │                          │  7. fetch() with CF pooling      │                      │
    │                          │                          │─────────────────────────────────────────────────────────▶│
    │                          │                          │  https://203.0.113.5.nip.io:10443│                      │
    │                          │                          │  (TLS session reuse automatic)   │                      │
    │                          │                          │                                  │                      │
    │                          │                          │  8. Response from app            │                      │
    │                          │                          │◀─────────────────────────────────────────────────────────│
    │                          │                          │                                  │                      │
    │                          │  9. Return response      │                                  │                      │
    │                          │◀─────────────────────────│                                  │                      │
    │                          │                          │                                  │                      │
    │  10. Response            │                          │                                  │                      │
    │◀─────────────────────────│                          │                                  │                      │
    │                          │                          │                                  │                      │

Total latency: ~169ms (p50)
├── Client → CF Edge: ~30ms
├── Worker execution: <10ms
├── Resolve API (cached): <5ms
├── CF → PCS (with pooling): ~100ms  ◀── Faster due to connection reuse
├── PCS processing: ~30ms
└── Return path: minimal (same connection)
```

### nip.io DNS Resolution

```
CF Workers cannot fetch() to raw IP addresses.
Solution: Use nip.io to convert IPs to hostnames.

┌─────────────────────────────────────────────────────────────┐
│                    nip.io Conversion                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  IPv4:                                                      │
│    192.168.1.1          → 192.168.1.1.nip.io               │
│    203.0.113.5:10443    → 203.0.113.5.nip.io:10443         │
│                                                             │
│  IPv6:                                                      │
│    2001:bc8:3021::1     → 2001-bc8-3021--1.nip.io          │
│    (colons replaced with hyphens)                           │
│                                                             │
│  DNS Resolution:                                            │
│    nip.io returns the embedded IP address                   │
│    203.0.113.5.nip.io → A record: 203.0.113.5              │
│                                                             │
└─────────────────────────────────────────────────────────────┘

IMPORTANT: PCS certificates must include nip.io SANs for this to work.
Example SANs:
  - *.alice.nsl.sh
  - alice.nsl.sh
  - 203.0.113.5.nip.io  ◀── Required for CF Worker
```

---

## Mode 3: CF Worker + Fallback Flow

### Complete Request Lifecycle (with fallback)

```
 CLIENT                    CF EDGE                  CF WORKER                        PCS / GATEWAY
    │                          │                          │                                │
    │  1. HTTPS Request        │                          │                                │
    │─────────────────────────▶│                          │                                │
    │                          │                          │                                │
    │                          │  2. Route to Worker      │                                │
    │                          │─────────────────────────▶│                                │
    │                          │                          │                                │
    │                          │                          │  3. Resolve domain             │
    │                          │                          │  4. Try direct to PCS          │
    │                          │                          │────────────────────────────────▶│ PCS
    │                          │                          │  fetch(nip.io:10443)           │
    │                          │                          │                                │
    │                          │                          │         ┌────────────────────┐ │
    │                          │                          │         │   TLS HANDSHAKE    │ │
    │                          │                          │         │   FAILED           │ │
    │                          │                          │         │   (COTS not set)   │ │
    │                          │                          │         └────────────────────┘ │
    │                          │                          │                                │
    │                          │                          │  5. Catch error or 502         │
    │                          │                          │     Check: server=cloudflare   │
    │                          │                          │                                │
    │                          │                          │  6. Fallback to Gateway        │
    │                          │                          │────────────────────────────────▶│ GATEWAY
    │                          │                          │  fetch(gateway.entrypoint...)  │
    │                          │                          │                                │
    │                          │                          │                         ┌──────┴──────┐
    │                          │                          │                         │   Gateway   │
    │                          │                          │                         │   resolves  │
    │                          │                          │                         │   & proxies │
    │                          │                          │                         └──────┬──────┘
    │                          │                          │                                │
    │                          │                          │                                ▼
    │                          │                          │                         ┌───────────┐
    │                          │                          │                         │    PCS    │
    │                          │                          │                         └─────┬─────┘
    │                          │                          │                               │
    │                          │                          │  7. Response via Gateway      │
    │                          │                          │◀───────────────────────────────│
    │                          │                          │                                │
    │                          │  8. Return response      │                                │
    │                          │◀─────────────────────────│                                │
    │                          │                          │                                │
    │  9. Response             │                          │                                │
    │◀─────────────────────────│                          │                                │
    │                          │                          │                                │
```

### Fallback Decision Logic

```
┌─────────────────────────────────────────────────────────────┐
│                    Fallback Decision Tree                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  try {                                                      │
│    response = await fetch(nip.io URL)                       │
│  }                                                          │
│                           │                                 │
│              ┌────────────┴────────────┐                    │
│              │                         │                    │
│          SUCCESS                    EXCEPTION               │
│              │                         │                    │
│              ▼                         ▼                    │
│     Check status code           Check if CF SSL error       │
│              │                         │                    │
│     ┌────────┴────────┐     ┌──────────┴──────────┐        │
│     │                 │     │                     │        │
│   200-499           502    SSL Error          Other        │
│     │                 │     │                     │        │
│     ▼                 ▼     ▼                     ▼        │
│  Return          Check        FALLBACK         Re-throw    │
│  response       headers       to Gateway                   │
│                    │                                       │
│            ┌───────┴───────┐                               │
│            │               │                               │
│     server:cloudflare    other                             │
│            │               │                               │
│            ▼               ▼                               │
│       FALLBACK        Return 502                           │
│       to Gateway      (PCS error)                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘

Fallback URL format:
  https://gateway.entrypoint.{server-domain}{path}

Why "gateway.entrypoint."?
  - Prevents routing loops
  - CF Worker ignores requests to this subdomain
  - Gateway handles it directly
```

---

## Route Types: Direct vs Tunnel

Both gateway modes support two route types:

```
┌─────────────────────────────────────────────────────────────┐
│                       ROUTE TYPES                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  DIRECT ROUTE (via mesh-router-agent)                       │
│  ─────────────────────────────────────                      │
│  • PCS has public IP                                        │
│  • Agent registers IP with backend                          │
│  • Gateway proxies directly to PCS                          │
│  • Priority: 1 (preferred)                                  │
│  • Latency: Lower                                           │
│                                                             │
│     Gateway ──────────────────────────▶ PCS (public IP)     │
│                                                             │
│  ─────────────────────────────────────────────────────────  │
│                                                             │
│  TUNNEL ROUTE (via mesh-router-tunnel)                      │
│  ─────────────────────────────────────                      │
│  • PCS behind NAT/firewall                                  │
│  • WireGuard tunnel to tunnel hub                           │
│  • Gateway proxies to tunnel hub                            │
│  • Priority: 2 (fallback)                                   │
│  • Latency: Higher (extra hop)                              │
│                                                             │
│     Gateway ──▶ Tunnel Hub ══WireGuard══▶ PCS (private IP)  │
│                                                             │
└─────────────────────────────────────────────────────────────┘

Route selection: Lower priority number wins.
If direct route is healthy, it's always used.
Tunnel is fallback when direct fails.
```

---

## Health Check Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    HEALTH CHECK FLOW                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Route Configuration:                                       │
│  {                                                          │
│    "ip": "203.0.113.5",                                    │
│    "port": 443,                                            │
│    "priority": 1,                                          │
│    "healthCheck": {                                        │
│      "path": "/.well-known/health",                        │
│      "host": "alice.nsl.sh"    // optional                 │
│    }                                                       │
│  }                                                          │
│                                                             │
│  ─────────────────────────────────────────────────────────  │
│                                                             │
│  Active Health Check (before routing):                      │
│                                                             │
│    1. Check health cache (TTL: 5 minutes)                   │
│                                                             │
│       Cache HIT & healthy    → Use route                    │
│       Cache HIT & unhealthy  → Skip route                   │
│       Cache MISS             → Probe endpoint               │
│                                                             │
│    2. HTTP HEAD probe                                       │
│       HEAD https://ip:port/.well-known/health               │
│       Timeout: 2 seconds                                    │
│                                                             │
│       Status 200  → Mark healthy, cache 5 min               │
│       Other       → Mark unhealthy, cache 5 min             │
│                                                             │
│  ─────────────────────────────────────────────────────────  │
│                                                             │
│  Passive Health Tracking (after routing):                   │
│                                                             │
│    Gateway tracks proxy failures per route.                 │
│                                                             │
│    Failure (502, timeout, connection refused):              │
│      → Increment failure counter                            │
│      → 3 consecutive failures → Mark unhealthy 1 min        │
│                                                             │
│    Success (any 2xx-5xx from PCS):                          │
│      → Reset failure counter                                │
│      → Mark healthy                                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Related Documentation

- [Gateway Modes](./GATEWAY_MODES.md) - Comparison and decision tree
- [Security in Transit](./SECURITY_IN_TRANSIT.md) - Certificate and encryption details
- [Routing Architecture](../mesh-router-gateway/doc/ROUTING_ARCHITECTURE.md) - Multi-route failover details
