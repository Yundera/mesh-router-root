# Gateway Modes

This document explains the three gateway configurations available in Mesh Router and helps you choose the right one for your deployment.

---

## Overview

Mesh Router supports three gateway modes for routing traffic to PCS instances:

| Mode | Description | Best For |
|------|-------------|----------|
| **Cloudflare Worker Only** | Edge-based routing via CF Workers | Best performance, production deployments |
| **OpenResty Gateway Only** | Self-hosted reverse proxy | Simple setup, full control |
| **CF Worker + Gateway Fallback** | CF Worker with OpenResty fallback | Maximum reliability |

---

## Mode 1: Cloudflare Worker Only

Traffic is routed entirely through Cloudflare's edge network using a Worker that resolves domains and proxies directly to PCS instances.

```
┌──────────┐     ┌─────────────────┐     ┌─────────────────┐     ┌─────────┐
│  Client  │────▶│  Cloudflare     │────▶│   CF Worker     │────▶│   PCS   │
│          │     │  Edge           │     │  (resolve +     │     │         │
└──────────┘     └─────────────────┘     │   proxy)        │     └─────────┘
                                         └─────────────────┘
                                                │
                                                ▼
                                         Uses nip.io DNS
                                         (IP → hostname)
```

### Configuration

```bash
# mesh-router-gateway-cf wrangler.toml
[vars]
RESOLVE_URL = "https://backend.example.com/resolve/v2"
# GATEWAY_FALLBACK_URL not set = no fallback
```

### Requirements

- Cloudflare account with Workers enabled
- **Advanced Certificate Manager (ACM)**: $10/month per zone
- **Custom Origin Trust Store (COTS)**: Upload mesh-router CA certificate
- PCS certificates must include nip.io SANs

### Performance

| Metric | Value |
|--------|-------|
| Throughput | ~249 req/s |
| p50 Latency | ~169ms |
| p95 Latency | ~246ms |

### Pros & Cons

| Pros | Cons |
|------|------|
| Best performance (CF connection pooling) | Requires COTS setup ($10/mo) |
| Global edge distribution | Cloudflare dependency |
| Automatic DDoS protection | nip.io dependency for IP routing |
| No self-hosted infrastructure | More complex initial setup |

---

## Mode 2: OpenResty Gateway Only

Traffic is routed through a self-hosted OpenResty (Nginx + Lua) gateway that handles domain resolution and proxying.

```
┌──────────┐     ┌─────────────────┐     ┌─────────────────┐     ┌─────────┐
│  Client  │────▶│  Cloudflare     │────▶│   OpenResty     │────▶│   PCS   │
│          │     │  CDN            │     │   Gateway       │     │         │
└──────────┘     └─────────────────┘     │  (Nginx + Lua)  │     └─────────┘
                                         └─────────────────┘
                                                │
                                                ▼
                                         Lua resolver queries
                                         mesh-router-backend
```

### Configuration

```bash
# Environment variables
RESOLVER_ENDPOINT=http://backend:8192/resolve/v2
CACHE_TTL=60
DEFAULT_BACKEND=https://default.example.com
```

### Requirements

- Server to host OpenResty gateway
- mesh-router-backend running and accessible
- SSL certificates (Let's Encrypt or custom)

### Performance

| Metric | Value |
|--------|-------|
| Throughput | ~159 req/s |
| p50 Latency | ~283ms |
| p95 Latency | ~517ms |

**Note**: Higher latency due to no SSL session reuse between gateway and PCS (architectural limitation of dynamic `proxy_pass`).

### Pros & Cons

| Pros | Cons |
|------|------|
| Full control over infrastructure | Higher latency than CF Worker |
| No external dependencies | Requires server maintenance |
| Simple setup | No automatic connection pooling |
| Works without COTS | Single point of failure |

---

## Mode 3: CF Worker + Gateway Fallback

CF Worker attempts direct connection to PCS; if that fails (e.g., COTS not configured), falls back to OpenResty gateway.

```
┌──────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Client  │────▶│  Cloudflare     │────▶│   CF Worker     │
│          │     │  Edge           │     │                 │
└──────────┘     └─────────────────┘     └────────┬────────┘
                                                  │
                                    ┌─────────────┴─────────────┐
                                    │                           │
                              TRY DIRECT                   ON FAILURE
                                    │                           │
                                    ▼                           ▼
                           ┌───────────────┐          ┌───────────────┐
                           │     PCS       │          │   OpenResty   │
                           │  (via nip.io) │          │   Gateway     │
                           └───────────────┘          └───────┬───────┘
                                                              │
                                                              ▼
                                                      ┌───────────────┐
                                                      │     PCS       │
                                                      └───────────────┘
```

### Configuration

```bash
# mesh-router-gateway-cf wrangler.toml
[vars]
RESOLVE_URL = "https://backend.example.com/resolve/v2"
GATEWAY_FALLBACK_URL = "https://gateway.entrypoint.example.com"
```

**Important**: The fallback URL must use a different subdomain (`gateway.entrypoint.`) to prevent routing loops.

### Fallback Triggers

The CF Worker falls back to the gateway when:
1. Direct HTTPS connection fails (SSL handshake error)
2. Cloudflare returns 502 with `server: cloudflare` header (COTS not working)
3. Connection timeout to PCS

### Requirements

- All requirements from Mode 1 (CF Worker)
- All requirements from Mode 2 (OpenResty Gateway)
- DNS configured for fallback subdomain

### Performance

| Scenario | Latency |
|----------|---------|
| Direct success | ~169ms (p50) |
| Fallback to gateway | ~280ms (p50) |

### Pros & Cons

| Pros | Cons |
|------|------|
| Maximum reliability | Most complex setup |
| Graceful degradation | Requires both infrastructures |
| Best of both modes | Higher cost (COTS + server) |
| Works during COTS issues | Two systems to maintain |

---

## Decision Tree

Use this flowchart to choose the right mode:

```
                        ┌─────────────────────────────┐
                        │  Do you need maximum        │
                        │  performance?               │
                        └─────────────┬───────────────┘
                                      │
                        ┌─────────────┴─────────────┐
                        │                           │
                       YES                          NO
                        │                           │
                        ▼                           ▼
         ┌──────────────────────────┐    ┌──────────────────────────┐
         │  Can you pay $10/mo      │    │  Do you want simplest    │
         │  for Cloudflare COTS?    │    │  setup possible?         │
         └────────────┬─────────────┘    └────────────┬─────────────┘
                      │                               │
         ┌────────────┴────────────┐    ┌─────────────┴─────────────┐
         │                         │    │                           │
        YES                        NO  YES                          NO
         │                         │    │                           │
         ▼                         │    ▼                           ▼
┌─────────────────┐                │  ┌─────────────────┐  ┌─────────────────┐
│ Mode 1:         │                │  │ Mode 2:         │  │ Mode 3:         │
│ CF Worker Only  │◀───────────────┘  │ OpenResty Only  │  │ CF + Fallback   │
│                 │                    │                 │  │                 │
│ Best for:       │                    │ Best for:       │  │ Best for:       │
│ - Production    │                    │ - Dev/staging   │  │ - High          │
│ - High traffic  │                    │ - Low budget    │  │   availability  │
│ - Global users  │                    │ - Full control  │  │ - Critical apps │
└─────────────────┘                    └─────────────────┘  └─────────────────┘
```

---

## Comparison Table

| Feature | CF Worker Only | OpenResty Only | CF + Fallback |
|---------|----------------|----------------|---------------|
| **Throughput** | ~249 req/s | ~159 req/s | ~249 req/s (direct) |
| **p50 Latency** | ~169ms | ~283ms | ~169ms (direct) |
| **Setup Complexity** | Medium | Low | High |
| **Monthly Cost** | $10 (COTS) | Server cost | $10 + server |
| **Reliability** | High | Medium | Highest |
| **SSL Session Reuse** | Yes (CF managed) | No | Yes (direct) |
| **Self-hosted** | No | Yes | Partial |
| **DDoS Protection** | CF included | Manual | CF included |

---

## Migration Paths

### From OpenResty to CF Worker

1. Deploy CF Worker alongside existing gateway
2. Configure COTS with mesh-router CA
3. Test with a subset of domains
4. Gradually route more traffic to CF Worker
5. Decommission gateway when confident

### From CF Worker to CF + Fallback

1. Ensure OpenResty gateway is deployed
2. Add `GATEWAY_FALLBACK_URL` to Worker config
3. Test fallback by temporarily breaking COTS
4. Monitor fallback rate in logs

---

## Related Documentation

- [Data Flow](./DATA_FLOW.md) - Detailed request flow diagrams
- [Security in Transit](./SECURITY_IN_TRANSIT.md) - Certificate and encryption details
- [OpenResty Gateway README](../mesh-router-gateway/README.md)
- [CF Worker README](../mesh-router-gateway-cf/README.md)
- [COTS Setup Guide](../mesh-router-gateway-cf/docs/CLOUDFLARE_COTS.md)
