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
| [**mesh-router-gateway**](./mesh-router-gateway) | HTTP reverse proxy gateway for wildcard domain routing | [README](./mesh-router-gateway/README.md) |
| [**mesh-router-tunnel**](./mesh-router-tunnel) | WireGuard-based VPN tunneling for NAT traversal | [README](./mesh-router-tunnel/README.md) |
| [**mesh-router-agent**](./mesh-router-agent) | Lightweight agent for direct IP registration | [README](./mesh-router-agent/README.md) |
| [**mesh-dashboard**](./mesh-dashboard) | Web dashboard for mesh network management | [README](./mesh-dashboard/README.md) |
| [**mesh-router-template-root**](./mesh-router-template-root) | Template for PCS instance configuration | - |

---

## Architecture

Architecture diagrams and design documents are available in the [doc/architecture](./doc/architecture) folder:

- [Mesh Router Architecture (Excalidraw)](./doc/architecture/mesh-router/)

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
