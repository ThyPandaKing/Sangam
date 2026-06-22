# Timeline & Key Deliverables — Integration Platform

> **Owner:** solo developer, part-time **~12–15 hrs/week**, Claude-assisted.
> **Profile baked into this plan:** strong app developer (TypeScript/React), **new to cloud & distributed systems**.
> **Target:** a **fully working, feature-complete solution by Monday Nov 2, 2026** (end of the Oct 26–Nov 1 week), leaving **Nov + Dec (~8 weeks) for testing and hardening**.
> **Companion doc:** see [ProjectPlan.md](ProjectPlan.md) for the architecture, tech rationale, and cost model. This file is the week-by-week execution tracker.

---

## How to use this tracker

Update the **State** column on each deliverable as you go. Legend:

| State | Meaning |
|-------|---------|
| 🔲 **Yet to start** | Not begun (everything starts here) |
| 🟡 **WIP** | In progress |
| ✅ **Done** | Complete and verified |
| 🔴 **Delayed** | Behind its target week — needs catch-up |
| ⏭️ **Moved to later phase** | Consciously pushed to a later week / the Nov–Dec buffer |

**Suggested weekly rhythm (~13 hrs):** 2 weeknight sessions × ~2.5 hrs (small, well-defined tasks) + 1 weekend block × ~7–8 hrs (the hard/integration work). Start each session by handing Claude the week's deliverable + the relevant code and asking it to scaffold or pair on the next concrete step.

---

## How this plan is customized to you

- **Local-first via LocalStack + Docker Compose.** Weeks 1–4 run entirely on your laptop — Postgres, Redis, SQS/SNS/Secrets Manager (LocalStack), Temporal (docker image), MailHog (email). The **AWS cutover in Week 5** is then a config/endpoint swap because you coded against the same AWS SDK the whole time. This keeps early momentum on features instead of fighting cloud consoles while you're learning.
- **Buffer concentrated on your two new areas.** You'll move fast on NestJS/React; the genuinely new weeks for you are **Week 5 (AWS deploy)** and **Week 6 (Temporal)**. Those are deliberately lighter on new features so the hours go to learning.
- **Basic AI in build, deepen in buffer.** Week 11 ships a thin Claude chat that builds a flow (with a confirm-before-write guardrail). Full connector-authoring-by-AI and cost-cap tooling are Dec items.
- **One datastore for the build.** FlowRun/ActionRun live in **Postgres JSONB**, not MongoDB — fewer moving parts for a part-timer. Mongo is a post-2026 scale decision.
- **Auth behind an interface.** A dev JWT provider locally → swap the implementation to **AWS Cognito** at the Week 5 deploy. RBAC lives in Postgres and is provider-agnostic, so no rework.

**Reality check:** ~13 weeks × ~13 hrs ≈ **~170 build hours**. Feature-complete (including a basic AI agent) in that budget is **ambitious but achievable** for a strong dev leaning hard on Claude — *if* scope discipline holds. The **cut-list at the bottom** is your pressure valve, and the Nov–Dec buffer absorbs overflow. Protect the **non-negotiables** below; everything else is negotiable.

---

## Definition of "fully working" (the Nov 1 bar)

A new user can, on the deployed AWS app:

1. Sign up / log in; belong to an **organization** with a **role** (RBAC enforced, tenant-isolated).
2. **Author/connect a connector** (declarative spec) and store its credentials securely (Secrets Manager).
3. **Build a multi-step flow** on a visual canvas using the **3 action types** (HTTP, Email, Data Transform).
4. **Trigger it** by an incoming **webhook** *or* a **schedule**.
5. Have it **run durably** (Temporal) with retries, surviving a worker crash.
6. **View per-flow / per-action logs** and a **Home dashboard** of running vs. draft flows.
7. Use a **basic AI chat** to scaffold a flow (propose → confirm).

If a feature isn't on this list, it is allowed to slip into Nov–Dec.

---

## Stack summary (one place)

| Layer | Technology | Introduced |
|-------|-----------|-----------|
| Language / API framework | **TypeScript**, **NestJS** (modular monolith: `control-plane` + `execution-engine` modules) | Wk 1 |
| Frontend | **React 19 + Vite** SPA, **React Query**, **@xyflow/react** (React Flow), **@rjsf** (schema forms), **Recharts** | Wk 1 / 4 (builder) |
| Relational DB | **PostgreSQL** + **JSONB** (connector specs, flow graphs, FlowRun/ActionRun), **Prisma** ORM | Wk 1 |
| Local dev infra | **Docker Compose** + **LocalStack** (SQS/SNS/Secrets Manager) + **MailHog** + **Temporal** image | Wk 1 |
| Templating / transforms | **JSONata** (safe expression/transform engine) | Wk 2 |
| Messaging / queue | **AWS SQS** (work queue) + **DB-driven fan-out** to subscriptions (SNS optional later) | Wk 3–4 |
| Cache / rate-limit / idempotency | **Redis** (ioredis) → **ElastiCache** at deploy | Wk 3 |
| Durable execution | **Temporal** (TS SDK; self-host Docker → small instance / Temporal Cloud) | Wk 6 |
| Scheduling | **node-cron** (dev) → **EventBridge Scheduler** / Temporal cron (prod) | Wk 4 |
| Compute / deploy | **Docker → ECR → ECS Fargate** (api behind **ALB** + worker), **Terraform** or **AWS Copilot** | Wk 5 |
| Auth | dev JWT (`jose`) behind an `IdentityProvider` interface → **AWS Cognito** | Wk 1 → Wk 5 |
| Secrets | LocalStack Secrets Manager → **AWS Secrets Manager** (+ KMS) | Wk 2 → Wk 5 |
| Email action | **MailHog** (dev) → **AWS SES** (prod) | Wk 7 |
| AI assistant | **Claude Messages API** (`@anthropic-ai/sdk`): Sonnet 4.6 default, tool use, structured outputs, prompt caching, SSE | Wk 11 |
| Logs / observability | structured logs → **CloudWatch**; **OpenTelemetry** + Grafana later | Wk 10 / 12 |
| CI/CD | **GitHub Actions** (lint/test/build → deploy via OIDC) | Wk 1 / 5 |

---

# Build Phase — Week by Week (Aug 3 → Nov 1)

## Week 1 — Aug 3–9 · Foundations & local dev environment
**Goal:** A running skeleton you can build on, fully local, with the module boundary that lets you split services later.
**Technologies:** TypeScript, NestJS, Prisma, PostgreSQL, Docker Compose, LocalStack, Vite/React, GitHub Actions, `jose` (dev JWT).

| Deliverable | Technology | State |
|-------------|-----------|-------|
| Monorepo (pnpm/Nx workspace): `api`, `web`, shared `types` | pnpm workspaces | 🔲 Yet to start |
| NestJS app with hard-bounded `control-plane` + `execution-engine` modules (no shared mutable state) | NestJS | 🔲 Yet to start |
| `docker-compose.yml`: Postgres, Redis, LocalStack, Temporal, MailHog | Docker, LocalStack | 🔲 Yet to start |
| Prisma schema + first migration; DB connection + health check | Prisma, Postgres | 🔲 Yet to start |
| `IdentityProvider` interface + dev JWT login (swap to Cognito in Wk 5) | jose | 🔲 Yet to start |
| React + Vite SPA shell: routing, layout, login page, API client (React Query) | React, Vite | 🔲 Yet to start |
| CI: lint + typecheck + unit-test + build on PR | GitHub Actions | 🔲 Yet to start |

**Definition of done:** `docker compose up` boots the full local stack; you can log in (dev JWT), hit a `/health` endpoint from the SPA, and CI is green on a PR.
**Claude-assist:** have Claude scaffold the NestJS module skeleton + the docker-compose file, then review the module boundary with it ("does anything in execution-engine import control-plane internals?").

---

## Week 2 — Aug 10–16 · Thin vertical slice — "one flow runs"
**Goal:** Prove the whole spine works on the dumbest possible stack: trigger → connector → outbound call → logged result. This is your single biggest de-risk.
**Technologies:** Postgres JSONB, JSONata (templating), undici/axios, LocalStack Secrets Manager, React Query.

| Deliverable | Technology | State |
|-------------|-----------|-------|
| Core schema: Org, User, Connector, Connection, Flow, FlowVersion, FlowRun, ActionRun (relational + JSONB) | Prisma, Postgres JSONB | 🔲 Yet to start |
| Declarative **connector spec** model (auth scheme, base URL, operation templates, IO schema) | JSON/JSONB | 🔲 Yet to start |
| **Template/expression engine** for resolving `{{input.x}}` in URL/headers/body | JSONata | 🔲 Yet to start |
| **Generic HTTP interpreter** (reads a connector op + resolves templates + makes the call) | undici | 🔲 Yet to start |
| One connector (generic HTTP / webhook.site or Slack outbound) | — | 🔲 Yet to start |
| 2-node flow (manual trigger → `action.http`) executed **synchronously**, writing FlowRun/ActionRun | NestJS | 🔲 Yet to start |
| `SecretsProvider` interface + LocalStack Secrets Manager impl | AWS SDK v3 | 🔲 Yet to start |
| Bare run-detail / log view in the SPA | React | 🔲 Yet to start |

**Definition of done:** From the UI you click **Run** on a flow, it makes a real external HTTP call, and you see the request/response recorded in the run detail.
**Claude-assist:** great week to have Claude generate the JSONata-based template resolver + a connector-spec TypeScript type + Zod validator.

---

## Week 3 — Aug 17–23 · Async execution (queue + worker) ⚠️ *new area — buffer week*
**Goal:** Move flow execution off the request path into a durable queue + worker. Learn work-queue semantics.
**Technologies:** AWS SDK v3 (SQS) against LocalStack, Redis (ioredis), separate worker process.

| Deliverable | Technology | State |
|-------------|-----------|-------|
| SQS queues (LocalStack): `flow-run` + a dead-letter queue | SQS, LocalStack | 🔲 Yet to start |
| Worker process (Nest standalone app in `execution-engine`) consuming `flow-run` | NestJS | 🔲 Yet to start |
| Switch "Run" from synchronous → **enqueue + worker executes async** | SQS | 🔲 Yet to start |
| Idempotency keys (dedupe re-delivered messages) | Redis | 🔲 Yet to start |
| Retry policy + DLQ routing + a "failed runs" view | SQS | 🔲 Yet to start |
| Redis cache-aside for connector specs / connection lookups | Redis | 🔲 Yet to start |

**Definition of done:** Clicking Run enqueues a job; the worker picks it up and executes; killing the worker mid-queue and restarting it loses nothing and never double-runs.
**Claude-assist:** ask Claude to explain at-least-once delivery + idempotency with a worked example before you code, then pair on the dedupe logic. *(Fallback if SQS is fiddly: BullMQ on Redis is a simpler drop-in — but SQS is the better learning + matches the deploy target.)*

---

## Week 4 — Aug 24–30 · Triggers & fan-out
**Goal:** Flows start from the outside world — incoming webhooks (with fan-out to all subscribed flows) and schedules.
**Technologies:** crypto (HMAC), DB-driven fan-out (Channel/Subscription), node-cron (dev).

| Deliverable | Technology | State |
|-------------|-----------|-------|
| `POST /ingest/:ingestKey` webhook endpoint + **HMAC/signature verification** | NestJS, crypto | 🔲 Yet to start |
| Channel + Subscription model; **dispatcher** resolves ingestKey → active subscriptions → fan-out enqueue | Postgres, SQS, Redis | 🔲 Yet to start |
| Per-key idempotency / replay dedupe on ingest | Redis | 🔲 Yet to start |
| Schedule trigger (cron expression on a flow) firing runs | node-cron → EventBridge later | 🔲 Yet to start |
| UI: configure a flow's trigger (webhook URL shown, or cron) | React | 🔲 Yet to start |

**Definition of done:** Hitting one webhook URL fans out to **2 subscribed flows**, each runs async; a cron-scheduled flow fires on time. *(End of all-local development.)*
**Claude-assist:** have Claude write the HMAC verification for a specific provider (e.g., Slack signing secret) and the subscription-fan-out query.

---

## Week 5 — Aug 31–Sep 6 · AWS deployment (the cutover) ⚠️ *new area — buffer week*
**Goal:** Get the working local app running on real AWS. Because you coded against AWS SDKs + LocalStack, this is mostly endpoint/config swaps + IaC.
**Technologies:** Docker, ECR, ECS Fargate, ALB, RDS Postgres, ElastiCache, real SQS, Secrets Manager, **Cognito**, ACM, Terraform (or AWS Copilot), GitHub Actions OIDC.

| Deliverable | Technology | State |
|-------------|-----------|-------|
| Dockerfiles for api + worker; push to ECR | Docker, ECR | 🔲 Yet to start |
| Infra as code: VPC, RDS Postgres, ElastiCache Redis, SQS, ALB, ECS services | Terraform / AWS Copilot | 🔲 Yet to start |
| Deploy api (behind ALB, HTTPS via ACM) + worker (no ingress) on Fargate | ECS Fargate, ALB, ACM | 🔲 Yet to start |
| Swap `IdentityProvider` impl → **Cognito**; swap `SecretsProvider` → Secrets Manager | Cognito, Secrets Manager | 🔲 Yet to start |
| Config/secrets via SSM/Secrets Manager; env parity with local | AWS SSM | 🔲 Yet to start |
| CI extended to build + deploy on merge to main | GitHub Actions OIDC | 🔲 Yet to start |

**Definition of done:** The Week-4 feature set runs on a public HTTPS URL on AWS, deployed via CI.
**Claude-assist:** have Claude generate the Terraform/Copilot config and the GitHub Actions deploy workflow; ask it to review IAM roles for least-privilege. Budget the full week — this is new for you.

---

## Week 6 — Sep 7–13 · Durable execution with Temporal ⚠️ *new area — buffer week*
**Goal:** Replace the bespoke worker loop with Temporal so multi-step flows survive crashes, get free retries, and keep per-step state.
**Technologies:** Temporal (TS SDK), Temporal workers on Fargate; Temporal via docker-compose locally.

| Deliverable | Technology | State |
|-------------|-----------|-------|
| Temporal running locally (docker) + a deployed Temporal (self-host small instance *or* Temporal Cloud) | Temporal | 🔲 Yet to start |
| Flow → Temporal **Workflow**; each action → an **Activity** (the connector interpreter) | Temporal TS SDK | 🔲 Yet to start |
| All non-deterministic work (HTTP/DB/time/random) lives in Activities, never workflow code | Temporal | 🔲 Yet to start |
| Default retry policies + per-step state surfaced into ActionRun | Temporal | 🔲 Yet to start |
| Schedule triggers migrated to Temporal cron (or kept on EventBridge) | Temporal | 🔲 Yet to start |

**Definition of done:** A multi-step flow **survives a worker crash mid-execution** and resumes correctly (kill the worker, watch it recover, no duplicate side effects).
**Claude-assist:** ask Claude to explain Temporal's determinism rules and review your workflow code for determinism violations — this is the #1 Temporal footgun.

---

## Week 7 — Sep 14–20 · The 3 action types + typed I/O
**Goal:** Complete the action catalog and let data flow between steps.
**Technologies:** AWS SES (MailHog locally), JSONata, JSON Schema.

| Deliverable | Technology | State |
|-------------|-----------|-------|
| `action.http` finalized (uses connector interpreter) | undici | 🔲 Yet to start |
| `action.email` (SES in prod, MailHog in dev) | AWS SES | 🔲 Yet to start |
| `action.dataTransform` (map/transform between steps) | JSONata | 🔲 Yet to start |
| Typed input/output passing: resolve `{{node-id.output.field}}` references between actions | JSONata | 🔲 Yet to start |
| ActionType catalog (defines IO schema + handler per type) | Postgres | 🔲 Yet to start |

**Definition of done:** A flow chains HTTP → DataTransform → Email, passing the HTTP response into the email body, running durably.
**Claude-assist:** Claude can generate the JSON Schemas for each action's config + the reference-resolution logic.

---

## Week 8 — Sep 21–27 · Visual flow builder UI
**Goal:** The client-side flow composer — the heart of the "configure from the client" requirement.
**Technologies:** React Flow (@xyflow/react), @rjsf (schema-driven forms), Zod, React Query.

| Deliverable | Technology | State |
|-------------|-----------|-------|
| React Flow canvas: add/connect/delete nodes, layout | @xyflow/react | 🔲 Yet to start |
| Per-node config panel driven by the action/connector JSON Schema | @rjsf/core | 🔲 Yet to start |
| Save canvas → new **FlowVersion**; load existing flow into canvas | React Query | 🔲 Yet to start |
| Draft vs. Active flow state + activate/deactivate | NestJS | 🔲 Yet to start |
| Inline validation (Zod) before save | Zod | 🔲 Yet to start |

**Definition of done:** You build a 3-step flow entirely in the UI, save it, reload it, and run it.
**Claude-assist:** have Claude scaffold the React Flow canvas + the rjsf config panel wiring; this is broad UI work where code-gen saves the most time.

---

## Week 9 — Sep 28–Oct 4 · OAuth2 + real connectors + connector authoring
**Goal:** Real third-party auth and the ability to author a connector from the client (requirement #1).
**Technologies:** OAuth2 (simple-oauth2 or custom), Slack + ServiceNow (or Teams) APIs, Secrets Manager.

| Deliverable | Technology | State |
|-------------|-----------|-------|
| OAuth2 authorization-code flow + token storage + auto-refresh | simple-oauth2, Secrets Manager | 🔲 Yet to start |
| Connection management UI (connect/disconnect, status) | React | 🔲 Yet to start |
| **2 real connectors** as declarative specs (e.g., Slack + ServiceNow) | connector spec | 🔲 Yet to start |
| Connector-authoring UI (create/edit a connector spec via form) | @rjsf | 🔲 Yet to start |

**Definition of done:** Build a flow using a real **OAuth-connected Slack** connector, triggered by a webhook, running durably end-to-end.
**Claude-assist:** Claude can draft the declarative specs for Slack/ServiceNow operations from their API docs and the OAuth refresh logic.

---

## Week 10 — Oct 5–11 · Multi-tenancy, RBAC, security baseline
**Goal:** Real orgs, roles, and tenant isolation enforced everywhere.
**Technologies:** Postgres RLS (or app-level tenant guard), NestJS guards, Redis rate-limiting, AWS WAF.

| Deliverable | Technology | State |
|-------------|-----------|-------|
| IAM model: Membership, Role, Permission, RolePermission | Postgres | 🔲 Yet to start |
| Tenant isolation by `org_id` (Postgres RLS or enforced guard on every query) | Postgres RLS | 🔲 Yet to start |
| RBAC guards across control-plane endpoints (e.g., `flow:execute`, `connector:author`) | NestJS guards | 🔲 Yet to start |
| Per-tenant rate limiting (token bucket) on `/ingest` + basic WAF rules | Redis, AWS WAF | 🔲 Yet to start |
| Resolved-permission cache per session | Redis | 🔲 Yet to start |

**Definition of done:** A user in Org A cannot see or trigger anything in Org B; roles gate actions; webhook floods are rate-limited.
**Claude-assist:** ask Claude to write a test suite that *attempts* cross-org access and asserts it's blocked (negative tests).

---

## Week 11 — Oct 12–18 · Basic AI assistant (thin slice)
**Goal:** A Claude chat that scaffolds a flow with a confirm-before-write guardrail. Scoped small on purpose.
**Technologies:** Claude Messages API (`@anthropic-ai/sdk`), Sonnet 4.6, tool use, structured outputs, prompt caching, SSE.

| Deliverable | Technology | State |
|-------------|-----------|-------|
| Agentic loop with tools: `list_connectors`, `create_flow`, `add_action`, `query_logs` | Claude tool use | 🔲 Yet to start |
| Structured output (json_schema) so generated flow graphs validate before any write | Claude structured outputs | 🔲 Yet to start |
| **Propose → confirm** guardrail: AI returns a preview/diff; DB write only on user confirm | NestJS | 🔲 Yet to start |
| SSE chat panel in the SPA | React, SSE | 🔲 Yet to start |
| Prompt caching of system prompt + connector catalog + tool defs; capture `usage` for cost | Claude prompt caching | 🔲 Yet to start |

**Definition of done:** "Build a flow that posts new ServiceNow incidents to Slack" produces a valid draft flow you confirm into existence; "summarize this flow's recent failures" returns a useful answer.
**Claude-assist:** you're literally building on Claude here — use the `claude-api` skill / official SDK docs for the tool-use loop and structured-output shapes. Keep model = Sonnet 4.6 for now.

---

## Week 12 — Oct 19–25 · Home dashboard, logs viewer, analytics rollups
**Goal:** The user-facing visibility surface (requirements #7, #8).
**Technologies:** CloudWatch (structured logs), Postgres rollup tables, Recharts.

| Deliverable | Technology | State |
|-------------|-----------|-------|
| **Home dashboard:** running vs. draft flows, recent runs, success/fail counts | React, Recharts | 🔲 Yet to start |
| **Logs viewer:** per-flow / per-action execution logs from FlowRun/ActionRun | React | 🔲 Yet to start |
| Analytics rollup worker → `analytics_daily`; Redis live counters | Postgres, Redis | 🔲 Yet to start |
| Structured app logging → CloudWatch; PII/secret redaction at write time | CloudWatch | 🔲 Yet to start |

**Definition of done:** Home shows real per-org flow/run stats; clicking a run shows its step-by-step logs; dashboards read rollups, never scan raw runs.
**Claude-assist:** Claude can generate the rollup aggregation query + the Recharts dashboard components.

---

## Week 13 — Oct 26–Nov 1 · Integration, end-to-end polish & feature freeze
**Goal:** Wire it all together, fix the seams, and declare feature-complete.
**Technologies:** OpenTelemetry (basic), everything above.

| Deliverable | Technology | State |
|-------------|-----------|-------|
| End-to-end smoke test of every user journey on AWS | manual + scripts | 🔲 Yet to start |
| Fix integration seams (auth ↔ RBAC ↔ execution ↔ logs ↔ AI) | — | 🔲 Yet to start |
| Basic OpenTelemetry traces (api → queue → worker) | OpenTelemetry | 🔲 Yet to start |
| UX rough-edge cleanup; empty/error states | React | 🔲 Yet to start |
| README + short runbook (how to deploy, env vars, how to recover) | docs | 🔲 Yet to start |
| **🎯 FEATURE FREEZE** — meets the "Definition of fully working" above | — | 🔲 Yet to start |

**Definition of done:** Every item in the **Nov 1 bar** checklist passes on the deployed app. From here, no new features until Jan — only tests, fixes, and hardening.

---

# 🎯 Milestone — Feature-complete: Monday Nov 2, 2026

If you hit the freeze: the remaining 8 weeks are pure quality. If you slipped 1–2 weeks: pull from the **cut-list** below and still freeze by mid-Nov, then compress testing.

---

# Testing & Hardening — Week by Week (Nov 2 → Dec 31)

## Week 14 — Nov 2–8 · Test harness + core unit/integration tests
**Tech:** Vitest/Jest, Supertest.

| Deliverable | State |
|-------------|-------|
| Test infra + coverage reporting in CI | 🔲 Yet to start |
| Unit tests: template engine, connector interpreter, reference resolution, idempotency, RBAC | 🔲 Yet to start |
| Integration tests: flow CRUD, trigger → enqueue → run path | 🔲 Yet to start |

## Week 15 — Nov 9–15 · End-to-end + workflow-correctness tests
**Tech:** Playwright, Temporal test framework.

| Deliverable | State |
|-------------|-------|
| Playwright E2E for the critical journeys (signup → build → trigger → logs) | 🔲 Yet to start |
| Temporal replay/determinism tests; retry + DLQ behavior verified | 🔲 Yet to start |
| Negative tests: cross-org access, RBAC denials, malformed webhooks | 🔲 Yet to start |

## Week 16 — Nov 16–22 · Load & resilience testing
**Tech:** k6 or Artillery.

| Deliverable | State |
|-------------|-------|
| Load test `/ingest` + fan-out; measure queue depth, worker throughput, latency | 🔲 Yet to start |
| Chaos: kill workers / drop DB connections mid-run; verify recovery + no double-runs | 🔲 Yet to start |
| Tune autoscaling (ALB requests + SQS depth) under load | 🔲 Yet to start |

## Week 17 — Nov 23–29 · Security hardening
**Tech:** AWS IAM review, dependency audit, the `security-review` skill.

| Deliverable | State |
|-------------|-------|
| Webhook abuse hardening (rate-limit, signature, replay) verified | 🔲 Yet to start |
| Secrets audit: nothing sensitive in logs/DB/AI context; KMS + least-privilege IAM | 🔲 Yet to start |
| Multi-tenant isolation + RBAC fuzzing; dependency/CVE audit | 🔲 Yet to start |

## Week 18 — Nov 30–Dec 6 · AI deepening (the buffer polish)
**Tech:** Claude (Haiku/Opus routing, Batch API).

| Deliverable | State |
|-------------|-------|
| Expand AI tools (connector authoring, richer log/failure analysis) | 🔲 Yet to start |
| Model routing (Haiku classify / Sonnet build / Opus debug) + prompt-cache tuning | 🔲 Yet to start |
| Per-org token budgets, cost caps + alerting; Batch API for overnight analytics | 🔲 Yet to start |

## Week 19 — Dec 7–13 · Observability completion (+ optional service split)
**Tech:** OpenTelemetry, Grafana LGTM / Grafana Cloud.

| Deliverable | State |
|-------------|-------|
| Full traces/metrics/dashboards + alerting (trace_id propagated across the queue) | 🔲 Yet to start |
| *(Optional learning milestone)* carve `execution-engine` into its own ECS service | 🔲 Yet to start |
| OpenSearch for log search *only if* CloudWatch search is too weak | 🔲 Yet to start |

## Week 20 — Dec 14–20 · Performance, cost & operability
| Deliverable | State |
|-------------|-------|
| Query/index tuning; cache hit-rate tuning; right-size Fargate/RDS | 🔲 Yet to start |
| Log retention tiers; AI cost review; backups + a restore drill | 🔲 Yet to start |
| Runbook + disaster-recovery notes finalized | 🔲 Yet to start |

## Weeks 21–22 — Dec 21–31 · Buffer, docs, demo
| Deliverable | State |
|-------------|-------|
| Catch up on anything that slipped (this is real slack — use it) | 🔲 Yet to start |
| User + developer docs; architecture diagram | 🔲 Yet to start |
| Record a demo; short retro / lessons-learned note | 🔲 Yet to start |

---

# If you fall behind: the cut-list (priority order)

At ~13 hrs/week, slippage is likely somewhere. **Mark the cut item ⏭️ Moved to later phase and keep momentum.** Cut in this order — top items hurt least:

1. **Second real connector** → ship with **one** (Slack). (Wk 9)
2. **Connector-authoring UI** → author connectors as JSON in the DB by hand for now; UI moves to Dec. (Wk 9)
3. **Analytics charts** → reduce Home to plain counts; drop Recharts visuals. (Wk 12)
4. **Postgres RLS** → enforce tenant scoping in an app-level guard instead (functionally equivalent, less new infra). (Wk 10)
5. **Basic AI agent** → move the *entire* AI slice to the Nov–Dec buffer (you chose "polish in buffer" — this just shifts the thin slice too). (Wk 11)
6. **OpenTelemetry in Wk 13** → defer all tracing to Wk 19. (Wk 13)
7. **Optional service split** → drop entirely (it's a learning bonus, zero user value pre-scale). (Wk 19)

**Never cut (the spine):** auth + RBAC + tenant isolation · connector spec + HTTP interpreter · flow CRUD + durable execution · webhook + schedule triggers · per-flow logs · Home dashboard. These *are* the product.

---

# The weeks most likely to slip (watch these)

| Week | Why it's risky | Pre-emptive move |
|------|----------------|------------------|
| **Wk 5 — AWS deploy** | New to cloud; first real infra; many moving parts | Use AWS Copilot for the fastest Fargate path if Terraform stalls; lean on Claude for IaC + IAM |
| **Wk 6 — Temporal** | Determinism model is subtle; replay bugs are silent | Treat Temporal as a black box first (1 workflow type); have Claude review for determinism violations |
| **Wk 8 — Flow builder UI** | Broadest single UI surface | Timebox; ship a minimal canvas first, polish in Dec |
| **Wk 11 — AI agent** | Easy to over-scope | Keep to "build a flow from chat" + confirm; everything else is Dec |

---

*Tracker initialized — all items 🔲 Yet to start. Update States as you go; revisit the cut-list at the end of each month against your actual velocity.*
