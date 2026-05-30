# SMS & Messaging Gateway — Phased Development Plan

> Project: 387-sms-messaging-gateway · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and `data-model-suggestion-1.md`. The database design adopts **Data Model 1 (Entity-Centric Normalized Relational)** — it is the recommended fit for a production gateway with multi-carrier failover, complex routing, and multi-jurisdiction compliance (full referential integrity, standard SQL for carrier/DLR/compliance analytics, indexed opt-out enforcement on the hot path).

---

## Core Requirements Summary

**What it does:** An AI-native, open-source SMS gateway that aggregates connections to multiple carriers (via SMPP 3.4 and carrier REST APIs) and routes each message through the optimal path based on cost, delivery speed, and reliability. It exposes a unified REST API (and an inbound SMPP ESME interface) for sending/receiving SMS, handles delivery receipts (DLRs) and inbound (MO) messages via reliable webhooks, manages sender IDs and number provisioning, and enforces cross-market compliance (10DLC/TCR, India DLT, GDPR/TCPA opt-out).

**Primary personas:**
- *Product engineers* needing OTP, transactional, and conversational SMS with a clean REST API.
- *Enterprise telecom integrators* needing SMPP-native multi-carrier routing and self-hosting.
- *Compliance/ops teams* needing consent records, opt-out enforcement, and audit trails.

**Key differentiators (the AI-native + transparency edge):**
- **Route transparency** — every message records *why* it took a route (matched rule, carrier chain walked, failover reason, cost).
- **DLR accuracy classification** — handset vs. network vs. intermediate vs. synthetic DLRs, addressing the unaddressed 2026 "fake DLR" problem.
- **Reliable webhooks** — ordered, deduplicated, signed, with retry + replay.
- **Compliance automation** — opt-out enforcement before every send; 10DLC/DLT registration workflows.
- **AI augmentation** — ML carrier selection, DLR-accuracy scoring, fraud/content moderation, latency/congestion prediction (surfaced as `ai_suggestions`, advisory-first).

**Deployment model:** Hybrid — self-hostable via Docker Compose (Postgres + Redis + API + workers + SMPP gateway) and cloud-hosted multi-tenant. No mandatory frontend for MVP; an optional read-only dashboard ships late.

**Integration surface:** Carrier SMPP SMSCs (port 2775), carrier REST APIs, HLR/number-lookup providers, The Campaign Registry (10DLC), India DLT portals, customer webhook endpoints, LLM provider for AI suggestions.

**Standards the build must honour:** SMPP 3.4, 3GPP TS 23.038 (GSM-7/UCS-2 encoding + segmentation), ITU-T E.164 (number format), OpenAPI 3.1 (published spec), OAuth 2.0 / API-key auth (RFC 6749), 10DLC/TCR + India DLT, GDPR/TCPA opt-out, HMAC-SHA256 webhook signing, NIST SP 800-63B disclosure for OTP.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | **Python 3.12** | Heavy carrier-protocol + ML/LLM workload; mature SMPP (`smpplib`), phone-number (`phonenumbers`), and async ecosystems. The AI-native features are first-class, not bolted on. |
| API framework | **FastAPI** | Async (essential for fan-out carrier I/O and webhooks), generates an **OpenAPI 3.1** spec automatically per `standards.md`, Pydantic v2 request/response validation. |
| ASGI server | **Uvicorn** behind **Gunicorn** workers | Standard production combo for FastAPI; multiple workers for throughput. |
| Database | **PostgreSQL 16** | Data Model 1 requires FK integrity, `TEXT[]`/GIN indexes (supported countries, capabilities), partial indexes (opt-out hot path), JSONB (HLR/provider responses), and **range partitioning** (messages, delivery_events, inbound_messages, audit_log). |
| Migrations | **Alembic** | Tracks the 14-table schema evolution; required by Definition of Done. |
| ORM / DB access | **SQLAlchemy 2.0 (async)** + raw SQL for hot-path dispatch reads | ORM for CRUD/admin; tuned raw queries for the latency-critical send path (routing-rule match, opt-out check). |
| Cache / queue / rate-limit | **Redis 7** | Carrier health cache, per-tenant token-bucket rate limiting, idempotency keys, and as the broker for the task queue. |
| Task queue | **Celery** (Redis broker) | Async workloads: message dispatch, carrier failover, DLR ingestion, webhook delivery with retry/backoff, scheduled sends, compliance polling. Beat for periodic health checks. |
| SMPP | **smpplib** (async wrapper) | SMPP 3.4 client binds (transmitter/receiver/transceiver) to carrier SMSCs; also basis for the inbound ESME server. |
| Phone numbers | **phonenumbers** (Google libphonenumber port) | E.164 validation/normalisation, country + region inference, number-type heuristics. |
| SMS encoding | Custom `encoding` module | GSM-7 vs UCS-2 detection and segment counting per 3GPP TS 23.038 (160/153 GSM-7, 70/67 UCS-2 with UDH). |
| LLM / AI | **Provider-agnostic client** (OpenAI-compatible), pluggable | Carrier-selection scoring, DLR-accuracy heuristics, content moderation, fraud signals. Advisory by default; gated behind config. |
| Auth | **API keys (hashed) + OAuth 2.0 client-credentials** | Per `standards.md`; API key on the hot path (prefix-indexed lookup), OAuth for dashboard/partner integrations. |
| Webhook signing | **HMAC-SHA256** | `standards.md` + Data Model 1 `webhook_signing_secret`. Detached-signature header + timestamp to prevent replay. |
| Testing | **pytest** + **pytest-asyncio** + **testcontainers** (Postgres/Redis) + **respx**/**httpx mock** | Unit + integration (real ephemeral PG/Redis) + mocked carrier HTTP/SMPP. |
| Lint / format / types | **ruff** (lint+format) + **mypy** (strict) | Single fast toolchain; type checking required by DoD. |
| Package / project | **uv** + `pyproject.toml` | Fast, reproducible dependency resolution and locking. |
| Containerisation | **Docker** + **docker-compose** | Self-hosted target (TeleOSS-style) and reproducible dev. |
| Frontend (optional, late) | **Next.js + Tremor** read-only dashboard | Delivery/cost/route-transparency analytics; not required for MVP. |

### Project Structure

```
sms-gateway/
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml              # postgres, redis, api, worker, beat, smpp-gateway
├── alembic.ini
├── .env.example
├── openapi.json                    # exported from FastAPI for SDK/Swagger
├── migrations/                     # Alembic
│   └── versions/
├── src/
│   └── sms_gateway/
│       ├── __init__.py
│       ├── config.py               # Pydantic Settings (env-driven)
│       ├── main.py                 # FastAPI app factory, router mounting
│       ├── db.py                   # async engine, session, partition helpers
│       ├── redis_client.py
│       ├── celery_app.py
│       ├── logging.py              # structured JSON logs + trace_id
│       ├── errors.py               # typed error hierarchy + RFC 7807 responses
│       ├── models/                 # SQLAlchemy models (Data Model 1, 14 tables)
│       │   ├── tenant.py user.py carrier.py phone_number.py sender_id.py
│       │   ├── routing_rule.py compliance.py opt_out.py
│       │   ├── message.py delivery_event.py inbound_message.py
│       │   └── webhook.py ai_suggestion.py audit_log.py
│       ├── schemas/                # Pydantic request/response DTOs
│       ├── auth/                   # api-key + oauth2, dependency injectors
│       ├── encoding/               # GSM-7/UCS-2 detection + segmentation
│       ├── numbers/                # E.164 normalisation, number-type inference
│       ├── routing/                # rule matching + carrier-chain selection
│       ├── carriers/               # carrier adapter abstraction
│       │   ├── base.py             # CarrierAdapter protocol
│       │   ├── smpp_adapter.py
│       │   ├── rest_adapter.py
│       │   └── registry.py         # protocol → adapter factory
│       ├── dispatch/               # send orchestration, failover, cost calc
│       ├── dlr/                    # DLR ingestion + classification
│       ├── inbound/                # MO handling, keyword detection (STOP/HELP)
│       ├── compliance/             # opt-out, 10DLC/TCR, DLT, HLR lookup
│       ├── webhooks/               # signing, ordered delivery, dedup, retry
│       ├── ai/                     # LLM client + suggestion generators
│       ├── analytics/              # delivery/cost/route-transparency queries
│       ├── smpp_server/            # inbound ESME server (enterprise integration)
│       └── api/
│           └── v1/                 # routers: messages, batch, numbers, carriers,
│                                   #   sender_ids, routing, compliance, webhooks,
│                                   #   inbound, analytics, suggestions, dlr_callback
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── e2e/
│   └── fixtures/                   # sample DLRs, SMPP PDUs, webhook payloads
└── scripts/
    ├── seed_dev.py
    └── export_openapi.py
```

Modules are grouped by concern, not by phase; every later phase only adds files/routers.

---

## Phase 1: Foundation, Config & Multi-Tenant Data Model

### Purpose
Stand up the project skeleton, configuration, database connectivity, the full Data Model 1 schema (so later phases never restructure), and structured logging. After this phase the app boots, migrations create all 14 tables, and health checks pass. Nothing sends yet.

### Tasks

#### 1.1 — Project scaffold, config, tooling
**What:** Initialise the repo, dependency management, FastAPI app factory, and settings.

**Design:**
- `pyproject.toml` with deps from the table above; `uv.lock` committed.
- `config.py` using Pydantic `BaseSettings`:
```python
class Settings(BaseSettings):
    database_url: str
    redis_url: str = "redis://localhost:6379/0"
    api_env: Literal["dev","test","prod"] = "dev"
    secret_encryption_key: str               # Fernet key for credential refs
    default_rate_limit_per_second: int = 100
    failover_mode: Literal["fail_open","fail_closed"] = "fail_closed"
    webhook_max_retries: int = 5
    ai_enabled: bool = False
    ai_base_url: str | None = None
    ai_api_key: str | None = None
    model_config = SettingsConfigDict(env_prefix="SMSGW_", env_file=".env")
```
- `main.py` exposes `create_app()` mounting `/v1` routers, registering exception handlers (RFC 7807 problem+json), and a `GET /healthz` returning `{status, db, redis, version}`.
- `errors.py`: `GatewayError` base → `ValidationError`, `RateLimitError`, `OptedOutError`, `NoRouteError`, `CarrierError`, `ComplianceBlockError`, each mapping to an HTTP status + `type` URI.

**Testing:**
- `Unit: Settings loads from env with SMSGW_ prefix → correct typed values, defaults applied`.
- `Unit: missing DATABASE_URL → ValidationError naming the field`.
- `Integration: GET /healthz with PG+Redis up → 200, status="ok"`.
- `Integration: GET /healthz with Redis down → 503, redis="down"`.

#### 1.2 — Database engine, sessions, partition helpers
**What:** Async SQLAlchemy engine/session and helpers to create monthly partitions.

**Design:**
- `db.py`: async engine, `async_session` factory, `get_session()` FastAPI dependency.
- `ensure_partition(table, month)` creating `messages_YYYYMM` etc. via `CREATE TABLE ... PARTITION OF ... FOR VALUES FROM (...) TO (...)`. A Celery beat task pre-creates next month's partitions.

**Testing:**
- `Integration (testcontainers PG): ensure_partition('messages', 2026-06) → child partition exists; inserting a June row lands in it`.
- `Integration: calling ensure_partition twice → idempotent, no error`.

#### 1.3 — Full schema migration (Data Model 1)
**What:** Alembic migration creating all 14 tables, indexes, CHECK constraints, and the 4 partitioned parents.

**Design:** Translate `data-model-suggestion-1.md` verbatim: `tenants, users, carriers, phone_numbers, sender_ids, routing_rules, compliance_registrations, opt_outs, messages (partitioned), delivery_events (partitioned), inbound_messages (partitioned), webhooks, ai_suggestions, audit_log (partitioned)`. Include all GIN/partial/composite indexes specified there. SQLAlchemy models mirror each table 1:1 in `models/`.

**Testing:**
- `Integration: alembic upgrade head → all 14 tables + expected indexes present (introspect pg_indexes)`.
- `Integration: insert carrier with protocol='smpp' and bad bind_mode → CHECK violation`.
- `Integration: alembic downgrade base → upgrade head round-trips cleanly`.

#### 1.4 — Structured logging & request tracing
**What:** JSON logs with a `trace_id` propagated onto messages.

**Design:** Middleware assigns/echoes `X-Trace-Id`; `logging.py` binds it to a contextvar so every log line and the `messages.trace_id` column share it.

**Testing:**
- `Unit: middleware with no header → generates UUID trace_id`.
- `Unit: middleware with X-Trace-Id → echoes it in response + log context`.

---

## Phase 2: Authentication, Tenancy & Rate Limiting

### Purpose
Make the API multi-tenant and safe to expose: API-key issuance/verification, OAuth2 client-credentials, per-tenant rate limiting, and the audit log. Every subsequent endpoint depends on the authenticated `tenant_id` and actor identity.

### Tasks

#### 2.1 — API key auth (hot path)
**What:** Issue and verify hashed API keys per `users` (Data Model 1: `api_key_hash`, `api_key_prefix`).

**Design:**
- Key format `sk_<env>_<prefix8>_<secret32>`. Store `api_key_prefix` (indexed) + `api_key_hash` (argon2/bcrypt of full key).
- `auth/api_key.py`: dependency `current_principal()` → resolves by prefix, verifies hash, returns `Principal(tenant_id, user_id, role)`. Caches positive lookups in Redis (short TTL).
- `POST /v1/api-keys` (owner/admin) returns the secret **once**.

**Testing:**
- `Unit: valid key → Principal with correct tenant/role`.
- `Unit: key with right prefix but wrong secret → 401, no timing leak (constant-time compare)`.
- `Integration: revoked/inactive user key → 401`.

#### 2.2 — OAuth 2.0 client-credentials
**What:** Token endpoint for partner/dashboard access (RFC 6749).

**Design:** `POST /v1/oauth/token` (grant_type=client_credentials) → JWT access token with `tenant_id` + scopes (`messages:write`, `numbers:read`, ...). `current_principal()` accepts either API key or Bearer JWT.

**Testing:**
- `Unit: valid client creds → JWT with expected scopes/exp`.
- `Integration: Bearer token lacking messages:write → 403 on POST /v1/messages`.

#### 2.3 — Per-tenant rate limiting
**What:** Token-bucket limiter using `tenants.rate_limit_per_second`.

**Design:** Redis Lua token-bucket keyed `rl:{tenant_id}`; dependency raises `RateLimitError` (429) with `Retry-After`. Applied to send endpoints.

**Testing:**
- `Integration (real Redis): N+1 sends within 1s at limit N → last call 429`.
- `Integration: tokens refill after window → subsequent call 200`.

#### 2.4 — Audit logging
**What:** Write `audit_log` entries for mutating actions.

**Design:** `audit(actor, action, entity_type, entity_id, changes, ip, ua)` helper invoked from service layers; `actor_type` ∈ user/system/api_key/carrier/ai.

**Testing:**
- `Integration: creating a carrier → audit_log row with action="carrier.create", correct actor_type and changes diff`.

---

## Phase 3: Encoding, Numbers & the Carrier Adapter Abstraction

### Purpose
Build the protocol-correct primitives the dispatch engine depends on: E.164 handling, GSM-7/UCS-2 segmentation, and a uniform `CarrierAdapter` interface implemented by SMPP and REST backends. This is the heart of "hides carrier complexity" and ships before dispatch so dispatch has a stable contract.

### Tasks

#### 3.1 — E.164 number handling
**What:** Normalise/validate numbers and infer country + type (ITU-T E.164).

**Design:**
```python
@dataclass
class ParsedNumber:
    e164: str          # "+14155550123"
    country_code: str  # "US"
    is_valid: bool
    number_type: str   # mobile/fixed/voip/unknown
def parse_number(raw: str, default_region: str | None = None) -> ParsedNumber
```
Backed by `phonenumbers`. All `from_number`/`to_number`/`opt_out.phone_number` stored as E.164.

**Testing:**
- `Unit: "(415) 555-0123" with region US → +14155550123, US, valid`.
- `Unit: "+9999" → is_valid=False`.
- `Unit: E.164 >15 digits → invalid`.

#### 3.2 — SMS encoding & segmentation (3GPP TS 23.038)
**What:** Choose encoding and compute segments.

**Design:**
```python
@dataclass
class EncodedBody:
    encoding: Literal["gsm7","ucs2","binary"]
    segment_count: int
    data_coding: int      # SMPP data_coding value
def encode_body(text: str) -> EncodedBody
```
Rules: all chars in GSM-7 basic+extension → `gsm7` (≤160 single / 153 per concatenated segment); else `ucs2` (≤70 / 67). `data_coding` 0 for GSM-7, 8 for UCS-2.

**Testing:**
- `Unit: "Hello" → gsm7, 1 segment, data_coding=0`.
- `Unit: 161 ASCII chars → gsm7, 2 segments`.
- `Unit: emoji/Chinese → ucs2, data_coding=8, correct 70/67 boundary`.
- `Unit: GSM-7 extension char (€) counts as 2 septets`.

#### 3.3 — CarrierAdapter interface + REST adapter
**What:** Uniform send contract; REST implementation.

**Design:**
```python
class SendResult(TypedDict):
    accepted: bool
    provider_message_id: str | None
    error_code: str | None
    raw: dict
    latency_ms: int
class CarrierAdapter(Protocol):
    async def send(self, msg: OutboundMessage) -> SendResult: ...
    async def healthcheck(self) -> bool: ...
```
`RestCarrierAdapter` builds requests per `carriers.rest_*` config; auth types basic/bearer/api_key/oauth2; credentials decrypted from `rest_credentials_ref` (Fernet). Maps HTTP/provider errors to canonical `error_code`s.

**Testing:**
- `Integration (respx-mocked): 200 with message id → accepted=True, provider_message_id set`.
- `Integration: provider 400 invalid number → accepted=False, error_code="invalid_destination"`.
- `Integration: timeout → CarrierError surfaced for failover`.

#### 3.4 — SMPP carrier adapter (SMPP 3.4)
**What:** Bind to a carrier SMSC and submit_sm.

**Design:** `SmppCarrierAdapter` manages a pooled bind (transmitter/transceiver per `smpp_bind_mode`, port 2775). `send()` issues `submit_sm` with computed `data_coding`, concatenation via UDH for multi-segment, and returns the SMSC `message_id`. Decrypts `smpp_password_ref`. Connection supervisor reconnects on drop.

**Testing:**
- `Integration (mock SMPP server fixture): bind transceiver → submit_sm returns message_id → accepted=True`.
- `Unit: 2-segment GSM-7 → two submit_sm PDUs with correct UDH sar references`.
- `Integration: SMSC ESME_RTHROTTLED response → CarrierError(throttled) for failover`.

---

## Phase 4: Routing, Compliance Gate & Message Dispatch (Core Value)

### Purpose
The product's beating heart: accept a send request, enforce opt-out/compliance, match a routing rule, walk the carrier chain with failover, and persist full route transparency. After this phase the gateway sends real SMS end-to-end with auditable routing decisions.

### Tasks

#### 4.1 — Routing rule matcher
**What:** Select the best `routing_rules` row for a message.

**Design:** Query by `tenant_id, message_type, destination_country, is_active` ordered by `priority DESC` (the exact query in Data Model 1), with NULL fields acting as wildcards; optional time-window check. Returns the rule (and its `carrier_chain`) or `None` → falls back to active carriers by `priority` filtered on `supported_countries`/`supported_message_types`.

**Testing:**
- `Unit: OTP to IN with a country+type rule and a wildcard rule → higher-priority specific rule chosen`.
- `Unit: no rule matches → fallback carrier list ordered by priority, filtered by supported_countries GIN`.
- `Unit: outside time_window → rule skipped`.

#### 4.2 — Compliance & opt-out gate
**What:** Block disallowed sends before dispatch (TCPA/GDPR/10DLC).

**Design:** Pre-send checks in order: (1) opt-out lookup (the indexed `EXISTS` query from Data Model 1) → `OptedOutError` (422); (2) promotional to a market requiring registration without an approved `compliance_registration` → `ComplianceBlockError`; (3) destination country supported. NIST SP 800-63B disclosure attached to OTP responses.

**Testing:**
- `Integration: send to opted-out number → 422 OptedOutError, no carrier call, no message row in 'sent'`.
- `Integration: opt-out scoped to promotional → transactional still allowed`.
- `Unit: promotional to US without approved 10DLC campaign → ComplianceBlockError`.

#### 4.3 — Dispatch orchestrator with failover
**What:** Persist message, walk carrier chain, record route transparency.

**Design:**
- `POST /v1/messages` validates → `encode_body` → `parse_number` → idempotency check (`idempotency_key` unique per tenant) → enqueue Celery `dispatch_message(message_id)`; respond `202` with message resource (`status="queued"`).
- Worker: state machine `queued → routing → sent | failed`; for each carrier in chain call `adapter.send()`; on `CarrierError` increment `carrier_attempts`, set `failover_from_carrier_id`, try next. On success persist `carrier_id`, `provider_message_id`, `cost_per_segment_microcents`, `total_cost_microcents`, `sent_at`. `failover_mode` decides behaviour when chain exhausts (fail_closed → `failed`).
- Route-transparency: a structured record of `{matched_rule_id, chain, attempts:[{carrier_id, error_code, latency_ms}]}` stored on `messages.provider_response`/trace.

**Testing:**
- `Integration (mocked adapters): first carrier throttled, second accepts → status sent, carrier_id=second, failover_from set, attempts=2`.
- `Integration: all carriers fail, fail_closed → status failed, no partial send`.
- `Integration: duplicate idempotency_key → returns original message, single dispatch`.
- `E2E: POST /v1/messages → 202 → poll GET /v1/messages/{id} → sent with route transparency populated`.

#### 4.4 — Scheduled & batch sending
**What:** `scheduled_at` sends and `POST /v1/messages/batch`.

**Design:** Beat task sweeps `idx_messages_scheduled` for due rows → enqueue dispatch. Batch endpoint accepts ≤1000 recipients, creates N message rows sharing a batch id, enqueues per-recipient dispatch, returns accepted/rejected counts.

**Testing:**
- `Integration: scheduled_at in past on create → dispatched immediately`.
- `Integration: batch of 3 (1 opted-out) → 2 queued, 1 rejected with reason`.

---

## Phase 5: DLR Ingestion, Inbound (MO) & Webhook Delivery

### Purpose
Close the loop: ingest carrier delivery receipts (classified by accuracy), handle inbound messages with keyword detection, and deliver both to customers over reliable, signed, ordered, deduplicated webhooks — a stated differentiator.

### Tasks

#### 5.1 — DLR ingestion & classification
**What:** Receive DLRs (SMPP `deliver_sm` receipt + carrier REST callbacks) → `delivery_events`, update message status.

**Design:** `POST /v1/dlr/{carrier_id}` and SMPP receiver path normalise carrier-specific receipts to canonical `event_type` + `dlr_type` (handset/network/intermediate/synthetic per Data Model 1). Classification heuristic + optional AI accuracy score (Phase 7) flags likely-synthetic DLRs. Updates `messages.status`/`delivered_at`. Emits a webhook event.

**Testing:**
- `Unit: SMPP receipt "stat:DELIVRD" → event delivered, message.delivered_at set`.
- `Unit: carrier returns "delivered" within 50ms of submit → flagged dlr_type=synthetic (implausible)`.
- `Integration: REST DLR callback for unknown provider_message_id → 200, logged, no crash`.

#### 5.2 — Inbound (MO) handling + keyword detection
**What:** Receive inbound SMS, detect STOP/START/HELP, create opt-outs.

**Design:** SMPP `deliver_sm` (non-receipt) and carrier inbound REST callbacks → `inbound_messages`. Keyword matcher sets `is_opt_out`/`is_opt_in`/`is_help`; STOP creates/activates an `opt_outs` row (source=`recipient_reply`) *before* webhook delivery; START reactivates; HELP triggers a templated auto-reply. Conversation grouping via shared `conversation_id`.

**Testing:**
- `Unit: body "STOP" → is_opt_out=True; opt_outs row created/active`.
- `Unit: "UNSUBSCRIBE", "Stop" (case-insensitive) recognised`.
- `Integration: inbound then immediate send to same number → blocked by opt-out gate`.

#### 5.3 — Reliable webhook delivery
**What:** Signed, ordered, deduplicated, retried webhook delivery.

**Design:**
- Events (`message.sent`, `message.delivered`, `message.failed`, `message.inbound`) enqueued to Celery `deliver_webhook`.
- Signature header `X-Signature: t=<ts>,v1=<hmac_sha256(secret, ts + "." + body)>`; reject window prevents replay.
- **Ordering** per destination via a Redis sequence + per-`(webhook_id)` ordered queue; a later event isn't delivered before an earlier one for the same message.
- **Dedup** via event UUID; receiver-side idempotency documented; sender retries same event id.
- Retry with exponential backoff up to `webhooks.max_retries`; on exhaustion increment `failure_count`, set `disabled_at` after threshold.
- `POST /v1/webhooks/{id}/replay` re-emits historical events.

**Testing:**
- `Integration (mock receiver): valid event → POST with correct HMAC; receiver verifies signature`.
- `Integration: receiver returns 500 thrice then 200 → delivered after backoff, attempts recorded`.
- `Integration: two events for one message → arrive in causal order`.
- `Unit: tampered body → signature verification fails`.

---

## Phase 6: Number Management, Sender IDs, Carriers Admin & HLR Lookup

### Purpose
Give tenants self-service management of the assets dispatch relies on, plus number intelligence (HLR/MNP) that improves routing — a key Infobip-class differentiator.

### Tasks

#### 6.1 — Carrier, number & sender-ID CRUD
**What:** Admin endpoints for `carriers`, `phone_numbers`, `sender_ids`.

**Design:** CRUD routers with role-gated writes; carrier credentials encrypted at rest (Fernet, stored as `*_ref`). Number provisioning records `number_type`, `capabilities`, `country_code`. Sender-ID registration tracks `status` + `registration_details`. Carrier create triggers an immediate `healthcheck()`.

**Testing:**
- `Integration: create SMPP carrier → secrets stored encrypted, not returned in GET`.
- `Unit: invalid number_type → 422`.
- `Integration: deactivating a carrier removes it from routing fallback`.

#### 6.2 — HLR / number-lookup
**What:** Resolve serving network (MCC/MNC), ported/roaming status before dispatch.

**Design:** `POST /v1/lookup` and an internal `lookup(number)` used by routing. Pluggable HLR provider adapter; result cached (Redis TTL) and, on send, stored as `messages.hlr_result` JSONB (per Data Model 1). Routing can target `destination_network` using the resolved current network.

**Testing:**
- `Integration (mocked HLR): ported number → hlr_result.current_network differs from original, ported=True`.
- `Integration: cached lookup within TTL → no second provider call`.
- `Unit: lookup failure → routing proceeds on country only (graceful degradation)`.

#### 6.3 — Carrier health monitoring
**What:** Periodic health checks driving `health_status` and failover bias.

**Design:** Beat task runs `healthcheck()` per active carrier; updates `health_status`, `consecutive_failures`, `last_health_check_at`. Routing deprioritises `degraded` and skips `down` carriers.

**Testing:**
- `Integration: carrier failing 3 checks → health_status=degraded then down; excluded from chain`.

---

## Phase 7: Compliance Automation & AI-Native Augmentation

### Purpose
Deliver the AI-native and compliance-automation differentiators: 10DLC/DLT registration workflows, and ML/LLM suggestions for carrier selection, DLR accuracy, fraud, and content moderation — surfaced advisory-first via `ai_suggestions`.

### Tasks

#### 7.1 — 10DLC (TCR) & India DLT registration workflows
**What:** Manage brand/campaign and DLT entity/template registrations.

**Design:** `compliance_registrations` CRUD + state machine `pending → submitted → approved/rejected/expired`. TCR adapter submits brand+campaign; DLT adapter handles entity+template with the post-2025 category suffix (`P/S/T/G`). Beat task polls external status. Approved registrations unblock the compliance gate (4.2).

**Testing:**
- `Integration (mock TCR): submit campaign → external_id stored, status submitted; poll → approved`.
- `Unit: DLT promotional template without category P/S/T/G → validation error`.

#### 7.2 — LLM client & suggestion framework
**What:** Provider-agnostic LLM client writing `ai_suggestions`.

**Design:** `ai/client.py` (OpenAI-compatible, gated by `ai_enabled`). Generators produce `ai_suggestions` rows with `suggestion_type`, `confidence`, `entity_type/id`, `suggestion_data`. **Advisory by default** — suggestions never silently override routing. `GET /v1/suggestions`, `POST /v1/suggestions/{id}/accept|dismiss`.

System prompt template (carrier selection example):
```
You are a routing analyst. Given recent per-carrier delivery rate, latency p95,
and cost for destination {country}/{network} and message_type {type}, recommend a
carrier ordering. Return JSON {ranking:[carrier_id...], rationale, confidence:0-1}.
```

**Testing:**
- `Integration (mocked LLM): generator → ai_suggestion row, confidence in [0,1]`.
- `Unit: ai_enabled=False → generators no-op, zero LLM calls`.
- `Unit: malformed LLM JSON → suggestion skipped, error logged, no crash`.

#### 7.3 — Content moderation, fraud & DLR-accuracy scoring
**What:** Pre-send content/fraud flags and post-hoc DLR scoring.

**Design:** Pre-send hook scores body for carrier-filter triggers and sender/number fraud signals → high-confidence `fraud_detection`/`content_moderation` suggestions can soft-block when configured. DLR-accuracy generator scores carriers' synthetic-DLR likelihood, feeding 5.1's classification.

**Testing:**
- `Unit: known spam pattern → content_moderation suggestion with high confidence`.
- `Integration: carrier with implausibly fast "delivered" DLRs → low DLR-accuracy score surfaced`.

---

## Phase 8: SMPP Server (Enterprise Inbound), Analytics, Dashboard & Hardening

### Purpose
Round out enterprise integration (SMPP ESME ingress), expose route-transparency/cost analytics, an optional read-only dashboard, and production hardening + published OpenAPI spec.

### Tasks

#### 8.1 — Inbound SMPP server (ESME)
**What:** Let enterprises submit via SMPP instead of REST (TeleOSS-style).

**Design:** `smpp_server/` accepts ESME binds (auth against API-key-derived credentials), receives `submit_sm`, maps to the same dispatch pipeline as 4.3, returns `submit_sm_resp` with the gateway message id; delivers DLRs back as `deliver_sm` receipts.

**Testing:**
- `Integration (smpp client fixture): bind transmitter → submit_sm → submit_sm_resp with message_id → message dispatched`.
- `Integration: bind with bad credentials → ESME_RBINDFAIL`.

#### 8.2 — Analytics & route-transparency API
**What:** Delivery/cost/carrier-performance aggregates and per-message route explanation.

**Design:** `GET /v1/analytics/delivery`, `/v1/analytics/carriers`, `/v1/analytics/cost` (time-range, leveraging partitioned tables); `GET /v1/messages/{id}/route` returns the matched rule, carrier chain, attempts, DLR-type breakdown, and total cost.

**Testing:**
- `Integration: seeded messages → delivery analytics counts match sent/delivered/failed`.
- `Integration: carrier analytics distinguishes handset vs network DLR rates`.
- `Integration: /route returns failover chain + reasons for a multi-attempt message`.

#### 8.3 — Optional read-only dashboard
**What:** Next.js + Tremor dashboard over the analytics API.

**Design:** Auth via OAuth2 token; pages: Overview, Carrier Performance, Route Transparency, Compliance status, AI Suggestions. Read-only; behind a Compose profile.

**Testing:**
- `E2E (Playwright, smoke): login → overview renders sent/delivered tiles from API`.

#### 8.4 — Hardening, OpenAPI export & Compose
**What:** Production readiness.

**Design:** Export `openapi.json` via `scripts/export_openapi.py` (OpenAPI 3.1); finalize `docker-compose.yml` (postgres, redis, api, worker, beat, smpp-gateway, optional dashboard); rate-limit + auth on every external route; secrets via env; partition pre-creation beat task verified; `.env.example` documented.

**Testing:**
- `Integration: exported openapi.json validates against OpenAPI 3.1 schema and includes all /v1 routes`.
- `E2E: docker compose up → healthz green → send via REST → DLR webhook received signed`.
- `Integration: every /v1 mutating route rejects unauthenticated requests (401)`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Schema        ─── required by everything
    │
Phase 2: Auth, Tenancy, Rate Limit  ─── requires 1
    │
Phase 3: Encoding/Numbers/Adapters  ─── requires 1 (auth not needed for primitives)
    │
Phase 4: Routing, Compliance Gate,  ─── requires 2 + 3  (CORE VALUE SHIPS HERE)
         Dispatch + Failover
    │
Phase 5: DLR, Inbound, Webhooks     ─── requires 4
    ├── Phase 6: Numbers/Carriers/HLR Admin   ─── requires 4 (parallel with 5)
    │
Phase 7: Compliance Automation + AI ─── requires 4 (+6 for HLR-fed AI); parallel with 5
    │
Phase 8: SMPP Server, Analytics,    ─── requires 4,5,6 (analytics needs DLR data);
         Dashboard, Hardening             8.3 dashboard parallel with 8.1/8.2
```

**Parallelism opportunities:**
- Phase 3 can start alongside Phase 2 (primitives don't need auth).
- After Phase 4: Phases 5, 6, and 7 can be developed concurrently by separate workstreams.
- Within Phase 8: dashboard (8.3) parallels SMPP server (8.1) and analytics (8.2).

---

## Definition of Done (per phase)

1. All tasks in the phase implemented.
2. All unit and integration tests pass (`pytest`), including mocked-carrier and real-PG/Redis (testcontainers) suites.
3. `ruff check` and `ruff format --check` pass.
4. `mypy --strict` passes for new/changed modules.
5. `docker compose build` succeeds and the relevant services start cleanly.
6. The phase's headline capability works end-to-end (demonstrated by an E2E test where applicable).
7. New config options added to `config.py` and documented in `.env.example`.
8. New API endpoints appear in the auto-generated OpenAPI 3.1 spec (`scripts/export_openapi.py`) with request/response schemas.
9. Alembic migration(s) created for any schema change and verified to upgrade/downgrade cleanly.
10. Mutating endpoints write an `audit_log` entry and enforce auth + rate limits.
11. Standards honoured where relevant: E.164 numbers, GSM-7/UCS-2 segmentation (3GPP TS 23.038), SMPP 3.4 PDUs, HMAC-SHA256 webhook signatures, opt-out enforcement before every send.
```
