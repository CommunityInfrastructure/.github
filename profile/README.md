# Community Infrastructure

Shared platform services for community crisis response organizations across New Jersey.

## Domains

| Domain | Community | Type |
|--------|-----------|------|
| [community-infra.org](https://community-infra.org) | Platform services | Backend |
| [frenchtownhelps.org](https://frenchtownhelps.org) | Frenchtown, NJ | Frontend |
| [frenchtowncares.org](https://frenchtowncares.org) | Frenchtown, NJ | Frontend |
| [hunterdoncares.org](https://hunterdoncares.org) | Hunterdon County, NJ | Frontend |
| [hunterdonhelp.org](https://hunterdonhelp.org) | Hunterdon County, NJ | Frontend |
| [hunterdonhelps.org](https://hunterdonhelps.org) | Hunterdon County, NJ | Frontend |
| [flemingtonhelps.org](https://flemingtonhelps.org) | Flemington, NJ | Frontend |
| [flemingtoncares.org](https://flemingtoncares.org) | Flemington, NJ | Frontend |
| [warrencares.org](https://warrencares.org) | Warren County, NJ | Frontend |
| [warrenhelps.org](https://warrenhelps.org) | Warren County, NJ | Frontend |
| [somersethelps.org](https://somersethelps.org) | Somerset County, NJ | Frontend |
| [somersetcares.org](https://somersetcares.org) | Somerset County, NJ | Frontend |
| [nohatenj.org](https://nohatenj.org) | New Jersey | Frontend |

## Website Repositories

Each domain has its own private repository containing static website content.

- `prod/` — live content served at the domain
- `test/` — staging content for review before promotion
- `archive/` — timestamped backups of previous production versions

Changes pushed to `prod/` are automatically synced to the production server every 5 minutes. Before each sync, the current live content is archived locally for rollback.

---

## Platform Architecture

### Overview

The platform provides SMS/MMS-to-chat bridging for community crisis response. Community members text a public phone number and their messages are routed into a secure Matrix chat workspace where trained volunteer operators can respond. The system is designed so operators never see raw phone numbers, all message content is redacted from logs, and every version of every public site is archived before updates.

**No AI or machine learning is used in the runtime system.** The entire backend is deterministic, auditable application code — TypeScript, SQL, and documented REST API calls to well-defined services (Twilio for SMS delivery, Google/DeepL for translation, Matrix client-server API for chat). There are no calls to black-box AI providers, no LLM inference, no probabilistic decision-making in the message pipeline. Every action the system takes is traceable to explicit code paths. When a community member texts for help, a real human operator reads and responds — the platform is the routing and security layer, not a chatbot.

```
Community Member                    Volunteer Operators
      |                                     |
  SMS/MMS                          Element Web (Matrix client)
      |                                     |
      v                                     v
  [Twilio] ──POST──> [Caddy] ──> [Bot] <──> [Synapse]
                        |                      |
                   TLS termination        Matrix homeserver
                   static sites            (PostgreSQL)
```

### Frontend Domains (Community-Facing)

Frontend domains are **read-only community announcement boards**. They serve static content — local news, resource directories, event information, and crisis updates — published by community coordinators. There is no user login, no database, and no dynamic server-side content.

Each frontend includes a **"Need help?" chat interface** that connects community members to trained helpdesk operators via SMS. Community members text a published phone number displayed on the site and their messages are routed to the operator workspace. The frontend itself never handles message data — it simply directs people to the SMS channel.

Frontend domains share no backend state. They are purely static HTML/CSS/JS served by Caddy from per-domain directories.

### Backend Services (community-infra.org)

Backend services are **not accessible to the general public**. They are used exclusively by enrolled helpdesk operators and system administrators. All backend interfaces are gated behind authentication:

| Service | Subdomain | Access |
|---------|-----------|--------|
| Synapse | `matrix.community-infra.org` | Matrix client API — requires authenticated Matrix account |
| Element Web | `element.community-infra.org` | **Tailscale VPN only** — not reachable from the public internet |
| SMS Bot | `sms.community-infra.org` | Twilio webhook endpoint only — HMAC signature validated, no public UI |

Operator accounts are created exclusively by administrators via the `!enroll` command. There is no self-registration.

**Element Web has no public DNS records and is restricted to Tailscale VPN connections only.** The Caddy reverse proxy enforces source IP validation — only the Tailscale CGNAT range (100.64.0.0/10) is permitted. Connections from the public internet receive a 403 response directing users to a native Matrix client instead. This eliminates the web application as an attack surface entirely. Operators who need Element Web must be on the Tailscale network; all other operators use native Matrix clients.

### Operator Mobile Access

Because the backend is built on the open [Matrix protocol](https://matrix.org/), operators are not limited to the hosted Element Web interface. Operators can respond to community members from any compatible Matrix client on any device, including mobile. This means helpdesk operators can work from their phones in the field during crisis events.

Compatible clients include [FluffyChat](https://fluffychat.im/), [Element X](https://element.io/), [SchildiChat](https://schildi.chat/), [Nheko](https://nheko.im/nheko-reborn/nheko), [Cinny](https://cinny.in/), and [dozens more](https://matrix.org/ecosystem/clients/) across iOS, Android, desktop, web, and terminal. Any Matrix client that supports threads (MSC3440) provides the full operator experience — claim conversations, reply to community members, run `!` commands, and receive delivery status updates.

### Voice and Multi-Geo Routing (In Development)

Voice call routing is in active development. Inbound calls to published phone numbers will be routed to available operators in the corresponding geographic area, with the same claim and dispatch model used for SMS conversations. Voice events are already captured and logged by the platform (the `voice_events` table and provider adapter interface support this today).

Operators will be able to work across multiple geographic areas simultaneously, handling both voice calls and text conversations from a single workspace. An operator enrolled in Frenchtown and Warren County, for example, would see inbound conversations from both communities and can claim and respond to either. Geographic assignment is managed per-operator, and operators can be added or removed from any geo at any time without service interruption.

### Container Stack

| Container | Image | Purpose |
|-----------|-------|---------|
| `caddy` | caddy:2-alpine | TLS termination, reverse proxy, static file serving |
| `synapse` | matrixdotorg/synapse | Matrix homeserver |
| `postgres` | postgres:16-alpine | Synapse database backend |
| `element` | vectorim/element-web | Operator chat UI |
| `bot` | interactiveagent-bot | SMS/MMS bridge, webhook server, command handler |

All containers run on a single Docker bridge network. Only Caddy exposes ports 80/443 to the internet. Inter-service communication is container-to-container via Docker DNS.

---

## SMS/Chat Message Flow

### Inbound (Community Member to Operator)

1. Community member sends SMS/MMS to a published phone number
2. Twilio delivers the message via HTTP POST to `sms.community-infra.org/sms`
3. Caddy terminates TLS and proxies to the bot container
4. Bot validates the Twilio HMAC-SHA1 signature (rejects forged webhooks)
5. Bot deduplicates the message (prevents double-processing of Twilio retries)
6. Bot tokenizes the sender's phone number (HMAC-SHA256 — irreversible, used for lookups)
7. Keyword check: `STOP`, `HELP`, `START` are handled inline per CTIA/TCPA requirements
8. Rate limit check (20 inbound messages per number per minute)
9. Bot finds or creates a conversation thread in the Matrix dispatch room
10. Message appears as a thread reply in Element Web for operators to see
11. For MMS: media is downloaded from Twilio (authenticated), uploaded to Synapse, and posted as an image in the thread

### Outbound (Operator to Community Member)

1. Operator replies in the Matrix thread
2. Bot receives the Matrix event via its sync loop
3. Bot looks up the conversation by room ID + thread root event ID
4. Claim enforcement: if another operator claimed this conversation, the send is blocked
5. Rate limit check (10 outbound per number per minute, 100 global per minute)
6. Bot decrypts the stored phone number (AES-256-GCM — only decrypted at send time)
7. Message is sent via Twilio REST API
8. Delivery status callbacks update the conversation record; failures are posted back to the thread

### Translation (Optional)

When configured with a translation provider (Google Translate or DeepL):
- Inbound messages are auto-detected for language and translated to English for operators
- Outbound messages can be auto-translated to the community member's detected language
- Original text is always preserved alongside the translation

---

## Security Architecture

### Design Principles

1. **Phone numbers are never stored in plaintext.** All phone storage uses cryptographic tokenization or encryption.
2. **Message content never appears in logs.** All log output is redacted by default.
3. **Operators never see raw phone numbers.** Only the last 4 digits are shown in the chat interface.
4. **Least privilege by default.** The sync agent uses a read-only credential. The bot's Matrix token is scoped to its own user. Caddy is the only internet-facing service.
5. **No registration.** All user accounts are created by administrators via the `!enroll` command.

### Phone Number Protection

| Operation | Method | Purpose |
|-----------|--------|---------|
| Lookup/dedup | HMAC-SHA256 (`PHONE_HMAC_KEY`) | Deterministic token for database queries without storing plaintext |
| Storage | AES-256-GCM (`PHONE_ENCRYPTION_KEY`) | Reversible encryption with per-record random IV; decrypted only at send time |
| Display | Last 4 digits only (`***1234`) | Operators see enough to confirm identity, not enough to contact directly |

The two keys (`PHONE_HMAC_KEY`, `PHONE_ENCRYPTION_KEY`) are independent 256-bit hex values. Compromise of the HMAC key allows correlation but not recovery of phone numbers. Compromise of the encryption key allows recovery only with access to the database.

### Log Redaction

Every log line passes through a redaction pipeline before output:

- **Phone numbers** (`+1XXXXXXXXXX` patterns) are replaced with `[REDACTED_PHONE]`
- **Email addresses** are replaced with `[REDACTED_EMAIL]`
- **API keys and tokens** (matched by key name patterns) are replaced with `[REDACTED_KEY]`
- **Message body fields** (`body`, `content`, `text`, `messageBody`, `smsBody`, `rawBody`) are replaced wholesale with `[REDACTED]`
- Redaction is applied recursively through nested objects

Log output is structured JSON (`{timestamp, level, service, component, message, context}`) for machine parsing. Message content is never included in any log level.

### Webhook Security

- All inbound Twilio webhooks are validated via HMAC-SHA1 signature verification
- The signature is computed over the full webhook URL + sorted POST parameters using the Twilio auth token
- Signature enforcement is mandatory in production (`enforceSignatureValidation: true`)
- Webhook deduplication prevents replay attacks (each `MessageSid` is processed exactly once)

### Network Security

- Caddy is the only container with published ports (80, 443)
- All inter-container traffic stays on the Docker bridge network
- Element Web is not accessible from the public internet — restricted to Tailscale VPN (100.64.0.0/10) with HTTPS Basic Auth as a second factor
- No public DNS records exist for Element subdomains
- Synapse has registration disabled — users are provisioned via admin API only
- Matrix federation endpoints are exposed for protocol compliance but registration is closed

### Compliance Controls

| Control | Implementation |
|---------|---------------|
| CTIA keyword handling | `STOP`, `HELP`, `START` processed before any other logic |
| Opt-out enforcement | Blocked numbers table (by token); all sends check before delivery |
| Rate limiting | Per-number and global limits on both inbound and outbound |
| Anomaly detection | Flags >50 messages/5min or 5+ identical messages from one number |
| Audit logging | All outbound sends logged with provider message ID and delivery status |
| PII containment | `containsPii()` utility flags phone/email/address patterns in operator notes |

---

## Multi-Organization Routing

The platform supports multiple community organizations on shared infrastructure. Routing is configured via `ORG_ROUTING_MAP`, a JSON mapping from intake phone number digits to organization identifiers:

```json
{"1XXXXXXXXXX": "frenchtown", "1XXXXXXXXXX": "hunterdon"}
```

Each conversation is tagged with an `organizationId` at creation time. All database queries (conversation lookup, queue, search, reporting) are scoped by organization, ensuring data isolation between communities sharing the same Twilio numbers or Matrix homeserver.

---

## Operator Commands

Operators interact with the system via `!` commands in Matrix:

| Command | Role | Description |
|---------|------|-------------|
| `!queue` | operator | List open/unclaimed conversations |
| `!claim` | operator | Claim a conversation thread for exclusive response |
| `!unclaim` | operator | Release a claimed conversation |
| `!close` | operator | Begin structured closeout flow (category, resolution, notes) |
| `!close cancel` | operator | Cancel an in-progress closeout |
| `!block` | operator | Block a phone number (by token) |
| `!unblock` | operator | Unblock a phone number |
| `!search` | operator | Search conversations by keyword, date, status |
| `!translate` | operator | Set auto-translation preferences |
| `!private` | operator | Move conversation to an encrypted private room |
| `!help` | operator | Show available commands |
| `!status` | supervisor | System health and statistics |
| `!report` | supervisor | Generate activity reports |
| `!enroll` | admin | Create a new operator account |
| `!deactivate` | admin | Deactivate an operator account |

### Conversation Lifecycle

```
New SMS → [open] → !claim → [claimed] → reply ↔ SMS → !close → [closeout] → [closed]
                                                           ↓
                                                    !close cancel → [claimed]
```

### Closeout Flow

When an operator runs `!close`, the bot walks them through a structured data collection:
1. **Category** — what type of request (e.g., housing, food, medical, information)
2. **Resolution** — how it was resolved (e.g., referred, resolved, escalated)
3. **Service area** — geographic area served
4. **Urgency** — low/medium/high/critical
5. **Follow-up needed** — yes/no
6. **Notes** — free-text (checked for PII before storage)

Closeout data is stored in the `interaction_log` table and used for aggregate reporting via `!report`.

---

## Data Storage

### Bot State (SQLite)

The bot maintains its own state in a SQLite database (separate from Synapse's PostgreSQL):

| Table | Contents |
|-------|----------|
| `conversations` | Active/closed conversations: org, phone token, encrypted phone, Matrix thread, status, claim state, detected language |
| `outbound_log` | Every outbound SMS with provider message ID, delivery status, Matrix event reference |
| `blocked_numbers` | Opted-out or manually blocked phone tokens (no plaintext) |
| `interaction_log` | Structured closeout records for reporting |
| `rate_limits` | Token-bucket counters with expiry |
| `processed_webhooks` | Deduplication: provider message IDs already processed |
| `users` | Operator accounts with roles (operator, supervisor, admin) |
| `voice_events` | Voice call events from provider |
| `operator_translation_prefs` | Per-operator auto-translation settings |

### Synapse State (PostgreSQL)

Matrix message history, room state, user accounts, and federation data. Managed entirely by Synapse — the bot interacts via the Matrix client-server API only.

---

## Quality Assurance and Production Promotion

### Automated QA via UAT MCP

All code changes are validated through a UAT (User Acceptance Testing) pipeline powered by a dedicated [MCP (Model Context Protocol) server](https://github.com/whm3/UAT_MCP). The UAT MCP orchestrates structured test scenarios and **engages human testers** to validate the system — there are always humans in the loop. Automated scaffolding sets up test conditions and collects results, but human testers execute the critical path validations:

- **End-to-end SMS flow tests** — human testers send real SMS messages through the full pipeline and verify delivery, thread creation, and operator response flow
- **Command validation** — every `!` command tested against expected behavior and error cases
- **Compliance checks** — CTIA keyword handling (STOP/HELP/START), opt-out enforcement, rate limit behavior
- **Security boundary tests** — webhook signature rejection, PII redaction verification, role-based access enforcement
- **Provider adapter tests** — messaging and translation adapters validated against contract interfaces

No code is promoted to production without passing the full UAT suite with human sign-off. Test results are recorded and traceable to specific build artifacts.

### Protected Production Promotion

Production deployments follow a gated promotion workflow:

1. **Development** — code changes are built and tested locally
2. **Automated QA** — UAT MCP runs the full test suite against a running instance
3. **Human UAT sign-off** — a human operator validates critical paths before production release. **No release is declared production-ready until human UAT is complete and declared successful.**
4. **Production deploy** — `deploy-prod.sh` executes the full deployment pipeline
5. **Post-deploy verification** — automated health checks confirm all services are responsive

For website content, the promotion model uses a `test/` → `prod/` pipeline:
- Collaborators push changes to `test/` for review
- `promote-site.sh` archives the current `prod/`, copies `test/` to `prod/`, and commits
- The sync pipeline detects the change and deploys to the server
- Every prior version of `prod/` is preserved in `archive/` with timestamped snapshots backed up to GitHub

Rollback is immediate via `promote-site.sh --backout <domain> <archive-name>`.

---

## Infrastructure Portability

### Provider-Agnostic Architecture

The platform is designed to be fully provider-agnostic at every layer. No component has a hard dependency on a specific cloud provider, SMS carrier, or hosting environment. The entire stack can be relocated by changing DNS records and redeploying.

**Messaging providers** are abstracted behind the `MessagingProviderAdapter` interface. The adapter defines a standard contract for webhook validation, inbound/outbound SMS and MMS, media handling, and delivery status tracking. Switching SMS carriers requires implementing this interface and setting a single environment variable (`MESSAGING_PROVIDER`) — no application code changes.

**Twilio is the current SMS provider for development and testing only.** Twilio is a US-based company and not the intended production carrier. The production roadmap targets a GDPR-compliant European telecom provider to ensure SMS message metadata and delivery logs remain under EU data protection jurisdiction. The adapter abstraction makes this a drop-in replacement: implement the `MessagingProviderAdapter` interface for the new carrier, deploy with `MESSAGING_PROVIDER=<new-carrier>`, and all message routing, webhook handling, and delivery tracking work unchanged. An Infobip adapter is already implemented as a second reference.

**Translation providers** follow the same pattern via the `TranslationAdapter` interface. Google Translate and DeepL are both implemented; adding a new provider is a single package with no core changes.

**Infrastructure components** are standard open-source software with no vendor lock-in:

| Component | Current | Production Target | Alternatives |
|-----------|---------|-------------------|-------------|
| SMS carrier | Twilio (test only) | EU GDPR-compliant telecom | Infobip (implemented), any carrier via adapter |
| Hosting | Hetzner (Germany) | Hetzner or EU equivalent | Any VPS/cloud provider |
| Container runtime | Docker Compose | Same | Kubernetes, Podman, any OCI runtime |
| Matrix homeserver | Synapse | Same | Conduit, Dendrite |
| Database | PostgreSQL | Same | Already portable |
| Reverse proxy | Caddy | Same | Nginx, Traefik |
| DNS/TLS | Cloudflare | Any EU DNS provider | Caddy handles Let's Encrypt directly |
| Object storage | Cloudflare R2 (planned) | EU-resident storage | S3, MinIO, Backblaze B2 |

### GDPR and Data Sovereignty

The current deployment runs on Hetzner infrastructure in **Germany**, providing EU data residency by default for all application data, Matrix message history, and operator activity. The production intent is to align **every external dependency** — including SMS carrier, DNS, translation API, and object storage — with GDPR-compliant, EU-jurisdictioned providers.

The provider-agnostic architecture means the platform can be moved to any privacy-first, GDPR-backed hosting provider by:

1. Provisioning a new VM with the target provider
2. Updating DNS records to point to the new IP
3. Running `deploy-prod.sh` — the full stack deploys from scratch

No data migration tooling is needed beyond the standard Docker volumes and database dumps. The entire infrastructure is defined as code and reproducible from a clean VM.

This portability is a deliberate architectural choice: if a hosting provider changes terms, pricing, or jurisdiction compliance, the platform can be relocated without any application changes — only DNS and a redeploy.

---

## Deployment

All deployment is automated via `scripts/deploy-prod.sh`:

1. Rsync source code to VM
2. Build bot Docker image on VM
3. Push config files (Caddyfile, docker-compose, homeserver.yaml, Element config)
4. Sync per-domain website content from local repos
5. Start/restart Docker Compose stack
6. Wait for Synapse health
7. Provision Matrix accounts if needed (first deploy only)
8. Start bot, verify health
9. Configure Twilio webhooks to point at production URL

New domains are onboarded via `scripts/onboard-domain.sh` which automates DNS, TLS, website repo creation, and infrastructure config updates.

Website content is synced independently via `scripts/sync-websites.sh`, which polls website repos for changes every 5 minutes, archives the current live content, and deploys updates. Archive commits are backed up to GitHub separately via `scripts/push-archives.sh` using write credentials distinct from the read-only sync agent.

---

## Security Goals (Ongoing)

- [ ] GDPR-compliant centralized logging (Loki on EU infrastructure)
- [ ] Log rotation and cold storage archival to R2
- [ ] Branch protection on website repos (pending GitHub Team plan)
- [ ] E2E encrypted Matrix rooms for sensitive conversations
- [ ] Periodic key rotation for PHONE_HMAC_KEY and PHONE_ENCRYPTION_KEY
- [ ] Automated vulnerability scanning of container images
- [ ] Backup and restore procedures for Synapse/PostgreSQL state
- [ ] GitHub App service identity for automated sync (replacing PAT)
