# Life after the Cloudflare worker: mitigations for direct-to-box traffic

> **Study, 2026-10-25.** Nothing here is implemented yet. Shared copy:
> https://claude.ai/artifact/4SrxHuTAbt9C6MeNmEKUVE

## Context

Direct-to-box answers are about 4x faster per request than the Cloudflare worker, so direct should be the default wherever a box can serve it. This study lists what we give up and what we can build on the box, in Maison/AppShield or in the backend to win part of it back.

Measured 2026-10-25 on wisera (mesh-console 1.9.0 probes, plus curl from a client in France):

| Path | TCP connect | Warm request p50 | Cold request | Tail |
| --- | --- | --- | --- | --- |
| Direct (`mesh-console.wisera.inojob.com`) | ~12 ms from a client | ~13 ms | ~50 ms | p99 137 ms (from the box) |
| Worker (`mesh-console-wisera.inojob.com`) | ~5 ms (Cloudflare edge) | ~50 ms | 65–257 ms | p99 420 ms (from the box) |

The worker terminates TLS closer to the visitor but then pays a second trip to the box on every request, plus a route lookup it caches 60 s. It caches no content, so there is no CDN effect to lose.

Out of scope: boxes behind NAT. They never get a direct answer and stay on gateway + tunnel.

## What we lose

Five things go away with the worker; volume DDoS is the least important for a fleet where one attack hits one box.

| Loss | What the worker did | What happens direct |
| --- | --- | --- |
| Usage telemetry (DAU) | Reports uid + app + day to the backend on HTML navigations (`notifyUsage` in `mesh-router-gateway-cf`); the gateway does the same (`mesh-router-gateway/nginx/lua/usage.lua`) | Direct traffic is never counted: DAU falls as boxes go direct, with no real change in use |
| Per-request failover | Tries `ip.<zone>`, then sslip.io, then the gateway (and its tunnel) on each request | The browser holds the box IP; a dead box or a changed IP fails until the nameserver flips and the 30 s TTL runs out |
| Abuse filtering | Cloudflare absorbs floods and filters bots and scanners before the box | Every scanner, brute-forcer and flood reaches Caddy and AppShield directly |
| IP privacy | The box IP is not in public DNS | The IP is in DNS: anyone can scan every published port (Samba, BitTorrent, game servers) |
| Certificate independence | Visitors see Cloudflare's certificate; the box can stay on the mesh CA | Each box needs its own Let's Encrypt wildcard by DNS-01; issuance and renewal can fail, and rate limits apply |

What stays the same: HTTP/3 and compression (Caddy does both), WebSockets, and NAT'd boxes (gateway + tunnel). Direct also gains something: Caddy and the apps see the visitor's real IP, which every mitigation below depends on.

## Mitigations

Most of what Cloudflare gave can be rebuilt per box. What no box can do alone is absorb a flood or share what it learns, so the two fleet-level items (a shared blocklist and a shield mode) carry the most weight.

### 1. Abuse protection on the box

Today: the AppShield gate puts almost every app behind a login, and Authelia runs on its default brute-force regulation (3 failures in 2 min means a 5 min ban; the template does not set it). Caddy (`mesh-router-caddy`, built with caddy-docker-proxy and acmedns only) has no rate limit and writes no per-site access log.

- **CrowdSec (recommended over fail2ban).** An agent reads the Caddy access log, Authelia's log and sshd's, and detects scans, brute force and floods with ready-made scenarios. A Caddy bouncer (an xcaddy plugin) refuses banned IPs at the proxy. It also pulls CrowdSec's community blocklist, which is the nearest thing to Cloudflare's network effect we can get for free.
- **fail2ban (the simpler option).** Lighter and well known, but every box learns alone. Docker-published ports skip the host's INPUT chain, so its bans must go in `DOCKER-USER`, and it needs Caddy to log to a file.
- **Caddy rate limits.** Add the `caddy-ratelimit` plugin to `mesh-router-caddy`: a tight per-IP limit on sign-in endpoints (`auth-<domain>`, Authelia's first-factor API) and a loose per-IP ceiling everywhere else. Its prerequisite, a per-site access log, also feeds CrowdSec; the log must be rotated (see the 2026-10 Caddy log-growth incident).
- **AppShield throttling.** The gate already sees every unauthenticated request. It can slow down repeated failed sign-ins and anonymous request bursts per IP, which covers apps that skip Authelia.
- **Authelia regulation, tuned.** Set it explicitly in the template, with a longer ban (for example 10 failures in 10 min means a 1 h ban), and send its log to CrowdSec.
- **Kernel-level limits.** nftables `connlimit` per source IP on 80/443 caps the connections one address can hold. SYN cookies are already on by default.

### 2. Floods: a shield mode

A box cannot absorb a volume attack; its uplink fills first. What we can do is make the edge a switch we flip back on in minutes.

- **Shield mode, per box.** One switch (owner, support, or automatic) makes the names answer the edge again, and firewalls 80/443 to Cloudflare's published ranges plus the gateway. The firewall half matters most: the attacker already knows the IP, so DNS alone does not stop them.
- **Detection.** A flood shows up as a jump in connection count or CPU, or as failed probes from outside. mesh-console's probes and Maison's resource alerts can trigger it, or at least page the owner.
- **Cost when the IP is the target.** If an attacker keeps flooding the IP itself, only a new IP helps (a box rebuild today). The provider's own DDoS filtering is our real floor here and needs checking (open question).

### 3. Failover

- **Health-checked answers.** The nameserver answers the box only while outside probes of the box pass, and the gateway otherwise. Probing from two or more places avoids flipping a box over one bad route.
- **Keep the TTL short.** 30 s today. Shorter mainly adds DNS load; resolvers that ignore short TTLs bound what we can win.
- **Report a new IP at once.** When a box's public IP changes, the agent pushes it so the nameserver follows within one TTL.
- **Rejected: answering both the box and the gateway.** Browsers do retry the second address when the first refuses, but DNS cannot rank them, so half the traffic would take the slow path all the time.

### 4. Usage telemetry

- **Report from the box.** AppShield sees every authenticated app open and knows the user and the app. It (or Maison) posts one deduplicated `{uid, app, day}` event to the existing usage ingest, signed with the box's Ed25519 key like route registration. The data stays the same as what the worker sends today.
- **Public apps.** Apps with no gate are counted from the Caddy access log instead (HTML navigations, as the worker does).
- **Bonus: reach telemetry.** mesh-console already measures ping and latency for both paths. Reported to the backend, that gives a fleet-wide view of direct against edge, which Cloudflare never gave us.

### 5. IP exposure

- **Default-deny firewall from the template.** Only 80/443 plus the ports apps declare (`needs-public-ip`) are open; everything else is closed at the host, whatever Docker publishes by mistake.
- **Exposure audit.** mesh-console lists every port reachable from outside (it can probe its own IP) next to the app that opened it.
- **Owner choice to stay hidden.** "Answer the gateway" already exists; offer it as the privacy setting for boxes on a home connection.

### 6. Certificates

- **Let's Encrypt rate limits.** 50 new certificates per registered domain per week; renewals do not count. A wave of new boxes under `nsl.sh` could hit it.
- **Public Suffix List.** Listing `nsl.sh` (and the other server domains) makes each user's domain its own registered domain for rate limits. It also stops browsers sharing cookies between users' domains, a security win of its own.
- **A second CA.** Caddy can fall back to another ACME issuer (ZeroSSL, Google Trust Services) when Let's Encrypt fails.
- **Already in place.** The nameserver answers the box directly only while it serves a trusted certificate, so a failed renewal falls back to the gateway rather than breaking.

## Comparison

Usage telemetry and the Caddy access log are cheap and unblock the rest; shield mode is the costliest and the only answer to floods. Effort and impact are rough estimates, sorted by impact.

| Mitigation | Recovers | Where it lives | Effort | Impact |
| --- | --- | --- | --- | --- |
| Box-side usage reporting | Usage telemetry | AppShield or Maison + backend ingest | Small | High: DAU is wrong without it |
| Shield mode (edge + firewall to edge ranges) | Floods, IP privacy | Nameserver, backend, template firewall, mesh-console switch | Large | High, rarely used |
| CrowdSec agent + Caddy bouncer | Abuse filtering | Template (new service), `mesh-router-caddy` plugin | Medium | High |
| Health-checked nameserver answers | Failover | Nameserver + outside probes | Medium | High |
| Caddy access log + rate limits | Abuse filtering | `mesh-router-caddy`, template Caddyfile | Small | Medium |
| Public Suffix List entry | Certificates (rate limits), cookie isolation | Request to the PSL maintainers | Small, weeks of waiting | Medium |
| Default-deny host firewall | IP exposure | Template | Medium | Medium |
| Authelia regulation tuned | Abuse filtering (sign-in) | Template auth stack | Small | Medium |
| AppShield throttling | Abuse filtering (gated apps) | AppShield | Medium | Medium |
| Second ACME CA | Certificates | Caddy config | Small | Low |
| Exposure audit in mesh-console | IP exposure | mesh-console | Small | Low |
| fail2ban (instead of CrowdSec) | Abuse filtering, per box only | Template | Small | Low |

## Roadmap and open questions

Fix the telemetry before moving more of the fleet to direct, then build abuse protection, and keep shield mode as the last step.

1. **Before going direct fleet-wide:** box-side usage reporting; the Caddy access log with rotation; Authelia regulation set explicitly; the Public Suffix List request filed (it takes weeks).
2. **Abuse protection:** CrowdSec with the Caddy bouncer and the community blocklist; Caddy rate limits on sign-in endpoints; the default-deny host firewall. Trial on holyhorse and wisera first.
3. **Failover:** health-checked answers in the nameserver, probed from two places.
4. **Shield mode:** the switch in mesh-console, the edge firewall profile, then automatic triggering from probes and resource alerts.

Open questions:

- [ ] What DDoS filtering does each provider (Contabo, the Proxmox cluster) give us before traffic reaches the box?
- [ ] Can shield mode put a name back behind Cloudflare now that `nsl.sh` is on our own nameservers, or does it have to answer the gateway, which does not scrub floods?
- [ ] CrowdSec's memory cost on the smallest box plan, and whether one shared fleet blocklist (bans from every box, pushed back to all) is worth running ourselves.
- [ ] Is `nsl.sh` already close to Let's Encrypt's 50-a-week limit during busy weeks?
- [ ] Should usage reporting live in AppShield (sees every gated open) or in Maison (one place, already talks to Yundera)?
