# BugRadar — Product Requirements Document (PRD)

**Version:** 1.0
**Status:** Draft — Hackathon MVP
**Owner:** [Your Name / Team Name]
**Last Updated:** August 2026

---

## 1. Overview

**BugRadar** is a self-hosted, developer-first error tracking and application monitoring platform — a lean, open, Sentry-inspired alternative built to run entirely on free-tier infrastructure and payment rails available in Pakistan.

It captures unhandled exceptions, logged errors, and performance traces from web and backend applications, groups them into de-duplicated "Issues," and gives engineering teams a real-time dashboard to triage, assign, and resolve them — closing the loop between "something broke in production" and "someone on the team knows and is fixing it."

### 1.1 Problem Statement

- Most small teams and student/hackathon projects in Pakistan either ship blind (no error visibility) or rely on `console.log` / manual bug reports from users.
- Sentry, Datadog, and similar tools are excellent but:
  - Priced in USD, billed via Stripe — **not available for direct signup/payment from Pakistan**.
  - Often overkill (feature-heavy, expensive at scale) for small teams, indie hackers, and student projects.
- There is a gap for a **lightweight, self-hostable, regionally-accessible** error tracking tool that a small team can deploy on free infrastructure and pay for (if ever) through payment rails that actually work in Pakistan.

### 1.2 Vision

Give every developer and small team — especially in Pakistan and similar markets underserved by global SaaS billing — a **production-grade, free-to-start error tracking platform** they can either self-host or use as a hosted service, paid for through Payoneer/local gateways instead of Stripe.

### 1.3 Goals (Hackathon MVP)

1. Capture and display real-time errors from at least one web SDK (JavaScript/Node) and one backend SDK (Python or Node/Express).
2. Automatically group duplicate errors into "Issues" with occurrence counts, first-seen/last-seen timestamps, and affected users.
3. Provide a clean dashboard: project list → issue list → issue detail (stack trace, breadcrumbs, environment, device/browser info).
4. Support basic alerting (email, and optionally Slack/Discord webhook) when a new issue is created or an issue spikes.
5. Multi-tenant: support multiple organizations/projects with simple role-based access (Owner, Admin, Member).
6. Ship a working billing skeleton using **Payoneer Checkout** (or a local gateway fallback) for a paid "Pro" tier — even if gated behind a "Coming Soon" flag for the hackathon demo.
7. Deploy fully on **Alibaba Cloud's free tier**, publicly accessible via HTTPS.

### 1.4 Non-Goals (Out of Scope for MVP)

- Full performance/APM tracing (distributed tracing, span waterfalls) — stretch goal only.
- Session replay (Sentry's screen-recording feature).
- Native mobile SDKs (iOS/Android) — stretch goal only.
- SAML/SSO enterprise auth.
- On-call scheduling / PagerDuty-style escalation policies.
- Multi-region deployment / advanced autoscaling.

---

## 2. Target Users & Personas

| Persona | Description | Needs |
|---|---|---|
| **Ali — Indie Hacker / Freelancer** | Builds side projects and freelance client apps in Pakistan | Wants free/cheap error monitoring without a US-billed credit card |
| **Sara — Startup Backend Engineer** | Works at a small Lahore-based startup, 5–10 person eng team | Wants issue triage, assignment, and Slack alerts without enterprise pricing |
| **Hamza — CS Student / Hackathon Participant** | Building a hackathon project or FYP | Needs to demo a "production-ready" monitoring integration quickly |
| **Team Lead / Engineering Manager** | Oversees multiple projects/repos | Needs an org-level dashboard with role management and audit visibility |

---

## 3. Competitive Landscape

| Product | Strength | Gap for our users |
|---|---|---|
| Sentry.io | Best-in-class error tracking, huge SDK ecosystem | Stripe-only billing, no Pakistan self-serve payment, pricier at scale |
| GlitchTip | Open-source, Sentry-protocol compatible | Requires self-hosting expertise; no polish for non-technical stakeholders |
| Bugsnag / Rollbar | Mature, enterprise-ready | Same billing/region issue as Sentry |
| **BugRadar** | Free-tier-first deployment, Pakistan-accessible billing (Payoneer/local gateways), Sentry-protocol-inspired simplicity | Smaller SDK ecosystem at launch (by design, MVP scope) |

---

## 4. Core Features (MVP Scope)

### 4.1 Authentication & Organization Management
- Email/password signup + login (JWT-based sessions).
- Email verification (via free-tier transactional email, e.g., Alibaba Cloud DirectMail free quota or Resend/Brevo free tier).
- Create Organization → Create Project (each project gets a unique **DSN**, Data Source Name / API key, mirroring Sentry's model).
- Roles: **Owner**, **Admin**, **Member** (Member = view + comment, Admin = manage projects, Owner = billing + org settings).

### 4.2 Error/Event Ingestion
- REST ingestion endpoint (`POST /api/{project_id}/store/`) accepting JSON error payloads.
- SDKs (or lightweight drop-in snippets) for:
  - JavaScript (browser) — captures unhandled `window.onerror` / `unhandledrejection`.
  - Node.js/Express — middleware to capture uncaught exceptions.
  - Python/Flask or FastAPI — exception handler integration.
- Payload includes: error message, stack trace, environment (`production`/`staging`/`dev`), release/version tag, user context (optional, privacy-respecting), browser/OS metadata, custom tags, breadcrumbs (last N actions before the error).

### 4.3 Issue Grouping & Deduplication
- Fingerprinting algorithm: hash of (error type + normalized stack trace top frames) groups repeated occurrences into a single **Issue**.
- Each Issue tracks: title, culprit (file/function), first seen, last seen, total occurrences, affected users count, status (`unresolved`, `resolved`, `ignored`).

### 4.4 Dashboard & Issue Management
- Organization → Project → Issue List (filterable by status, environment, time range, search by error message).
- Issue Detail view: full stack trace (syntax highlighted), breadcrumbs timeline, tags, occurrence graph (events/day sparkline), affected users list, comment thread, assign-to-teammate.
- Bulk actions: resolve, ignore, delete.

### 4.5 Alerting
- Rule: "Notify when a new Issue is created" (default, on by project).
- Rule: "Notify when Issue occurrences exceed N in M minutes" (spike detection).
- Channels: Email (MVP) → Slack/Discord webhook (stretch).

### 4.6 Release Tracking (Stretch)
- Tag events with a release/version string; show issues-per-release to catch regressions after deploys.

### 4.7 Billing & Monetization (Hackathon-Scoped)
- Free tier: 1 organization, up to 3 projects, 5,000 events/month, 30-day event retention.
- Pro tier (paywall UI only, functional checkout stubbed/sandboxed for demo): higher event volume, longer retention, unlimited projects.
- Payment integration target: **Payoneer Checkout** as primary (Payoneer is available for receiving payments into Pakistan); fallback/local options evaluated: **Safepay**, **PayPro**, **JazzCash/Easypaisa merchant APIs** for PKR-denominated local billing, since Stripe is not directly available to Pakistani merchants without an offshore entity.

---

## 5. Success Metrics (Hackathon Judging Criteria Alignment)

| Metric | Target for Demo |
|---|---|
| Time from unhandled exception → visible Issue in dashboard | < 5 seconds |
| SDK integration time (from docs) | < 10 minutes for a new developer |
| Working deployment on Alibaba Cloud free tier | 100% uptime during judging window |
| Alerting round-trip (issue created → email received) | < 2 minutes |
| Live demo: intentionally-broken sample app reporting to BugRadar | End-to-end working |

---

## 6. Assumptions & Constraints

- **Infrastructure:** Must run within Alibaba Cloud's free-tier limits (see TRD for specific service quotas) — this bounds compute, storage, and database size for the MVP.
- **Payments:** Stripe is unavailable for direct Pakistani merchant signup; Payoneer Checkout and/or local Pakistani gateways (Safepay, PayPro, JazzCash, Easypaisa) are the only viable payment rails without incorporating offshore.
- **Team size/time:** Hackathon timeline — scope is deliberately trimmed vs. full Sentry feature parity.
- **Data residency:** No specific compliance requirement for MVP (not handling regulated data), but event payloads should support scrubbing of PII/sensitive fields as a good practice.

---

## 7. Risks

| Risk | Mitigation |
|---|---|
| Alibaba Cloud free-tier quota exhaustion during demo/judging | Add rate-limiting on ingestion endpoint; monitor usage; use lightweight always-free services where possible |
| Payoneer Checkout not fully supporting recurring/subscription billing | Ship billing as a "sandbox/demo" flow for the hackathon; document real production path in TRD |
| Event storage growth exceeding free DB tier | Implement retention policy (auto-delete events older than 30 days on Free plan) |
| SDK complexity scope creep | Freeze SDK scope to JS + one backend language for MVP; document extension points |

---

## 8. Milestones (Suggested Hackathon Timeline)

1. **Day 1:** Auth, Org/Project model, DSN generation, ingestion API skeleton, DB schema on Alibaba Cloud.
2. **Day 2:** Issue grouping logic, dashboard UI (project list, issue list, issue detail), JS SDK.
3. **Day 3:** Alerting (email), Node/Python SDK, deploy pipeline to Alibaba Cloud.
4. **Day 4:** Billing UI stub with Payoneer Checkout sandbox, polish, seed demo data, prepare live demo app.

---

## 9. Appendix: Feature Parity Snapshot vs. Sentry

| Sentry Feature | BugRadar MVP | BugRadar Stretch |
|---|---|---|
| Error tracking & grouping | ✅ | — |
| Multi-project orgs | ✅ | — |
| Alerting rules | ✅ (basic) | Advanced conditions |
| Release tracking | Stretch | ✅ |
| Performance monitoring/tracing | ❌ | Stretch |
| Session Replay | ❌ | Not planned |
| Native mobile SDKs | ❌ | Stretch |
| SSO/SAML | ❌ | Not planned |
