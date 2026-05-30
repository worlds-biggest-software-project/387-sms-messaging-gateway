# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: SMS Messaging Gateway · Created: 2026-05-26

## Philosophy

Every domain concept — carrier, phone number, routing rule, compliance registration, message, delivery receipt — lives in its own table with strict foreign key relationships. This approach mirrors how telecom systems have historically been built: each entity has a well-defined lifecycle, and relationships between entities (which carrier delivered which message, which number is registered for which 10DLC campaign) are expressed through foreign keys rather than embedded documents.

The normalized design is particularly well-suited for SMS gateways because the domain has clear, stable entity boundaries defined by telecom standards. A carrier connection has SMPP-specific fields (system_id, bind mode, port 2775) that are structurally different from REST API credentials. Phone numbers have distinct types (long code, short code, toll-free) with different regulatory requirements. Delivery receipts have a well-defined taxonomy (handset vs. network DLRs) that benefits from typed columns rather than freeform JSON.

This model prioritises query flexibility — finding all messages routed through a specific carrier, all numbers registered for 10DLC, or all delivery events for a message — through standard JOIN operations. The trade-off is more tables and more complex write paths, particularly for message dispatch where carrier selection, compliance checking, and opt-out verification all require cross-table lookups.

**Best for:** Teams building a production SMS gateway with complex routing rules, multi-carrier failover, and regulatory compliance requirements across multiple jurisdictions.

**Trade-offs:**
- Pro: Full referential integrity across carriers, numbers, routing rules, and compliance registrations
- Pro: Standard SQL queries for carrier performance analysis, DLR accuracy auditing, and compliance reporting
- Pro: Each entity has its own lifecycle and can be independently versioned, suspended, or archived
- Pro: Opt-out enforcement via direct index lookup before every send
- Con: Message dispatch requires multiple JOINs (routing rules → carrier → compliance → opt-out check)
- Con: 14 tables increases migration complexity when schema evolves
- Con: HLR lookup results stored per-message create data volume at scale
- Con: Carrier failover chain stored as UUID array in routing_rules — less queryable than a junction table

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| SMPP 3.4 | Carrier table models SMPP-specific fields: system_id, port (IANA 2775), bind_mode (transmitter/receiver/transceiver), data_coding on messages |
| 3GPP TS 23.038 | Message encoding column tracks GSM 7-bit (160 chars), UCS-2 (70 chars), or binary; segment_count derived from encoding |
| ITU-T E.164 | All phone number fields store E.164 format (+ prefix, max 15 digits); indexed for routing lookups |
| 10DLC / TCR | Compliance registrations table models brand and campaign registration with TCR IDs, use cases, and approval status |
| India DLT | Compliance registrations table models DLT entity/template registration with 2025 message category suffixes (-P, -S, -T, -G) |
| GDPR | Opt-outs table tracks consent withdrawal with source attribution; audit_log records all data access |
| TCPA | Opt-out enforcement via indexed lookup before every outbound message; source tracking for compliance proof |
| SHAKEN/STIR (RFC 8226) | Sender ID registration tracks authentication status for caller/sender ID verification |
| NIST SP 800-63B | Message type 'otp' flagged separately; AI suggestions include fraud detection for OTP abuse patterns |
| OAuth 2.0 (RFC 6749) | User API key authentication; carrier REST credentials support oauth2 auth type |
| OpenAPI 3.x | REST API carrier connections reference OpenAPI-compatible endpoint structures |
| GSMA IR.75 | Carrier table models direct connections and hubbing relationships; routing rules support MCC+MNC destination targeting |

---

## Entity Management

### tenants

```sql
CREATE TABLE tenants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  slug TEXT NOT NULL UNIQUE,
  status TEXT NOT NULL DEFAULT 'active'
    CHECK (status IN ('active','suspended','cancelled')),
  default_sender_id TEXT,
  failover_mode TEXT NOT NULL DEFAULT 'fail_closed'
    CHECK (failover_mode IN ('fail_open','fail_closed')),
  webhook_signing_secret TEXT NOT NULL,
  rate_limit_per_second INTEGER NOT NULL DEFAULT 100,
  default_message_type TEXT NOT NULL DEFAULT 'transactional'
    CHECK (default_message_type IN ('otp','transactional','promotional')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### users

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  email TEXT NOT NULL,
  name TEXT NOT NULL,
  role TEXT NOT NULL DEFAULT 'developer'
    CHECK (role IN ('owner','admin','developer','analyst','billing','service_account')),
  api_key_hash TEXT,
  api_key_prefix TEXT,
  is_active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);
CREATE INDEX idx_users_api_key ON users(api_key_prefix) WHERE api_key_prefix IS NOT NULL;
```

---

## Carrier & Number Management

### carriers

```sql
CREATE TABLE carriers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  name TEXT NOT NULL,
  protocol TEXT NOT NULL CHECK (protocol IN ('smpp','rest')),
  -- SMPP configuration (SMPP 3.4, IANA port 2775)
  smpp_host TEXT,
  smpp_port INTEGER DEFAULT 2775,
  smpp_system_id TEXT,
  smpp_password_ref TEXT,
  smpp_system_type TEXT,
  smpp_bind_mode TEXT
    CHECK (smpp_bind_mode IN ('transmitter','receiver','transceiver')),
  -- REST API configuration
  rest_base_url TEXT,
  rest_auth_type TEXT
    CHECK (rest_auth_type IN ('basic','bearer','api_key','oauth2')),
  rest_credentials_ref TEXT,
  -- Throughput and routing
  throughput_per_second INTEGER NOT NULL DEFAULT 50,
  priority INTEGER NOT NULL DEFAULT 0,
  is_active BOOLEAN NOT NULL DEFAULT true,
  supported_countries TEXT[] NOT NULL DEFAULT '{}',
  supported_message_types TEXT[] NOT NULL DEFAULT '{otp,transactional,promotional}',
  -- Cost tracking
  cost_per_segment_microcents INTEGER,
  -- DLR quality
  dlr_accuracy TEXT NOT NULL DEFAULT 'unknown'
    CHECK (dlr_accuracy IN ('handset_confirmed','network_only','mixed','unknown')),
  -- Health monitoring
  health_status TEXT NOT NULL DEFAULT 'healthy'
    CHECK (health_status IN ('healthy','degraded','down')),
  last_health_check_at TIMESTAMPTZ,
  consecutive_failures INTEGER NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_carriers_tenant ON carriers(tenant_id);
CREATE INDEX idx_carriers_active ON carriers(tenant_id, is_active, priority)
  WHERE is_active = true;
CREATE INDEX idx_carriers_countries ON carriers USING GIN (supported_countries);
```

### phone_numbers

```sql
CREATE TABLE phone_numbers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  carrier_id UUID NOT NULL REFERENCES carriers(id),
  number TEXT NOT NULL UNIQUE,
  country_code TEXT NOT NULL,
  number_type TEXT NOT NULL
    CHECK (number_type IN ('long_code','short_code','toll_free','virtual_mobile')),
  capabilities TEXT[] NOT NULL DEFAULT '{sms}',
  status TEXT NOT NULL DEFAULT 'active'
    CHECK (status IN ('pending','active','suspended','released')),
  is_10dlc_registered BOOLEAN NOT NULL DEFAULT false,
  monthly_cost_cents INTEGER,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_phone_numbers_tenant ON phone_numbers(tenant_id);
CREATE INDEX idx_phone_numbers_country ON phone_numbers(tenant_id, country_code, status)
  WHERE status = 'active';
CREATE INDEX idx_phone_numbers_capabilities ON phone_numbers USING GIN (capabilities);
```

### sender_ids

```sql
CREATE TABLE sender_ids (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  alphanumeric_id TEXT NOT NULL,
  country_code TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending'
    CHECK (status IN ('pending','approved','rejected','suspended')),
  registration_details JSONB,
  -- Example registration_details:
  -- {
  --   "registrar": "ofcom",
  --   "registration_number": "REG-2026-001",
  --   "approved_use_cases": ["transactional", "otp"]
  -- }
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(tenant_id, alphanumeric_id, country_code)
);
```

---

## Routing & Compliance

### routing_rules

```sql
CREATE TABLE routing_rules (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  name TEXT NOT NULL,
  priority INTEGER NOT NULL DEFAULT 0,
  message_type TEXT
    CHECK (message_type IN ('otp','transactional','promotional')),
  destination_country TEXT,
  destination_network TEXT,
  carrier_chain UUID[] NOT NULL,
  time_window_start TIME,
  time_window_end TIME,
  time_window_timezone TEXT DEFAULT 'UTC',
  max_cost_per_segment_microcents INTEGER,
  is_active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_routing_rules_tenant ON routing_rules(tenant_id);
CREATE INDEX idx_routing_rules_lookup ON routing_rules(
  tenant_id, message_type, destination_country, is_active
) WHERE is_active = true;
```

**Example routing query — find matching rule for an OTP to India:**

```sql
SELECT * FROM routing_rules
WHERE tenant_id = $1
  AND is_active = true
  AND (message_type IS NULL OR message_type = 'otp')
  AND (destination_country IS NULL OR destination_country = 'IN')
ORDER BY priority DESC
LIMIT 1;
```

### compliance_registrations

```sql
CREATE TABLE compliance_registrations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  registration_type TEXT NOT NULL
    CHECK (registration_type IN (
      '10dlc_brand','10dlc_campaign',
      'dlt_entity','dlt_template',
      'sender_id_registration'
    )),
  -- 10DLC fields (TCR)
  tcr_brand_id TEXT,
  tcr_campaign_id TEXT,
  campaign_use_case TEXT,
  -- India DLT fields (post-Feb 2025)
  dlt_entity_id TEXT,
  dlt_template_id TEXT,
  dlt_message_category TEXT
    CHECK (dlt_message_category IN ('P','S','T','G')),
  -- Common
  status TEXT NOT NULL DEFAULT 'pending'
    CHECK (status IN ('pending','submitted','approved','rejected','suspended','expired')),
  external_id TEXT,
  submitted_at TIMESTAMPTZ,
  approved_at TIMESTAMPTZ,
  expires_at TIMESTAMPTZ,
  registration_data JSONB,
  rejection_reason TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_compliance_tenant ON compliance_registrations(tenant_id);
CREATE INDEX idx_compliance_status ON compliance_registrations(tenant_id, registration_type, status);
```

### opt_outs

```sql
CREATE TABLE opt_outs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  phone_number TEXT NOT NULL,
  source TEXT NOT NULL
    CHECK (source IN ('recipient_reply','api','admin','compliance','carrier_feedback')),
  message_type TEXT,
  opted_out_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  is_active BOOLEAN NOT NULL DEFAULT true,
  resubscribed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(tenant_id, phone_number, message_type)
);

CREATE INDEX idx_opt_outs_lookup ON opt_outs(tenant_id, phone_number, is_active)
  WHERE is_active = true;
```

**Pre-send opt-out check:**

```sql
SELECT EXISTS(
  SELECT 1 FROM opt_outs
  WHERE tenant_id = $1
    AND phone_number = $2
    AND is_active = true
    AND (message_type IS NULL OR message_type = $3)
) AS is_opted_out;
```

---

## Messaging

### messages

```sql
CREATE TABLE messages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL,
  from_number TEXT NOT NULL,
  to_number TEXT NOT NULL,
  message_type TEXT NOT NULL
    CHECK (message_type IN ('otp','transactional','promotional')),
  body TEXT NOT NULL,
  encoding TEXT NOT NULL DEFAULT 'gsm7'
    CHECK (encoding IN ('gsm7','ucs2','binary')),
  segment_count INTEGER NOT NULL DEFAULT 1,
  status TEXT NOT NULL DEFAULT 'queued'
    CHECK (status IN (
      'queued','routing','sent','delivered',
      'failed','rejected','expired','undeliverable'
    )),
  direction TEXT NOT NULL DEFAULT 'outbound'
    CHECK (direction IN ('outbound','inbound')),
  -- Carrier routing
  carrier_id UUID,
  routing_rule_id UUID,
  phone_number_id UUID,
  carrier_attempts INTEGER NOT NULL DEFAULT 0,
  failover_from_carrier_id UUID,
  -- Provider tracking
  provider_message_id TEXT,
  provider_response JSONB,
  -- SMPP fields
  smpp_command_status INTEGER,
  smpp_data_coding INTEGER,
  -- Cost
  cost_per_segment_microcents INTEGER,
  total_cost_microcents INTEGER,
  -- HLR lookup result
  hlr_result JSONB,
  -- Example hlr_result:
  -- {
  --   "mcc": "310", "mnc": "260",
  --   "original_network": "T-Mobile US",
  --   "current_network": "Verizon",
  --   "ported": true, "roaming": false,
  --   "valid": true, "lookup_ms": 45
  -- }
  -- Compliance
  compliance_registration_id UUID,
  -- Conversation
  conversation_id UUID,
  -- Tracing
  trace_id TEXT,
  -- Scheduling
  idempotency_key TEXT,
  scheduled_at TIMESTAMPTZ,
  sent_at TIMESTAMPTZ,
  delivered_at TIMESTAMPTZ,
  failed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_messages_tenant ON messages(tenant_id, created_at DESC);
CREATE INDEX idx_messages_to_number ON messages(to_number, created_at DESC);
CREATE INDEX idx_messages_status ON messages(tenant_id, status)
  WHERE status IN ('queued','routing','sent');
CREATE INDEX idx_messages_idempotency ON messages(tenant_id, idempotency_key)
  WHERE idempotency_key IS NOT NULL;
CREATE INDEX idx_messages_conversation ON messages(conversation_id, created_at)
  WHERE conversation_id IS NOT NULL;
CREATE INDEX idx_messages_provider ON messages(carrier_id, provider_message_id);
CREATE INDEX idx_messages_scheduled ON messages(scheduled_at)
  WHERE scheduled_at IS NOT NULL AND status = 'queued';
```

### delivery_events

```sql
CREATE TABLE delivery_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  message_id UUID NOT NULL,
  tenant_id UUID NOT NULL,
  event_type TEXT NOT NULL
    CHECK (event_type IN (
      'queued','submitted','accepted','sent','delivered',
      'failed','rejected','expired','buffered','unknown'
    )),
  dlr_type TEXT
    CHECK (dlr_type IN ('handset','network','intermediate','synthetic')),
  status_code TEXT,
  error_code TEXT,
  error_message TEXT,
  carrier_id UUID,
  carrier_timestamp TIMESTAMPTZ,
  raw_receipt JSONB,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_delivery_events_message ON delivery_events(message_id, created_at);
CREATE INDEX idx_delivery_events_tenant ON delivery_events(tenant_id, created_at DESC);
CREATE INDEX idx_delivery_events_dlr_type ON delivery_events(tenant_id, dlr_type, created_at DESC);
```

### inbound_messages

```sql
CREATE TABLE inbound_messages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL,
  from_number TEXT NOT NULL,
  to_number TEXT NOT NULL,
  body TEXT NOT NULL,
  encoding TEXT NOT NULL DEFAULT 'gsm7'
    CHECK (encoding IN ('gsm7','ucs2','binary')),
  segment_count INTEGER NOT NULL DEFAULT 1,
  carrier_id UUID,
  phone_number_id UUID,
  conversation_id UUID,
  provider_message_id TEXT,
  -- Keyword detection (STOP/START/HELP)
  is_opt_out BOOLEAN NOT NULL DEFAULT false,
  is_opt_in BOOLEAN NOT NULL DEFAULT false,
  is_help BOOLEAN NOT NULL DEFAULT false,
  -- Webhook delivery tracking
  webhook_delivered BOOLEAN NOT NULL DEFAULT false,
  webhook_delivered_at TIMESTAMPTZ,
  webhook_attempts INTEGER NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_inbound_tenant ON inbound_messages(tenant_id, created_at DESC);
CREATE INDEX idx_inbound_conversation ON inbound_messages(conversation_id, created_at)
  WHERE conversation_id IS NOT NULL;
CREATE INDEX idx_inbound_from ON inbound_messages(from_number, created_at DESC);
```

---

## Operations

### webhooks

```sql
CREATE TABLE webhooks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  url TEXT NOT NULL,
  event_types TEXT[] NOT NULL,
  signing_secret TEXT NOT NULL,
  signing_algorithm TEXT NOT NULL DEFAULT 'hmac-sha256',
  is_active BOOLEAN NOT NULL DEFAULT true,
  failure_count INTEGER NOT NULL DEFAULT 0,
  max_retries INTEGER NOT NULL DEFAULT 5,
  last_failure_at TIMESTAMPTZ,
  last_success_at TIMESTAMPTZ,
  disabled_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_webhooks_tenant ON webhooks(tenant_id);
CREATE INDEX idx_webhooks_event_types ON webhooks USING GIN (event_types);
```

### ai_suggestions

```sql
CREATE TABLE ai_suggestions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL,
  suggestion_type TEXT NOT NULL
    CHECK (suggestion_type IN (
      'carrier_selection','dlr_accuracy_warning','compliance_inference',
      'fraud_detection','send_time_optimization','cost_optimization',
      'content_moderation','number_validation','congestion_prediction',
      'routing_optimization'
    )),
  title TEXT NOT NULL,
  description TEXT NOT NULL,
  confidence NUMERIC(3,2) NOT NULL CHECK (confidence BETWEEN 0 AND 1),
  entity_type TEXT NOT NULL,
  entity_id UUID NOT NULL,
  suggestion_data JSONB NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending'
    CHECK (status IN ('pending','accepted','dismissed','expired')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  resolved_at TIMESTAMPTZ
);

CREATE INDEX idx_ai_suggestions_tenant ON ai_suggestions(tenant_id, status);
CREATE INDEX idx_ai_suggestions_entity ON ai_suggestions(entity_type, entity_id);
```

### audit_log

```sql
CREATE TABLE audit_log (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL,
  actor_id UUID,
  actor_type TEXT NOT NULL
    CHECK (actor_type IN ('user','system','api_key','carrier','ai')),
  action TEXT NOT NULL,
  entity_type TEXT NOT NULL,
  entity_id UUID NOT NULL,
  changes JSONB,
  ip_address INET,
  user_agent TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_log_tenant ON audit_log(tenant_id, created_at DESC);
CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id, created_at DESC);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Users | 2 | Core multi-tenancy |
| Carrier & Numbers | 3 | carriers, phone_numbers, sender_ids |
| Routing & Compliance | 3 | routing_rules, compliance_registrations, opt_outs |
| Messaging | 3 | messages (partitioned), delivery_events (partitioned), inbound_messages (partitioned) |
| Operations | 3 | webhooks, ai_suggestions, audit_log (partitioned) |
| **Total** | **14** | 4 partitioned tables |

---

## Key Design Decisions

1. **Carrier protocol polymorphism via nullable columns** — SMPP fields (host, port, system_id, bind_mode) and REST fields (base_url, auth_type, credentials_ref) coexist on the carriers table rather than using inheritance or separate tables. Only one set is populated per row based on the protocol discriminator. This avoids JOIN overhead on the hot path (carrier selection during message dispatch).

2. **Routing rules with carrier_chain UUID array** — Failover order is expressed as an ordered array of carrier IDs rather than a junction table. This trades queryability ("which rules reference carrier X?") for simplicity on the dispatch hot path where the entire chain is needed in one read. The array is walked left-to-right during failover.

3. **Opt-out table separate from messages** — Opt-out state is its own entity rather than a flag on a subscriber record because TCPA compliance requires proving when and how consent was withdrawn. The source column distinguishes STOP keyword replies from API-driven or carrier-feedback removals. The pre-send lookup uses a partial index on is_active for sub-millisecond checks.

4. **DLR type classification on delivery_events** — The dlr_type column (handset/network/intermediate/synthetic) directly addresses the 2026 DLR accuracy concern identified in the market research. This allows tenants to distinguish genuine handset confirmations from carrier-level acceptance receipts, which no incumbent currently exposes clearly.

5. **HLR result stored per-message** — Network lookup results (MCC, MNC, ported status, roaming) are stored as JSONB on the message rather than in a separate cache table. This provides full audit trail of routing decisions but increases storage. For high-volume tenants, a TTL-based HLR cache table could be added as an optimisation.

6. **E.164 as canonical phone number format** — All phone number fields (from_number, to_number, opt_out phone_number) store E.164 format per ITU-T E.164. No separate country code parsing — the number itself is the lookup key. This aligns with how every carrier API accepts and returns numbers.

7. **Four partitioned tables** — messages, delivery_events, inbound_messages, and audit_log are all partitioned by created_at for time-range pruning. These are the highest-volume tables and benefit most from partition elimination on date-range queries.

8. **Compliance registration as a polymorphic table** — 10DLC (brand + campaign), India DLT (entity + template), and sender ID registrations all share one table with a type discriminator. The alternative (separate tables per jurisdiction) would be cleaner per-entity but creates unbounded table growth as new jurisdictions are added. The JSONB registration_data column holds jurisdiction-specific fields that don't fit the common schema.

9. **Inbound keyword detection as boolean flags** — is_opt_out, is_opt_in, and is_help are pre-computed booleans set during inbound processing rather than computed at query time. This ensures STOP/UNSUBSCRIBE keywords trigger immediate opt-out creation regardless of downstream webhook delivery success.

10. **Conversation ID as a nullable foreign-key-less UUID** — Two-way conversations are tracked by a shared conversation_id on both messages and inbound_messages, but there is no conversations table. The conversation is an implicit grouping rather than a first-class entity. This keeps the schema simpler for the MVP; a conversations table with session state can be added when conversational messaging matures.
