# Architecture Guardrails (Project-Specific)

This file holds the **architecture-derived guardrails for the current project**.
It is the place to encode rules that fall out of THIS project's architecture
document — tenancy model, financial integrations, async patterns, security
boundaries, deployment topology — that the generic rules files cannot know about.

It sits ON TOP of `code-standards.md` and `core-behaviors.md`, never replaces them.

## How to use this file in a new project

1. Replace the "Current project" block below with the active project's stack.
2. Keep the 12 section headings (§1–§12) — they are the universal concern areas.
3. Each section states a **generic pattern** followed by an **"In <project>:"**
   block that shows how the pattern is populated for the active project. When
   forking, rewrite the project blocks — leave the patterns alone unless the
   pattern itself doesn't fit.
4. Delete sections that genuinely don't apply (e.g. §2 if the project never
   handles money, §5 if there is no real-time channel), but pause before
   deleting — most apply to most products.

## Current project

- **Product:** Elimu360 — Integrated Digital Education Management Platform
- **Backend:** Django 5 + DRF + Channels + Celery + Redis
- **Clients:** React Native (mobile, iOS + Android), Next.js (web, SSR + SPA)
- **Data:** Aurora PostgreSQL (multi-AZ) + TimescaleDB + Redis 7 + AWS S3 + OpenSearch
- **Infra:** AWS ECS Fargate + ALB + CloudFront + WAF/Shield + API Gateway + SageMaker + Secrets Manager
- **Money rails:** Safaricom M-Pesa (Daraja), Mambu CBS, multiple credit-partner REST APIs
- **Tenancy:** Multi-school, row-level via `school_id` tenant column

The generic Tech Stack table in `CLAUDE.md` describes the onboarding kit, not
this project. For Elimu360 work, load the `python-dev`, `postgresql`, and
`vector-database` skills (not the Vue/NestJS ones).

---

## 1. Tenancy Isolation is a Safety Rule, Not a Feature

**Pattern.** In any multi-tenant system, a cross-tenant data leak (one customer
sees another's records) is a product-killing incident. Tenancy enforcement is
input-sanitisation-grade — applied at the lowest layer that can guarantee it,
never re-derived per ViewSet or per query.

**Required regardless of stack:**

- Every business table carries a tenant column with a non-null FK to the tenant
  table. No exceptions for "global" entities beyond the user, tenant, and
  platform-admin audit tables themselves.
- The tenant filter is applied by the **default model manager / repository**,
  not by application code. App code that forgets to filter must NOT be able to
  leak — the default must be safe.
- API endpoints derive the tenant from the **authenticated principal** (JWT
  claim, API key, session), never from a request-supplied path or query param.
- Raw SQL is forbidden unless the query has an explicit parameterised tenant
  clause. String interpolation of the tenant value is forbidden — always bound.
- Background jobs receive the tenant ID as an explicit argument. Workers do
  not share request context — anything implicit at the web layer becomes
  explicit at the worker layer.

**In Elimu360:** Tenant = `school_id` FK to `schools.School`.

```python
# app/managers.py — every model with school_id uses this
class SchoolScopedManager(models.Manager):
    def get_queryset(self):
        school_id = get_current_school_id()  # from request context
        if school_id is None:
            raise TenantNotResolved("school_id missing on request context")
        return super().get_queryset().filter(school_id=school_id)
```

DRF ViewSets inherit from `SchoolScopedViewSet` that injects `school_id` from
`request.user.school_id` into both `get_queryset()` AND `perform_create()`.
Celery tasks take `school_id` as a required positional argument.

**Reviewer checklist (run on every PR adding a model, endpoint, or worker task):**

- [ ] Model has tenant FK with `db_index=True` and `on_delete=PROTECT`
- [ ] Default manager filters by tenant
- [ ] API permission class enforces `principal.tenant_id == obj.tenant_id`
- [ ] Background-task signature includes tenant as a required argument
- [ ] Test covers cross-tenant access denial — and returns **404, not 403**
      (403 reveals the row exists in another tenant)

---

## 2. Money Touches Need Idempotency, Signature Verification, and Audit

**Pattern.** Any integration that moves money (or instructs a third party to
move money) must satisfy three properties: idempotent retries, cryptographic
verification of inbound webhooks, and an immutable audit trail of state
transitions.

### Idempotency

- Every outbound call to a money rail (payment, refund, loan disbursement,
  wallet debit) sends an application-generated idempotency key, stored on the
  local transaction row **before** the API call is issued.
- Worker retries reuse the same key — never regenerate. The idempotency key
  is the deduplication boundary across retries, network failures, and
  partial successes.
- Inbound webhooks are usually NOT idempotent on the provider's side
  (providers retry on timeout). Handlers must upsert by the provider's
  natural identifier and treat repeat callbacks as a no-op.

### Signature / HMAC verification

- Inbound webhook endpoints validate the provider's signature **before**
  loading the payload into any model. Reject with 401 on mismatch.
- Secrets used to verify signatures live in the secrets manager, never in
  config files, `.env` committed to git, or the codebase.
- Raw webhook payloads are NOT logged at INFO — they may contain PII or
  payment identifiers. Log a redacted summary; archive the raw payload only
  to a secured audit bucket.

### Audit trail

- Every state change (capture, disburse, debit, refund) writes a row to an
  append-only `financial_event` table with: actor, tenant, amount, currency,
  external reference, before-state, after-state, IP, user-agent, request_id.
- The audit table is enforced append-only at the database (Postgres row-level
  policy, trigger, or revoke UPDATE/DELETE at role level). Application-level
  "we just won't update it" is not enough.
- No payment instrument data (PAN, CVV, cardholder name) is ever stored.
  Only references to the provider's tokens.

**In Elimu360:** Money rails are M-Pesa (Daraja STK Push / C2B / B2C), Mambu
CBS (wallets, loans, repayments), and credit-partner REST APIs.

- M-Pesa callback handler validates Safaricom HMAC before parsing; upserts by
  `(checkout_request_id, mpesa_receipt_number)`.
- Mambu webhooks validate configured HMAC header. Secrets in AWS Secrets Manager.
- Financial events land in `audit.financial_event`, append-only Postgres policy.

---

## 3. Async-First — Never Block the HTTP Cycle

**Pattern.** Long-running, CPU-heavy, fan-out, or externally-dependent work
must not run inline in a request handler. It belongs in a background task
queue with named lanes, retry policy, and dead-letter handling.

### What must move to background workers

| Operation | Inline forbidden because |
|-----------|--------------------------|
| Document generation (PDF, Excel, CSV) | CPU-bound, blocks request worker |
| Outbound payment / financial API calls | External latency, must retry with backoff |
| SMS / push / email send | Network I/O, fan-out, bulk batching |
| ML inference | Model load + compute can take seconds |
| Search-index sync | I/O burst, batched for efficiency |
| Bulk operations over a threshold (e.g. >100 rows) | Always — batch into a task |

### Named queue discipline

Tasks declare a **queue**, never the default. Worker fleets are sized per
queue. Mixing latency-sensitive traffic (user-facing notifications) with
CPU-heavy traffic (PDFs) on the same queue starves users.

Recommended baseline lanes: `payments` (high-priority, low-volume),
`documents` (CPU-heavy), `notifications` (fan-out), `reports` (low-priority,
can backlog), `ml` (long-running).

### Failure handling

- Every task sets explicit retry config: which exceptions retry, exponential
  backoff with jitter, maximum attempts.
- Exhausted retries route to a **dead-letter store** for ops review — never
  silently logged and dropped.
- Task bodies are idempotent — the queue may execute them more than once.

**In Elimu360:** Async layer is Celery 5.4 + Redis broker, with the named
queues above. Failed tasks route to a separate Redis dead-letter queue;
Sentry pages on queue-depth threshold.

```python
@shared_task(
    queue="payments",
    autoretry_for=(RequestException,),
    retry_backoff=True,
    retry_jitter=True,
    max_retries=5,
)
def initiate_stk_push(school_id, payment_id, idempotency_key):
    ...
```

---

## 4. Stateless Workers — In-Process State is a Bug

**Pattern.** Web workers and background workers hold no per-request or
per-user state in memory between requests. Any worker must be able to serve
any request. State lives in the durable layer (database, cache, object store)
or on the request itself.

### Forbidden in any worker process

- Module-level dicts or caches keyed by user, tenant, or session
- Memoisation (e.g. Python `lru_cache`, JS module-level memo) on functions
  whose result depends on the current request
- Singleton "service" classes that accumulate state across requests
- Thread-locals or async-locals for anything beyond the current-request
  context, and that context must be cleared by middleware on exit

### Allowed

- Stateless memoisation on pure functions (deterministic, no request input)
- Distributed cache (Redis, Memcached) accessed by key
- Database, read replicas, object storage — the durable layer

**Acceptance test:** kill any worker mid-stream. The user's next request,
routed to a different worker, behaves identically. If state lived in-process,
this fails.

**In Elimu360:** Both Django ASGI workers (Gunicorn + Uvicorn on ECS Fargate)
and Celery workers are stateless. Sessions and channel state live in Redis;
business state lives in Aurora. Auto-scaling can add or remove workers at any
time.

---

## 5. Real-Time Channels — Authenticate and Scope at Connect Time

**Pattern.** WebSocket / SSE / long-lived stream connections bypass the
normal HTTP authorisation path. They must authenticate at connect time, scope
their subscriptions to the principal's tenant + role, and re-verify on every
inbound and outbound message.

### Required regardless of stack

- The handshake authenticates the principal **before** the socket is
  accepted. Anonymous connections are closed with a defined close code, not
  upgraded then dropped.
- Subscription group names embed the tenant. Naming a group by a non-tenant
  identifier (e.g. just a class ID) means a typo silently broadcasts across
  tenants.
- Subscribing to a group requires verifying (a) the principal belongs to that
  tenant and (b) the principal's role is allowed on that channel.
- Outbound messages on the channel layer carry the tenant. The consumer
  re-checks it against the connected principal before forwarding to the
  client.

**In Elimu360:** Django Channels 4.2, channel layer backed by Redis pub/sub.

- Every `AsyncJsonWebsocketConsumer.connect()` authenticates the JWT before
  `await self.accept()`. Anonymous → close 4001.
- Group naming convention: `school_{school_id}_class_{class_id}_announcements`.
- Channel messages include `school_id`; consumers verify match before
  `self.send_json(...)`.

---

## 6. Data Layer — Replicas, Pooling, and Time-Series

**Pattern.** A primary database is a precious resource. Reads that don't need
linearisability go to replicas. Connection pooling must match the application's
session expectations. Time-series data uses purpose-built partitioning.

### Read routing

- Reporting, analytics, search-index sync, and ML training pulls hit a **read
  replica**, never the primary. Batch reads on the primary saturate it during
  business-peak windows.
- Transactional writes and the consistency-sensitive read immediately after
  them go to the primary.

### Pool / proxy gotchas

- **Transaction-mode connection pooling** (e.g. PgBouncer transaction mode)
  breaks anything that depends on a long-lived session: session-scoped
  advisory locks, `SET LOCAL` outside an explicit transaction, raw prepared
  statements relying on session state, `LISTEN/NOTIFY`. ORMs are usually safe;
  raw drivers are not without configuration.
- Move pub/sub out of the application connection — use the in-memory store's
  pub/sub instead.

### Time-series data

- Time-series tables (events, metrics, attendance, audit streams) use a
  purpose-built engine (e.g. TimescaleDB hypertables, ClickHouse, BigQuery)
  with partitioning on tenant + time.
- Rollups (weekly, monthly) are computed by continuous-aggregate features of
  the time-series engine — not recomputed in application code.

**In Elimu360:** Aurora PostgreSQL (multi-AZ) primary + read replicas, accessed
via the Django `replica` database alias. PgBouncer in transaction mode.
Attendance and academic performance time-series in TimescaleDB hypertables
partitioned on `school_id` + event timestamp; weekly/monthly dashboards driven
by continuous aggregates.

---

## 7. Security Baseline — Beyond `code-standards.md` §"Security"

**Pattern.** Security at the architecture layer is about where secrets live,
how privileged files are served, when step-up auth is forced, and how
abuse-resistant the public surface is.

### Secrets

- All non-trivial secrets (DB credentials, payment-provider keys, third-party
  API keys, signing keys) live in a managed secret store. Not in config
  files, `.env` in git, container images, or build artefacts.
- Application code reads secrets from environment variables or a request-time
  fetch — never embeds them.
- A `.env` committed to git triggers credential rotation, not just a revert.

### Private file access

- Files containing PII, financial records, or other sensitive content live in
  private object storage. Access is via short-lived signed URLs (recommended
  TTL ≤ 15 minutes).
- The signing function verifies the requesting principal has tenant-scoped
  permission to the object's key prefix **before** signing.

### Step-up auth (MFA)

- Privileged roles (admin, finance, platform admin) require MFA. Token
  issuance for these roles returns an `mfa_required` step-up token, not a
  full access token, until MFA is satisfied.

### Rate limits

- Every endpoint declares a throttle. No silent unlimited endpoints.
- Suggested baseline: anonymous 60 req/min, authenticated 300 req/min, OTP
  and password endpoints 5 req/min per phone/email (separate scope),
  webhook callbacks unthrottled but protected via signature verification.

**In Elimu360:** Secrets in AWS Secrets Manager, injected as ECS task env
vars at launch. Private files in S3 served via CloudFront signed URLs with
≤15 min TTL. MFA enforced at JWT issue for `{ADMIN, FINANCE_OFFICER,
PLATFORM_ADMIN}`. DRF throttle classes mandatory on every endpoint;
M-Pesa callback unthrottled, validated via HMAC.

---

## 8. ML Models are Versioned Artefacts, Not Live Code

**Pattern.** When a model output affects a high-stakes decision (financial,
safety, child welfare, medical), the bar is higher than typical "ML in
production." Models are versioned, pinned, audited, and human-promoted.

### Required

- Every deployed model has an explicit version identifier. Application code
  pins a specific version. "Latest" endpoints are forbidden.
- Every inference call logs: model version, feature vector hash, tenant,
  output, latency, request id. This is the audit trail when a decision is
  challenged.
- Feature schemas exclude protected attributes (gender, religion, ethnicity,
  tribe, refugee status, disability) when the decision is regulated or
  consequential. Adding a feature requires a written fairness review.
- Retraining pipelines run on the read replica or an exported snapshot,
  never against the transactional primary.

### Forbidden

- Auto-promoting a freshly trained model to the production endpoint.
  Promotions require human approval and a Lock Document
  (`verification-and-reporting.md` §"Quality Gates").
- Calling the model from inside an HTTP handler. ML inference goes through
  the async layer (§3).
- Logging the full feature vector for production traffic — store a hash.

**In Elimu360:** Academic risk and credit scoring models are deployed on
AWS SageMaker, pinned by `model_name_v{N}`. Weekly credit-model retrain on
S3 exports + read replica. Promotions human-approved via Lock Document.
Inference dispatched through the `ml` Celery queue.

---

## 9. Mobile & Web Clients — Offline, Consent, Push

**Pattern.** Cross-platform clients need three things called out at the
architecture level: which views work offline, how user consent is collected
and revoked, and how push payloads handle PII.

### Offline behaviour

- Views that the architecture marks as offline-capable function read-only
  without network. Local store is the source of truth between syncs.
- Writes (data entry, state mutations) queue locally with a client-generated
  idempotency key and replay on reconnect. The server accepts the same key
  + same payload as a no-op return of the prior response.
- Conflict policy is explicit per entity (e.g. server-wins for grades,
  last-writer-wins with a tolerance window for attendance).

### Consent

- First-run onboarding collects **explicit, granular** consent for each
  channel and processing purpose (SMS, push, AI processing, third-party
  contact). Stored with timestamp and IP.
- Withdrawal is reachable from in-app settings and takes effect within a
  bounded SLO (e.g. 60 seconds) — purging subscriptions, push topics,
  and contact opt-ins via the async layer.

### Push notifications

- Payloads contain **no PII or sensitive amounts**. Use generic templates
  ("New grade posted", "Payment due") and require app open + authentication
  before details are visible.

**In Elimu360:** React Native mobile app caches gradebook and attendance in
local SQLite; refreshes on foreground or push trigger. Writes queue with
UUIDv4 idempotency keys. Conflict policy: server-wins (grades),
last-write-wins with 30s window (attendance). Consent stored in
`users.UserConsent`. FCM payloads are non-sensitive templates only.

---

## 10. Audit Logging — Tamper-Evident

**Pattern.** Privileged actions, financial state changes, sensitive PII
access, and bulk exports leave a tamper-evident audit trail. The trail
itself is verifiable — not just durable.

### What gets logged

- Every administrative action (create / update / delete on users, tenants,
  subscriptions, pricing, configuration)
- Every financial state change (see §2)
- Every read of a sensitive user record by anyone other than the user or a
  pre-authorised guardian
- Every bulk export (document download, spreadsheet export, API bulk fetch)

### Schema rules

- Audit rows are append-only — enforced at the database, not just at the
  application.
- Each row stores a hash of the previous row (within a tenant partition),
  giving a chain that's verifiable end-to-end. Chain breaks are detected by
  a scheduled integrity check that pages ops.
- Audit writes for high-stakes events happen in the **same transaction** as
  the change being audited. Never fire-and-forget through the async layer
  for these — broker loss could erase the record.

**In Elimu360:** `audit_log` table in Postgres, append-only via row-level
policy. Each row stores `prev_row_hash` (SHA-256, partitioned by school_id).
Nightly Celery integrity check pages Sentry on chain mismatch. Financial and
admin audit writes share the transaction with the underlying change.

---

## 11. Deployment & Operations

**Pattern.** Production runs on a managed orchestrator behind a load balancer.
Auto-scaling reacts to the right signal (queue depth as well as CPU).
Migrations are gated by a separate step. Rollback is to a previous artefact,
not via reverse-migration.

### Required

- Compute runs on a managed orchestrator (container service, serverless
  functions, managed Kubernetes). No bare VMs hand-managed by an engineer.
- Auto-scaling uses **leading indicators**, not just request count — queue
  depth, P95 latency, and CPU together. Many spikes are queue-driven
  (notification fan-out, document generation).
- New images pass: unit tests, integration tests, container security scan,
  and a smoke test against staging before promotion.
- Schema migrations run as a **separate task** before the rolling update of
  application workers. Never embedded in the container entrypoint — a slow
  migration would block worker boot and cascade into outage.
- Rollback is via the orchestrator (previous task definition / image tag),
  not via reverse-migration. Destructive schema changes often don't have a
  reverse migration.

**In Elimu360:** All services on AWS ECS Fargate behind ALB. Auto-scaling
target-tracks CPU + Celery queue depth. CI runs unit + integration + Trivy
scan; staging smoke test before prod promotion. Migrations run as a
dedicated ECS task in the deploy pipeline. Rollback = ECS service update to
the previous task definition.

---

## 12. What Counts as "Done" for a PR

In addition to `verification-and-reporting.md` §"Quality Gates":

- [ ] Tenancy enforcement verified — §1 checklist passed
- [ ] If touching money: idempotency key + signature verification + audit row — §2
- [ ] If introducing long-running work: lives in a named background queue — §3
- [ ] If touching state in a worker: state confirmed in durable layer, not memory — §4
- [ ] If touching a real-time channel: auth at connect + tenant-scoped group — §5
- [ ] No new connection-mode-incompatible patterns introduced — §6
- [ ] Secrets in the secret manager, no plaintext in repo — §7
- [ ] If shipping a new model version: pinned, fairness reviewed, human-promoted — §8
- [ ] If touching sensitive user data: consent path verified, audit row written — §10
- [ ] Migrations gated separately from app rollout; rollback path identified — §11
- [ ] Error monitoring release tag updated, performance trace verified on staging

If any item is missing, the PR is not ready — regardless of test pass count.

---

## Cross-References

- Coding standards that apply everywhere: `code-standards.md`
- Behavioural rules: `core-behaviors.md`
- Verification + status reporting: `verification-and-reporting.md`
- When in doubt about tenancy, money, or sensitive user data, escalate to the
  human BEFORE coding (per `core-behaviors.md` §10 Overconfidence Prevention).
