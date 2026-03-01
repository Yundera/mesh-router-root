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
    │                          │                          │     appPrefix = "app"            │                      │
    │                          │                          │                                  │                      │
    │                          │                          │  4. Query resolve API            │                      │
    │                          │                          │─────────────────────────────────▶│                      │
    │                          │                          │  GET /resolve/v2/alice           │                      │
    │                          │                          │  (cached 60s at CF edge)         │                      │
    │                          │                          │                                  │                      │
    │                          │                          │  5. Return routes (pre-validated)│                      │
    │                          │                          │◀─────────────────────────────────│                      │
    │                          │                          │  {routes: [                      │                      │
    │                          │                          │    {type:"domain",               │                      │
    │                          │                          │     domain:"88-x.sslip.io",      │                      │
    │                          │                          │     priority:2},                 │                      │
    │                          │                          │    {type:"domain",               │                      │
    │                          │                          │     domain:"88-x.nip.io",        │                      │
    │                          │                          │     priority:3}                  │                      │
    │                          │                          │  ]}                              │                      │
    │                          │                          │                                  │                      │
    │                          │                          │  6. Filter type=domain           │                      │
    │                          │                          │     Select lowest priority       │                      │
    │                          │                          │     Prepend appPrefix            │                      │
    │                          │                          │                                  │                      │
    │                          │                          │  7. fetch() with CF pooling      │                      │
    │                          │                          │─────────────────────────────────────────────────────────▶│
    │                          │                          │  https://app-88-x.sslip.io:443   │                      │
    │                          │                          │  (domain from route, no IP→DNS)  │                      │
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

### Domain Route Selection

```
Routes are pre-registered and pre-validated by the agent.
CF Worker filters and uses domain routes directly.

┌─────────────────────────────────────────────────────────────┐
│                  DOMAIN ROUTE SELECTION                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Routes from API (already validated at registration):       │
│  [                                                          │
│    {type:"ip", ip:"88.187.147.189", priority:1},  ◀─ Skip  │
│    {type:"domain", domain:"88-187-147-189.sslip.io", p:2}, │
│    {type:"domain", domain:"88-187-147-189.nip.io", p:3}    │
│  ]                                                          │
│                                                             │
│  Worker processing:                                         │
│  1. Filter: type="domain" only                              │
│     → ["88-187-147-189.sslip.io", "88-187-147-189.nip.io"]  │
│                                                             │
│  2. Sort by priority (lowest first)                         │
│     → sslip.io (priority 2) selected                        │
│                                                             │
│  3. Skip if marked unhealthy (from previous failures)       │
│                                                             │
│  4. Prepend app prefix from request                         │
│     Request: app.alice.nsl.sh                               │
│     → appPrefix = "app"                                     │
│     → "app-88-187-147-189.sslip.io"                         │
│                                                             │
│  5. Build URL and fetch                                     │
│     → https://app-88-187-147-189.sslip.io:443/path          │
│                                                             │
└─────────────────────────────────────────────────────────────┘

Key change: Domain is pre-registered, not constructed from IP.
If sslip.io was unreachable at registration, it won't be in the list.
```

### DNS Services for Domain Routes

```
CF Workers cannot fetch() to raw IP addresses.
Solution: Use wildcard DNS services that embed the IP in the hostname.

┌─────────────────────────────────────────────────────────────┐
│                    DNS SERVICE OPTIONS                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  sslip.io (preferred, priority 2):                          │
│    88.187.147.189     → 88-187-147-189.sslip.io            │
│    2001:bc8:3021::1   → 2001-bc8-3021--1.sslip.io          │
│                                                             │
│  nip.io (fallback, priority 3):                             │
│    88.187.147.189     → 88-187-147-189.nip.io              │
│    2001:bc8:3021::1   → 2001-bc8-3021--1.nip.io            │
│                                                             │
│  Why two services?                                          │
│    • Redundancy: if one DNS service has issues, use other   │
│    • Validated at registration: only working ones stored    │
│    • sslip.io often faster/more reliable (hence priority 2) │
│                                                             │
└─────────────────────────────────────────────────────────────┘

PCS certificates must include SANs for these domains.
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

## Route Types

Routes are categorized by `type` field and selected based on the routing component:

```
┌─────────────────────────────────────────────────────────────┐
│                       ROUTE TYPES                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  IP ROUTES (type: "ip")                                     │
│  ─────────────────────────────────────                      │
│  Used by: OpenResty Gateway (can connect to IPs directly)   │
│                                                             │
│  ┌──────────┬──────────┬────────────────────────────────┐   │
│  │ Priority │ Source   │ Example                        │   │
│  ├──────────┼──────────┼────────────────────────────────┤   │
│  │ 1        │ agent    │ 88.187.147.189:443 (public)    │   │
│  │ 10       │ tunnel   │ 172.30.0.3:80 (WireGuard)      │   │
│  └──────────┴──────────┴────────────────────────────────┘   │
│                                                             │
│  ─────────────────────────────────────────────────────────  │
│                                                             │
│  DOMAIN ROUTES (type: "domain")                             │
│  ─────────────────────────────────────                      │
│  Used by: CF Worker (cannot connect to IPs, uses DNS)       │
│                                                             │
│  ┌──────────┬──────────┬────────────────────────────────┐   │
│  │ Priority │ Source   │ Example                        │   │
│  ├──────────┼──────────┼────────────────────────────────┤   │
│  │ 2        │ agent    │ 88-187-147-189.sslip.io:443    │   │
│  │ 3        │ agent    │ 88-187-147-189.nip.io:443      │   │
│  └──────────┴──────────┴────────────────────────────────┘   │
│                                                             │
│  Domain routes use wildcard DNS services (sslip.io, nip.io) │
│  that resolve embedded IPs back to the actual IP address.   │
│                                                             │
└─────────────────────────────────────────────────────────────┘

Route selection: Lower priority number wins.
CF Worker uses domain routes, Gateway uses IP routes.
If all domain routes fail, CF Worker falls back to Gateway.
```

## Route Registration & Validation

Routes are validated at registration time before being stored:

```
┌─────────────────────────────────────────────────────────────┐
│                  ROUTE VALIDATION FLOW                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Agent submits routes to backend:                           │
│  POST /routes/:userid/:sig                                  │
│  {                                                          │
│    "routes": [                                              │
│      {"type": "ip", "ip": "88.187.147.189", "port": 443},  │
│      {"type": "domain", "domain": "88-187-147-189.sslip.io"}│
│      {"type": "domain", "domain": "88-187-147-189.nip.io"} │
│    ]                                                        │
│  }                                                          │
│                           │                                 │
│                           ▼                                 │
│  Backend validates each route:                              │
│                                                             │
│  For each route:                                            │
│  ├─ Build test URL:                                         │
│  │   IP route:     https://88.187.147.189:443/health        │
│  │   Domain route: https://88-187-147-189.sslip.io/health   │
│  │                                                          │
│  ├─ Test connectivity (5 second timeout)                    │
│  │   ├─ Connection refused? → Reject                        │
│  │   ├─ Timeout? → Reject                                   │
│  │   └─ Connected? → Continue                               │
│  │                                                          │
│  └─ Verify SSL (if HTTPS)                                   │
│      ├─ Cert valid? → Accept                                │
│      └─ Cert invalid? → Reject                              │
│                           │                                 │
│                           ▼                                 │
│  Store only healthy routes in Redis                         │
│  Return: { accepted: [...], rejected: [...] }               │
│                                                             │
└─────────────────────────────────────────────────────────────┘

Benefits:
• Bad routes never reach Redis → No timeout delays
• nip.io/sslip.io outages detected at registration
• Agent knows which routes failed and why
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
