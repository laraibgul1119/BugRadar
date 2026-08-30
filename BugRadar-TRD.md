# BugRadar — Technical Requirements Document (TRD)

**Version:** 1.0
**Status:** Draft — Hackathon MVP
**Related:** BugRadar-PRD.md, BugRadar-AppFlow.md
**Last Updated:** August 2026

---

## 1. Architecture Overview

BugRadar follows a simple, monolith-first architecture (easiest to build and deploy within a hackathon timeline and free-tier compute limits), with clear internal module boundaries so it can be split into services later.

```
                          ┌────────────────────────────┐
                          │      Client Applications    │
                          │  (Web app instrumented with │
                          │   BugRadar JS/Node/Python   │
                          │           SDK)               │
                          └──────────────┬───────────────┘
                                         │ HTTPS (event payload + DSN key)
                                         ▼
                     ┌───────────────────────────────────┐
                     │   Alibaba Cloud ECS (1 vCPU/1GB,   │
                     │   Free Tier, 12 months)            │
                     │  ┌───────────────────────────────┐│
                     │  │   Nginx (reverse proxy + TLS)  ││
                     │  └───────────────┬────────────────┘│
                     │                  ▼                  │
                     │  ┌───────────────────────────────┐ │
                     │  │  BugRadar Backend (Node.js/    │ │
                     │  │  Express or FastAPI) — API,    │ │
                     │  │  Ingestion, Auth, Alerts       │ │
                     │  └──────┬───────────────┬──────────┘│
                     │         │               │            │
                     │         ▼               ▼            │
                     │  ┌────────────┐   ┌───────────────┐ │
                     │  │ PostgreSQL │   │ Redis (queue/  │ │
                     │  │ (ApsaraDB  │   │ cache, free    │ │
                     │  │ RDS free   │   │ tier or ECS-   │ │
                     │  │ tier /     │   │ hosted)        │ │
                     │  │ self-      │   │                │ │
                     │  │ hosted on  │   └───────────────┘ │
                     │  │ ECS)       │                      │
                     │  └────────────┘                      │
                     └───────────────────────────────────────┘
                                         │
                                         ▼
                     ┌───────────────────────────────────┐
                     │  Alibaba Cloud OSS (Object Storage) │
                     │  — stores source maps, large stack  │
                     │  traces, static frontend build      │
                     └───────────────────────────────────────┘
                                         │
                                         ▼
                     ┌───────────────────────────────────┐
                     │  Notification layer:                │
                     │  - Transactional email (free tier   │
                     │    provider e.g. Brevo/Resend)      │
                     │  - Slack/Discord webhook (stretch)  │
                     └───────────────────────────────────────┘

                     ┌───────────────────────────────────┐
                     │  Billing: Payoneer Checkout API /   │
                     │  local gateway (Safepay/PayPro)     │
                     │  — isolated module, sandbox mode    │
                     └───────────────────────────────────────┘
```

**Design principle:** Keep the system a single deployable unit (backend API + background worker + static frontend) for MVP simplicity, but structure code into clear modules (`ingestion`, `auth`, `issues`, `alerts`, `billing`) so it can be decomposed into microservices post-hackathon if usage grows beyond free-tier capacity.

---

## 2. Technology Stack

| Layer | Choice | Rationale |
|---|---|---|
| Frontend | React + Vite + TailwindCSS | Fast dev velocity, small bundle, free to host as static build on OSS/CDN |
| Backend API | Node.js (Express) or Python (FastAPI) | Both have first-class SDK ecosystems for error capture; pick one per team skillset |
| Database | PostgreSQL | Relational integrity for orgs/projects/users/issues; JSONB columns for flexible event payloads |
| Cache/Queue | Redis | Rate limiting, async event processing queue, session cache |
| Object Storage | Alibaba Cloud OSS | Store source maps, large stack traces, static assets |
| Auth | JWT (access + refresh tokens), bcrypt/argon2 password hashing | Stateless, simple, no paid auth-as-a-service needed |
| Email | Brevo (Sendinblue) free tier / Alibaba Cloud DirectMail free quota | Free transactional email quota available and works for recipients in Pakistan |
| Payments | Payoneer Checkout (primary), Safepay/PayPro/JazzCash/Easypaisa (local fallback) | Stripe unavailable for direct PK merchant onboarding |
| Hosting/Compute | Alibaba Cloud ECS (Elastic Compute Service, free tier instance) | Meets "must deploy on Alibaba Cloud free tier" constraint |
| Reverse Proxy/TLS | Nginx + Let's Encrypt (free certs) | Free HTTPS termination |
| CI/CD | GitHub Actions (free for public/private repos within limits) | Automated build/test/deploy on push |
| Monitoring BugRadar itself | Self-hosted BugRadar instance ("dogfooding") + Alibaba Cloud CloudMonitor free tier | Free infra-level metrics (CPU/mem/disk) |
| Error/Log format | Sentry-compatible envelope format (subset) | Lets us optionally reuse existing open-source Sentry SDKs pointed at a custom DSN, saving SDK-authoring time |

> **Note:** Adopting a Sentry-protocol-compatible ingestion endpoint (a subset of the open `sentry-Envelope`/DSN model) is a deliberate technical shortcut: existing Sentry SDKs (`@sentry/browser`, `@sentry/node`, `sentry-sdk` for Python) can point at BugRadar's ingestion URL, drastically reducing the SDK-build effort during the hackathon while still shipping "our own" dashboard, grouping, alerting, and billing.

---

## 3. Alibaba Cloud Free-Tier Service Mapping

As of the current free-tier program, Alibaba Cloud offers **new-account trial credits (commonly $200–$1,200+ depending on account type, valid ~60 days) plus a 12-month free ECS trial** (typically a 1-core/1GB instance), and **always-free monthly quotas** on select services (e.g., OSS storage, some CDN/DNS quotas) that renew monthly as long as the account stays active. Exact figures vary by region/promotion and should be re-verified in the Alibaba Cloud console at project start, since these change over time.

| Service | Free Tier Component Used | Purpose in BugRadar |
|---|---|---|
| **ECS** (Elastic Compute Service) | 12-month free trial instance (e.g., 1-core/1GB or 2-core/2GB depending on account type) | Hosts backend API, worker, Nginx, and (optionally) self-hosted Postgres/Redis |
| **OSS** (Object Storage Service) | Always-free monthly quota (several hundred GB class) | Stores frontend static build, source maps, large payloads |
| **ApsaraDB RDS** (PostgreSQL) | Trial credits (from new-user credit pool) OR self-hosted Postgres on the free ECS instance if RDS credits run out | Primary relational datastore |
| **CDN** | Free tier quota | Serve frontend assets with lower latency |
| **DirectMail** | Free monthly sending quota | Alert emails, verification emails |
| **CloudMonitor** | Free basic monitoring | CPU/memory/disk alerts on the ECS instance itself |
| **SSL Certificates** | Free via Let's Encrypt (not an Alibaba product, but zero-cost and standard practice) | HTTPS |
| **Security Group / VPC** | Free (included with ECS) | Network isolation, firewall rules for the ECS instance |

**Cost-control safeguards (since free tiers/trial credits expire or are capped):**
- Use a single right-sized ECS instance rather than multiple; run Postgres + Redis as containers on the same instance if RDS credits are limited, to avoid falling back to paid RDS.
- Set up **Alibaba Cloud budget alerts** immediately at account creation to avoid surprise charges once trial credits deplete.
- Cap ingestion volume with application-level rate limiting per project (aligned with the Free/Pro plan quotas in the PRD) so storage/compute stay within free bounds.
- Implement a scheduled job to purge events older than the plan's retention window (e.g., 30 days) to bound DB/OSS growth.

---

## 4. Data Model (Core Entities)

```
Organization
 ├── id, name, slug, plan (free/pro), created_at
 ├── has many: Project, Membership

User
 ├── id, email, password_hash, name, created_at
 ├── has many: Membership

Membership (join table: User ↔ Organization)
 ├── user_id, organization_id, role (owner/admin/member)

Project
 ├── id, organization_id, name, platform (js/node/python), dsn_key, created_at
 ├── has many: Issue

Issue
 ├── id, project_id, title, culprit, fingerprint_hash,
 │   status (unresolved/resolved/ignored),
 │   first_seen, last_seen, event_count, user_count, assigned_to (nullable User)

Event
 ├── id, issue_id, timestamp, environment, release,
 │   message, stack_trace (JSONB), breadcrumbs (JSONB),
 │   tags (JSONB), user_context (JSONB, PII-scrubbed), raw_payload_ref (OSS pointer for large payloads)

AlertRule
 ├── id, project_id, trigger_type (new_issue/spike), threshold, channel (email/webhook), enabled

Subscription (Billing)
 ├── id, organization_id, plan, status, payment_provider (payoneer/safepay/etc.),
 │   external_subscription_id, current_period_end
```

---

## 5. API Design (Representative Endpoints)

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/api/auth/signup` | Create user account |
| `POST` | `/api/auth/login` | Authenticate, issue JWT |
| `POST` | `/api/orgs` | Create organization |
| `POST` | `/api/orgs/:orgId/projects` | Create project, returns DSN |
| `POST` | `/ingest/:projectDsnKey/store/` | **Ingestion endpoint** — receives event payload from SDKs (public, keyed by DSN, rate-limited) |
| `GET` | `/api/projects/:id/issues` | List issues (filter/sort/paginate) |
| `GET` | `/api/issues/:id` | Issue detail + recent events |
| `PATCH` | `/api/issues/:id` | Update status (resolve/ignore), assign |
| `POST` | `/api/projects/:id/alert-rules` | Create alert rule |
| `POST` | `/api/billing/checkout` | Initiate Payoneer/local-gateway checkout session |
| `POST` | `/api/billing/webhook` | Receive payment provider webhook (payment success/failure) |

**Ingestion pipeline (async):**
1. `POST /ingest/...` validates DSN + rate limit → pushes raw event to Redis queue → returns `202 Accepted` immediately (keeps SDK non-blocking).
2. Background worker consumes queue → computes fingerprint → upserts `Issue` (increment count / create new) → inserts `Event` row → evaluates `AlertRule`s → dispatches notification if triggered.

---

## 6. Security Requirements

- All traffic over HTTPS (Let's Encrypt cert via Nginx).
- Passwords hashed with bcrypt/argon2; never logged.
- DSN keys are per-project, rotatable, and only grant **write** access to the ingestion endpoint (not read access to dashboard data).
- JWT access tokens short-lived (15 min) + refresh tokens (7 days), stored in httpOnly cookies for the dashboard.
- Input validation/sanitization on all ingested payloads (size caps, JSON schema validation) to prevent abuse of the public ingestion endpoint.
- Rate limiting per DSN key (Redis token bucket) to protect free-tier compute/storage from abuse or runaway loops.
- PII scrubbing option: allow projects to configure fields (e.g., email, IP) to be redacted before storage.
- Alibaba Cloud Security Group rules restrict inbound traffic to 80/443 (public) and SSH (restricted to admin IP).

---

## 7. Scalability & Performance Notes (Free-Tier Realities)

- A single free-tier ECS instance (1 vCPU/1GB) realistically supports a **low-to-moderate event volume** (hundreds to low thousands of events/day) comfortably — sufficient for hackathon demo and early free-tier users, not for high-traffic production use.
- Async ingestion (queue + worker) prevents event bursts from blocking the API or timing out SDK requests.
- Database indices on `(project_id, fingerprint_hash)` and `(issue_id, timestamp)` are critical for issue-list and issue-detail query performance at small scale.
- Documented upgrade path (post free-tier / post-hackathon): move to paid ECS tier or split into managed RDS + separate compute + managed Redis (e.g., Alibaba Cloud ApsaraDB for Redis) once traffic exceeds a single-instance ceiling.

---

## 8. Deployment & CI/CD

1. **Source control:** GitHub monorepo (`/frontend`, `/backend`, `/sdks`).
2. **CI:** GitHub Actions — lint, unit test, build Docker images on every PR.
3. **CD:** On merge to `main`, GitHub Actions builds and pushes Docker images, then SSHs into the Alibaba Cloud ECS instance to pull and restart containers (`docker compose up -d`) — simplest zero-cost deployment approach avoiding paid container registries/orchestration.
4. **Environment config:** `.env` file on the ECS instance (not committed), managed manually or via GitHub Actions secrets injected at deploy time.
5. **Domain/DNS:** Free subdomain (e.g., from Alibaba Cloud DNS free tier, or a free provider like a `.tech`/`.xyz` hackathon domain, or a Cloudflare-proxied free domain) pointed at the ECS public IP.

---

## 9. Payment Integration Notes

- **Primary:** Payoneer Checkout — supports receiving payments into Pakistan; integrate via Payoneer's checkout/API for one-time and (where supported) recurring charges. Because Payoneer's recurring/subscription billing tooling is less mature than Stripe's, MVP billing may be implemented as **manual/periodic re-checkout** (user re-authorizes payment monthly) rather than true auto-renewing subscriptions, with a clear upgrade note for post-hackathon iteration.
- **Local fallbacks to evaluate:** Safepay and PayPro (Pakistani payment gateways with Shopify/API support, PKR settlement), and direct JazzCash/Easypaisa merchant APIs for mobile-wallet based billing — useful if targeting purely local PKR-paying customers rather than international USD customers.
- **Webhook handling:** All payment provider webhooks verified via signature before updating `Subscription` status, to prevent spoofed payment confirmations.
- Billing module is isolated (`/billing`) so the payment provider can be swapped without touching core product logic.

---

## 10. Testing Strategy

- Unit tests: fingerprinting/grouping logic, auth flows, alert rule evaluation.
- Integration tests: ingestion endpoint → issue creation → alert dispatch (using a test SMTP catcher).
- Load smoke test: simulate burst of events against ingestion endpoint to confirm queue absorbs load without dropping events, within free-tier instance limits.
- Manual QA checklist: SDK integration on a sample broken app, end-to-end from error thrown → dashboard visible → email received.

---

## 11. Open Technical Decisions (To Finalize During Build)

- Node/Express vs. Python/FastAPI for backend (team skillset dependent).
- Self-hosted Postgres on ECS vs. ApsaraDB RDS trial credits (depends on remaining credit balance).
- Whether to implement true Sentry-envelope protocol compatibility (enables reuse of official Sentry SDKs) vs. a simpler custom JSON payload (faster to build, requires writing thin custom SDKs).
