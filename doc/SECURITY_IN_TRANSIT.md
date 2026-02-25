# Security in Transit

This document explains how data is protected as it flows through Mesh Router, including certificate management, encryption details, and the trust model.

---

## Overview

Mesh Router uses a private PKI (Public Key Infrastructure) to secure communications between components. All traffic is encrypted with TLS, and PCS instances receive certificates signed by the mesh-router Certificate Authority.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           ENCRYPTION LAYERS                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Client ═══TLS═══▶ Cloudflare ═══TLS═══▶ Gateway ═══TLS═══▶ PCS           │
│           (public)             (origin)            (mesh-ca)               │
│                                                                             │
│   Every hop is encrypted. No plaintext transmission.                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Certificate Architecture

### Trust Hierarchy

```
                    ┌─────────────────────────────────────┐
                    │     mesh-router-backend             │
                    │     Certificate Authority           │
                    │                                     │
                    │  • Generates CA keypair at startup  │
                    │  • Signs PCS certificates           │
                    │  • Exposes CA cert at /ca-cert      │
                    └──────────────┬──────────────────────┘
                                   │
                                   │ Signs
                                   │
            ┌──────────────────────┼──────────────────────┐
            │                      │                      │
            ▼                      ▼                      ▼
   ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
   │  PCS Alice      │   │  PCS Bob        │   │  PCS Carol      │
   │                 │   │                 │   │                 │
   │  Certificate:   │   │  Certificate:   │   │  Certificate:   │
   │  *.alice.nsl.sh │   │  *.bob.nsl.sh   │   │  *.carol.nsl.sh │
   │                 │   │                 │   │                 │
   │  Signed by:     │   │  Signed by:     │   │  Signed by:     │
   │  mesh-router CA │   │  mesh-router CA │   │  mesh-router CA │
   └─────────────────┘   └─────────────────┘   └─────────────────┘
```

### PCS Certificate Details

Each PCS instance receives a certificate with:

| Field | Value | Purpose |
|-------|-------|---------|
| **Subject CN** | `*.username.server-domain` | Primary domain wildcard |
| **Issuer** | mesh-router CA | Trust chain |
| **Validity** | 72 hours | Short-lived for security |
| **Key Type** | RSA 2048 or ECDSA | PCS generates keypair |

**Subject Alternative Names (SANs):**

```
SANs included in each PCS certificate:

1. *.username.server-domain     (e.g., *.alice.nsl.sh)
   └── Covers: app.alice.nsl.sh, api.alice.nsl.sh, etc.

2. username.server-domain       (e.g., alice.nsl.sh)
   └── Covers: bare domain access

3. {public-ip}.nip.io           (e.g., 203.0.113.5.nip.io)
   └── Required for CF Worker direct connections
   └── IPv6: 2001-bc8-3021--1.nip.io (colons → hyphens)
```

---

## Certificate Lifecycle

### 1. PCS Registration and Certificate Issuance

```
┌─────────────┐                                    ┌──────────────────────┐
│    PCS      │                                    │  mesh-router-backend │
│  (Agent)    │                                    │       (CA)           │
└──────┬──────┘                                    └───────────┬──────────┘
       │                                                       │
       │  1. Detect public IP (via STUN)                       │
       │                                                       │
       │  2. Generate keypair                                  │
       │     (private key stays on PCS)                        │
       │                                                       │
       │  3. Create CSR (Certificate Signing Request)          │
       │                                                       │
       │  4. POST /cert/{userid}/{signature}                   │
       │     Body: { csr, publicIp }                           │
       │─────────────────────────────────────────────────────▶│
       │                                                       │
       │                    5. Verify Ed25519 signature        │
       │                       (proves PCS owns userid)        │
       │                                                       │
       │                    6. Generate certificate            │
       │                       - Add SANs (domain + nip.io)    │
       │                       - Set 72h validity              │
       │                       - Sign with CA key              │
       │                                                       │
       │  7. Return signed certificate + CA cert               │
       │◀─────────────────────────────────────────────────────│
       │                                                       │
       │  8. Store certs:                                      │
       │     /certs/cert.pem (signed cert)                     │
       │     /certs/key.pem (private key)                      │
       │     /certs/ca.pem (CA cert for chain)                 │
       │                                                       │
```

### 2. Certificate Renewal

Certificates are renewed automatically before expiry:

```
┌─────────────────────────────────────────────────────────────┐
│                  RENEWAL TIMELINE                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  T+0h        Certificate issued (72h validity)              │
│     │                                                       │
│     │        Normal operation                               │
│     │                                                       │
│  T+48h       Agent checks: validity < 24h remaining?        │
│     │        └── No: continue normal operation              │
│     │                                                       │
│  T+48h+      Agent checks again                             │
│     │        └── Yes: initiate renewal                      │
│     │                                                       │
│  T+48h+      New certificate issued                         │
│     │        └── Caddy hot-reloads new cert                 │
│     │                                                       │
│  T+72h       Old certificate expires (unused)               │
│                                                             │
└─────────────────────────────────────────────────────────────┘

Renewal is seamless - no downtime during certificate rotation.
```

---

## Encryption by Connection

### Connection Map

```
┌───────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                       │
│    CLIENT                     CLOUDFLARE                  GATEWAY                 PCS │
│       │                           │                          │                     │  │
│       │◄════════ TLS 1.3 ════════▶│                          │                     │  │
│       │   Cloudflare Edge Cert    │                          │                     │  │
│       │   (managed by CF)         │                          │                     │  │
│       │                           │                          │                     │  │
│       │                           │◄════════ TLS 1.2+ ══════▶│                     │  │
│       │                           │   Cloudflare Origin Cert │                     │  │
│       │                           │   (Full Strict mode)     │                     │  │
│       │                           │                          │                     │  │
│       │                           │                          │◄═══ TLS 1.2+ ══════▶│  │
│       │                           │                          │   mesh-router CA    │  │
│       │                           │                          │   signed cert       │  │
│       │                           │                          │                     │  │
└───────────────────────────────────────────────────────────────────────────────────────┘
```

### Connection Details

| Connection | Protocol | Certificate | Verified By | Notes |
|------------|----------|-------------|-------------|-------|
| **Client → Cloudflare** | TLS 1.3 | Cloudflare Edge | Browser CA store | Universal SSL, automatic |
| **Cloudflare → Gateway** | TLS 1.2+ | Let's Encrypt | Cloudflare | Origin server cert |
| **Gateway → Backend API** | HTTPS | mesh-ca signed | Gateway (Lua) | Route resolution |
| **Gateway → PCS** | TLS 1.2+ | mesh-ca signed | Gateway (nginx) | Proxied traffic |
| **CF Worker → PCS** | TLS 1.2+ | mesh-ca signed | COTS | Requires custom trust |
| **Tunnel (WireGuard)** | WireGuard | Ed25519 keys | Peer public key | Encrypted tunnel |

---

## Trust Configuration

### OpenResty Gateway Trust

The gateway trusts the mesh-router CA for connections to PCS:

```nginx
# nginx.conf
http {
    # Trust mesh-router CA for Lua HTTP client
    lua_ssl_trusted_certificate /etc/ssl/certs/mesh-ca.pem;
    lua_ssl_verify_depth 2;
}

# gateway.conf
server {
    location / {
        # Verify PCS certificate against mesh-router CA
        proxy_ssl_verify on;
        proxy_ssl_trusted_certificate /etc/ssl/certs/mesh-ca.pem;
        proxy_ssl_verify_depth 2;

        proxy_pass $backend;  # https://[pcs-ip]:port
    }
}
```

### Cloudflare Worker Trust (COTS)

CF Workers only trust public CAs by default. To trust mesh-router CA:

```
┌─────────────────────────────────────────────────────────────┐
│            CUSTOM ORIGIN TRUST STORE (COTS)                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Obtain mesh-router CA certificate                       │
│     GET https://backend.example.com/ca-cert                 │
│                                                             │
│  2. Upload to Cloudflare Dashboard                          │
│     SSL/TLS → Custom Origin Trust Store → Upload            │
│                                                             │
│  3. Enable compatibility flag in wrangler.toml              │
│     compatibility_flags = [                                 │
│       "nodejs_compat",                                      │
│       "cots_on_external_fetch"                              │
│     ]                                                       │
│                                                             │
│  4. Worker fetch() now trusts mesh-router CA                │
│                                                             │
│  Cost: $10/month (Advanced Certificate Manager add-on)      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### PCS Trust (Caddy)

PCS instances run Caddy which serves the mesh-router signed certificate:

```
# Caddy configuration (simplified)
{
    auto_https off  # Using mesh-router CA, not Let's Encrypt
}

:10443 {
    tls /certs/cert.pem /certs/key.pem

    reverse_proxy /* localhost:80
}
```

---

## Authentication Mechanisms

### Ed25519 Signature Authentication

Route registration uses cryptographic signatures for authentication without requiring full Firebase auth:

```
┌─────────────────────────────────────────────────────────────┐
│              Ed25519 SIGNATURE FLOW                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  PCS (Agent/Tunnel)                    Backend              │
│         │                                 │                 │
│         │  1. Sign userid with Ed25519    │                 │
│         │     private key                 │                 │
│         │     signature = sign(userid)    │                 │
│         │                                 │                 │
│         │  2. POST /route/{userid}/{sig}  │                 │
│         │─────────────────────────────────▶                 │
│         │                                 │                 │
│         │     3. Lookup user's public key │                 │
│         │        (stored at registration) │                 │
│         │                                 │                 │
│         │     4. Verify: pubkey.verify(   │                 │
│         │           userid, signature)    │                 │
│         │                                 │                 │
│         │     5. If valid: register route │                 │
│         │◀─────────────────────────────────                 │
│         │                                 │                 │
│                                                             │
│  Benefits:                                                  │
│  • No shared secrets transmitted                            │
│  • Signature proves ownership of private key                │
│  • Resistant to replay (userid is unique)                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### WireGuard Tunnel Authentication

Tunnel connections use WireGuard's built-in authentication:

```
┌─────────────────────────────────────────────────────────────┐
│              WIREGUARD AUTHENTICATION                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  PCS (Tunnel Client)               Tunnel Hub               │
│         │                              │                    │
│         │  1. Exchange public keys     │                    │
│         │     (during registration)    │                    │
│         │◀────────────────────────────▶│                    │
│         │                              │                    │
│         │  2. WireGuard handshake      │                    │
│         │     (Noise protocol)         │                    │
│         │══════════════════════════════│                    │
│         │                              │                    │
│         │  3. Encrypted tunnel         │                    │
│         │     (ChaCha20-Poly1305)      │                    │
│         │══════════════════════════════│                    │
│         │                              │                    │
│                                                             │
│  Security properties:                                       │
│  • Perfect forward secrecy                                  │
│  • Mutual authentication via public keys                    │
│  • Resistant to DoS (silent drop of invalid packets)        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Security Considerations

### What's Protected

| Threat | Protection | Notes |
|--------|------------|-------|
| **Eavesdropping** | TLS encryption on all hops | No plaintext data in transit |
| **MITM (public internet)** | Cloudflare + browser CA verification | Standard web PKI |
| **MITM (gateway → PCS)** | mesh-router CA verification | Private PKI |
| **Route hijacking** | Ed25519 signature verification | Cryptographic proof of ownership |
| **Replay attacks** | Signature over unique userid | Cannot replay old registrations |
| **DDoS** | Cloudflare protection | Included with CF |

### Known Limitations

| Limitation | Impact | Mitigation |
|------------|--------|------------|
| **72h certificate validity** | Requires regular renewal | Automatic renewal by agent |
| **Single CA** | CA compromise affects all PCS | Monitor CA, rotate if needed |
| **COTS cost** | $10/month for CF Worker mode | Use gateway-only mode if cost-sensitive |
| **nip.io dependency** | CF Worker requires nip.io | Fallback to gateway if nip.io fails |

### Security Best Practices

1. **Protect the CA private key** - The mesh-router-backend CA key should be secured
2. **Use short certificate validity** - 72h limits exposure from compromised certs
3. **Enable Cloudflare Full (Strict)** - Verify origin certificates
4. **Monitor certificate expiry** - Alert if renewal fails
5. **Rotate Ed25519 keys** - Consider periodic key rotation for agents

---

## Troubleshooting SSL Errors

### Common Issues

```
┌─────────────────────────────────────────────────────────────┐
│  ERROR: "certificate verify failed"                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Gateway → PCS:                                             │
│  • Check /etc/ssl/certs/mesh-ca.pem exists                  │
│  • Verify nginx config: proxy_ssl_trusted_certificate       │
│  • Ensure PCS cert signed by current CA (not old CA)        │
│                                                             │
│  CF Worker → PCS:                                           │
│  • Verify COTS is configured in Cloudflare dashboard        │
│  • Check cots_on_external_fetch flag in wrangler.toml       │
│  • Ensure PCS cert has nip.io SAN                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  ERROR: "certificate has expired"                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  • Check agent is running and can reach backend             │
│  • Verify agent renewal process: logs, connectivity         │
│  • Manual renewal: restart agent to force cert refresh      │
│  • Check system clock on PCS (time drift causes failures)   │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  ERROR: "hostname mismatch"                                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  • Certificate SANs don't match requested hostname          │
│  • For nip.io: ensure cert includes {ip}.nip.io SAN         │
│  • For domain: ensure cert includes *.user.domain SAN       │
│  • Check if IP changed since cert was issued                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Related Documentation

- [Gateway Modes](./GATEWAY_MODES.md) - Comparison of gateway configurations
- [Data Flow](./DATA_FLOW.md) - Request flow diagrams
- [COTS Setup Guide](../mesh-router-gateway-cf/docs/CLOUDFLARE_COTS.md) - Cloudflare trust store configuration
- [mesh-router-backend API](../mesh-router-backend/readme.md) - Certificate endpoints
