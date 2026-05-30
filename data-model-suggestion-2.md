# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: SMS Messaging Gateway · Created: 2026-05-26

## Philosophy

The hybrid approach keeps relational columns for fields that drive queries and indexing — message status, phone numbers, timestamps — while consolidating configuration, carrier details, routing rules, and compliance registrations into JSONB documents on their parent entities. A tenant document contains its entire carrier fleet, phone number inventory, and routing configuration in structured JSONB columns, making tenant provisioning a single-row operation.

This design recognises that SMS gateway configuration is read-heavy and tenant-scoped: when dispatching a message, the system needs the tenant's carriers, routing rules, and opt-out policy as a single unit. Rather than JOINing across 6+ tables, the dispatch path reads one tenant row and has everything needed to select a carrier, check compliance, and format the SMPP or REST request. Messages themselves remain relational because they are the hot path for writes, status updates, and time-range queries — but their delivery event timeline is embedded as a JSONB array rather than a separate table.

The trade-off is denormalisation: updating a single carrier's health status means patching a JSONB array within the tenant row. This is acceptable for configuration that changes infrequently (carrier additions, number provisioning) but would be problematic if carrier health checks updated every second. For real-time carrier health, an external Redis cache is assumed.

**Best for:** Rapid MVP development where tenant self-service configuration and fast message dispatch matter more than fine-grained per-carrier analytics across the fleet.

**Trade-offs:**
- Pro: Tenant provisioning is a single INSERT with all configuration embedded
- Pro: Message dispatch reads one tenant row for carrier selection, routing, and compliance
- Pro: Adding jurisdiction-specific compliance fields requires no schema migration — just extend the JSONB structure
- Pro: Fewer tables (6) means simpler migrations and smaller operational surface
- Con: Cross-tenant carrier analytics (e.g., "which tenants use carrier X?") require JSONB path queries
- Con: Updating individual carrier health within the tenant JSONB document requires careful JSONB patching
- Con: Delivery event timeline embedded in messages can grow unbounded for long-running conversations
- Con: No foreign key enforcement between embedded carrier IDs and the carriers they reference

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| SMPP 3.4 | Carrier objects in tenants.carriers_json include SMPP-specific nested fields: host, port (2775), system_id, bind_mode |
| 3GPP TS 23.038 | Message encoding field for GSM 7-bit / UCS-2 / binary; segment count derived from encoding |
| ITU-T E.164 | All phone number fields in E.164 format; embedded in carriers_json and phone_numbers_json |
| 10DLC / TCR | Compliance registrations embedded in tenants.compliance_json with TCR brand/campaign IDs |
| India DLT | DLT entity/template registrations embedded in tenants.compliance_json with post-2025 category suffixes |
| GDPR / TCPA | Opt-outs table remains relational for fast pre-send enforcement; source attribution preserved |
| GSMA IR.75 | Carrier objects model hub vs. direct connections; routing_json supports MCC+MNC destination targeting |
| OAuth 2.0 | Carrier REST credentials support multiple auth types in nested JSONB structure |
| NIST SP 800-63B | Message type 'otp' tracked relationally for OTP-specific latency and fraud analytics |

---

## Tenant Configuration

### tenants

```sql
CREATE TABLE tenants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  slug TEXT NOT NULL UNIQUE,
  status TEXT NOT NULL DEFAULT 'active'
    CHECK (status IN ('active','suspended','cancelled')),

  -- Embedded users
  users_json JSONB NOT NULL DEFAULT '[]',
  -- Example users_json:
  -- [
  --   {
  --     "id": "uuid", "email": "dev@example.com", "name": "Dev User",
  --     "role": "developer", "api_key_hash": "sha256...",
  --     "api_key_prefix": "sk_live_abc", "is_active": true
  --   }
  -- ]

  -- Carrier connections (SMPP and REST)
  carriers_json JSONB NOT NULL DEFAULT '[]',
  -- Example carriers_json:
  -- [
  --   {
  --     "id": "uuid", "name": "Infobip Direct",
  --     "protocol": "smpp",
  --     "smpp": {
  --       "host": "smpp.infobip.com", "port": 2775,
  --       "system_id": "myapp", "password_ref": "vault://smpp/infobip",
  --       "bind_mode": "transceiver"
  --     },
  --     "throughput_per_second": 100,
  --     "priority": 10, "is_active": true,
  --     "supported_countries": ["US","GB","DE"],
  --     "supported_message_types": ["otp","transactional","promotional"],
  --     "cost_per_segment_microcents": 750,
  --     "dlr_accuracy": "handset_confirmed"
  --   },
  --   {
  --     "id": "uuid", "name": "Twilio REST Backup",
  --     "protocol": "rest",
  --     "rest": {
  --       "base_url": "https://api.twilio.com/2010-04-01",
  --       "auth_type": "basic",
  --       "credentials_ref": "vault://rest/twilio"
  --     },
  --     "throughput_per_second": 50,
  --     "priority": 5, "is_active": true,
  --     "supported_countries": ["US","CA"],
  --     "supported_message_types": ["otp","transactional"],
  --     "cost_per_segment_microcents": 900,
  --     "dlr_accuracy": "network_only"
  --   }
  -- ]

  -- Phone numbers
  phone_numbers_json JSONB NOT NULL DEFAULT '[]',
  -- Example phone_numbers_json:
  -- [
  --   {
  --     "id": "uuid", "number": "+14155551234",
  --     "country_code": "US", "number_type": "long_code",
  --     "carrier_id": "uuid", "capabilities": ["sms","mms"],
  --     "status": "active", "is_10dlc_registered": true,
  --     "monthly_cost_cents": 100
  --   },
  --   {
  --     "id": "uuid", "number": "12345",
  --     "country_code": "US", "number_type": "short_code",
  --     "carrier_id": "uuid", "capabilities": ["sms"],
  --     "status": "active", "monthly_cost_cents": 50000
  --   }
  -- ]

  -- Sender IDs (alphanumeric)
  sender_ids_json JSONB NOT NULL DEFAULT '[]',
  -- Example sender_ids_json:
  -- [
  --   {
  --     "id": "uuid", "alphanumeric_id": "MyApp",
  --     "country_code": "GB", "status": "approved",
  --     "registration_details": {"registrar": "ofcom"}
  --   }
  -- ]

  -- Routing rules
  routing_json JSONB NOT NULL DEFAULT '[]',
  -- Example routing_json:
  -- [
  --   {
  --     "id": "uuid", "name": "OTP India via Infobip",
  --     "priority": 10,
  --     "message_type": "otp",
  --     "destination_country": "IN",
  --     "carrier_chain": ["uuid-infobip", "uuid-twilio"],
  --     "max_cost_per_segment_microcents": 1200
  --   },
  --   {
  --     "id": "uuid", "name": "Default US",
  --     "priority": 0,
  --     "message_type": null,
  --     "destination_country": "US",
  --     "carrier_chain": ["uuid-twilio", "uuid-infobip"]
  --   }
  -- ]

  -- Compliance registrations (10DLC, DLT, etc.)
  compliance_json JSONB NOT NULL DEFAULT '[]',
  -- Example compliance_json:
  -- [
  --   {
  --     "id": "uuid", "type": "10dlc_brand",
  --     "tcr_brand_id": "BXXXXXX", "status": "approved",
  --     "submitted_at": "2026-01-15T00:00:00Z",
  --     "approved_at": "2026-01-18T00:00:00Z"
  --   },
  --   {
  --     "id": "uuid", "type": "10dlc_campaign",
  --     "tcr_campaign_id": "CXXXXXX",
  --     "use_case": "otp", "status": "approved",
  --     "brand_id": "uuid"
  --   },
  --   {
  --     "id": "uuid", "type": "dlt_entity",
  --     "dlt_entity_id": "110XXXXXXXXX",
  --     "status": "approved"
  --   },
  --   {
  --     "id": "uuid", "type": "dlt_template",
  --     "dlt_template_id": "1107XXXXXXXXX",
  --     "dlt_message_category": "T",
  --     "template_body": "Your OTP is {#var#}",
  --     "status": "approved"
  --   }
  -- ]

  -- Webhooks
  webhooks_json JSONB NOT NULL DEFAULT '[]',
  -- Example webhooks_json:
  -- [
  --   {
  --     "id": "uuid", "url": "https://api.example.com/sms/events",
  --     "event_types": ["delivery_report","inbound_message","opt_out"],
  --     "signing_secret": "whsec_...",
  --     "signing_algorithm": "hmac-sha256",
  --     "is_active": true, "failure_count": 0
  --   }
  -- ]

  -- General configuration
  config_json JSONB NOT NULL DEFAULT '{}',
  -- Example config_json:
  -- {
  --   "default_sender_id": "+14155551234",
  --   "default_message_type": "transactional",
  --   "failover_mode": "fail_closed",
  --   "rate_limit_per_second": 100,
  --   "hlr_lookup_enabled": true,
  --   "hlr_cache_ttl_hours": 24,
  --   "opt_out_keywords": ["STOP","UNSUBSCRIBE","CANCEL","QUIT"],
  --   "opt_in_keywords": ["START","SUBSCRIBE","YES"],
  --   "help_keywords": ["HELP","INFO"],
  --   "auto_reply_opt_out": "You have been unsubscribed. Reply START to resubscribe.",
  --   "auto_reply_help": "Reply STOP to unsubscribe. Msg rates may apply."
  -- }

  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tenants_slug ON tenants(slug);
CREATE INDEX idx_tenants_carriers ON tenants USING GIN (carriers_json);
CREATE INDEX idx_tenants_compliance ON tenants USING GIN (compliance_json);
```

**Carrier selection query — find active carriers for OTP to India:**

```sql
SELECT
  c->>'id' AS carrier_id,
  c->>'name' AS carrier_name,
  c->>'protocol' AS protocol,
  (c->>'priority')::int AS priority,
  (c->>'cost_per_segment_microcents')::int AS cost
FROM tenants,
  jsonb_array_elements(carriers_json) AS c
WHERE id = $1
  AND (c->>'is_active')::boolean = true
  AND c->'supported_countries' ? 'IN'
  AND c->'supported_message_types' ? 'otp'
ORDER BY (c->>'priority')::int DESC;
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

  -- Carrier used (ID references carriers_json entry)
  carrier_id TEXT,
  carrier_attempts INTEGER NOT NULL DEFAULT 0,

  -- Routing decision
  routing_json JSONB,
  -- Example routing_json:
  -- {
  --   "rule_matched": "OTP India via Infobip",
  --   "carrier_chain": ["uuid-infobip", "uuid-twilio"],
  --   "carrier_selected": "uuid-infobip",
  --   "selection_reason": "highest_priority",
  --   "failover_attempts": [
  --     {
  --       "carrier_id": "uuid-infobip", "carrier_name": "Infobip Direct",
  --       "attempted_at": "2026-05-26T10:00:00.123Z",
  --       "result": "success",
  --       "latency_ms": 45
  --     }
  --   ]
  -- }

  -- HLR lookup result
  hlr_json JSONB,
  -- Example hlr_json:
  -- {
  --   "mcc": "404", "mnc": "45",
  --   "original_network": "Airtel India",
  --   "current_network": "Jio",
  --   "ported": true, "roaming": false,
  --   "valid": true, "lookup_ms": 120
  -- }

  -- Delivery event timeline (appended as events arrive)
  delivery_json JSONB NOT NULL DEFAULT '[]',
  -- Example delivery_json:
  -- [
  --   {
  --     "event": "queued", "at": "2026-05-26T10:00:00.100Z"
  --   },
  --   {
  --     "event": "submitted", "at": "2026-05-26T10:00:00.150Z",
  --     "carrier_id": "uuid", "provider_message_id": "SM1234"
  --   },
  --   {
  --     "event": "sent", "at": "2026-05-26T10:00:00.200Z",
  --     "dlr_type": "network"
  --   },
  --   {
  --     "event": "delivered", "at": "2026-05-26T10:00:01.500Z",
  --     "dlr_type": "handset",
  --     "carrier_timestamp": "2026-05-26T10:00:01.200Z"
  --   }
  -- ]

  -- Cost
  total_cost_microcents INTEGER,

  -- Compliance
  compliance_registration_id TEXT,

  -- Conversation grouping
  conversation_id UUID,

  -- Tracing and deduplication
  trace_id TEXT,
  idempotency_key TEXT,

  -- Scheduling
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
CREATE INDEX idx_messages_scheduled ON messages(scheduled_at)
  WHERE scheduled_at IS NOT NULL AND status = 'queued';
CREATE INDEX idx_messages_type ON messages(tenant_id, message_type, created_at DESC);
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

  -- Carrier info
  carrier_id TEXT,
  phone_number_id TEXT,
  provider_message_id TEXT,

  -- Conversation
  conversation_id UUID,

  -- Keyword detection and auto-responses
  keywords_json JSONB NOT NULL DEFAULT '{}',
  -- Example keywords_json:
  -- {
  --   "detected": "STOP",
  --   "is_opt_out": true, "is_opt_in": false, "is_help": false,
  --   "auto_reply_sent": true,
  --   "auto_reply_message_id": "uuid"
  -- }

  -- Webhook delivery
  webhook_json JSONB NOT NULL DEFAULT '{}',
  -- Example webhook_json:
  -- {
  --   "delivered": true,
  --   "delivered_at": "2026-05-26T10:00:00.500Z",
  --   "attempts": 1,
  --   "webhook_id": "uuid",
  --   "response_status": 200
  -- }

  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_inbound_tenant ON inbound_messages(tenant_id, created_at DESC);
CREATE INDEX idx_inbound_from ON inbound_messages(from_number, created_at DESC);
CREATE INDEX idx_inbound_conversation ON inbound_messages(conversation_id, created_at)
  WHERE conversation_id IS NOT NULL;
```

---

## Compliance & Operations

### opt_outs

```sql
CREATE TABLE opt_outs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL,
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
  entity_id TEXT NOT NULL,
  suggestion_data JSONB NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending'
    CHECK (status IN ('pending','accepted','dismissed','expired')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  resolved_at TIMESTAMPTZ
);

CREATE INDEX idx_ai_suggestions_tenant ON ai_suggestions(tenant_id, status);
```

### audit_log

```sql
CREATE TABLE audit_log (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL,
  actor_id TEXT,
  actor_type TEXT NOT NULL
    CHECK (actor_type IN ('user','system','api_key','carrier','ai')),
  action TEXT NOT NULL,
  entity_type TEXT NOT NULL,
  entity_id TEXT NOT NULL,
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
| Tenant Configuration | 1 | Embeds users, carriers, numbers, sender IDs, routing, compliance, webhooks |
| Messaging | 2 | messages (partitioned), inbound_messages (partitioned) |
| Compliance | 1 | opt_outs — relational for fast pre-send lookup |
| Operations | 2 | ai_suggestions, audit_log (partitioned) |
| **Total** | **6** | 3 partitioned tables |

---

## Key Design Decisions

1. **Entire carrier fleet embedded in tenant** — All carrier connections (SMPP and REST) live in `carriers_json` on the tenant row. During message dispatch, one `SELECT` on the tenant gives the dispatch engine the full carrier list, routing rules, and compliance state. The alternative (relational carrier table with JOINs) adds latency on the critical path. The trade-off: cross-tenant carrier queries require `jsonb_array_elements` unnesting.

2. **Routing rules embedded alongside carriers** — The `routing_json` array stores rules that reference carrier IDs from `carriers_json` on the same row. This ensures routing decisions never require a second query. Rule priority ordering and message type/country matching happen in application code after a single row fetch.

3. **Delivery events as embedded timeline** — Each message carries its own `delivery_json` array containing the ordered sequence of DLR events. This eliminates the need for a separate delivery_events table and makes message status queries self-contained. For DLR accuracy analysis across carriers, the `dlr_type` field within each event enables JSONB path queries. Unbounded growth is mitigated by archiving messages older than the retention window.

4. **Opt-outs remain relational** — Despite the JSONB-heavy design, opt-outs stay in their own indexed table because pre-send opt-out checking is on the critical path for every message and must be sub-millisecond. TCPA statutory damages ($500–$1,500 per violation) make this lookup too important to embed in a JSONB document.

5. **Compliance registrations embedded in tenant** — 10DLC brand/campaign, India DLT entity/template, and sender ID registrations are embedded in `compliance_json`. This makes tenant configuration self-contained and allows new jurisdiction types (e.g., future RCS business messaging registrations) to be added without schema migration — just extend the JSONB structure with a new `type` discriminator.

6. **Carrier ID as TEXT not UUID** — Since carriers are embedded in tenant JSONB rather than in their own table, the `carrier_id` on messages is a TEXT field containing the JSONB-internal UUID string. No foreign key enforcement exists — consistency is maintained by the application layer. This is the primary trade-off of the hybrid approach.

7. **Routing decision recorded per-message** — The `routing_json` on each message captures which rule matched, the full carrier chain attempted, and per-attempt results including latency. This provides the routing transparency identified as an underserved area in the market — tenants can see exactly why each message took its path, including failover attempts.

8. **Inbound keyword detection as embedded JSONB** — Rather than separate boolean columns, inbound messages store keyword detection results in `keywords_json` including the detected keyword, classification, and whether an auto-reply was sent. This keeps the inbound path simple while allowing future keyword types without column additions.

9. **Webhook configuration embedded in tenant** — Webhook endpoints, signing secrets, and event type subscriptions live in `webhooks_json` on the tenant. The dispatch engine reads webhooks alongside carrier config in the same tenant row fetch. Webhook delivery tracking (attempts, failures) is managed by the webhook delivery service and recorded in `audit_log` rather than updating the tenant row on every webhook attempt.

10. **HLR results as per-message JSONB** — Mobile Number Portability lookup results are stored on the message as `hlr_json` rather than in a shared cache table. This ensures routing audit trail completeness — each message records the network state at the moment of dispatch. A Redis-based HLR cache with TTL sits in front of this for cost optimisation, but the canonical record is on the message.
