# Mesh Router

**A distributed mesh networking solution for self-hosted infrastructure**

This repository serves as the root "bookmark" for all Mesh Router projects, bringing together the various components that make up the mesh networking ecosystem.

---

## Sponsors

This project is made possible by:

| Sponsor | Description |
|---------|-------------|
| [**Yundera**](https://yundera.com) | Easy to use cloud server for open source container applications |
| [**NSL.SH**](https://nsl.sh) | Free domain for open source projects |

---

## Overview

Mesh Router enables secure, distributed routing of traffic between public servers and private instances. It supports both direct IP routing (for low latency) and WireGuard VPN tunneling (for instances behind NAT).

```
                    ┌─────────────────────────────────────────┐
                    │            mesh-router-gateway          │
                    │         (public entry point)            │
                    └──────────────────┬──────────────────────┘
                                       │
                         ┌─────────────┴─────────────┐
                         │                           │
                         ▼                           ▼
              ┌──────────────────┐       ┌──────────────────┐
              │   Direct Route   │       │   VPN Tunnel     │
              │  (Public IP via  │       │  (WireGuard via  │
              │   agent)         │       │   tunnel)        │
              └──────────────────┘       └──────────────────┘
                         │                           │
                         ▼                           ▼
              ┌──────────────────────────────────────────────┐
              │              PCS Instance                    │
              │         (self-hosted services)               │
              └──────────────────────────────────────────────┘
```

---

## Components

| Component | Description | Documentation |
|-----------|-------------|---------------|
| [**mesh-router-backend**](./mesh-router-backend) | Express.js API for domain management and IP resolution | [README](./mesh-router-backend/README.md) |
| [**mesh-router-gateway**](./mesh-router-gateway) | OpenResty reverse proxy gateway for wildcard domain routing | [README](./mesh-router-gateway/README.md) |
| [**mesh-router-gateway-cf**](./mesh-router-gateway-cf) | Cloudflare Worker gateway for edge-based routing | [README](./mesh-router-gateway-cf/README.md) |
| [**mesh-router-tunnel**](./mesh-router-tunnel) | WireGuard-based VPN tunneling for NAT traversal | [README](./mesh-router-tunnel/README.md) |
| [**mesh-router-agent**](./mesh-router-agent) | Lightweight agent for direct IP registration | [README](./mesh-router-agent/README.md) |
| [**mesh-dashboard**](./mesh-dashboard) | Web dashboard for mesh network management | [README](./mesh-dashboard/README.md) |
| [**mesh-router-template-root**](./mesh-router-template-root) | Template for PCS instance configuration | - |

---

## Gateway Modes

Mesh Router supports three gateway configurations:

| Mode | Description | Performance |
|------|-------------|-------------|
| **CF Worker Only** | Edge-based routing via Cloudflare Workers | Best (~249 req/s, 169ms p50) |
| **OpenResty Only** | Self-hosted reverse proxy | Simpler setup (~159 req/s, 283ms p50) |
| **CF + Fallback** | CF Worker with OpenResty fallback | Maximum reliability |

For detailed guidance on choosing and configuring gateway modes:

- [Gateway Modes Guide](./doc/GATEWAY_MODES.md) - Decision tree and comparison
- [Data Flow](./doc/DATA_FLOW.md) - How traffic flows through each mode
- [Security in Transit](./doc/SECURITY_IN_TRANSIT.md) - Certificates and encryption

---

## Debugging & API Reference

### Debugging Headers

Use these headers to debug routing issues:

#### X-Mesh-Trace

Add `X-Mesh-Trace: 1` request header to see the routing path in the response:

```bash
curl -s -D- -o /dev/null -H "X-Mesh-Trace: 1" https://app.alice.domain.com/ | grep -i x-mesh-route
# Response: x-mesh-route: cf-worker,nip.io,direct,pcs
```

Route path segments:
- `cf-worker` - Cloudflare Worker processed the request
- `nip.io` - Direct path via nip.io (~4x faster than gateway)
- `gateway-fallback` - OpenResty gateway fallback
- `direct` / `agent` - Route from mesh-router-agent
- `tunnel` - Route from mesh-router-tunnel (WireGuard)
- `pcs` - Final destination

#### X-Mesh-Force

Force a specific routing path for testing:

```bash
curl -H "X-Mesh-Force: gateway" https://app.alice.domain.com/  # Use gateway fallback
curl -H "X-Mesh-Force: direct" https://app.alice.domain.com/   # Prefer agent routes
curl -H "X-Mesh-Force: tunnel" https://app.alice.domain.com/   # Prefer tunnel routes
```

### Backend API

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/routes/:userid/:sig` | Ed25519 | Register routes |
| `GET` | `/routes/:userid` | Public | Get routes for user |
| `DELETE` | `/routes/:userid/:sig` | Ed25519 | Delete all routes |
| `GET` | `/resolve/v2/:domain` | Public | Resolve domain to routes |
| `GET` | `/available/:domainName` | Public | Check domain availability |

Example - resolve a domain:
```bash
curl https://backend.domain.com/resolve/v2/alice
# Response: {"userId":"...","routes":[...],"routesTtl":580}
```

See [mesh-router-backend/README.md](./mesh-router-backend/README.md) for full API documentation.

---

## Architecture

Architecture diagrams and design documents are available in the [doc/architecture](./doc/architecture) folder:

- [Architecture Overview](./doc/architecture/README.md)
- [Excalidraw Diagrams](./doc/architecture/mesh-router/)

---

## Getting Started

Clone this repository with all submodules:

```bash
git clone --recursive https://github.com/Yundera/mesh-router-root.git
```

Or if you've already cloned:

```bash
git submodule update --init --recursive
```

Refer to each component's README for specific setup instructions.

---

## Local Development

To run the complete Mesh Router stack locally, see the [dev/](./dev) folder which includes:

- Docker Compose configuration for all services
- Example environment file
- Caddy reverse proxy configuration
- Step-by-step setup instructions

```bash
cd dev
cp .env.example .env
# Edit .env with your credentials
docker compose up -d
```

See [dev/README.md](./dev/README.md) for detailed instructions.

---

## License

See [LICENSE](./LICENSE) for details.
