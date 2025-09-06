
## 1) Client SDK — *“Error sensor” inside your applications*


**Architecture (simple terms):**

* Just 1–2 lines of code to initialize.
* Catches unhandled exceptions and HTTP errors.
* Sends events in batches (so apps don’t slow down).

**Technologies & Why:**

* **TypeScript** (targets: Browser, Node.js): strong typing, stable public API, tree-shaking for smaller browser size.
* **Wrappers/plugins:** React hook, Next.js middleware, Express plugin → quick adoption by teams.

**Key decisions:**

* **Reliable delivery:** batching + retries with backoff, local buffer if offline.
* **Noise control:** sampling (e.g., keep 10% of identical errors), short-term deduplication.
* **Privacy by default:** auto-scrubbing of sensitive data (emails, tokens), `beforeSend` hook for custom filtering.
* **Security:** project token (DSN) + HMAC signature of payload.

---

## 2) API Backend — *“Reception desk & processing pipeline”*

**Architecture:**

* Public endpoint `/v1/events` accepts JSON → immediately enqueues (responds with `202 Accepted`).
* Workers process events: clean noise, apply source maps (decode stack traces), calculate fingerprints, and create/update “Issues.”

**Technologies & Why:**

* **NestJS (Node 20) + Prisma:** modular, DI, validation, fast I/O, easy for devs to onboard.
* **Queue:** Kafka (or Redis Streams for MVP) → scalable, guaranteed delivery.
* **Redis:** caching & rate limiting, lightweight state management.

**Key decisions:**

* **Rate limits:** token-bucket per project and per IP to protect against traffic spikes.
* **Idempotency:** dedupe with hash keys (error type + stack + message + release).
* **Schema validation & versioning:** safe SDK evolution.
* **Server-side PII scrubbing:** second layer of protection.

---

## 3) Web Dashboard — *“Control panel for release quality”*

**Architecture:**

* Main sections: Issues, Event Stream, Release Health, Search/Filters, Alerts, Settings (projects, tokens, roles).
* Live updates via SSE/WebSocket.

**Technologies & Why:**

* **Next.js (App Router) + React + TypeScript + Tailwind + shadcn/ui:** fast development, SSR/ISR for speed, unified design kit.
* **ClickHouse:** analytics queries.
* **Postgres:** metadata (projects, users, tokens, roles).

**Key decisions:**

* **Search & filters:** by time, project, environment, version, severity, free text.
* **RBAC & ownership:** roles (Owner/Admin/Member/Read-only), assign responsible devs, Jira/Linear integrations.
* **Debugging UX:** full stack traces, breadcrumbs, tags, releases, suspicious commits.

---

## 4) Real-time Alerts — *“Know before users complain”*

**Architecture:**

* Worker runs sliding-window queries in ClickHouse (e.g., “error X > 50 times in 10 minutes”).
* Triggers notifications, respects cooldowns & deduplication.

**Technologies & Why:**

* **Email:** AWS SES / SendGrid.
* **Chat & On-call:** Slack/Teams webhooks, PagerDuty for critical incidents.

**Key decisions:**

* **Rule types:** new issue, frequency spike, regression after release, “error came back.”
* **Noise reduction:** thresholds, grouping, only notify responsible teams.
* **Traceability:** alert log with reasons and history.

---

## 5) DevOps — *“Reliable and cost-efficient platform”*

**Architecture:**

* **Containers:** Kubernetes (MVP can start with Docker Compose/ECS).
* **IaC:** Terraform + Helm → reproducible environments.
* **Monitoring:** Prometheus + Grafana; logs in Loki; alerts via Alertmanager.
* **Secrets:** AWS Secrets Manager / SOPS.
* **CDN/WAF/TLS:** CloudFront + AWS WAF, TLS via ACM.

**Storage:**

* **Postgres:** organizations, projects, users, roles, rules.
* **ClickHouse:** error events analytics (cheap, fast).
* **S3 (MinIO for dev, AWS S3 in prod):** source maps, attachments, cold storage (tiered to Glacier).

**Key decisions:**

* **Retention policy:** 14–30 days hot in ClickHouse, older → S3 (Parquet). Restoreable for analytics.
* **Autoscaling:** HPA, resource quotas per API & workers.
* **Security:** least privilege IAM, encryption in transit & at rest, full audit logs.
* **Cost optimization:** compression in CH, TTL/rollups, cold archive in S3.

---

## Why this stack works for you

* **Developer speed:** NestJS/Next.js/TypeScript = fast, familiar, single language across SDK, backend, frontend.
* **Scalability:** queues + ClickHouse handle growth smoothly.
* **Cost efficiency:** S3 for heavy/old data, ClickHouse for cheap real-time queries.
* **Privacy by default:** double PII scrubbing, project tokens + HMAC, strict RBAC.

---

## Key Decisions (Summary)

* **Real-time:** queue + workers → alerts within seconds to minutes.
* **Retention:** hot 30 days in CH, long-term in S3 (customizable).
* **Rate limits:** per-project/IP safety net during error storms.
* **Security:** HTTPS + HMAC, RBAC, IAM, audit logs.
* **DevOps process:** IaC, CI/CD with canaries, monitoring, auto-healing.

---

## Developer Questions for You

**Business & Product**

* What counts as *critical* (P0/P1)? When should we send night-time alerts?
* What SLA is expected (e.g., alerts under 60s p95)?
* Do you want “crash-free sessions” metrics and release stability reports?

**Integrations & Users**

* Which alert channels are needed first: Email, Slack/Teams, PagerDuty?
* Do you want Jira/Linear integrations and auto-assignment rules?

**Data & Privacy**

* Which PII must never be stored?
* Do you need “delete all user data X” (GDPR compliance)?
* What default retention fits: 30/90/180 days? Any legal constraints?

**Tech & Support**

* SDK platforms at launch: just Browser + Node, or also Mobile?
* What’s your expected peak error traffic (e.g., during releases)?

**Security & Access**

* Do you need SSO (Okta/Google Workspace), IP allowlist, audit logs?
* Should data storage be region-specific (EU-only)?

**Economics**

* What are storage/traffic budget limits?
* Is longer retention more important than cost savings?

