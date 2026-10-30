# User mail: receiving on `<zone>`

Status (2026-10-30): phases 0, 0.5 and 1 implemented (`mesh-router-backend/src/services/InboundMail.ts`,
`UserMail.ts`; the owner's card in mesh-console's Email page), the rest is design (§11).

## 1. Goal and scope

Today a box can only **send** mail: apps talk SMTP to the `smtp` container (`mail-gateway`),
which relays over HTTPS to the backend (`POST /email/send`), which sends through SendGrid as
`<app>.<user>@<zone>`. Nothing can **receive** mail for a user, and inbound mail for the zone
(`contact@nsl.sh`) depends on Cloudflare Email Routing, which ends with the cutoff.

This feature makes the platform receive mail for `<zone>` and lets each user choose where theirs goes:

| Mode | What happens | Who it is for |
|---|---|---|
| `off` (default) | Mail to the user is dropped | everyone until they opt in |
| `forward` | Each message is re-sent to one verified external address (Gmail, …) | users who keep their usual inbox |
| `box` | Messages are stored on the box; any IMAP client or webmail reads them, and replies go out through the existing relay | users who self-host their mail |

The zone's own addresses (`contact@`, `admin@`, `postmaster@`, `abuse@`, `dmarc@`) are handled by the same
receiver as forward-mode entries (§5.6).

**Non-goals**

- **No MX of our own.** SendGrid Inbound Parse receives the mail; we run no port 25 anywhere.
- **No full mail server per box.** The box gets a mailbox (IMAP), not an MTA. A user who wants
  port 25 delivered straight to their own Postfix/mailcow is a different feature (publish the
  port, `needs-public-ip`), not this one.

## 2. Addresses

### 2.1 Base shape: `<user>@<zone>` (Gmail model)

| Address | Meaning |
|---|---|
| `alice@nsl.sh` | Alice's mailbox: receives, and the client sends as it |
| `alice+<tag>@nsl.sh` | Same mailbox; the tag is kept for filtering (`+shop` → folder `shop`) |
| `<app>.alice@nsl.sh` | Alice's app `<app>` sending notifications (exists today); replies to it reach Alice's mailbox too |
| `contact@`, `admin@`, `postmaster@`, `abuse@`, `dmarc@nsl.sh` | the zone's own aliases (§5.6) |

- User labels are DNS labels (no dots), so `<user>` and `<app>.<user>` never collide.
- Zone aliases share the namespace with user labels: `contact`, `admin`, `abuse`, `postmaster`,
  `dmarc` are in `RESERVED_DOMAINS` (`mesh-router-backend/src/services/Domain.ts`, next to
  `mail`). Holders of a name reserved after the fact keep it (existing rule); none of the five
  was held on staging or prod (2026-10-30).
- One MX for the whole zone (§4); nothing is needed in the per-user zones.

### 2.2 Add-on: `*@<user>.<zone>`

Later, a user may also get unlimited addresses on their own subdomain (`mom@smith.nsl.sh`,
`shop@alice.nsl.sh`) — families and small businesses on one box. It does not conflict with §2.1:
it adds an MX in the user's zone (only for delegated users, so after the cutoff) and one Inbound
Parse setting for that hostname, routed through the same backend code with the same modes.
Everything below is written for §2.1; §2.2 changes only the address check.

## 3. Overview

```
 sender
   │ SMTP
   ▼
 mx.sendgrid.net  ── MX of <zone> (apex)
   │ Inbound Parse: spam check, raw MIME, POST
   ▼
 mesh-router-backend  POST /email/inbound/<token>
   │ local part → zone alias | user (+tag, <app>.) | unknown (drop)
   │ user → mode
   ├─ forward ─▶ SendGrid send (forwarding subuser) ─▶ verified external address
   └─ box ─────▶ HTTPS POST https://mail.<user>.<zone>/inbound   (normal route: direct or tunnel)
                    │
                    ▼  box, mesh stack
                 smtp (mail-gateway) /inbound ── LMTP ──▶ dovecot (maildir)
                                                            ▲ IMAP :143 on pcs
                                       webmail store app ───┘   (SnappyMail / Roundcube)
                                              │ SMTP :587
                                              ▼
                                 smtp (mail-gateway) ─▶ backend /email/send ─▶ SendGrid
```

## 4. Receiving: SendGrid Inbound Parse

- **One parse setting for the zone**: hostname `<zone>`, URL
  `https://<backend>/router/api/email/inbound/<token>`, `spam_check: true`, `send_raw: true`.
  Inbound Parse requires the hostname's domain to be an authenticated domain (`<zone>` already
  is, for app mail). Set up once per deployment, by hand or by `mesh-cli mail setup`.
- **MX record**: `<zone> MX 10 mx.sendgrid.net` in the static zone file (and in Cloudflare until
  the cutoff). SPF stays `include:sendgrid.net`; the Cloudflare include and `cf2024-1` DKIM go.
- **Authenticating the webhook**: the `<token>` path segment is a random per-deployment secret
  (the endpoint is otherwise public). Prefer signed payloads if Inbound Parse offers them.
- **Recipient resolution** (from the envelope recipient, not the `To:` header): zone alias →
  user (`<user>`, `<user>+<tag>`, `<app>.<user>`) → otherwise drop. Inbound Parse has already
  accepted the message, so an unknown address cannot be refused at SMTP time; it is dropped and
  counted, never bounced (no backscatter).
- **Spam**: messages above `MAIL_SPAM_DROP` (e.g. 8) are dropped; between `MAIL_SPAM_MARK` and that,
  they get `X-Spam: yes` and land in Junk (box mode) or are not forwarded (forward mode — never
  relay doubtful mail on our reputation).
- **Box unreachable**: the backend answers 5xx and relies on Inbound Parse's retries. If those
  turn out to be short, the backend keeps a small Redis queue (`mail:queue:<uid>`, capped,
  oldest dropped first) and drains it when the box's route comes back.
- **Duplicates**: a retry can follow a delivery that actually succeeded (slow answer, timeout).
  The backend remembers each processed `Message-ID` + recipient in Redis for a day and answers
  2xx to a repeat without sending it again.

Observed on staging (2026-10-30, `inojob.com`, catcher at webhook.site): the bare zone is
accepted as parse hostname; the POST is `multipart/form-data` from `Sendlib/1.0`, with
`envelope` (JSON, real recipients), `email` (raw MIME, a 590 kB PNG attachment intact), `to`,
`from`, `subject`, `spam_score` / `spam_report` (SpamAssassin), `SPF`, `dkim`, `sender_ip`,
`charsets`; no signature header. A 500 was retried with the same `Message-ID` 31 s later.

## 5. Mode `forward`

The owner sets it **from the box**: the Email page of mesh-console calls
`GET|PUT|DELETE /email/forward/<userid>/<sig>` with the box's signature, as the agent does for
routes. The setting is stored on the server (`nsl-router/<uid>.mail`), so it follows the user
across box rebuilds and a FOSS box without a Yundera account gets it too.

1. **Verified target.** The user enters an address; we send a confirmation link; forwarding
   starts only after the click. Without it a user could point the forward at a stranger and
   mail-bomb them through us. The link opens a page with a button (`POST /email/confirm/<token>`):
   mail scanners prefetch links and must not confirm on the owner's behalf. 48 h validity, at
   most 5 confirmation mails per user and day; the link's base is `MAIL_CONFIRM_BASE`, default
   `https://<zone>/router/api`. Targets in or under the zone are refused (loop).
2. **Sender rewrite** (mailing-list style). Re-sending `bob@gmail.com`'s message from our
   SendGrid fails Gmail's DMARC, so the forwarded copy is
   `From: "Bob via <zone>" <alice@<zone>>` (the address it was sent to), `Reply-To: bob@gmail.com`,
   original subject, body and attachments, plus `X-Original-From` / `X-Original-To`. Replies
   from the user's inbox go straight to Bob.
3. **Separate reputation.** Later: forwarded mail sent by a dedicated SendGrid **subuser**, so
   reported forwarded spam hurts it rather than the account app mail depends on. Phase 1 uses
   the main account (decision 2026-10-30), protected by the spam drop and the daily cap.
4. **Limits.** `MAIL_FORWARD_DAILY` messages per user per day (default 100, counted after the
   spam and loop drops; aliases uncapped), size cap equal to Inbound Parse's.
5. **Visible failure.** When SendGrid suppresses the target (bounce, block, spam report,
   invalid), the box shows the forward as *paused* with the reason (read from SendGrid's
   suppression lists, best effort: it needs an API key with suppression access). Setting the
   address again clears the suppression and sends a new confirmation link.
6. **Zone aliases** are forward entries with no user: a static map in the backend's config,
   targets trusted (no verification click), same rewrite, same subuser. `MAIL_ALIAS_TARGET` is
   the default target; `MAIL_ALIASES` lists the aliases as `name` (→ default target) or
   `name:target` entries, default `contact,admin,abuse,postmaster,dmarc`. `contact+tag@` is
   `contact@`. One copy per distinct target, sent from the first alias it was addressed to;
   each copy is deduplicated on Message-ID + target, so a retry re-sends only the failed ones.
   Mail to any other address of the zone is dropped (logged, never bounced).

## 6. Mode `box`

### 6.1 Registration

Box mode is only offered when the box says it can take mail. At start-up the box's
`mail-gateway` (provider mode, already holding `userid:signature`) calls
`POST /email/inbound/register` with `{ url, token }`:

- `url` = `https://mail.<user>.<zone>/inbound`, a dot-form name, so the gateway passes TLS through
  to the box (one key per name) and the backend talks to the box end to end;
- `token` = a random secret the box generates and keeps; the backend sends it back on every
  delivery (`Authorization: Bearer`), so only the backend can inject mail.

Re-registration replaces the previous token. A box that stops registering (older template,
stack removed) is shown as *mailbox unavailable*; mail then waits in the queue (§4) rather than
being lost.

### 6.2 On the box (mesh stack)

| Piece | Change |
|---|---|
| `smtp` (`mail-gateway`) | new `/inbound` HTTP endpoint behind Caddy at `mail.<user>.<zone>`: checks the token, hands the raw MIME to Dovecot over LMTP, returns 2xx only once Dovecot accepted it |
| `dovecot` (new service) | IMAP on `pcs` only (143, no port published), LMTP for delivery, maildir under the mesh stack's data dir, one mailbox per box, Sieve for `+tag` and `X-Spam` filing, quota (`MAIL_QUOTA`, default a few GB) |
| Maison managed settings | new managed `imap` value `{host, port, user, pass}` beside `smtp`, `pass` a secret reference, so apps get mailbox credentials the same way they get SMTP |
| Backup | the maildir is part of the mesh stack's data and goes into the kopia snapshot |

POP3 is one Dovecot line and stays off unless someone asks.

### 6.3 Clients

- **On the box**: a webmail store app (SnappyMail or Roundcube) reading the managed `imap` and
  `smtp` settings. This is the supported client for the first release.
- **Outside the box**: see §8.

## 7. Sending: enforce, don't rewrite

### 7.1 Sender rule

Today the relay **rewrites** every sender: `mail-gateway` keeps only the local part of
`MAIL FROM` as an app name and the backend builds `<app>.<user>@<zone>` from it. A reply written
in webmail as `alice@nsl.sh` would therefore leave as `alice.alice@nsl.sh` — an address the other
side never wrote to.

Target: `mail-gateway` passes the full `MAIL FROM`, and the backend **enforces** it against the
signature-verified identity:

| `MAIL FROM` | Result |
|---|---|
| `<user>@<zone>` | sent as is (mailbox) |
| `<app>.<user>@<zone>` (`<app>` = `[a-z0-9-]{1,20}`) | sent as is (app) |
| any other address in `<zone>` (another user, a zone alias, `x.y.z@`) | **rejected**: 403 from the backend, `554 5.7.1` to the app |
| any other domain (`noreply@vaultwarden-alice.nsl.sh`, `me@gmail.com`) | **transition:** rewritten to `<local>.<user>@<zone>` and logged; later rejected like the row above |

The domain still comes only from the verified identity, so sending as another user stays
impossible, as today.

**Why the transition row.** Every app on every box currently sends from an address outside the
zone and relies on the rewrite: Maison's managed `smtp` leaves `from` empty, which falls back to
`noreply@APP_DOMAIN` (`maison/docs/config.md`). Rejecting those today would stop all app mail.
Order: (1) Maison's managed `smtp` sets `from = <app>.<user>@<zone>` per app; (2) the relay
logs every rewrite; (3) once the logs show only stragglers, the rewrite becomes a reject.

Open: whether `<user>+<tag>@<zone>` may send (keeps the tag in replies). Not in the first cut.

### 7.2 Human mail

- **mail-gateway**: for a mailbox sender (`<user>@<zone>`), send **one** message with all
  recipients (To/Cc/Bcc) instead of one copy per recipient — copy-per-recipient is fine for
  notifications but breaks Cc for human mail.
- **backend `/email/send`**: accept `from`, `cc`, `bcc` and threading headers (`In-Reply-To`,
  `References`, `Reply-To`).
- **Deliverability**: DKIM `d=<zone>` and the `em.<zone>` return path align as today; for §2.2
  addresses they align relaxed and subdomains inherit the apex DMARC policy.
- **Abuse**: human mail goes to arbitrary recipients, so it gets its own lower rate limit and,
  like forwarding, its own SendGrid subuser.

## 8. External clients (phase 3)

Thunderbird or a phone needs **IMAPS 993** and **submission 465**. Both are implicit TLS with SNI,
so the gateway can route them exactly like 443: `mesh-router-gateway/nginx/lua/sni_router.lua`
already passes dot-form names through to the box over its best route (public IP or tunnel). Work:

- gateway: extra stream listeners on 993/465 using the same SNI decision, passthrough only
  (never terminate mail TLS on the gateway);
- box: Dovecot 993 and mail-gateway 465 on `mail.<user>.<zone>` with a real certificate (DNS-01
  through the backend's acme-dns API, as Caddy does), 465 requiring SMTP AUTH with the mailbox
  credentials — this is the first time `mail-gateway` is reachable from outside the box;
- boxes in direct mode get the same names resolving to their own IP.

Until then, external clients are not supported; webmail on the box covers access from anywhere.

## 9. Dashboard and CLI

- Owner dashboard: mail mode `off | forward | box`, forward target + verification state, box
  availability, last deliveries (time, sender domain, status; never subjects or bodies).
- `mesh-cli mail setup|status|mode|resync` in the backend container for admin actions
  (create the parse setting, inspect a user, drain the queue), consistent with `mesh-cli dns …`.
- mesh-console: inbound counters next to the existing `mail-gateway stats`.

## 10. Privacy and abuse

- Mail content passes through Twilio SendGrid (receive, and send for forward/reply) and the
  backend in memory; nothing is stored centrally except the queue in §4 while a box is down.
  The privacy policy must say so.
- Logs and stats record time, sender domain, recipient, status and size — never subject or body.
- A user turned off or deleted stops receiving at once; queued mail is dropped.

## 11. Phases

| Phase | Content | Depends on |
|---|---|---|
| 0 — before the cutoff | Same perimeter as Cloudflare Email Routing: Inbound Parse on `<zone>`, `/email/inbound/<token>` forwards **every** message of the zone to `MAIL_CATCHALL` (rewrite of §5.2, spam drop, Message-ID dedupe, loop guard); MX switched in Cloudflare. Implemented in `mesh-router-backend/src/services/InboundMail.ts` | nothing — replaces Cloudflare Email Routing |
| 0.5 | Zone aliases (§5.6) replace the catch-all, everything else dropped; reserve `contact`, `admin`, `abuse`, `postmaster`, `dmarc`. Implemented (`MAIL_CATCHALL` replaced by `MAIL_ALIAS_TARGET` / `MAIL_ALIASES`) | phase 0 |
| 1 | Users: recipient resolution (§4), mode `forward`, set from the box (mesh-console Email page). Implemented | phase 0 |
| 1.5 | Sender rule step 1–2 (§7.1), split from phase 1 (decision 2026-10-30) | phase 1 |
| 2 | Mode `box`: registration, `/inbound` + Dovecot in the mesh stack, managed `imap`, human mail (§7.2), webmail store app; sender rule step 3 | phase 1 |
| 3 | IMAPS / submission through the gateway | phase 2 |
| add-on | `*@<user>.<zone>` (§2.2) | user zones (after the cutoff) |

## 12. To verify before phase 0

- ~~Inbound Parse accepts the bare zone as hostname~~ — yes (staging, 2026-10-30).
- Inbound Parse retry policy on non-2xx: it retries (first retry after ~30 s); how long it keeps
  trying is still unknown, and decides whether the §4 queue is needed.
- Inbound Parse message size limit. Webhook authentication: no signature header seen, so the
  URL token stays.
- SendGrid plan: subusers available, and monthly volume once forwarding and replies count.
- Before §2.2: how many parse settings one account may hold, and whether wildcards are accepted.
