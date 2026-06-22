# Integration Platform — Technical Project Plan

> **Goal:** A multi-tenant integration/automation platform (an iPaaS, in the spirit of n8n / Zapier / Workato) where users build connectors, wire them into scheduled or event-triggered **Flows** made of typed **Actions**, with IAM/RBAC, per-flow logs, and an AI assistant that can author connectors/flows. Target: a working platform by **end of 2026**.
>
> **Document status:** Planning only — no code yet. Every technology decision below states **what / how / why (incl. rejected alternatives) / cost**. Dollar figures are approximate 2026 USD at a dev/hobby tier vs. a small-production tier.

## Assumptions (correct any of these and I'll revise)

1. **Solo developer / very small team**, building **part-to-full time** over the **~6.5 months** from today (2026-06-19) to end of 2026. This is primarily a **learning project** that you also want to be genuinely functional.
2. There is a real tension to manage: you *want* to learn the heavy distributed stack (microservices, queues, autoscaling, k8s) and designed it into your HLD — but a solo dev in 6 months must ship. **The plan resolves this by sequencing: a modular monolith + one worker first, with clean seams to split later, and "learn the heavy thing" scheduled as explicit milestones rather than launch blockers.**
3. **Default cloud: AWS** — largest managed-service ecosystem and best learning ROI. GCP/Azure are noted where relevant; the architecture is portable.
4. **AI layer uses Claude (Anthropic)** via the official SDK. Pricing in this doc uses real 2026 per-MTok rates.

---

## 1. Review of Your High-Level Design

Your HLD is solid — the instincts (gateway, control-plane vs execution-plane split, two data shapes, durable queue with fan-out, Redis, analytics) are right for this class of system. Most adjustments below are about **sequencing for a solo 6-month build**, not the destination. Verdicts: ✅ keep as-is · 🟡 keep the idea, change the *how/when* · 🔴 rethink before building.

| Component | You proposed | Verdict | Recommendation (short) |
|-----------|--------------|---------|------------------------|
| **API Gateway (HLD #1)** | An API gateway fronting incoming requests, doing rate-limiting, logging, and IAM. | 🟡 Adjust | Keep the concept, but DON'T put end-user authn/authz INTO the gateway. Use AWS API Gateway (HTTP API, ~$1/M requests) OR an Application Load Balancer + an app-layer middleware. Let the gateway do TLS termination, coarse rate-limiting (per-IP/per-key), request logging, and routing. Do tenant-scoped RBAC inside the app where you have org/role context. For the dev tier, just run your framework's router (FastAPI/Express) behind one ALB and add rate-limiting via Redis — skip a dedicated gateway product until you actually split services. |
| **2-Microservice split: static-data vs execution (HLD #2)** | Two services: (a) static-data/config CRUD (flows, integrations, login, access grants); (b) execution (running flows, inbound/outbound requests). | 🔴 Reconsider | Start as a MODULAR MONOLITH with clear internal module boundaries (auth, connectors, flows, execution, ai), deployed as ONE service, plus ONE separate async worker process for flow execution. The control-plane/data-plane instinct is correct — so enforce it as a code boundary now and as a network boundary later. The seam that actually matters first is synchronous-API vs long-running-worker (so a slow flow can't starve your HTTP threads), NOT static-vs-execution. Plan the split for when you have load/team to justify it; document the module interfaces so the later extraction is mechanical. |
| **SQL + NoSQL two-database split (HLD #3 / NFR)** | SQL for fixed-schema data (Integrations, IAM, Credentials); NoSQL for flexible/changing data (Requests, Actions). | 🟡 Adjust | Use ONE Postgres (AWS RDS/Aurora Serverless v2) as the system of record for everything: IAM, orgs, connectors, flows, credentials-metadata. Store the flexible/variable parts (request configs, action input/output schemas, connector definitions, flow DAGs) as JSONB columns inside Postgres. Add a second store only where access patterns genuinely diverge: execution LOGS and analytics events, which are high-volume append-only time-series — those are the real NoSQL/columnar candidates (DynamoDB, or just Postgres + partitioning early). Defer the second DB until logs volume forces it. |
| **Servers, load balancers, autoscaling (HLD #4)** | Multiple hosts sized to scale, with load balancers and auto-scaling. | 🟡 Adjust | For dev/learning: containerize and run on AWS ECS Fargate (or App Runner) behind one ALB — you get horizontal scaling and rolling deploys without managing nodes (~$15-40/mo dev tier for 1-2 small tasks + ALB ~$18/mo). Defer Kubernetes/EKS: it's the single biggest time-sink and EKS control plane alone is ~$73/mo before any workload. Autoscaling: turn on ECS target-tracking on CPU/queue-depth once you have a worker — but a fixed 1-2 task baseline is fine to ship. Treat 'learn k8s' as an explicit Phase-2 milestone, not a launch dependency. |
| **Queue + streaming fan-out for incoming requests (HLD #5 / NFR)** | Queue incoming requests and stream/broadcast to every active flow subscribed to that input. | ✅ Keep | Keep it — this is the strongest part of the HLD and the right pattern. Concretely: webhook ingress writes the event durably first (Postgres outbox or direct), then publishes. Use the topic/fan-out shape: one event must reach N subscribed flows. On AWS, EventBridge or SNS->SQS (one SQS queue per consumer type) gives true fan-out cheaply; a single SQS queue alone does NOT fan out. For the learning-heavy path, Kafka/Redpanda or AWS MSK teaches real streaming + consumer groups + replay, but adds ops weight — start with SNS+SQS (~$0.40/M + $0.50/M msgs, near-free at dev volume) and graduate to Kafka as an explicit learning milestone. Crucially: workers must be idempotent and the queue must be durable so a redeploy/crash doesn't lose or double-fire flows. |
| **Redis cache (HLD #6)** | Redis for frequently accessed data. | 🟡 Adjust | Add Redis (AWS ElastiCache, ~$12-15/mo dev tier t4g.micro, or Upstash serverless ~free-to-cheap at low volume) but be specific about WHAT it does, because 'cache hot data' is the least valuable use here. Highest-value uses for this platform: (1) rate-limiting counters for the webhook gateway, (2) idempotency keys / dedup for incoming requests, (3) distributed locks for scheduled-flow leader election so a cron doesn't double-fire across workers, (4) ephemeral session/token storage. Generic read-through caching of config is premature until Postgres is actually a bottleneck. |
| **Analytics per user/org (HLD #7)** | Store analytics per user/organization. | 🟡 Adjust | Don't build an analytics subsystem in v1. Emit structured execution events (flow_started, action_completed, request_failed, tokens_used) to the same append-only log store, and compute per-user/org metrics with simple aggregate queries on demand. Reach for a dedicated analytics store (ClickHouse, or AWS Athena over S3 event files) ONLY when log volume makes Postgres aggregation slow. Capture the event schema now (tenant_id, flow_id, ts, type, payload) so you can backfill into a warehouse later without re-instrumenting. |
| **MISSING: Secrets & credential management** | Not addressed — HLD mentions a Credentials table but no secret handling. | 🔴 Reconsider | Treat this as a first-class, non-deferrable component. Store only credential METADATA (provider, scopes, owner) in Postgres; store the actual secrets (API keys, OAuth tokens, refresh tokens) in AWS Secrets Manager (~$0.40/secret/mo + API calls) or KMS-encrypted columns with envelope encryption. Build an OAuth2 broker early — most real connectors (Slack, Teams, ServiceNow) need OAuth with token refresh, not static keys. Never log secrets; redact in execution logs. |
| **MISSING: Durable workflow execution engine** | HLD treats 'execution service' as a single box; no story for multi-step durability, retries, or resume-after-crash. | 🔴 Reconsider | Decide explicitly how a Flow (sequence of Actions, #4) executes durably. Two viable paths: (A) DIY state machine — persist each action's status/output to Postgres, drive transitions via the queue, make each step idempotent and retryable (high learning value, more code). (B) Adopt a durable-execution engine — Temporal (self-host or Temporal Cloud) or AWS Step Functions — which gives retries, timeouts, and crash-resume for free. For a learning project, build (A) for v1 to understand the problem, and evaluate Temporal as a Phase-2 learning milestone. |
| **MISSING: Idempotency & delivery semantics** | Not addressed; NFR implies at-least-once queue delivery but no dedup story. | 🔴 Reconsider | Make idempotency a platform-wide rule: every incoming webhook gets a dedup key (provider event id or hash), stored in Redis/Postgres so replays don't re-trigger flows; every outgoing request uses an idempotency key where the third party supports one. Queues (SQS, Kafka) are at-least-once, so workers WILL see duplicates — design every action to be safely re-runnable. |
| **MISSING: Connector / plugin model (the core extensibility requirement)** | Requirement #1 wants UI-authored, user-defined connectors, but the HLD has no design for how a connector is defined, stored, validated, or executed safely. | 🔴 Reconsider | Design a DECLARATIVE connector spec (JSON/YAML: base URL, auth type, endpoints, request templating, input/output schema) stored as JSONB in Postgres and interpreted by a generic HTTP execution engine — do NOT let users upload arbitrary code in v1. This makes 'add an integration from the UI' a data-entry problem, not a code-deploy problem, and it's what the AI agent (#6) generates. Validate specs against a schema before activation. Defer custom-code/sandboxed connectors (a serious security problem) to a later phase. |
| **MISSING: Frontend / client architecture** | Requirements demand heavy UI (UI-authored connectors #1, client-configurable requests #2, flow builder #4, AI chat #6, Home dashboard #8) but the HLD is backend-only. | 🟡 Adjust | Plan the frontend explicitly: a React/Next.js (or SvelteKit) SPA. Highest-effort piece is the visual flow builder — use React Flow (open-source) for the node/edge canvas rather than building one. Dynamic connector/request config forms should be schema-driven (render from the same JSON schema the connector spec defines) so adding a connector needs zero new UI code. The AI chat is a streaming chat panel over the Messages API. Host on Vercel/Amplify/CloudFront+S3 (dev tier ~free-to-$20/mo). |
| **MISSING: Scheduler for time-triggered flows** | Requirement #3 says flows can be schedule-triggered, but the HLD has no scheduling component. | 🟡 Adjust | Add a scheduler that turns cron definitions into queued flow-trigger events feeding the SAME execution path as webhooks. Use a single leader (Redis lock or DB advisory lock) to enqueue due jobs so multiple workers don't double-fire. On AWS, EventBridge Scheduler can also drive this. Keep schedule defs in Postgres alongside the flow. |
| **AI Agent layer (requirement #6) — implied in HLD execution service** | An AI assistant using Claude to build connectors/flows/actions via chat, parse requests into tool calls, and analyze flow outputs. | 🟡 Adjust | Build it on the Anthropic Messages API with tool use (function calling) using the official SDK; the agent's tools are the SAME internal APIs your UI calls (create_connector, create_flow, add_action), so it generates the declarative specs above — no separate AI code path. Model tiering: Haiku 4.5 ($1/$5 per 1M) for intent classification/routing, Sonnet 4.6 ($3/$15, 1M ctx) for the main connector/flow-building agent, Opus 4.8 ($5/$25) only for the hardest reasoning. Use prompt caching (cache reads ~0.1x input) for the large system prompt + connector-schema context, and the Batch API (50% off) for non-interactive 'analyze my flows' jobs. Gate AI-generated specs behind the same validation + human confirmation before activation. |

### Rationale per decision

**API Gateway (HLD #1) — 🟡 Adjust**  
An API gateway is the right place for cross-cutting transport concerns, but multi-tenant RBAC (who can edit which org's flows) needs domain context the gateway lacks, so it ends up duplicated. Two distinct ingress paths matter here: the management API (authenticated users in the UI) and the webhook ingress (untrusted third parties hitting your incoming-request endpoints). The latter needs per-tenant-token rate limits and abuse protection more than the former. Rejected: Kong/Apigee — operationally heavy and overkill for a solo 6-month build. AWS-managed API Gateway is a good learning vehicle and removes you from running gateway infra.

**2-Microservice split: static-data vs execution (HLD #2) — 🔴 Reconsider**  
Two deployable services on day one doubles your CI/CD, observability, local-dev, and inter-service-contract overhead before you have a single working flow — a heavy tax on a solo ~6-month timeline. But the underlying separation (config plane vs runtime plane) is genuinely sound and worth designing toward, so capturing it as enforced module boundaries gives you the learning and the future optionality without the early ops cost. The real first cut you DO need is a separate worker (queue consumer) so flow execution runs out-of-band from the request path. Modular-monolith-first → extract-later is the standard pragmatic path and matches the stated learning-vs-shipping tension.

**SQL + NoSQL two-database split (HLD #3 / NFR) — 🟡 Adjust**  
Postgres JSONB gives you schema flexibility for connector/request/action definitions WITHOUT a second database's operational burden, and you keep transactions and joins across flows/users/credentials — which you will need (e.g., 'delete an org, cascade its flows and creds'). Two databases on day one means two backup strategies, two consistency models, and cross-store referential integrity you hand-maintain. Rejected as primary store: MongoDB/DynamoDB for the core domain — you'd lose multi-entity transactions that IAM and billing-like cascades require. The legitimate NoSQL win is the write-heavy log/analytics path, so spend the 'learn NoSQL' budget there, not on the config data.

**Servers, load balancers, autoscaling (HLD #4) — 🟡 Adjust**  
Multi-host + LB + autoscaling is the correct production shape and a good thing to learn, but provisioning it before there's a workload to scale burns weeks of your 6.5-month window on infra yak-shaving. Fargate gives you 80% of the distributed-systems learning (containers, load balancing, scaling policies, health checks, rolling deploys) at a fraction of the operational cost of self-managed k8s. Rejected for launch: EKS/self-managed k8s (steep learning curve + idle control-plane cost), raw EC2 (you'd reinvent orchestration). Tie autoscaling to queue depth, not just CPU, since execution is the bursty dimension.

**Queue + streaming fan-out for incoming requests (HLD #5 / NFR) — ✅ Keep**  
Decoupling ingress from execution via a durable queue is exactly how iPaaS platforms absorb bursts and isolate failures, and fan-out maps directly to your 'multiple flows subscribed to one input' requirement. The distinction worth internalizing: a work-queue (SQS, competing consumers, one message one worker) is for distributing a flow's actions; pub/sub fan-out (SNS/EventBridge/Kafka) is for one event triggering many flows — you need BOTH and they are not interchangeable. Rejected as the only mechanism: a bare single SQS queue (no fan-out) and in-memory pub/sub (lost on restart, no durability).

**Redis cache (HLD #6) — 🟡 Adjust**  
Redis is justified, but framing it as a read-cache understates its real role in a distributed iPaaS — the coordination primitives (rate limits, idempotency, locks) are what keep correctness under concurrency, and you need them earlier than you need a cache. Caching config data to relieve Postgres is a real optimization but introduces invalidation bugs, so add it reactively when profiling shows a hot path. Rejected: Memcached (no data structures, no persistence — Redis's sorted sets/locks are exactly what you want); in-process cache only (breaks the moment you run >1 worker).

**Analytics per user/org (HLD #7) — 🟡 Adjust**  
Per-tenant analytics is a real product need (it feeds the Home dashboard and the AI 'analyze my flows' feature), but a separate analytics pipeline is over-engineering for a solo 6-month build with no traffic to analyze yet. The cheap, correct move is to instrument events well from day one and aggregate lazily — instrumentation is the irreversible part, the storage choice is reversible. Rejected for v1: standing up ClickHouse/warehouse + ETL (operational weight with no data to justify it). The Home dashboard's 'running vs draft flows' is just a query on flow state, not analytics.

**MISSING: Secrets & credential management — 🔴 Reconsider**  
An iPaaS is fundamentally a custody-of-other-people's-credentials business — getting this wrong is the catastrophic failure mode, and it's far cheaper to design in than to retrofit after secrets have leaked into logs/DB. OAuth token refresh is load-bearing for the connector requirement (#1, #2) and is easy to underestimate. This is also high learning ROI (KMS, envelope encryption, OAuth flows). Rejected: plaintext or app-symmetric-key-in-env (one repo leak compromises every tenant).

**MISSING: Durable workflow execution engine — 🔴 Reconsider**  
A flow is a multi-step distributed transaction across third parties — partial failure (action 3 of 5 fails, action 2 already sent an email) is the normal case, not the exception, and 'just run the steps in a loop' silently loses or double-executes work on any crash or redeploy. This is the hardest correctness problem in the whole platform and the HLD currently has no answer for it. Persisting per-action state + idempotency + retry-with-backoff is the irreducible minimum. Rejected: pure in-memory orchestration (no crash recovery); Step Functions as primary (less portable, AWS-locked, weaker fit for user-defined dynamic DAGs than Temporal/DIY).

**MISSING: Idempotency & delivery semantics — 🔴 Reconsider**  
Once you choose durable queues with at-least-once delivery (the right choice), duplicate delivery is guaranteed, not hypothetical — without dedup keys a single retry sends a customer two emails or files two ServiceNow tickets. This is cheap to bake in early and very expensive to bolt on after the execution model is built around the assumption of exactly-once. Directly protects the Action correctness in requirement #4.

**MISSING: Connector / plugin model (the core extensibility requirement) — 🔴 Reconsider**  
The 'every user authors their own connector from the UI' requirement is the platform's defining feature, and whether a connector is declarative-data or arbitrary-code is the single most consequential architecture decision — it dictates the DB model, the execution engine, the security boundary, and what the AI agent emits. A declarative spec interpreted by one engine is dramatically safer and simpler than per-user code execution, and it directly enables both the AI builder and the request-config requirement (#2). Rejected for v1: user-supplied code connectors (arbitrary code execution = a sandboxing/security project on its own; only revisit with strong isolation like microVMs/WASM).

**MISSING: Frontend / client architecture — 🟡 Adjust**  
For an iPaaS, the UI is not a thin layer — it IS the product surface for four of your requirements, and the flow builder plus schema-driven config forms are substantial enough that omitting them from the plan would blow the timeline estimate. Schema-driven forms are the linchpin that makes the 'easy to add connectors from the UI' goal real, by reusing the connector spec for both execution and rendering. Rejected: hand-building a flow-graph editor (React Flow already solves it); bespoke per-connector forms (defeats the extensibility requirement).

**MISSING: Scheduler for time-triggered flows — 🟡 Adjust**  
Scheduled triggering is an explicit requirement and a classic distributed-systems trap: run the cron on every worker and every scheduled flow fires N times. Routing schedule triggers into the same queue/fan-out path as incoming requests keeps the execution engine uniform (one code path for 'something triggered this flow'), which simplifies the whole runtime. Cheap to add, but must be designed so it isn't an afterthought that double-executes.

**AI Agent layer (requirement #6) — implied in HLD execution service — 🟡 Adjust**  
Reusing the platform's own internal APIs as the agent's tools means the AI builds via the exact validated, declarative path a human uses — this is what makes the AI feature safe and tractable rather than a free-text-to-arbitrary-config risk. Model tiering plus prompt caching on the (large, stable) connector-schema context is where the real cost control lives, since that context is re-sent on every turn. Rejected: a single always-Opus pipeline (5-25x the cost for routing/classification with no quality gain); fine-tuning (unnecessary — tool use + cached context covers it). Human-in-the-loop confirmation is required because AI specs touch others' credentials and live integrations.

---

## 2. Domain Model & Data Architecture

The platform splits cleanly into **config/identity data** (low-volume, relational, transactional integrity matters) and **execution data** (high-volume, append-heavy, schema-flexible, time-bound). That split drives every datastore decision below. Net recommendation:

- **Postgres** (managed: AWS RDS/Aurora) — the system of record for identity, tenancy, connectors, connections, flow definitions, triggers/subscriptions. Anything you query relationally, join, or need ACID on.
- **A document store for per-run execution records** — `FlowRun` / `ActionRun` payloads.
- **A time-series/log store for raw logs + analytics rollups.**

I'll be specific about the document and log stores at the end rather than hand-waving "NoSQL."

---

### Entity-by-entity: what it is, where it lives, why

**Organization** — tenant boundary. **Postgres.** Root of multi-tenancy; every other row carries (or inherits) `org_id`. Relational, tiny volume, needs FK integrity. This is the partition key for row-level security.

**User** — a person, belongs to one or more orgs (membership is a join table, so users can be invited across orgs). **Postgres.** Auth identity, transactional, joined constantly.

**Role / Permission** — RBAC. **Postgres.** Model as: `Role` (per-org, e.g. Admin/Builder/Viewer), `Permission` (action+resource, e.g. `flow:execute`, `connector:author`), `RolePermission` (join), `Membership(user, org, role)`. Relational by nature; you will join `User → Membership → Role → Permission` on every authz check. Cache the resolved permission set per session in Redis. **Rejected:** storing roles as a JSON blob on the user — kills the ability to query "who can execute flows in org X" and makes permission changes non-atomic.

**Connector** (reusable integration *type* / template — possibly user-authored) — **this is the keystone of requirement #1 and #6b.** A connector is **declarative config, not code**, so it can be authored from the UI and by the AI agent. **Postgres for the row + metadata; the connector *spec* as a `jsonb` column** (or document-store reference for very large specs, but `jsonb` is correct here). The spec describes:
  - auth scheme (`none | api_key | bearer | oauth2 | basic`) and where the secret goes (header/query/body),
  - base URL, default headers,
  - a set of **operation templates** (named requests: method, path template, header template, body template using a templating syntax like `{{input.x}}`),
  - typed input/output JSON Schemas per operation.

Because it's data, "add a new integration from the client" = insert a connector row. A generic **runtime interpreter** in the execution service reads the spec and performs the HTTP call — no per-connector code is deployed. **Why Postgres+jsonb over a pure document store:** you need relational links (`connector.org_id`, `connector.author_id`, visibility = private/org/public), versioning, and you query/list connectors constantly. `jsonb` gives schema flexibility *and* GIN-indexable querying inside the spec. **Rejected:** connectors as deployed code/plugins (Pythonic n8n nodes) — defeats UI-authoring and the AI agent goal, and turns every new integration into a deploy.

**Connection / Credential** (authenticated instance of a connector for a user/org) — **Postgres for the record, secrets in a dedicated secrets manager (AWS Secrets Manager / KMS-encrypted), NOT in the row.** The Postgres row holds `connection_id, org_id, connector_id, owner_user_id, display_name, status, secret_ref` where `secret_ref` points at the secret manager entry; OAuth tokens/refresh tokens live encrypted there. **Why:** credentials are relational (belong to a connector + org), but raw secrets must never sit in app DB plaintext or app logs. **Rejected:** encrypting secrets into a jsonb column with an app-held key — workable but worse key rotation/audit story than Secrets Manager + KMS, which is also better for the learning goal.

**Trigger** (schedule OR incoming webhook) — **Postgres.** A flow has exactly one trigger.
  - **Schedule trigger:** stores a cron expression; a scheduler (EventBridge Scheduler, or a cron worker) enqueues a run when it fires.
  - **Webhook/incoming trigger:** the platform exposes an endpoint keyed by a stable `trigger_id` / `ingest_key`. The Trigger row holds the endpoint config + a reference to the **Subscription** model below.
  Relational because triggers are looked up by key on every inbound request and join to flow + org.

**Subscription** (the fan-out model — **requirement (a)**) — **Postgres, lookup-cached in Redis.** This is how "incoming request → broadcast to all active flows subscribed to that input" is modeled without hard-coding. Define a **Channel/Topic** = a named input stream (e.g. `slack:message`, or a specific inbound webhook endpoint). Active flows declare a `Subscription(channel_id, flow_id, flow_version, filter_predicate, active)`. On an inbound request:
  1. Gateway resolves the endpoint → `channel_id`.
  2. Enqueue the event onto a queue/stream once.
  3. A dispatcher reads active `Subscription` rows for that channel (cached set in Redis), applies each `filter_predicate`, and fans out one `FlowRun` enqueue per matching flow.
  
  So fan-out = a query over a relational subscription table, not a coded broadcast. Subscriptions are relational (join flow↔channel, must be transactionally toggled active/inactive), and the hot read ("which flows listen to channel X") is cached. **Rejected:** embedding subscriber lists inside the channel document — race-prone on concurrent activate/deactivate and hard to query "what does flow Y subscribe to."

**Flow** (definition, versioned) — **Postgres for the row; the action graph as `jsonb` (or a document reference).** A Flow row: `flow_id, org_id, owner_user_id, name, status (draft|active|disabled), current_version_id, trigger_id`. **Versioning:** a separate `FlowVersion(flow_version_id, flow_id, version_no, graph, created_by, created_at)` table where `graph` is the serialized action DAG. Editing a live flow creates a new version; runs pin to the version they started on (so a mid-flight edit can't corrupt an in-progress run — critical for the AI agent editing flows). **Why Postgres:** flows are listed, permission-checked, joined to triggers/owners, and versioned — all relational. The mutable, nested graph is the only flexible part → `jsonb`.

**Action / Step storage — how the graph is stored (requirement (c)):** store the whole graph as a **single `jsonb` document on `FlowVersion`**, NOT as one row per step. The graph is:
```
{
  "nodes": [
    {"id":"n1","type":"trigger.webhook","config":{...}},
    {"id":"n2","type":"action.dataTransform","config":{...},"inputSchema":{...},"outputSchema":{...}},
    {"id":"n3","type":"action.email","config":{...}}
  ],
  "edges": [{"from":"n1","to":"n2"},{"from":"n2","to":"n3"}]
}
```
Start with ~3 node `type`s (Email, DataTransform, HTTP/Request) and extend the `type` registry — types are looked up in a small **ActionType** catalog (Postgres) that defines the typed input/output schema and which runtime handler executes it. Storing the graph as one document keeps reads/edits atomic (load whole flow, save whole flow) and matches how the UI canvas thinks. **Rejected:** normalized `Step` + `Edge` tables — more joins, painful atomic versioning, and you'd reassemble the graph on every load for zero query benefit (you never query "all step n2s across flows").

**FlowRun** (execution instance) — **Document store.** `flow_run_id, flow_id, flow_version_id, org_id, trigger_event_ref, status (queued|running|succeeded|failed|cancelled), started_at, finished_at, trigger_payload, final_output`. High-volume, append-then-update, schema varies by trigger/flow, never JOINed heavily — queried by `flow_id`/`org_id`/time. This is the textbook document-store case. **Why not Postgres:** write volume and unbounded payload shapes; you'd be storing blobs in Postgres anyway and paying for VACUUM on a write-heavy table.

**ActionRun** (per-step execution record) — **Document store**, either embedded in the `FlowRun` document (if steps-per-run is bounded and you always read them together — usually true) or a sibling collection keyed by `flow_run_id`. Holds `step_id, type, status, input, output, error, started_at, finished_at, duration_ms, attempt`. Embedding is the default since you read a run's steps as a unit on the Logs screen. High-volume, flexible payloads, no cross-run joins.

**Request config (incoming + outgoing) — requirement #2:** **not a separate top-level entity — it lives as config inside Connector operation templates and/or inside a flow node's `config`.** Outgoing request config = the operation template on the connector (URL/headers/auth/body templating) resolved at run time against step input. Incoming request config = the Trigger/Channel endpoint definition (path, expected auth, payload schema). Per-execution *resolved* requests/responses (the actual URL hit, status, response body) are recorded in the `ActionRun` document. **Postgres** for the reusable config; **document store** for the per-execution instance.

**Log / Event** (per-flow execution logs, #7) — **Time-series / log store, not Postgres.** Structured log lines (`flow_run_id, step_id, ts, level, message, attrs`) are append-only, extremely high-volume, queried by time-range and `flow_run_id`. Use a log/observability store (see options below). Keep a pointer (`log_stream_ref`) on the `FlowRun` document so the UI can fetch the run's logs. **Rejected:** logs in Postgres — write amplification and table bloat; relational queries add nothing for log search.

**Analytics rollups** (#7 NFR) — **Pre-aggregated rollups in Postgres (or a columnar/OLAP store at larger scale).** Don't compute dashboards by scanning raw runs/logs. A background job rolls up per-(org, flow, day): run counts, success/fail, avg duration, etc., into a compact `analytics_daily` table. Small, relational, fast to query for Home/dashboards. At real scale, ship raw events to a columnar warehouse (ClickHouse / Redshift / BigQuery); for this project, a Postgres rollup table is the right first move. **Rejected:** querying raw `FlowRun`/log data live for the dashboard — too slow and expensive per page load.

---

### Concrete datastore picks (decisive)

- **Relational/config/identity:** **PostgreSQL via AWS RDS** (single instance) for dev; **Aurora PostgreSQL** for small-prod. *Rejected:* DynamoDB for this tier — you'd fight its lack of joins for IAM/connectors; *rejected:* self-managed Postgres on EC2 — needless ops for a learning project. Use `jsonb` heavily for connector specs and flow graphs.
- **Document store (FlowRun/ActionRun):** **MongoDB (Atlas)** — best learning ROI, rich queries on flexible run documents, easy local dev. *Rejected:* DynamoDB (awkward query model, painful for ad-hoc run inspection), *rejected:* shoving runs into Postgres jsonb (write-heavy bloat). Acceptable alt if you want one-fewer-vendor: Postgres `jsonb` runs early, migrate to Mongo when volume bites — call this out as a sequencing choice.
- **Logs:** **OpenSearch (AWS) or Loki/Grafana** for searchable execution logs; *rejected:* logs in the primary DB. For dev, write structured logs to stdout → CloudWatch Logs, add OpenSearch when search matters.
- **Hot lookups:** **Redis (ElastiCache)** for resolved permission sets, active-subscription sets per channel, connector spec cache.

---

### Compact entity-relationship sketch

```
Organization 1───* Membership *───1 User
Membership *───1 Role
Role *───* Permission            (via RolePermission)

Organization 1───* Connector         (Connector.spec = jsonb: auth + operation templates + IO schemas; visibility: private|org|public)
User        1───* Connector         (author_id; user-authored connectors)
Connector   1───* Connection         (Connection.secret_ref → Secrets Manager; never store raw secret)
Organization 1───* Connection

Organization 1───* Flow
Flow         1───1 Trigger           (schedule cron | incoming webhook endpoint)
Flow         1───* FlowVersion       (FlowVersion.graph = jsonb DAG of nodes+edges)
ActionType   *catalog* ──< node.type (referenced by graph nodes; defines IO schema + handler)

Channel/Topic 1───* Subscription *───1 Flow     (fan-out: inbound event → query active subs → enqueue 1 FlowRun each)
Trigger(webhook) ──1 Channel

Flow         1───* FlowRun           [DOCUMENT STORE]   (pinned to a flow_version_id)
FlowRun      1───* ActionRun         [DOCUMENT STORE]   (embedded; per-step input/output/error)
FlowRun      1───* LogEvent          [LOG/TS STORE]     (linked via log_stream_ref)
(Org, Flow, day) ── AnalyticsRollup  [POSTGRES rollup]  (background-aggregated)

Legend: no marker = PostgreSQL. [DOCUMENT STORE] = MongoDB. [LOG/TS STORE] = OpenSearch/Loki.
```

**HLD verdict on the data layer:** the "SQL for fixed schema, NoSQL for flexible" split (HLD #3) is **correct — keep it**, with two **adjustments**: (1) it's really a *three*-way split — add a dedicated log/time-series store rather than dumping logs in the document DB; (2) connector specs and flow graphs belong in Postgres `jsonb`, not the document store, because they're relationally linked and versioned config, not high-volume execution data. The analytics requirement (#7/HLD #7) should be **pre-aggregated rollups**, not live scans of run/log data.

---

## 3. Technology Decisions by Domain

Each subsection states what to use, how to use it here, why (with rejected alternatives), and cost.

### 3.1 API Gateway & Edge

This layer is the single front door for two distinct traffic classes: **client API/UI calls** (authenticated dashboard/CRUD against the config service) and **inbound webhooks** (third-party events hitting per-flow ingest endpoints, requirement #2-incoming and the fan-out in NFR). Both need TLS termination, authn/authz, per-tenant rate limiting, structured request logging, and routing to the two services from the domain model (config vs execution).

**WHAT / recommendation:** **AWS Application Load Balancer (ALB) + AWS WAF**, fronting your service(s), with **app-layer middleware** doing authz, tenant rate-limit accounting, and webhook routing. Skip Amazon API Gateway as the primary edge.

**HOW, concretely:**
- **TLS:** ACM-issued cert on the ALB (free certs, auto-renew). HTTPS only; HTTP→HTTPS redirect at the listener.
- **Routing:** ALB listener rules split by path — `/api/*` → config service, `/ingest/*` → execution service (or one modular-monolith target early; same rules survive the later split, so this choice is forward-compatible).
- **Authn/authz at the edge vs app:** terminate TLS and do coarse WAF filtering at the edge; do **JWT/session validation and RBAC in app middleware**, because authz needs the resolved per-tenant permission set (cached in Redis per the domain model) — too tenant-aware to live in a generic gateway. WAF blocks the obvious bad traffic (IP rate rules, common exploits, oversized bodies) before it touches compute.
- **Dynamic webhook provisioning:** do **NOT** create a new gateway route per flow. Expose one wildcard route `POST /ingest/{ingest_key}`; the execution service resolves `ingest_key` → Trigger → Channel (Postgres, Redis-cached), authenticates the inbound request per the connector's incoming auth config, enqueues the event **once**, and the dispatcher fans out. New webhook = insert a Trigger row, zero infra change. This is the only model that scales to user-authored connectors.
- **Per-tenant rate limits:** enforce in **app middleware backed by Redis** (token-bucket keyed by `org_id`), because limits are per-tenant/per-plan and live in your data — a generic edge limiter only knows IPs. Use WAF/ALB rate rules as a crude **global DoS shield**, not the tenant quota.
- **Logging:** ALB access logs → S3; structured app request logs (method, path, `org_id`, `trigger_id`, latency, status) → stdout → CloudWatch, later OpenSearch (matches the domain-model log store).

**WHY / rejected alternatives:**
- **Amazon API Gateway (rejected as primary):** per-request pricing punishes high webhook volume, payload/timeout limits hurt long flow triggers, and its usage plans/API keys don't map to *your* `org_id` tenancy — you'd reimplement authz anyway. Fine for a quick managed HTTP endpoint, wrong as the system edge.
- **Kong / Traefik / NGINX self-hosted (rejected for a solo learner):** Kong/Traefik teach great patterns but add a stateful component to run, secure, and scale — pure ops tax against a 6.5-month solo timeline. Revisit Kong when you split services and want declarative gateway plugins.
- **ALB + WAF + app middleware (chosen):** managed, cheap, fixed-cost, integrates with ACM/CloudWatch/auto-scaling target groups, and keeps the tenant-aware logic (authz, rate limits, fan-out) in code you control and can test locally.

**HLD verdict (point #1):** **Keep it — adjust placement.** Rate-limiting/logging/IAM "at the gateway" is right conceptually, but split it: WAF/ALB do TLS, global rate shielding, and access logging; **per-tenant rate limiting and IAM/RBAC belong in app middleware**, not the edge box, because they depend on tenant data.

**COST (approximate 2026 USD/month):**
- **Dev/hobby:** ALB ~$16–22 (hourly + minimal LCU) + WAF ~$5–8 (web ACL + a few rules) + ACM $0 + CloudWatch logs a few $. **~$25–35/mo.** Cheaper still: skip ALB locally, run the app directly behind a single host/ngrok for webhook testing — **~$0**.
- **Small-prod:** ALB ~$25–45 (more LCUs from real traffic) + WAF ~$10–20 (more rules/requests) + access-log S3/CloudWatch ~$5–15. **~$45–80/mo.** Amazon API Gateway at, say, 5M webhook calls/mo would alone run ~$15–17 just in request fees plus data — another reason ALB's flat-ish cost wins as volume grows.

### 3.2 Compute & Service Architecture

**WHAT — Start as a modular monolith, not two microservices.** One deployable, two hard-bounded internal modules: a **control-plane** (CRUD/config: IAM, connectors, connections, flow definitions, subscriptions — HLD service "a") and an **execution-engine** (trigger dispatch, run orchestration, inbound/outbound request execution — HLD service "b"). Plus a separately-deployed **worker pool** for flow runs (queue consumers) and the **AI agent** as a module that talks to the Messages API. The control-plane/execution seam is enforced in code (separate modules, no shared mutable state, communication only via the queue and the shared Postgres/Mongo contracts) so it can be carved into two services later by changing a transport, not a rewrite.

**HLD verdict — adjust.** The "two microservices now" instinct is right about the *boundary* but wrong about the *deployment* for a solo dev with ~6.5 months. Two services on day one means two deploy pipelines, network auth between them, distributed tracing, and partial-failure handling — pure tax before you have a working flow. Keep the two boundaries as modules; split when a real driver appears (execution needs independent autoscaling from config CRUD — which it will, so design the seam now). The one thing that **is** separate from the start is the worker pool, because flow execution is bursty and must scale independently of the API — that is the genuinely different workload, not "config vs execution."

**Language/runtime — Node.js 22 + TypeScript with NestJS.** WHY: this workload is I/O-bound glue (HTTP fan-out to third parties, queue I/O, DB calls), where Node's async model shines and CPU work is minimal. TypeScript types map directly onto the typed Action input/output and connector JSON Schemas. NestJS gives module boundaries, DI, and queue/guard plumbing out of the box — it *structures* the control-plane/execution split for you. One language across API, workers, and the React UI lowers solo-dev load. **Rejected:** Go (better raw concurrency/footprint, but more boilerplate for this CRUD-heavy app and weaker JSON-Schema/templating ergonomics — revisit only if execution becomes CPU-bound); Python/FastAPI (great for the AI agent and fine otherwise, but weaker typing story for the action graph and you'd still want TS on the front end — the official Anthropic SDK exists for both, so this isn't a blocker either way).

**Containers + orchestration — Docker, on AWS ECS Fargate.** Two Fargate services: `api` (the monolith, behind an ALB) and `worker` (queue consumers, no public ingress). WHY ECS Fargate: serverless containers, no node management, native ALB + target-tracking autoscaling — this *is* HLD #4 (load balancers + auto-scaling) without the k8s tax. **Rejected:** Kubernetes/EKS (huge learning sink, ~$73/mo control plane before a single pod — defer until multi-service scale truly demands it); Fly.io/Render (excellent DX and cheaper to start, but ECS keeps you in the AWS ecosystem you're already learning for RDS/Secrets Manager/SQS). Autoscaling: ALB request-count target for `api`; SQS queue-depth target for `worker`.

**COST (approx. 2026 USD/mo).** Dev/hobby: 1 small Fargate task each for api+worker (~0.25 vCPU/0.5 GB) ≈ $18–30; ALB ≈ $18; total compute **~$40–60/mo** (excludes RDS/Mongo/Redis covered elsewhere). Small-prod: 2× api (0.5 vCPU/1 GB) + 2–4 burst workers + ALB ≈ **$120–220/mo**. AI agent calls are usage-based on top (Sonnet $3/$15 per 1M tokens for most agent work; near-zero at dev volumes).

### 3.3 Data Layer (SQL + NoSQL + more)

**Verdict on HLD #3:** The SQL-for-fixed-schema / NoSQL-for-flexible split is **correct — keep it**, with one adjustment: it's a *three-store* design. Config/identity → SQL; high-volume per-run execution → document store; logs → a dedicated log store. Don't dump logs into the document DB.

### SQL — PostgreSQL (system of record)
**What:** PostgreSQL 16. **AWS RDS for Postgres** (single instance) at dev tier; **Aurora PostgreSQL** at small-prod (for read replicas + storage autoscaling, tying into HLD #4).

**How:** Holds everything relational and ACID-sensitive — `Organization`, `User`, `Membership`, `Role/Permission/RolePermission`, `Connector` (+ spec as `jsonb`), `Connection` (with `secret_ref`, not raw secrets), `Trigger`, `Subscription`, `Channel`, `Flow`, `FlowVersion` (graph as `jsonb`), `ActionType` catalog, and `analytics_daily` rollups. Use `jsonb` + GIN indexes for connector specs and flow graphs — schema flexibility *with* queryability, which is exactly why these declarative-config pieces stay in Postgres rather than the document store.

**Why / rejected:** Postgres gives you joins (every authz check walks `User→Membership→Role→Permission`), FK integrity, transactions for activate/deactivate toggles, and `jsonb` for the flexible bits — one engine covers config + flexible-config. *Rejected: DynamoDB* — no joins, brutal for IAM and "list connectors in org X"; *Rejected: self-managed Postgres on EC2* — needless backup/patching ops for a learning solo dev (RDS gives you that managed-service learning anyway).

### Document store — MongoDB Atlas (execution records)
**What:** **MongoDB Atlas** for `FlowRun` and embedded `ActionRun` documents (per-run resolved request/response payloads live here).

**How:** Append-on-create, update-on-finish; query by `flow_id`/`org_id`/time for the Logs screen. Embed `ActionRun` steps in the `FlowRun` doc (read as a unit). Index `{org_id, flow_id, started_at}`.

**Why / rejected:** Unbounded, schema-varying, write-heavy run payloads with no cross-run joins — textbook document case, and Atlas has the best learning ROI + local-dev parity. *Rejected: DynamoDB* (awkward ad-hoc run inspection); *Rejected: runs in Postgres jsonb long-term* (VACUUM/bloat on a write-heavy table).

### Logs — start in CloudWatch, add OpenSearch when search hurts
**What:** Structured JSON logs to stdout → **CloudWatch Logs** (dev); **AWS OpenSearch** (or self-hosted Loki/Grafana) when you need full-text/time-range search at scale. Keep `log_stream_ref` on the `FlowRun` doc. *Rejected: logs in Postgres* — write amplification, table bloat, zero relational benefit.

### Solo-dev sequencing (the pragmatic path)
**Start with Postgres + `jsonb` for everything**, including `FlowRun`/`ActionRun` — one datastore to operate, joins available, and `jsonb` absorbs flexible payloads. Introduce MongoDB Atlas only when run volume causes write/bloat pain; OpenSearch only when CloudWatch search is too weak. This defers two vendors and matches the modular-monolith-first strategy. Design `FlowRun` access behind a repository interface so the Postgres→Mongo swap is localized.

### Cost (approximate 2026 USD/month)

| Store | Dev / hobby | Small-prod |
|---|---|---|
| Postgres | RDS `db.t4g.micro`, ~$15–25 (free tier ~$0 first 12mo) | Aurora PG, 1 writer + 1 replica, ~$120–250 |
| MongoDB | Atlas M0 free, or M10 ~$60 | M20/M30, ~$150–350 |
| Logs | CloudWatch ingest/retention, ~$5–20 | OpenSearch small cluster, ~$80–200 |
| **Phase-1 total** | **~$20–45** (Postgres-only path) | **~$350–800** (all three) |

**Bottom line:** Postgres (RDS→Aurora) is the backbone; MongoDB Atlas and OpenSearch are *deferred* additions you grow into. Begin Postgres-only to ship in the ~6.5-month window, split out the document and log stores as volume demands.

### 3.4 Messaging: Queue & Streaming (fan-out)

This layer turns "an event arrived" into "every subscribed flow runs reliably." Two distinct semantics are in play, and conflating them is the classic mistake:

- **Pub/sub fan-out (one event → many flows):** requirement NFR #3 / HLD #5. An inbound webhook on a `Channel` must reach *all* active `Subscription`s for that channel. One producer, N independent subscribers.
- **Work queue (one job → exactly one worker):** each resulting `FlowRun` (and each retryable outbound HTTP `ActionRun`) is a unit of work that exactly one executor should pick up, with retries and a dead-letter path.

The dispatcher in the domain model bridges them: it *consumes* the channel event, queries active subscriptions (Redis-cached), then *produces* one work item per matching flow.

### What to use

**Starting choice (solo dev, dev → small-prod): AWS SQS + SNS.**
- **SNS topic per Channel** = fan-out. The dispatcher publishes once; SNS does not itself store work — instead use the **SNS→SQS fan-out pattern**: each consumer type subscribes an SQS queue to the topic. But for *this* design, fan-out is data-driven (subscriptions live in Postgres), so the simpler shape is: gateway → **one SQS "ingest" queue** → dispatcher worker resolves subscriptions → publishes one message per flow to a **"flow-run" SQS queue** → executor workers consume. SNS is only needed when you have multiple *static* subscriber services; given DB-driven fan-out, **start with SQS-only** and add SNS later if you split services.
- **SQS Standard** for the flow-run work queue: at-least-once, near-unlimited throughput. **SQS FIFO** only for the rare flow that needs strict per-key ordering (FIFO `MessageGroupId` = per-flow or per-entity ordering, exactly-once dedup window).
- **DLQ:** native — set `maxReceiveCount` (e.g. 5) with a redrive policy to a `*-dlq` queue. Failed runs land there for inspection on the Logs screen.

**Why SQS/SNS over alternatives:** fully managed (zero ops — decisive for a solo learner), pay-per-use, native DLQ/retry/visibility-timeout, trivial AWS SDK integration. It teaches core distributed concepts (at-least-once, idempotency, backpressure) without running a broker.

**Rejected:**
- **Kafka / AWS MSK:** real log-streaming with replay and consumer groups — the "right" tool at large scale and great learning, but MSK is ~$150–300+/mo minimum and operationally heavy; massive overkill for month one. *This is the scale-up target, not the start.*
- **RabbitMQ / NATS (self-hosted):** excellent routing/JetStream, but you now run, patch, and monitor a broker. Rejected for the same ops-burden reason; revisit only if avoiding AWS lock-in matters.
- **Redis Streams:** you already run Redis (ElastiCache) for caches; consumer groups give work-queue + light fan-out cheaply. Viable as a *zero-extra-cost* starter, but weaker DLQ/redrive ergonomics and you'd hand-roll retry/visibility logic. Use SQS instead so delivery semantics are managed; keep Redis for hot lookups.
- **EventBridge (bus):** great for *schedule triggers* (EventBridge Scheduler replaces a cron worker — recommend it for the schedule-trigger path) and cross-service event routing, but pricier per-event and not a work queue. Use it for scheduling, not for the run fan-out hot path.

### Delivery guarantees (how, concretely)

- **At-least-once:** SQS guarantees this; design every consumer to tolerate duplicates.
- **Idempotency:** stamp each event with a deterministic `event_id` (hash of channel + payload + ingest timestamp) and each fan-out message with `flow_run_id` derived from `(event_id, flow_id)`. Executor does a conditional insert of the `FlowRun` doc keyed on `flow_run_id`; a duplicate delivery no-ops. This is the safety net for at-least-once.
- **Ordering:** Standard SQS is best-effort order — fine, since flows are independent. Use FIFO queues only where a flow mutates ordered external state.
- **Retries:** visibility-timeout redelivery + `maxReceiveCount` → DLQ. Outbound HTTP actions also retry *in-handler* with exponential backoff for transient 5xx/429 before letting the message redeliver.

### Scale-up path

SQS-only → add **SNS** when you split static subscriber services → introduce **MSK/Kafka** when you need event replay, high-throughput streaming, or analytics tap-off (raw events → ClickHouse). The domain's `Subscription` table means fan-out logic doesn't change when the transport does.

### Cost (approximate, 2026 USD)

- **Dev/hobby:** SQS first 1M requests/mo free, then ~$0.40/M; SNS ~$0.50/M publishes. Realistic dev usage: **$0–2/mo.** EventBridge Scheduler: negligible.
- **Small-prod (~10–50M messages/mo):** SQS ~$4–20/mo + SNS ~$5–25/mo + minor data transfer ≈ **$15–60/mo.**
- **Contrast — MSK (scale-up):** ~$150–300+/mo for a minimal multi-AZ cluster, before storage. Defer until volume or replay needs justify it.

**HLD verdict (#5 / NFR #3):** **Keep** the "queue + stream/fan-out" concept — it's correct and central. **Adjust** the implementation: don't reach for Kafka on day one; start SQS-only with DB-driven fan-out, model pub/sub explicitly only when multiple static subscribers exist, and treat MSK as the deliberate scale-up milestone.

### 3.5 Flow Execution Engine

**What it is.** The runtime that takes a pinned `FlowVersion.graph` and executes its node DAG to completion: fire trigger → enqueue a `FlowRun` → run each `ActionRun` in topological order, threading typed output of one step into the typed input of the next, persisting per-step state, retrying transient failures, and recording everything to the document/log stores.

**Build bespoke vs. adopt durable execution — the central decision.** *Durable execution* means the framework persists the state of a workflow after every step, so a process crash, deploy, or host failure resumes from the last completed step instead of restarting or losing work. Your workflow code reads as a normal function; the framework checkpoints each step's result and replays deterministically on recovery. This matters here because a Flow may make several outbound HTTP calls (`action.email`, `action.http`); without durability a crash mid-flow either silently drops the run or re-sends emails. Durable execution gives exactly-once-effect semantics, free retries with backoff, timers for long/async actions, and a queryable run history — i.e. most of your retry/partial-failure/per-step-state requirements come for free.

**Recommendation: Temporal (self-hosted via `docker compose` for dev, Temporal Cloud for small-prod).** Workflows model the DAG; each node is an Activity (the generic connector interpreter for `action.http`, a transform for `action.dataTransform`, an SMTP/SES call for `action.email`). Activity retry policies handle retries/backoff per step; Workflow history *is* your per-step state and partial-failure record; `workflow.sleep`/signals handle long-running and async (webhook-callback) actions natively. You learn the single most valued distributed-systems pattern of 2026.

*Rejected:* **AWS Step Functions** — durable and managed, but Amazon States Language is JSON-config, not real code; awkward to interpret a *user-authored* dynamic graph, and weaker local-dev story. **Inngest / Restate** — Inngest is the easiest DX and a fine fallback, but more SaaS-magic, less mechanism learned; Restate is excellent but smaller community for a learner. **Bespoke engine on SQS + a state machine in Mongo** — you *will* reinvent retries, timers, idempotency, and replay, and get them subtly wrong; only justified as a deliberate "learn the internals" detour, not as the shipping engine.

**Scheduling (cron triggers).** Use **EventBridge Scheduler** (one schedule per cron Trigger) to invoke a small dispatcher that starts a Temporal workflow; or, simpler, a single Temporal **Schedule** per cron trigger (Temporal has native cron). Prefer Temporal Schedules — one fewer moving part and it inherits durability. *Webhook triggers* hit the API gateway → enqueue once → dispatcher queries active `Subscription` rows (Redis-cached) → starts one workflow per matching flow (the fan-out from the domain model).

**Compute + queue mapping.** Temporal *is* the durable queue (task queues), so you don't also need SQS for orchestration. Run Temporal Workers as containers on **ECS Fargate** (dev: 1 task; small-prod: 2–3 with autoscaling on task-queue backlog). Keep Redis (ElastiCache) for subscription/permission/connector-spec lookups only.

**HLD verdict:** your "execution service + queue/stream" (HLD #2b, #5) is **correct — keep it**, but **adjust**: let Temporal be both the queue and the state engine rather than hand-rolling queue + per-step state on raw SQS/Mongo.

**Cost (approx. 2026 USD/month).**
- **Dev:** self-hosted Temporal in Docker on one `t4g.small` EC2 or local — ~$0–15. Fargate single task ~$15. **~$15–30 total.**
- **Small-prod:** Temporal Cloud (usage-based, ~$25 floor + actions, realistically ~$50–150 at light load) + Fargate workers ~$60–120 + ElastiCache `t4g.micro` ~$12. **~$130–280 total.** *Cheaper alt:* keep self-hosting Temporal on a `t4g.medium` (~$30) to defer Temporal Cloud.

### 3.6 Integration / Connector Framework & Secrets

**WHAT — a declarative, config-driven connector model (not code plugins).** A connector is **data**, not deployed code: a Postgres row plus a `jsonb` *spec*. The spec carries `auth` (scheme + secret placement), `baseUrl`, default headers, and a set of named **operation templates** (method, path template, header/body templates, typed input/output JSON Schema). A single generic **runtime interpreter** in the execution service reads any spec and performs the HTTP call. Adding ServiceNow/Slack/WhatsApp/Teams = inserting a connector row; "author from the UI" and "AI-authored" (req #1, #6b) fall out for free because there is nothing to deploy.

**HOW — outgoing requests.** Each operation template resolves at run time against the step input:
```
POST {{connection.baseUrl}}/api/now/table/{{input.table}}
Headers: Authorization: Bearer {{secret.token}}, Content-Type: application/json
Body: {"short_description": "{{input.summary}}"}
```
Templating uses a **sandboxed, logic-less engine — Jinja2 (StrictUndefined) or a JSONata/JMESPath expression layer — NOT raw `eval`/JS `Function`.** Secrets are injected by reference (`{{secret.*}}` resolves from the secrets manager at execution, never persisted into the resolved request that gets logged). **HOW — incoming.** The Trigger/Channel endpoint defines path, expected auth (HMAC signature verification for Slack/WhatsApp/Teams, shared-secret, or none), and a payload schema; the gateway authenticates, normalizes, and enqueues once for fan-out.

**WHY / rejected.** Code-plugin model (n8n-style TypeScript nodes, Workato SDK) is more powerful for gnarly APIs but defeats UI authoring, blocks the AI agent, and turns every integration into a deploy + supply-chain risk — **rejected** for this tier; the declarative model covers ~90% of REST/webhook APIs. Templating via `eval` — **rejected** as a remote-code-execution hole on multi-tenant user-authored specs. A future "advanced/custom code action" sandboxed in a Lambda/Firecracker microVM can cover the long tail later.

**CRITICAL — credentials & secrets.** The Postgres `Connection` row holds only relational metadata + a `secret_ref`; **raw API keys, OAuth access/refresh tokens, and HMAC signing secrets live in AWS Secrets Manager (KMS-encrypted at rest)**, never in the app DB or logs. Per-tenant isolation: namespace secret paths `org/{org_id}/conn/{connection_id}` and scope IAM/KMS grants per org. **OAuth2:** the static-data service runs the authorization-code flow (redirect → callback → exchange), stores access + refresh tokens in Secrets Manager; the execution interpreter checks expiry and **auto-refreshes** (refresh-token grant) on 401 or pre-expiry, writing the rotated token back. **Rejected:** encrypting secrets into a `jsonb` column with an app-held key — weaker rotation/audit story and a single key-compromise blast radius. HashiCorp Vault is the stronger learning play (dynamic secrets, fine-grained leasing) but adds an HA service to operate — **rejected for dev**, reconsider at scale.

**COST (approx. 2026 USD/month).**
- **Dev/hobby:** AWS Secrets Manager $0.40/secret/mo + ~$0.05/10K API calls; a few dozen secrets ≈ **$1–5**. KMS default key free; CMK ~$1. Total **~$5**.
- **Small-prod:** hundreds–low-thousands of connections → **$50–400** (Secrets Manager dominated by per-secret count). At thousands of secrets, switch to one secret per *org* holding a map, or self-hosted **Vault on a single node (~$15–40 EC2)** to cap per-secret fees.

**HLD verdict:** the "Credentials in SQL" line (HLD #3) is **adjust** — keep the *record* in Postgres, but the *secret material* must live in Secrets Manager/KMS, not the SQL row.

### 3.7 IAM, RBAC & Multi-tenancy

**What — use a managed identity provider, not hand-rolled auth.** Recommendation for a solo learner: **AWS Cognito** as the IdP (user pools = authentication, JWT issuance, hosted login, social/OAuth, MFA, password reset), with **RBAC and tenancy modeled in your own Postgres** (the domain model already defines `Organization / User / Membership / Role / Permission / RolePermission`). Cognito owns *authentication*; your control-plane owns *authorization*.

**How.**
- **Auth flow:** client logs in against a Cognito user pool → receives a signed **JWT access token** (15-min TTL) + refresh token (rotating). Tokens are validated at the API gateway by verifying the RS256 signature against Cognito's JWKS — stateless, no per-request DB hit.
- **Sessions vs JWT:** stateless JWT for API calls; refresh tokens handled by Cognito. Store nothing server-side except a Redis denylist for forced logout/revocation.
- **Identity→authz bridge:** the JWT carries `sub` (Cognito user id) and a custom `org_id` claim for the *active* org. On login/org-switch, your control-plane resolves `Membership → Role → Permission` and **caches the resolved permission set in Redis** keyed by `(user_id, org_id)` (TTL ~5 min, invalidated on role change). Authz checks read Redis, not Postgres.
- **Enforcement across services:** the **API gateway** verifies the JWT and rejects anonymous/expired calls (coarse gate). The **control-plane service** does fine-grained permission checks (`flow:execute`, `connector:author`) against the cached set before any CRUD/config mutation. The **execution service** does NOT re-auth user JWTs at run time — a run already passed authz when it was triggered/scheduled; instead it carries an internal signed service token + the originating `org_id`, and every datastore query is scoped by `org_id`. This keeps execution off the hot auth path.

**Multi-tenancy — shared DB with `org_id` + Postgres Row-Level Security (RLS).** Every table carries `org_id`; set `SET app.current_org` per connection and enforce RLS policies so a query physically cannot return another tenant's rows. Document store (Mongo) and logs (OpenSearch) are filtered by mandatory `org_id` predicates in a shared data-access layer. **Rejected: schema-per-tenant** — thousands of schemas balloon migration/connection-pool complexity and crush a solo dev; **rejected: DB-per-tenant** — operationally absurd at this tier. Shared-table + RLS is the iPaaS-standard middle ground and the best learning ROI.

**Securing per-tenant webhook endpoints.** Each incoming Trigger gets a stable, unguessable `ingest_key` in the path (e.g. `/ingest/{ingest_key}`); the gateway resolves it to `(org_id, channel_id)` without a user JWT. Require an **HMAC signature** (shared secret in Secrets Manager) or provider-native verification (Slack signing secret, etc.), apply **per-key rate limits** at the gateway, and reject on tenant/flow `disabled`. The endpoint is anonymous-to-users but cryptographically bound to one tenant.

**Why Cognito over alternatives.** **Rejected build-your-own** — password hashing, MFA, token rotation, and breach risk are exactly what you should *not* hand-roll. **Rejected Auth0/Clerk** — best DX but per-MAU pricing bites at scale and teaches a vendor, not the cloud. **Rejected Keycloak/Ory self-hosted** — most learning but you'd babysit an auth server for 6 months instead of shipping; revisit later. Cognito wins on AWS-native integration (matches the rest of the stack) and free tier.

**Cost (approx. 2026 USD/month).**
- **Dev/hobby:** Cognito free up to 10K MAU → **$0**. Redis (ElastiCache `t4g.micro` or local) ~$0–12. **~$0–12/mo.**
- **Small-prod:** Cognito ~$0.015/MAU above 10K (e.g. 25K MAU ≈ **$225**); ElastiCache small node ~$12–25; Secrets Manager $0.40/secret/mo. **~$250–280/mo**, dominated by MAU count.

*Alternative if you outgrow Cognito's clunky UI/customization: migrate to Keycloak self-hosted (license-free, ~$15–30/mo compute) once you have ops bandwidth — the Postgres RBAC model above is IdP-agnostic and won't change.*

### 3.8 AI Agent Layer (Claude)

The assistant is an **agentic loop on the Messages API with tool use** (official Anthropic SDK — `anthropic` for Python, `@anthropic-ai/sdk` for TypeScript). Claude never mutates platform state directly; it calls **tools that your execution/static-data services own**, and those tools enforce IAM/RBAC and write to Postgres/Mongo exactly as the UI would. This is the keystone of requirement #6: because connectors and flows are *declarative JSON* (per the domain model), "build me an integration" reduces to "emit a valid connector spec" and "build me a flow" to "emit a valid flow graph."

**Model choice per task** (use these exact IDs):
- **Haiku 4.5** (`claude-haiku-4-5`, $1/$5) — intent classification ("is this a build request or an analysis request?"), connector-field extraction, log-line labeling. Cheap, 200K context, fast.
- **Sonnet 4.6** (`claude-sonnet-4-6`, $3/$15) — the default agent driving most tool-use turns: authoring connectors, assembling flow graphs, routine log analysis. Best cost/intelligence balance.
- **Opus 4.8** (`claude-opus-4-8`, $5/$25) — only the hardest reasoning: debugging a failing multi-step flow from logs, designing a non-trivial connector with OAuth + pagination + body templating. 1M context fits a whole flow + its run history.
- Rejected: Fable 5 ($10/$50) — over-budget for a learning project; reserve for nothing here. Rejected: defaulting everything to Opus — 5x Haiku's cost for classification you don't need it for.

Use `thinking: {type: "adaptive"}` and tune `output_config: {effort: ...}` (`low` for Haiku-style tasks, `high` for Opus debugging). Do **not** use `budget_tokens` — it 400s on these models.

**Tools (function calling).** Define typed tools the agent loop dispatches to your services: `create_connector(spec)`, `create_flow(graph)`, `add_action(flow_id, node)`, `query_logs(flow_id, time_range)`, `get_flow(flow_id)`, `list_connectors(org_id)`. Use the SDK tool runner for the loop, or a **manual agentic loop** when you need human-in-the-loop gating (recommended here).

**Structured outputs.** Generate connector specs and flow graphs with `output_config: {format: {type: "json_schema", schema: ...}}` (or `strict: true` on the create tools) so the JSON validates against your connector/flow schemas before it ever reaches the DB. This eliminates a whole class of malformed-spec bugs. (Prefilling is removed on these models — use structured outputs, not assistant prefill.)

**Guardrails — AI proposes, human/validation confirms.** Mutating tools (`create_connector`, `create_flow`) run in a **propose → validate → confirm** flow: the tool handler validates the spec against schema + RBAC, returns a *diff/preview* as the tool result, and the agent surfaces it; the actual DB write only fires after the user clicks confirm in the UI. Destructive actions (delete flow, disable active flow) are never auto-executed — they require an explicit second confirmation event. This maps cleanly onto the manual loop (intercept `tool_use`, gate, then return `tool_result`).

**Reading logs for analysis.** `query_logs` reads from the log/time-series store (OpenSearch/Loki) and the `FlowRun`/`ActionRun` documents, returns structured run summaries (not raw log spew), and the agent reasons over them — pinning to a `flow_version_id` so analysis matches what actually ran.

**Prompt caching.** The connector-authoring system prompt + JSON schemas + tool definitions are a large stable prefix — cache them (`cache_control: {type: "ephemeral"}`, ~0.1x reads vs 1.25x writes). Keep the frozen prompt/tools first, put the user's varying request after the last breakpoint. Verify via `usage.cache_read_input_tokens`.

**Interactive vs batch.** Interactive chat streams (`.stream()` / `.get_final_message()`). Bulk overnight analysis ("summarize failures across all flows this week") uses the **Batch API** (`messages.batches`) at **50% off** — non-latency-sensitive and the natural fit for the per-flow analytics rollups.

**Cost estimate — one "build me a flow" turn (Sonnet 4.6).** Cached system prompt + schemas + tools ≈ 8K tokens (cache read ≈ 8K × $3/1M × 0.1 = **$0.0024**); user request + uncached context ≈ 2K input (× $3/1M = **$0.006**); generated flow-graph JSON + reasoning ≈ 3K output (× $15/1M = **$0.045**). **≈ $0.05/turn.** A multi-tool agentic build (3–4 turns) lands around **$0.15–0.25**. The same turn on Opus ≈ 1.7x; on Haiku ≈ 0.3x.

**Cost per tier.**
- **Dev/hobby:** a single developer testing → mostly Haiku/Sonnet, a few hundred turns/month. **~$5–20/month**, easily under the $5 minimum spend most months. Code execution server-side tooling has a free monthly allowance you won't exceed.
- **Small production:** ~50 active users, light agent use (a few builds + analyses each per day) ≈ 10–20K turns/month, Sonnet-weighted with batch analytics at half price and caching cutting input cost ~80%. **~$150–500/month.** Opus-heavy debugging usage pushes the top of that range; aggressive caching + routing classification to Haiku keeps it near the bottom.

### 3.9 Caching (Redis)

**What to use:** A single **Redis 7.x** instance (managed). Dev: **Upstash Redis** (serverless, pay-per-request, scales to zero) or a local Docker Redis. Small-prod: **AWS ElastiCache for Redis** (`cache.t4g.micro`/`small`), same VPC as the app. **Rejected:** self-hosted Redis on EC2 (needless failover/patching ops for a learning project); **Memcached** (no persistence, no Streams, no rich data structures — disqualifies the queue/rate-limiter double-duty below); in-process caches like Caffeine (don't survive across multiple app hosts behind the load balancer — HLD #4).

**What to cache (and how):**

| Data | Structure | TTL | Invalidation |
|---|---|---|---|
| Resolved permission set per `(user, org)` | `SET` / serialized blob, key `perm:{org}:{user}` | 5–15 min | Delete key on role/membership change |
| Active subscriptions per channel (fan-out hot path) | `SET` of `flow_id`, key `subs:{channel}` | 10 min | Delete key on activate/deactivate |
| Connector spec (interpreter reads it every outbound call) | `STRING` (JSON), key `connector:{id}:{version}` | 1 h | Version-keyed → never stale; old key just expires |
| Flow definition (`FlowVersion.graph`) | `STRING` (JSON), key `flowver:{flow_version_id}` | 1 h | Immutable per version → no active invalidation |
| Rate-limit counters (HLD #1) | `INCR` + EXPIRE, key `rl:{key}:{window}` | window length | Auto-expire |
| Idempotency keys (inbound dedup) | `SET key NX EX`, key `idem:{ingest_key}:{hash}` | 24 h | Auto-expire |
| Session / token introspection | `STRING`, key `sess:{token}` | token lifetime | Delete on logout |

**Invalidation strategy:** prefer **version/immutability-keyed caching** for config (connector specs and flow graphs are versioned — a new version is a new key, so reads are never stale and you never invalidate, you just let old keys TTL out). For mutable lookups (permissions, subscriptions), use **write-through delete**: the static-data service deletes the affected key inside the same transaction commit hook that changes the underlying Postgres row, so the next read repopulates (cache-aside). **Rejected:** TTL-only invalidation for permissions (a revoked user keeps access for the whole TTL — unacceptable for RBAC); **rejected:** pub/sub cache busting across nodes (over-engineered at this scale; single shared Redis means one delete suffices).

**What NOT to cache:** raw secrets/OAuth tokens (those live in Secrets Manager, requirement from the Connection model — caching them widens the blast radius); `FlowRun`/`ActionRun` execution records and logs (write-once, read-rarely, already in Mongo/OpenSearch — caching adds no hit-rate); anything you can't cheaply invalidate or that mutates faster than it's read.

**Double-duty — Redis as queue + rate-limiter:** for the early modular-monolith phase, lean into it. Use **Redis Streams** (`XADD`/consumer groups) as the inbound event queue and FlowRun dispatch queue (HLD #5), and `INCR`+`EXPIRE` for rate limiting (HLD #1) — one dependency covers cache, queue, and limiter, maximizing learning-per-component. **Separate concerns when:** you need durable, replayable, multi-day-retention event streams or guaranteed delivery semantics across services — migrate the queue to **SQS / Amazon MSK (Kafka)** and keep Redis purely as cache + rate-limiter. Streams' in-memory durability and trimming make it the wrong long-term home for the fan-out backbone, but it's the right first move.

**Cost (approximate 2026 USD/month):**
- **Dev:** local Docker Redis $0; or Upstash free/pay-as-you-go ~**$0–5**.
- **Small-prod:** ElastiCache `cache.t4g.micro` (~0.5 GB) single node ~**$12–16**; `cache.t4g.small` with one replica for HA ~**$45–60**. Upstash fixed-tier alt ~**$10–20**.

**HLD verdict (#6): keep.** Redis is the right call. One adjustment: be explicit that early on it triple-duties as cache + queue + rate-limiter, and plan to peel the queue off to SQS/MSK once durability/replay matters — don't build the long-term fan-out backbone on Streams.

### 3.10 Observability & Per-Flow Logs

Two distinct concerns that people conflate. **(a) Execution logs** are a *product feature* — the user opens a FlowRun on the Logs/Home screen and sees what each Action did. **(b) Operational observability** is for *you* — app health, latency, error rates, traces across the gateway → static-data → execution services. Different audiences, different stores, different retention. Build them separately.

### (a) Per-flow / per-action execution logs (product feature, #7)

**What/How.** The structured per-step record (`step_id, status, resolved request URL, response status, input, output, error, duration_ms, attempt`) lives in the **`ActionRun` document embedded in the `FlowRun` (MongoDB Atlas)** — already decided in the data model; the Logs screen reads one run as a unit. Fine-grained **log *lines*** (`flow_run_id, step_id, ts, level, message, attrs`) are append-only and high-volume: write them to a **log store keyed by `flow_run_id`**, with a `log_stream_ref` pointer on the FlowRun doc so the UI query is a single indexed range scan.

**Store + retention.** Dev: structured JSON to stdout → **CloudWatch Logs**, queried with Logs Insights — zero infra. Small-prod: **AWS OpenSearch** (or Grafana **Loki**) for full-text/attr search over millions of lines. Tier retention to control cost: keep hot/searchable logs **14–30 days**; the `FlowRun`/`ActionRun` summary docs (the part the dashboard and AI-analysis agent need) stay in Mongo **90 days**, then roll to S3/Glacier or drop. Raw debug lines expire fastest.

**PII.** Action inputs/outputs carry user data (emails, Slack messages, webhook bodies). **Redact at write time**, not at read: a field-level scrubber on the log pipeline (regex/keypath denylist for tokens, auth headers, known PII fields) before persistence; never log resolved `Authorization` headers or `secret_ref` values; mark sensitive node config so its payload is stored truncated/hashed. This matters doubly because the AI agent (#6) reads run outputs.

**Rejected:** execution logs in Postgres — write amplification + VACUUM bloat on an append-heavy table, and relational queries buy nothing for time-range log search.

### (b) Operational observability (app/metrics/tracing)

**What/How.** Instrument every service with **OpenTelemetry SDKs** (vendor-neutral — the load-bearing choice; it lets you swap backends without re-instrumenting). Emit traces with a `trace_id` propagated from the **API gateway** through the queue into the execution worker, so one inbound request → one trace spanning fan-out to N FlowRuns. Metrics: queue depth, dispatch lag, per-connector HTTP latency/error rate, run success/fail.

**Backend.** Dev: **self-host Grafana + Loki + Tempo + Prometheus** (the "LGTM" stack) via Docker Compose — best learning ROI, free, teaches the OTel pipeline end-to-end. Small-prod: **Grafana Cloud** free/pro tier (managed LGTM, same dashboards, no ops). **Rejected: Datadog** — excellent but cost balloons fast (host + ingestion + custom-metric pricing) and overkill for a learning project. **Rejected: CloudWatch-only** for tracing — X-Ray is weaker than Tempo and locks you to AWS; fine for logs, not the whole stack.

### Cost (approximate, 2026 USD/month)

- **Dev/hobby:** CloudWatch Logs (~few GB) **$5–15**; self-hosted LGTM on your existing dev box **$0**. **~$5–15.**
- **Small-prod:** OpenSearch smallest cluster **~$25–70**; Grafana Cloud Pro **~$0–50** (generous free tier, then usage); CloudWatch overflow **~$10–20**. **~$60–140**, dominated by OpenSearch — defer it (stay on CloudWatch Logs Insights) until log search volume actually justifies the spend.

**HLD verdict:** gateway logging (HLD #1) is **correct but too narrow** — log at the gateway *and* propagate one `trace_id` through the queue into execution so a fanned-out request stays correlated; and split user-facing execution logs from operational telemetry rather than treating "logging" as one thing.

### 3.11 Analytics (per user/org)

Split analytics into two products with different SLAs, datastores, and consumers. Conflating them is the most common iPaaS mistake.

- **Operational analytics** — "is my automation healthy *right now*." Powers the **Home dashboard** (running vs draft flows), per-flow run history, alerting. Read by end users, freshness matters (seconds–minutes), low cardinality per page.
- **Product analytics** — "how is the platform/org being used over time." Powers usage-per-org/user, connector adoption, AI cost, billing, capacity planning. Read by admins/you, freshness is fine at hours, high-volume aggregate scans.

### Metrics that matter

| Metric | Source event | Operational | Product |
|---|---|---|---|
| Executions (count) | `FlowRun` finished | ✓ runs today/flow | ✓ runs/org/day |
| Success / failure rate | `FlowRun.status` | ✓ red flows on Home | ✓ reliability trend |
| Latency (p50/p95/p99) | `FlowRun.duration_ms` | ✓ slow-flow alert | ✓ SLO tracking |
| Volume per connector | `ActionRun.type`/connector_id | — | ✓ adoption, hot connectors |
| Incoming-event throughput | queue ingest | ✓ backlog/lag | ✓ peak load |
| **AI usage + cost** | per-call `usage` (input/output/cache tokens × model) | ✓ per-flow AI spend | ✓ org AI bill |
| Running vs draft flows | `Flow.status` (Postgres) | ✓ Home headline | — |

**AI cost specifically** — record `usage` on every Claude call (the SDK returns `usage.input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`) tagged with `model`, `org_id`, `flow_run_id`, and a `purpose` (agent/classify/analyze). Compute cost from these 2026 rates (input / output per 1M tokens), with cache reads at ~0.1× input and writes at ~1.25×/2× input:

- `claude-opus-4-8` — $5 / $25 (hardest reasoning; flow/output analysis)
- `claude-sonnet-4-6` — $3 / $15 (most agent work)
- `claude-haiku-4-5` — $1 / $5 (cheap classification/routing — e.g. "which connector does this request describe")

Batch API halves cost for non-interactive jobs (bulk flow analysis); prompt caching cuts the connector-catalog/system-prompt prefix to ~0.1×. Store cost as a derived column so a future price change re-prices historically from raw token counts.

### How to compute rollups: pre-aggregate, never scan-on-read

The dashboard must not scan raw `FlowRun`/log data per page load — that is slow and expensive at scale. Pipeline:

1. **Event stream** — emit a structured `run.finished` event (and AI `usage` events) onto the same queue (Kafka/Kinesis/SQS) that already fans out incoming requests. Reuse the infra; don't add a second bus.
2. **Aggregation worker** — a consumer increments per-(org, flow, connector, model, day/hour) counters: run counts, success/fail, sum+count for avg latency, a t-digest/HDR sketch for percentiles, token sums.
3. **Pre-aggregated tables** — write rollups into compact Postgres tables (`analytics_hourly`, `analytics_daily`). The Home dashboard reads these with sub-millisecond indexed lookups.

For *operational* near-real-time (today's counts before the hourly rollup lands), keep live counters in **Redis** (`INCR` per org/flow/day with TTL); the dashboard reads Redis for the current bucket and Postgres for history.

### Storage: pragmatic start → scale-up path

**Start (dev/small-prod):** Postgres rollup tables (Aurora Postgres you already run for config/IAM — no new vendor). A `analytics_daily(org_id, flow_id, day, runs, ok, fail, p95_ms, in_tok, out_tok, cost_usd)` table stays tiny (one row per org/flow/day) and serves every dashboard query. Raw `FlowRun` documents in Mongo + logs in the log store remain the source of truth for drill-down.

**Scale-up:** when raw-event volume makes Postgres aggregation jobs slow (tens of millions of runs/day), ship raw events to a **columnar/OLAP store — ClickHouse** (best learning ROI + cost; self-host or ClickHouse Cloud) and run rollups there, keeping the thin Postgres summary tables as the dashboard-serving layer. Materialized views in ClickHouse compute the rollups continuously.

**Rejected:** *Redshift/BigQuery* as the first move — overkill and pricier for this volume; revisit only if you standardize on AWS-native warehousing or already have BigQuery. **Rejected:** querying raw runs/logs live for the dashboard — the whole reason rollups exist. **Rejected:** a dedicated TSDB (Prometheus/InfluxDB) as the system of record for business analytics — great for infra metrics, awkward for per-org/per-connector business rollups and joins to org/flow metadata; use Prometheus only for service-level ops metrics.

### HLD verdict

HLD #7 ("store analytics per user/organization") is **correct — keep it**, with one adjustment: make it explicitly a **pre-aggregated rollup**, not live scans, and feed it from the same queue as the execution pipeline rather than a separate analytics ingestion path. The Home dashboard (HLD #8) is then powered cheaply by `analytics_daily` + Redis live counters — no raw-data scan on any page load.

### Cost per tier (approximate 2026 USD/month)

- **Dev/hobby:** $0 extra — rollup tables live in your existing RDS Postgres; Redis counters in your existing ElastiCache/local Redis. Marginal cost is storage (cents).
- **Small-prod:** ~$15–60/mo if you add managed ClickHouse Cloud (entry tier) once volume justifies it; $0 incremental if you stay on Aurora rollups. AI-cost analytics adds no infra cost — it's metadata you already receive in each Claude `usage` response.

### 3.12 Frontend / Client App

**What.** A single-page **React 19 + TypeScript** app built with **Vite**, not Next.js. The entire product is an authenticated internal tool (flow builder, dashboards, config forms, chat) — there is no SEO/marketing surface, no need for SSR or server components. A Vite SPA gives the fastest dev loop, simplest mental model, and cheapest static hosting. **Rejected: Next.js** — its SSR/RSC/routing machinery is dead weight for a logged-in app dashboard, adds a Node server to operate, and muddies the clean SPA-talks-to-API boundary that matches the two-service backend. **Rejected: Angular/Vue** — React has by far the largest ecosystem for the load-bearing piece below (flow builder) and the best learning ROI.

**Visual flow builder (requirement #3/#4).** Use **React Flow (@xyflow/react)**. It gives nodes, edges, drag-to-connect, pan/zoom, minimap, and custom node rendering out of the box — exactly the canvas the `FlowVersion.graph` (nodes+edges jsonb) maps onto 1:1. Serialize the canvas straight to that graph shape. **Rejected: building a canvas from scratch** (SVG/Canvas + custom hit-testing/edge-routing) — months of work for a solved problem; the wrong place to spend a 6.5-month budget. **Rejected: Rete.js/Drawflow** — smaller communities, weaker TS support. xyflow is MIT and free.

**State & data.** **TanStack Query (React Query)** for all server state (connectors, flows, runs, logs) — caching, refetch, polling the Logs/Home views, optimistic flow saves. **Zustand** for local UI/canvas state (selected node, panel open). **Rejected: Redux Toolkit** — too much boilerplate; server-state belongs in React Query, not a global store. Forms via **React Hook Form + Zod**; reuse the connector/action JSON Schemas to drive dynamic config forms (**`@rjsf` / react-jsonschema-form**) so a new connector's UI renders itself from its spec — directly serving "add an integration from the client side."

**AI chat panel (requirement #6).** A side panel streaming responses over SSE from the backend AI service (the backend holds the Anthropic SDK + tool definitions; the client never sees keys). Render streamed tokens and tool-call cards (e.g. "creating connector…"). Standard streaming UI, low frontend risk.

**UI kit.** **Tailwind CSS + shadcn/ui** (Radix primitives) — fast, accessible, copy-in components, no runtime lock-in. **Auth:** SPA holds a short-lived JWT/access token (from the IAM service), refresh via httpOnly cookie; route guards from the cached resolved-permission set. If outsourcing auth, **Auth0/Clerk** SDKs drop in cleanly.

**Hosting.** Dev/hobby: **CloudFront + S3** (static build) or **Vercel/Netlify free tier** — effectively **$0–5/mo**. Small-prod: **CloudFront + S3** with a custom domain/WAF, **~$5–25/mo** (mostly egress); Vercel Pro is **~$20/seat/mo** if you want zero-config CI/previews. Recommend **CloudFront+S3** for AWS-consistency and learning ROI; **rejected: Amplify Hosting** as it obscures the underlying CloudFront/S3 you want to learn.

**MVP vs later.** **MVP:** auth/login, Home (running + draft flows), connector authoring form (schema-driven), connection/credential setup, flow builder canvas with the 3 action types, run-trigger, run-detail + logs viewer, AI chat panel. **Later:** real-time canvas collaboration, version-diff UI, drag-reorder palette polish, analytics charts (Recharts), template marketplace.

**Effort:** roughly **6–9 focused weeks** solo for the MVP, the flow-builder + schema-driven forms being the bulk.

---

## 4. Consolidated Cost Model

For a solo learner, the cheapest viable path keeps the real spend near zero in the first months by leaning on free tiers and a single small box: a modular monolith on one container (or local Docker), Postgres-only (RDS free tier or Neon free), Upstash/local Redis, MongoDB Atlas M0, Cognito free up to 10K MAU, CloudWatch logs, SQS/SNS free tier, a CloudFront+S3 or Vercel-free SPA, and Claude usage that stays under ~$20/mo at dev test volume. Expect roughly $25-70/month all-in at dev/hobby scale (most of which is the optional ALB/WAF edge you can skip locally for ~$0). Small-production is a different order of magnitude — roughly $700-1,800/month — dominated by three drivers: Aurora Postgres with a read replica, Cognito MAU fees once past 10K users, and managed durable-execution/observability (Temporal Cloud + OpenSearch). The single biggest variable cost is Cognito MAU at scale; the biggest deferrable costs are Temporal Cloud, OpenSearch, and a second Mongo tier — all of which can stay on free/self-hosted alternatives until volume forces the upgrade. Claude AI is usage-based and cheap if you route classification to Haiku, default agent work to Sonnet, reserve Opus for hard debugging, and exploit prompt caching (~0.1x reads) plus the 50%-off Batch API for analytics.

### Per-line-item monthly cost (approximate 2026 USD)

| Category | Product | Dev / hobby | Small production | Notes |
|----------|---------|-------------|------------------|-------|
| Edge / API Gateway | AWS ALB + AWS WAF + ACM (TLS) | $0 (skip ALB locally; ngrok for webhooks) or ~$25-35 if provisioned | $45-80 | ACM certs free. WAF ~$5-8 dev / $10-20 prod. Rejected Amazon API Gateway: per-request fees punish webhook volume. Per-tenant rate-limit + IAM done in app middleware, not the edge box. |
| Compute | AWS ECS Fargate (api task + worker task), Node 22 / NestJS | $40-60 (or ~$0 fully local Docker) | $120-220 | Modular monolith, two Fargate services (api behind ALB, worker no ingress). Rejected EKS (~$73/mo control plane before any pod). Autoscale api on ALB requests, worker on queue depth. |
| Database (SQL) | RDS Postgres db.t4g.micro (dev) -> Aurora PostgreSQL 1 writer + 1 replica (prod) | $0-25 (free tier first 12mo, else ~$15-25); Neon free tier ~$0 | $120-250 | System of record: IAM/RBAC, connectors, connections (secret_ref only), flow versions (jsonb), analytics rollups. Biggest controllable prod DB cost is the read replica. |
| Database (Document) | MongoDB Atlas (M0 free / M10 dev -> M20-M30 prod) | $0 (M0 free) or ~$60 (M10) | $150-350 | FlowRun + embedded ActionRun execution records. Deferrable: start Postgres-only with jsonb behind a repository interface; introduce Atlas only when write volume/bloat hurts. |
| Cache / Rate-limit | Upstash Redis (dev) -> AWS ElastiCache Redis cache.t4g.micro/small (prod) | $0-5 (local Docker or Upstash free) | $12-60 | Permission sets, subscription fan-out lists, connector/flow specs (version-keyed), rate-limit counters, idempotency keys. t4g.micro ~$12-16; +replica HA ~$45-60. |
| Queue / Messaging | AWS SQS + SNS (+ EventBridge Scheduler for cron) | $0-2 (1M req/mo free) | $15-60 | DB-driven fan-out: ingest queue -> dispatcher -> flow-run queue + DLQ. Rejected MSK/Kafka (~$150-300+/mo) until replay/streaming needed. Redis Streams a $0 starter alternative. |
| Flow Execution Engine | Temporal (self-hosted Docker dev -> Temporal Cloud prod) | $0-30 (self-host on t4g.small/local) | $130-280 | Durable execution = free retries, per-step state, replay on crash. Temporal Cloud ~$50-150 light load + Fargate workers ~$60-120 + ElastiCache. Cheaper alt: self-host on t4g.medium (~$30) to skip Temporal Cloud. Rejected Step Functions (JSON config, weak for dynamic user graphs). |
| Secrets / Credentials | AWS Secrets Manager + KMS | ~$1-5 | $50-400 | $0.40/secret/mo + $0.05/10K calls. Cost scales with connection count; at thousands switch to one secret per org (map) or self-host Vault on a ~$15-40 EC2. Raw tokens never in app DB/logs. |
| IAM / Auth | AWS Cognito (user pools) + Postgres RBAC + Redis perm cache | $0 (free <=10K MAU) | $225-280 at ~25K MAU | $0.015/MAU above 10K — the single biggest per-user prod driver. Rejected Auth0/Clerk (per-MAU bite + vendor lock). RBAC model is IdP-agnostic; can migrate to self-hosted Keycloak (~$15-30 compute) later. |
| AI | Claude via Messages API (Haiku 4.5 classify, Sonnet 4.6 agent, Opus 4.8 hard debug) | $5-20 (often under monthly minimum) | $150-500 | Per 1M tok in/out: Haiku $1/$5, Sonnet $3/$15, Opus $5/$25. One Sonnet 'build a flow' turn ~$0.05 (~$0.15-0.25 multi-turn). Prompt caching cuts stable prefix to ~0.1x; Batch API 50% off for analytics. Route classification to Haiku to stay near bottom of range. |
| Logs (product) | CloudWatch Logs (dev) -> AWS OpenSearch (prod) | $5-15 | $25-70 | Per-flow execution log lines keyed by flow_run_id; FlowRun/ActionRun summaries stay in Mongo. Redact PII/secrets at write time. Defer OpenSearch until CloudWatch Logs Insights search is too weak. |
| Observability (ops) | OpenTelemetry SDKs -> self-hosted Grafana LGTM (dev) / Grafana Cloud (prod) | $0 (self-host on dev box) | $0-50 (Grafana Cloud free tier then usage) | Traces/metrics with trace_id propagated gateway->queue->worker. Rejected Datadog (cost balloons). Vendor-neutral OTel lets backend swap without re-instrumenting. |
| Analytics | Postgres rollup tables (analytics_daily/hourly) + Redis live counters -> ClickHouse Cloud at scale | $0 (reuses existing RDS + Redis) | $0-60 | Pre-aggregated rollups fed from the same queue; never scan raw runs on dashboard load. AI cost analytics adds no infra (derived from each Claude usage response). ClickHouse only when tens of M runs/day. |
| Frontend / Hosting | React 19 + Vite SPA + React Flow on CloudFront + S3 (or Vercel/Netlify free) | $0-5 | $5-25 | SPA, no SSR needed (authenticated internal tool). React Flow (MIT, free) for the flow builder canvas. Rejected Next.js (SSR dead weight). Prod cost mostly CloudFront egress; Vercel Pro ~$20/seat if zero-config CI wanted. |

### Total-cost scenarios

| Tier | Monthly total | What it buys |
|------|---------------|--------------|
| Learning / hobby (solo dev) | ~$25-70/mo (as low as ~$5-15 fully local) | Modular monolith on one Fargate task or local Docker; Postgres-only via RDS free tier or Neon free (defer Mongo); Upstash/local Redis; self-hosted Temporal in Docker (or SQS-only, defer Temporal); SQS/SNS free tier; Cognito free (<=10K MAU); CloudWatch logs + self-hosted Grafana LGTM; analytics in Postgres rollups; SPA on Vercel/Netlify free or CloudFront+S3. Claude ~$5-20 on Haiku/Sonnet test traffic. Skip ALB/WAF locally (use ngrok for webhooks) to push the floor toward ~$5-15. Biggest line item is the optional managed edge; everything else rides free tiers. |
| Small production (~50 users / ~25K MAU) | ~$700-1,800/mo | ALB+WAF edge ($45-80); 2x api + burst workers on Fargate ($120-220); Aurora Postgres writer+replica ($120-250); MongoDB Atlas M20-M30 ($150-350); ElastiCache w/ HA ($12-60); SQS/SNS ($15-60); Temporal Cloud + workers ($130-280); Secrets Manager ($50-400, connection-count driven); Cognito at 25K MAU (~$225); Claude ($150-500); OpenSearch logs ($25-70); Grafana Cloud ($0-50); analytics ($0-60); SPA hosting ($5-25). Big drivers: Cognito MAU, Aurora replica, Secrets Manager per-secret count, Temporal Cloud, OpenSearch, and Claude. Defer Temporal Cloud (self-host), OpenSearch (stay on CloudWatch), and the Mongo tier to land near the low end. |
| Scaled (high volume / replay / streaming) | ~$3,000-8,000+/mo | Adds the deliberately-deferred heavy components once volume justifies them: Amazon MSK/Kafka for replayable event streaming and analytics tap-off (~$150-300+ minimum cluster), ClickHouse for OLAP rollups over tens of millions of runs/day, larger Aurora with multiple read replicas and storage autoscaling, multi-AZ ElastiCache cluster, higher Mongo tiers, OpenSearch cluster scale-out, and Cognito MAU fees that scale linearly with users (the dominant variable). Claude cost grows with agent adoption but stays controlled via Haiku routing, prompt caching, and the 50%-off Batch API for bulk flow analysis. At this tier the architecture's modular seams (control-plane vs execution, DB-driven fan-out) are split into real microservices and EKS becomes defensible. |

---

## 5. Roadmap & Timeline (to End of 2026)

The governing principle: **ship a thin end-to-end vertical slice early, then layer breadth, then AI, then scale/hardening.** A solo learner's biggest risk is spending months on infrastructure (k8s, microservices, Kafka) before a single flow has ever run. So the sequencing is deliberately *outcome-first*: by mid-July one real flow executes end-to-end on the simplest possible stack, and every later phase adds a layer to a thing that already works rather than building a layer on faith.

**Resolving the modular-monolith-vs-split tension in the schedule.** The split is a *deployment* decision deferred to Phase 6, not an *architecture* decision. From day one (Phase 1) the code is organized into two hard-bounded modules — `control-plane` and `execution-engine` — with no shared mutable state and communication only through the queue + shared DB contracts. They deploy as **one container** through Phase 5. The seam is real in code the whole time; carving it into two ECS services in Phase 6 is then a transport change, not a rewrite. This buys the microservices *learning* (Phase 6) without paying the microservices *tax* during the months that matter for shipping. Concretely: do NOT stand up two services, inter-service auth, or distributed tracing in June. Do enforce the module boundary in June so the December split is cheap.

### Milestone table

| Phase | Window | Goal | Key deliverables | Tech/domains introduced | Learning focus |
|---|---|---|---|---|---|
| **0. Foundations** | late June 2026 (~1.5 wk) | Repo, skeleton, "hello deploy" | Monorepo (TS), NestJS modular monolith with `control-plane`/`execution-engine` modules, Postgres via RDS, Docker, one ECS Fargate task behind an ALB, CI (GitHub Actions) | TypeScript/NestJS, Docker, RDS Postgres, ECS Fargate, ALB, ACM/HTTPS | AWS account hygiene, container deploy pipeline, the module-boundary discipline |
| **1. Thin vertical slice — "one flow runs"** | July 2026 (~3.5 wk) | A single flow, manually triggered, executes one real outbound action and logs it | Auth (Cognito login + JWT verify), minimal Postgres schema (Org/User/Connector/Connection/Flow/FlowVersion), one declarative connector (e.g. Slack outbound), the generic HTTP interpreter, a 2-node flow (trigger → `action.http`), synchronous in-process run, `FlowRun`/`ActionRun` written to Postgres `jsonb`, a bare run-detail/logs view | Cognito, JWT middleware, declarative connector spec + templating engine, secrets in Secrets Manager, React+Vite SPA shell | The connector-as-data model; secrets never in DB; end-to-end request flow |
| **2. Triggers, queue & fan-out** | late July → Aug 2026 (~4 wk) | Flows trigger by incoming webhook and by schedule; runs go async through a queue | Wildcard `POST /ingest/{ingest_key}` endpoint + HMAC verification, SQS ingest + flow-run queues, dispatcher resolving `Subscription`s and fanning out, idempotency keys, DLQ + retries, EventBridge/Temporal schedule for cron triggers, Redis (ElastiCache) for subscription + permission caching | SQS, SNS concepts, Redis/ElastiCache, idempotency/at-least-once, dispatcher fan-out | Async messaging semantics; at-least-once + idempotency; cache-aside |
| **3. Durable execution + breadth** | Sept 2026 (~4 wk) | Multi-step flows survive crashes; 3 action types; usable builder UI | Temporal (self-hosted Docker → workers on Fargate), DAG executor threading typed I/O, all 3 action types (`action.http`, `action.email` via SES, `action.dataTransform`), React Flow visual builder, schema-driven connector/action config forms, OAuth2 authorization-code flow + token auto-refresh, 2–3 real connectors (ServiceNow, Slack, Teams) | Temporal durable execution, React Flow, react-jsonschema-form, SES, OAuth2 | Durable-execution mental model; typed action graph; OAuth flows |
| **4. Multi-tenancy, RBAC & data split** | Oct 2026 (~3.5 wk) | Real orgs/users/roles; isolation enforced; execution records in proper stores | Full IAM model (Membership/Role/Permission/RolePermission), Postgres Row-Level Security by `org_id`, RBAC checks in control-plane, per-tenant rate limiting (Redis token bucket) + WAF, **migrate `FlowRun`/`ActionRun` from Postgres `jsonb` → MongoDB Atlas** behind the repository interface, logs → CloudWatch (structured) | RLS, RBAC, AWS WAF, MongoDB Atlas, repository-pattern store swap | Multi-tenant isolation; least-privilege; when/why to introduce a document store |
| **5. AI agent layer** | Nov 2026 (~4 wk) | Chat assistant builds connectors/flows and analyzes runs, with guardrails | Agentic loop on Messages API (official SDK) with typed tools (`create_connector`, `create_flow`, `add_action`, `query_logs`), Haiku/Sonnet/Opus routing, structured-output JSON-schema validation against connector/flow schemas, propose→validate→confirm guardrails, prompt caching of the connector-catalog prefix, streaming SSE chat panel in the SPA, `usage`-based AI cost capture | Claude Messages API, tool use, structured outputs, prompt caching, SSE streaming UI | Tool-use agent design; AI-proposes-human-confirms safety; cost control via routing/caching/batch |
| **6. Hardening, observability, service split** | Dec 2026 (~3.5 wk) | Production-shaped, observable, and the two services split out | OpenTelemetry traces propagated gateway→queue→worker, Grafana/LGTM (or Grafana Cloud), per-flow logs to OpenSearch *if needed* (else stay CloudWatch), analytics rollup worker + `analytics_daily` + Redis live counters powering the Home dashboard, autoscaling (ALB request-count + SQS depth), **carve `execution-engine` into its own ECS service** with internal service token, runbook/backup checks | OpenTelemetry, Grafana LGTM, OpenSearch (optional), pre-aggregated analytics, ECS service split, autoscaling | Distributed tracing across a queue; rollups not live scans; the actual microservice split |

### MVP definition (target: end of Phase 4, ~end of October)

The platform is a **functional MVP** when a real user in a real org can: log in (Cognito + RBAC); author a connector from the UI and connect credentials (stored in Secrets Manager); build a multi-step flow on the React Flow canvas using the 3 action types; have it trigger via incoming webhook **or** schedule; have it run durably (Temporal) with retries; and view per-flow execution logs on the Home dashboard — all isolated per tenant via RLS. **The AI agent (Phase 5) and full observability/service-split (Phase 6) are explicitly NOT part of the MVP** — they are the value-add and production-shaping layers on top of a platform that already works end-to-end. This ordering means that even if Nov/Dec slip, you finish 2026 with a genuinely usable product.

### Sequencing rationale (narrative)

- **Why a slice in July, not foundations through August.** The single highest-leverage de-risking move is proving "trigger → connector → outbound call → log" works on the dumbest possible stack (synchronous, Postgres-only, one connector). Everything after is swapping one component at a time on a working baseline: sync→queue (Phase 2), in-process→durable (Phase 3), `jsonb`→Mongo (Phase 4). Each swap is independently testable and reversible.
- **Why queue before durable execution.** Phase 2 establishes the async/fan-out backbone (the core NFR) on simple SQS; Phase 3 then introduces Temporal *as the durable engine over that flow*. Doing durability first would mean rework when fan-out lands.
- **Why Mongo and OpenSearch are deferred to Phase 4/6, not Phase 1.** Start Postgres-only (the data-layer decision); introduce the document store only when run volume justifies it and behind a repository interface so the swap is localized. This keeps one datastore to operate during the fragile early months.
- **Why AI is Phase 5, not earlier.** The agent's tools are thin wrappers over connector/flow CRUD that must already exist and be schema-validated. Building the agent before the declarative connector/flow model is stable means building tools against a moving target.
- **Why the service split is dead last.** It is the one piece of the HLD that is pure learning-with-cost and zero user value until scale demands it. Done in December against a mature, well-bounded codebase, it is a transport change; done in June it would have taxed every prior phase.

### Stretch / consciously deferred to post-2026

- **Kafka/MSK** for event streaming + replay (stay on SQS; MSK is the scale-up milestone, ~$150–300+/mo, not a 2026 need).
- **Kubernetes/EKS** (ECS Fargate covers autoscaling without the k8s tax; revisit only at multi-service scale).
- **Aurora read replicas / DB-per-region**, ClickHouse OLAP analytics (Postgres rollup tables suffice at this volume).
- **Custom-code/sandboxed action type** (Lambda/Firecracker microVM) for APIs the declarative model can't express.
- **Real-time canvas collaboration, version-diff UI, template marketplace, connector marketplace.**
- **HashiCorp Vault** (dynamic secrets) and **Keycloak** self-hosted IdP — stronger learning plays, but Secrets Manager + Cognito ship faster.
- **Batch-API overnight analytics**, advanced AI cost dashboards, and Opus-heavy auto-debugging of failing flows.
- **SOC2/compliance, formal DR/multi-AZ failover, load/chaos testing** — production-grade hardening beyond a learning MVP.

A realistic read: Phases 0–4 (the MVP) are the firm commitment for the ~6.5-month window; Phase 5 (AI) is high-value and very achievable in November given the declarative foundation; Phase 6 is the "if time allows / continue into Q1 2027" buffer — and slipping it costs you observability polish and the service split, neither of which blocks a working, demoable platform by December 31, 2026.

---

## 6. Risks & Mitigations

| # | Risk | Likelihood | Impact | Concrete mitigation |
|---|------|------------|--------|---------------------|
| 1 | **Scope creep / never shipping.** The HLD describes a Workato-scale platform; a solo dev has ~6.5 months. Trying to build all of it at once means nothing works by EOY 2026. | High | Critical | Lock an MVP contract: auth + Home + schema-driven connector authoring + 1 connection + flow-builder canvas with the 3 action types + manual trigger + run/log viewer + AI chat. Ship modular-monolith first; defer microservice split, MSK, OpenSearch, ClickHouse, EKS to explicit "scale-up" milestones gated on real volume, not calendar. Timebox the flow-builder to 6-9 weeks. |
| 2 | **Durable-execution complexity.** Temporal is powerful but has a real learning curve (workers, task queues, determinism rules, versioning); a wrong determinism assumption silently breaks replay. | Medium | High | Adopt Temporal as a black box first: one workflow type, activities = the connector interpreter, lean on default retry policies. Keep all non-deterministic work (HTTP, DB, random, time) inside Activities, never in workflow code. Self-host via Docker for dev to avoid Temporal Cloud cost while learning. Have a fallback: if Temporal stalls the timeline, the SQS + idempotent-handler path still ships a working (less elegant) engine. |
| 3 | **Secrets mishandling.** User-authored connectors hold OAuth tokens / API keys / HMAC secrets across tenants; a leak into logs, DB, or the AI agent's context is a breach. | Medium | Critical | Secret *material* only in AWS Secrets Manager (KMS), never in Postgres rows, never in resolved-request payloads. Field-level scrubber redacts `Authorization`/`secret_ref`/known PII at log *write* time. `{{secret.*}}` resolves at execution and is excluded from the logged request. AI `query_logs` reads redacted summaries only. Namespace `org/{org_id}/conn/{id}` with per-org KMS grants. |
| 4 | **Webhook security & abuse.** Public `/ingest/{ingest_key}` endpoints are anonymous-to-users and a natural DoS/replay/forgery target; one tenant's flood can starve others. | High | High | Unguessable `ingest_key`; mandatory HMAC / provider-native signature verification (Slack signing secret, etc.); per-key + per-`org_id` token-bucket rate limits in app middleware (Redis), WAF/ALB as global DoS shield; idempotency dedup (`idem:{ingest_key}:{hash}`, 24h) to absorb replays; reject on disabled tenant/flow; enqueue-once then fan out so a flood is bounded by queue, not synchronous fan-out. |
| 5 | **AI cost runaway.** An agentic loop defaulting to Opus, no caching, runaway tool loops, or a malicious user hammering "build me a flow" can spike spend unpredictably. | Medium | Medium | Route by task: Haiku for classify, Sonnet default, Opus only for hardest debugging. Cache the connector-catalog/system-prompt/tool-def prefix (~0.1x reads). Cap agent loop iterations and per-turn max output. Batch API (50% off) for non-interactive analytics. Per-org token budgets + alerting off the recorded `usage` metadata; hard monthly ceiling that disables agent on breach. |
| 6 | **Log/observability cost & volume runaway.** High-volume per-action logs and traces in OpenSearch/Datadog can quietly become the biggest line item. | Medium | Medium | Stay on CloudWatch Logs Insights + self-hosted LGTM until search volume justifies OpenSearch. Tier retention: hot logs 14-30d, FlowRun summaries 90d then S3/Glacier, raw debug lines shortest. Sample high-volume debug logs; never store full payloads unredacted. Pre-aggregate analytics into `analytics_daily` rather than scanning raw runs. |
| 7 | **Multi-tenant data isolation failure.** A single missing `org_id` predicate leaks one tenant's flows/runs/secrets to another — catastrophic for a platform holding customer credentials. | Medium | Critical | Postgres Row-Level Security with `SET app.current_org` so a query *physically cannot* cross tenants even on a forgotten WHERE. Mandatory `org_id` filter enforced in a single shared data-access layer for Mongo/OpenSearch. Execution service carries originating `org_id` and scopes every query. Add an automated test that asserts cross-tenant reads return empty. |
| 8 | **Solo-dev bus factor / burnout.** One person owns every layer; illness, a hard bug, or fatigue stalls the whole project, and undocumented decisions are unrecoverable. | High | High | Favor managed black boxes (Cognito, RDS, Fargate, Secrets Manager, Temporal Cloud) over self-hosted infra to shed ops load. Keep this planning doc + ADRs in-repo. IaC (CDK/Terraform) so the environment is reproducible. Ruthless phasing so each phase is independently demoable. Use the AI agent itself for code review/rubber-ducking. |
| 9 | **Third-party API rate limits & flakiness.** ServiceNow/Slack/WhatsApp/Teams enforce quotas and return 429/5xx; naive outbound calls cause cascading flow failures and possible account bans. | High | Medium | In-handler exponential backoff + jitter on 429/5xx before letting the message redeliver; respect `Retry-After`. Per-connection concurrency/rate caps in the interpreter. Circuit-breaker per connector to fail fast when a provider is down. Let Temporal Activity retries + DLQ catch the rest; surface failures on the Logs screen rather than silently dropping. |
| 10 | **Premature distributed-systems complexity.** Building two services + MSK + EKS + autoscaling on day one (the literal HLD) multiplies failure modes (network auth, distributed tracing, partial failure) before a single flow runs. | Medium | High | Enforce the modular-monolith-first decision: two code modules, one deployable, seam communication via queue + DB contracts only. Split to services / introduce MSK / move to EKS only when a concrete driver appears (execution needs independent autoscaling, replay needed, etc.). Design the seam now (so the split is a transport change, not a rewrite) but don't pay for it yet. |

## 7. Learning Path

The guiding principle: **learn the high-leverage, transferable, mechanism-rich technologies deeply; consume the rest as managed black boxes** so you don't drown. A solo dev cannot master every layer in 6.5 months — spend learning budget where it compounds.

**Highest-leverage to learn deeply (these are the point of the project):**
- **Distributed workflow / durable execution (Temporal)** — the single most career-valuable distributed-systems pattern here; understand workflows vs activities, determinism, retries, replay.
- **Queue + fan-out semantics (SQS → later Kafka/MSK)** — at-least-once, idempotency, DLQ, backpressure, pub/sub vs work-queue. Transferable everywhere.
- **Multi-tenancy + RBAC + Postgres RLS** — schema design, the `User→Membership→Role→Permission` walk, tenant isolation. Core backend skill.
- **Agentic AI on the Messages API (tool use, structured outputs, caching)** — the differentiating skill of 2026; build the manual agentic loop yourself.
- **Containers + ECS Fargate + IaC + autoscaling** — modern cloud deployment fundamentals.
- **Connector-interpreter design** (declarative spec → templated HTTP) — the heart of *this* product.

**Consciously use as black boxes (do NOT go deep early):**
- **Cognito** (auth/JWT issuance — don't hand-roll crypto), **RDS/Aurora & Atlas** (don't self-host DBs), **Secrets Manager + KMS** (don't build a vault), **ALB + WAF + ACM** (managed edge), **CloudFront+S3** (static hosting), **OpenSearch/ClickHouse/MSK/EKS** (defer entirely until volume forces them). Learn their *interfaces*, not their internals, until a real need arises.

**Phased learning order, tied to the roadmap:**

**Phase 0 — Foundations (weeks 1-2).** TypeScript + NestJS module structure; Docker + docker-compose; Postgres schema design + `jsonb`; CDK/Terraform basics. *Black-box: nothing yet — this is all hands-on.* Outcome: local modular monolith skeleton, two modules, runs in containers.

**Phase 1 — Identity & data backbone (weeks 3-5).** Cognito integration (black box), JWT verification middleware, **Postgres RLS + RBAC** (deep), repository pattern, Redis cache-aside. Outcome: auth works, tenants are isolated, config CRUD exists.

**Phase 2 — Connectors & secrets (weeks 6-8).** **Connector-interpreter + templating** (deep — Jinja2/JSONata sandbox), Secrets Manager + KMS + OAuth code flow (interface-level), schema-driven forms on the client. Outcome: author a connector from the UI, make a real outbound call to one real API (ServiceNow or Slack).

**Phase 3 — Execution & queueing (weeks 9-13).** **SQS + idempotency + DLQ** (deep), then **Temporal** (deep — the biggest learning investment; workflows/activities/retries/replay). Webhook ingest + fan-out via subscriptions. Outcome: a real Flow runs end-to-end, triggered by schedule and webhook, with retries and per-step state.

**Phase 4 — AI agent (weeks 14-16).** **Anthropic Messages API agentic loop** (deep): tool use, structured outputs against your connector/flow JSON schemas, prompt caching, propose→validate→confirm guardrails, model routing (Haiku/Sonnet/Opus), `usage`-based cost tracking. Outcome: "build me a flow" works with human confirmation.

**Phase 5 — Frontend polish, logs, analytics, observability (weeks 17-21).** React Flow canvas (deep — it's load-bearing UX), TanStack Query, the Logs viewer, pre-aggregated `analytics_daily` rollups, OpenTelemetry instrumentation (interface-level), Home dashboard. Outcome: demoable product.

**Phase 6 — Scale-up learning (post-MVP, only if time/volume warrant).** *Now* go deep on the deferred heavy stack as deliberate learning exercises: split the monolith into the two services; introduce **MSK/Kafka** for replay/streaming; **OpenSearch** for log search; **ClickHouse** for analytics; **EKS** if you want Kubernetes. Each is a discrete, optional learning module — pursue for the skill, not because the MVP needs it.

This sequencing front-loads the transferable distributed-systems and AI skills (Phases 3-4, the project's whole reason for existing), keeps every other layer a managed service until it earns complexity, and ensures a working, demoable product by ~week 21 — comfortably inside the EOY 2026 window with buffer for the inevitable overruns.

---

The AI layer is named (Claude), but no model/pricing facts are load-bearing in an adversarial-review section — the costs are already supplied authoritatively in the prompt. No skill needed. Writing directly.

## 8. Tradeoffs & Contrarian View (read this one twice)

The plan above is a genuinely good *target architecture* — and that is precisely the risk. It is the system you build in year two with a team, dressed up as the system a solo developer builds in 6.5 months. Every section says "start lean, defer the heavy thing," but the document still introduces ~14 distinct technologies (Postgres, Mongo, Redis, SQS/SNS, Temporal, Cognito, Secrets Manager, ECS/Fargate, ALB, WAF, CloudWatch/OpenSearch, ClickHouse, React Flow, Anthropic SDK). "Defer" is doing a lot of load-bearing work, and deferral discipline is the first thing that collapses under deadline pressure. Here is where I'd push back, hard.

**1. Three datastores on day one is the real over-engineering — not microservices.** The plan correctly kills the 2-service split and k8s, but then quietly keeps Postgres + MongoDB + a log store as "phase 1-ish." That is the same mistake in a different costume. MongoDB earns nothing in v1 that Postgres `jsonb` doesn't: `FlowRun`/`ActionRun` fit fine in a partitioned Postgres table at any volume a solo learner will generate this year (you will not hit Postgres write-bloat at 10K runs/day). Running a second database means a second connection pool, second backup story, second local-dev container, second failure mode — for zero v1 benefit. **Verdict: Postgres-only until it provably hurts. The "learn NoSQL" budget is not worth a production dependency you don't need.**

**2. Durable execution (Temporal) is the single biggest distraction risk.** Durable execution is the *correct* answer to the hardest correctness problem in the platform — and it is also a large, opinionated framework with its own mental model (workflows vs activities, determinism constraints, versioning, replay) that you will spend 2-3 weeks fighting before your first flow runs end-to-end. For a learning project with no live customers, crash-resume of a half-finished flow is a *nice-to-have*, not a launch blocker. The honest minimum is: persist each `ActionRun` status to Postgres, make handlers idempotent, retry transient failures, and on restart resume any run stuck in `running`. That is a few hundred lines and teaches you the actual problem. **Verdict: build the DIY state machine for v1. If you want Temporal, use Temporal Cloud (skip self-hosting entirely) and treat it as an explicit Phase-2 learning week — not a v1 dependency.** Self-hosting Temporal in v1 is the kind of yak-shave that eats a month.

**3. Kafka/MSK is correctly deferred — but even SNS+SQS is probably premature.** The fan-out requirement is real, but with a modular monolith and one worker process, your "queue" can be a Postgres-backed job table (`SELECT ... FOR UPDATE SKIP LOCKED`) or Redis Streams — both of which you already have. You get durability, at-least-once, and fan-out (one row per subscribed flow) without provisioning SNS topics and SQS queues and IAM policies and DLQ redrive configs. SQS is the right *second* move; it is not the cheapest *first* move when you're still proving a flow can run at all. **Verdict: `FOR UPDATE SKIP LOCKED` on Postgres for v1; migrate to SQS when you split the worker onto its own host.**

**4. The AI feature is scoped sanely in mechanism but dangerously in sequencing.** "The agent's tools are the same internal APIs the UI calls, emitting declarative specs" is exactly right and is the only sane design. But the AI agent is listed as MVP. It cannot be: an agent that *builds connectors and flows* is only buildable *after* connectors, flows, validation, and the declarative spec all exist and are stable. If you start the AI layer before the core CRUD is solid, you'll be debugging tool-call JSON against a moving schema. The cost estimates ($5-20/mo dev, $150-500/mo small-prod) are realistic *for the model usage* — that's not where AI risk lives. The risk is timeline: a polished agentic chat with human-in-the-loop confirmation is easily 3-4 weeks of work. **Verdict: AI is the LAST thing you build, not part of the MVP. A "generate a connector spec from a description" single-shot call (no agent loop, no tools) gets you 70% of the demo value in 3 days.**

**5. The cost estimates are individually plausible but collectively optimistic — and miss the real bill.** Summing the "small-prod" tiers across sections lands around **$900-1,800/mo** (Aurora $120-250, Mongo $150-350, OpenSearch $80-200, Cognito at 25K MAU $225, Temporal Cloud $50-150, Fargate $120-220, ElastiCache, ALB, NAT gateway…). For a learning project that is absurd, and you will not run it. More importantly, two silent line items are missing everywhere: **NAT Gateway (~$32/mo + data) is mandatory the moment your private-subnet Fargate tasks call third-party APIs** — which is the entire point of the platform — and **data egress**. The *dev*-tier numbers ($20-60/mo total realistically) are the only ones that matter, and they're fine. **Verdict: the small-prod column is a fantasy you'll never provision; stop pricing it as if you will. Budget the dev tier honestly and add NAT gateway.**

**6. Cognito is the wrong managed service to learn.** The plan picks Cognito for AWS-consistency, then spends a sentence admitting its "clunky UI/customization" and planning a Keycloak escape hatch. That's a tell. Cognito is notoriously painful (rigid hosted UI, awkward custom attributes, irritating token/claim handling), and for a solo dev the time-to-first-login matters more than ecosystem purity. **Verdict: use Clerk or Auth0 free tier for v1 — login works in an hour, and your Postgres RBAC model is IdP-agnostic so switching later is cheap.** Or, since you're already running Postgres, a Lucia-style / library-based session auth teaches you more and removes a vendor entirely. Don't burn days on Cognito's quirks.

**7. The declarative connector model is right — and its hardest 20% is unscoped.** "A connector is JSONB + a generic HTTP interpreter" is the best decision in the whole plan. But the plan waves at the parts that are actually hard: **pagination** (cursor vs offset vs link-header), **OAuth2 token refresh with concurrent-refresh races**, **rate-limit/429 backoff per third party**, and **response-shape mapping into the next action's typed input**. These are 80% of why real connectors are hard, and a templating engine alone doesn't address them. This isn't a reason to abandon the model — it's a warning that "ServiceNow + Slack + WhatsApp + Teams" connectors are each more work than "insert a row." **Verdict: scope v1 to ONE connector done properly (Slack, with real OAuth + refresh) before claiming the model generalizes.**

**8. The frontend estimate is the most likely number to blow up.** "6-9 weeks solo for the MVP" for a flow-builder canvas + schema-driven dynamic forms + streaming chat + dashboards + auth screens is optimistic by roughly 2x for someone also building the entire backend. Schema-driven forms (rjsf) look easy in a demo and become a swamp the moment you need conditional fields, custom widgets for auth config, and good error UX. **Verdict: this is the hidden critical-path item.** A solo dev who is "primarily learning backend" will under-invest here and stall.

**The single most likely reason this does NOT ship by end of 2026:** **scope, via the "defer" trap.** Not any one technology — the *accumulation* of fourteen technologies each individually justified as "good learning ROI" and "we'll defer the hard part." Six months disappears into infrastructure plumbing (Temporal determinism bugs, Cognito claims, SNS→SQS IAM, Mongo+Postgres dual-write consistency) and the actual product — *a user builds a flow that fires on a webhook and posts to Slack* — never gets a vertical slice. Solo learning projects die in the infra layer, not the product layer.

**Leanest alternative path:** Build **one vertical slice end-to-end on the smallest possible stack first**: Postgres (everything, including runs and a `SKIP LOCKED` job queue) + one Fargate task (or even one EC2/Fly box) + a Vite SPA with React Flow. One trigger type (incoming webhook), one connector (Slack, real OAuth), three action types, a DIY persisted-state executor, library-based auth. No Mongo, no SQS, no Temporal, no Cognito, no microservices, no k8s. Get a webhook to fire a flow that posts to Slack, with logs, in ~6 weeks. *Then* re-introduce the deferred tech one at a time as deliberate, scoped learning milestones (week of Temporal, week of SQS fan-out, week of the AI agent) — each on top of a thing that already works. Studying n8n's source (it's the closest open-source analog — declarative-ish nodes, visual builder, queue mode) for a few days before you start is higher-ROI than reading any cloud docs.

**If you only do ONE thing differently:** **Ship a single working vertical slice — webhook → flow → Slack message → visible log — on a Postgres-only, single-service stack within the first 6 weeks, before adding a second datastore, a queue product, a workflow engine, or the AI agent.** Everything else in this plan is a Phase-2 learning milestone layered onto that slice. If you cannot demo that slice by early August, no amount of architecture will save the deadline.
