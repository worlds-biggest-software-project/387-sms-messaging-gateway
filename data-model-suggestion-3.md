# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: SMS Messaging Gateway · Created: 2026-05-26

## Philosophy

Every state change in the SMS gateway — carrier added, message queued, DLR received, opt-out processed, compliance registration approved — is captured as an immutable event in a single append-only event store. The current state of any entity is derived by replaying its event stream. Read-optimised materialised views (read models) are projected from the event store to serve the dispatch hot path, analytics dashboards, and compliance queries.

This architecture is a natural fit for SMS gateways because the domain is inherently event-driven: a message moves through a well-defined lifecycle (queued → routing → submitted → sent → delivered/failed), delivery receipts arrive asynchronously from carriers, and regulatory compliance requires proving exactly what happened and when. An event-sourced model turns the audit trail from an afterthought into the source of truth — every routing decision, carrier failover, DLR receipt, and opt-out action is a first-class event that can be replayed, queried, and analysed.

The CQRS pattern separates the write path (append events) from the read path (query materialised views). The dispatch engine reads pre-computed carrier configurations from `rm_routing_config`. The analytics dashboard reads from `rm_delivery_analytics`. The compliance team queries `rm_compliance_status`. Each read model is optimised for its specific access pattern while the event store preserves the complete, immutable history.

**Best for:** Regulatory-heavy deployments where full audit trail, temporal queries ("what was the carrier configuration on date X?"), and event replay for debugging routing decisions are critical requirements.

**Trade-offs:**
- Pro: Complete, immutable audit trail — every routing decision, DLR, and compliance action is a first-class event
- Pro: Temporal queries are native ("what carriers were active when message X was sent?")
- Pro: Event replay enables debugging of routing decisions and DLR accuracy analysis
- Pro: Read models can be rebuilt from scratch if a projection bug is found
- Pro: New analytics dimensions can be added by creating new projections over existing events
- Con: Eventual consistency between event store and read models (milliseconds in practice)
- Con: Dispatch hot path depends on read model freshness — stale routing config could route to a deactivated carrier
- Con: Event store grows rapidly for high-volume tenants (millions of message lifecycle events per day)
- Con: Schema evolution of events requires careful versioning — old events must remain readable

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CloudEvents 1.0 | Event envelope follows CloudEvents spec: ce_source, ce_type, ce_specversion, ce_time |
| SMPP 3.4 | Carrier lifecycle events capture SMPP-specific configuration; message events record smpp_command_status and data_coding |
| 3GPP TS 23.038 | Message events record encoding (gsm7/ucs2/binary) and segment count per 3GPP character set rules |
| ITU-T E.164 | All phone numbers in events stored in E.164 format |
| 10DLC / TCR | Compliance stream events track brand/campaign registration lifecycle with TCR external IDs |
| India DLT | Compliance stream events track DLT entity/template registration with post-2025 category suffixes |
| GDPR / TCPA | Opt-out events are immutable compliance proof; temporal replay shows exact consent state at any point |
| GSMA IR.75 | Carrier events model hub/direct connection changes; routing events reference GSMA network identifiers |
| W3C Trace Context | Message events carry trace_id for distributed tracing across the dispatch pipeline |
| NIST SP 800-63B | OTP message events flagged separately; security events track OTP fraud patterns |

---

## Event Store Infrastructure

### event_store

```sql
CREATE TABLE event_store (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  stream_type TEXT NOT NULL
    CHECK (stream_type IN (
      'tenant','user','carrier','phone_number','sender_id',
      'routing_rule','compliance','message','opt_out',
      'conversation','ai','config'
    )),
  stream_id UUID NOT NULL,
  version BIGINT NOT NULL,
  event_type TEXT NOT NULL,
  event_data JSONB NOT NULL,
  metadata JSONB NOT NULL DEFAULT '{}',
  -- CloudEvents 1.0 envelope
  ce_source TEXT NOT NULL DEFAULT '/sms-gateway',
  ce_specversion TEXT NOT NULL DEFAULT '1.0',
  ce_type TEXT NOT NULL,
  ce_time TIMESTAMPTZ NOT NULL DEFAULT now(),
  -- Actor
  actor_id UUID,
  actor_type TEXT NOT NULL
    CHECK (actor_type IN (
      'user','system','api_key','carrier','smpp_gateway',
      'hlr_service','keyword_processor','webhook_engine','ai'
    )),
  tenant_id UUID NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(stream_type, stream_id, version)
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_events_stream ON event_store(stream_type, stream_id, version);
CREATE INDEX idx_events_tenant ON event_store(tenant_id, created_at DESC);
CREATE INDEX idx_events_type ON event_store(event_type, created_at DESC);
CREATE INDEX idx_events_ce_type ON event_store(ce_type, ce_time DESC);
```

### stream_snapshots

```sql
CREATE TABLE stream_snapshots (
  stream_type TEXT NOT NULL,
  stream_id UUID NOT NULL,
  version BIGINT NOT NULL,
  snapshot_data JSONB NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (stream_type, stream_id)
);
```

### projection_checkpoints

```sql
CREATE TABLE projection_checkpoints (
  projection_name TEXT PRIMARY KEY,
  last_event_id UUID NOT NULL,
  last_event_time TIMESTAMPTZ NOT NULL,
  events_processed BIGINT NOT NULL DEFAULT 0,
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Event Catalogue

### Tenant Stream

| Event Type | Key Fields | Actor Types |
|------------|------------|-------------|
| tenant.created | name, slug, config | user |
| tenant.updated | changed_fields | user |
| tenant.suspended | reason | user, system |
| tenant.reactivated | — | user |
| tenant.rate_limit_changed | old_limit, new_limit | user, ai |
| tenant.failover_mode_changed | old_mode, new_mode | user |

### Carrier Stream

| Event Type | Key Fields | Actor Types |
|------------|------------|-------------|
| carrier.added | name, protocol, smpp_config/rest_config, supported_countries, throughput | user |
| carrier.updated | changed_fields | user |
| carrier.activated | — | user |
| carrier.deactivated | reason | user, system |
| carrier.health_changed | old_status, new_status, consecutive_failures | system, ai |
| carrier.health_check_completed | latency_ms, success, error_code | system |
| carrier.throughput_adjusted | old_tps, new_tps, reason | user, ai |
| carrier.dlr_accuracy_updated | old_accuracy, new_accuracy, sample_size | system, ai |

### Phone Number Stream

| Event Type | Key Fields | Actor Types |
|------------|------------|-------------|
| phone_number.provisioned | number (E.164), country_code, number_type, carrier_id, capabilities | user |
| phone_number.activated | — | system |
| phone_number.suspended | reason | user, system |
| phone_number.released | — | user |
| phone_number.10dlc_registered | tcr_campaign_id | system |
| phone_number.carrier_changed | old_carrier_id, new_carrier_id | user |

### Routing Rule Stream

| Event Type | Key Fields | Actor Types |
|------------|------------|-------------|
| routing_rule.created | name, priority, message_type, destination_country, carrier_chain | user |
| routing_rule.updated | changed_fields | user |
| routing_rule.activated | — | user |
| routing_rule.deactivated | reason | user |
| routing_rule.reordered | old_priority, new_priority | user, ai |

### Compliance Stream

| Event Type | Key Fields | Actor Types |
|------------|------------|-------------|
| compliance.10dlc_brand_submitted | tcr_brand_id, brand_data | user |
| compliance.10dlc_brand_approved | tcr_brand_id | system |
| compliance.10dlc_brand_rejected | tcr_brand_id, rejection_reason | system |
| compliance.10dlc_campaign_submitted | tcr_campaign_id, use_case | user |
| compliance.10dlc_campaign_approved | tcr_campaign_id | system |
| compliance.10dlc_campaign_rejected | tcr_campaign_id, rejection_reason | system |
| compliance.dlt_entity_registered | dlt_entity_id | user |
| compliance.dlt_entity_approved | dlt_entity_id | system |
| compliance.dlt_template_submitted | dlt_template_id, category (P/S/T/G), template_body | user |
| compliance.dlt_template_approved | dlt_template_id | system |
| compliance.sender_id_submitted | alphanumeric_id, country_code | user |
| compliance.sender_id_approved | — | system |
| compliance.sender_id_rejected | rejection_reason | system |
| compliance.registration_expired | registration_type, external_id | system |

### Message Stream

| Event Type | Key Fields | Actor Types |
|------------|------------|-------------|
| message.queued | from, to (E.164), body, encoding, segment_count, message_type, idempotency_key | api_key |
| message.opt_out_checked | is_opted_out, lookup_ms | system |
| message.compliance_checked | registration_id, is_compliant, check_type | system |
| message.hlr_lookup_completed | mcc, mnc, ported, roaming, current_network, lookup_ms | hlr_service |
| message.routing_decided | rule_matched, carrier_chain, carrier_selected, selection_reason | system, ai |
| message.submitted | carrier_id, provider_message_id, smpp_command_status, latency_ms | smpp_gateway, system |
| message.submission_failed | carrier_id, error_code, error_message | smpp_gateway, system |
| message.failover_triggered | from_carrier_id, to_carrier_id, attempt_number, reason | system |
| message.sent | dlr_type (network), carrier_timestamp | carrier |
| message.delivered | dlr_type (handset), carrier_timestamp | carrier |
| message.failed | error_code, error_message, is_permanent | carrier |
| message.rejected | reason, carrier_id | carrier, system |
| message.expired | ttl_seconds | system |
| message.cost_calculated | segment_count, cost_per_segment_microcents, total_cost_microcents | system |
| message.scheduled | scheduled_at | api_key |
| message.schedule_fired | — | system |

### Opt-Out Stream

| Event Type | Key Fields | Actor Types |
|------------|------------|-------------|
| opt_out.created | phone_number (E.164), source, message_type | keyword_processor, api_key, user, carrier |
| opt_out.resubscribed | phone_number, source | keyword_processor, api_key, user |
| opt_out.bulk_imported | count, source_file | user |
| opt_out.carrier_feedback_received | phone_number, carrier_id, feedback_type | carrier |

### Conversation Stream

| Event Type | Key Fields | Actor Types |
|------------|------------|-------------|
| conversation.started | phone_number, our_number, first_message_id, direction | system |
| conversation.message_added | message_id, direction | system |
| conversation.inbound_received | from, to, body, keyword_detected | keyword_processor |
| conversation.auto_reply_sent | reply_type (opt_out/help/custom), message_id | system |
| conversation.webhook_delivered | webhook_id, response_status, latency_ms | webhook_engine |
| conversation.webhook_failed | webhook_id, error, attempt_number | webhook_engine |
| conversation.closed | reason, duration_seconds, message_count | system |

### AI Stream

| Event Type | Key Fields | Actor Types |
|------------|------------|-------------|
| ai.suggestion_generated | suggestion_type, entity_type, entity_id, confidence, suggestion_data | ai |
| ai.suggestion_accepted | suggestion_id, applied_changes | user |
| ai.suggestion_dismissed | suggestion_id, reason | user |
| ai.carrier_prediction | carrier_id, destination, predicted_delivery_rate, predicted_latency_ms | ai |
| ai.dlr_accuracy_alert | carrier_id, suspected_fake_dlr_rate, sample_size | ai |
| ai.compliance_inference | message_id, detected_category, confidence, applied_rules | ai |
| ai.fraud_detected | entity_type, entity_id, pattern, risk_score | ai |
| ai.congestion_predicted | carrier_id, predicted_window, confidence | ai |

---

## Read Models

### rm_routing_config

Pre-computed carrier selection data for the dispatch hot path.

```sql
CREATE TABLE rm_routing_config (
  tenant_id UUID NOT NULL,
  config_hash TEXT NOT NULL,
  carriers JSONB NOT NULL,
  -- Example carriers:
  -- [
  --   {
  --     "id": "uuid", "name": "Infobip Direct",
  --     "protocol": "smpp", "is_active": true,
  --     "health_status": "healthy",
  --     "smpp": {"host": "smpp.infobip.com", "port": 2775, "system_id": "myapp"},
  --     "throughput_per_second": 100,
  --     "supported_countries": ["US","GB","IN"],
  --     "supported_message_types": ["otp","transactional","promotional"],
  --     "cost_per_segment_microcents": 750,
  --     "dlr_accuracy": "handset_confirmed"
  --   }
  -- ]
  routing_rules JSONB NOT NULL,
  -- Example routing_rules:
  -- [
  --   {
  --     "name": "OTP India", "priority": 10,
  --     "message_type": "otp", "destination_country": "IN",
  --     "carrier_chain": ["uuid-infobip", "uuid-twilio"]
  --   }
  -- ]
  phone_numbers JSONB NOT NULL,
  sender_ids JSONB NOT NULL,
  compliance_registrations JSONB NOT NULL,
  config JSONB NOT NULL,
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (tenant_id)
);
```

### rm_message_status

Searchable message status with delivery timeline for API queries.

```sql
CREATE TABLE rm_message_status (
  message_id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL,
  from_number TEXT NOT NULL,
  to_number TEXT NOT NULL,
  message_type TEXT NOT NULL,
  body_preview TEXT,
  encoding TEXT NOT NULL,
  segment_count INTEGER NOT NULL,
  status TEXT NOT NULL,
  direction TEXT NOT NULL,
  -- Carrier and routing
  carrier_id TEXT,
  carrier_name TEXT,
  routing_rule_name TEXT,
  carrier_attempts INTEGER NOT NULL DEFAULT 0,
  -- HLR
  destination_network TEXT,
  is_ported BOOLEAN,
  -- Delivery timeline
  delivery_timeline JSONB NOT NULL DEFAULT '[]',
  -- DLR details
  last_dlr_type TEXT,
  last_dlr_at TIMESTAMPTZ,
  -- Cost
  total_cost_microcents INTEGER,
  -- Conversation
  conversation_id UUID,
  -- Timestamps
  queued_at TIMESTAMPTZ,
  sent_at TIMESTAMPTZ,
  delivered_at TIMESTAMPTZ,
  failed_at TIMESTAMPTZ,
  error_code TEXT,
  error_message TEXT,
  -- Tracing
  trace_id TEXT,
  idempotency_key TEXT,
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_message_tenant ON rm_message_status(tenant_id, queued_at DESC);
CREATE INDEX idx_rm_message_to ON rm_message_status(to_number, queued_at DESC);
CREATE INDEX idx_rm_message_status ON rm_message_status(tenant_id, status);
CREATE INDEX idx_rm_message_conversation ON rm_message_status(conversation_id, queued_at)
  WHERE conversation_id IS NOT NULL;
CREATE INDEX idx_rm_message_carrier ON rm_message_status(carrier_id, queued_at DESC);
CREATE INDEX idx_rm_message_idempotency ON rm_message_status(tenant_id, idempotency_key)
  WHERE idempotency_key IS NOT NULL;
```

### rm_carrier_health

Per-carrier performance metrics and health status.

```sql
CREATE TABLE rm_carrier_health (
  tenant_id UUID NOT NULL,
  carrier_id TEXT NOT NULL,
  carrier_name TEXT NOT NULL,
  protocol TEXT NOT NULL,
  is_active BOOLEAN NOT NULL,
  health_status TEXT NOT NULL,
  -- Performance metrics (rolling window)
  messages_sent_1h INTEGER NOT NULL DEFAULT 0,
  messages_delivered_1h INTEGER NOT NULL DEFAULT 0,
  messages_failed_1h INTEGER NOT NULL DEFAULT 0,
  delivery_rate_1h NUMERIC(5,4),
  avg_latency_ms_1h INTEGER,
  p95_latency_ms_1h INTEGER,
  -- DLR accuracy
  dlr_accuracy TEXT NOT NULL DEFAULT 'unknown',
  handset_dlr_rate NUMERIC(5,4),
  network_only_dlr_rate NUMERIC(5,4),
  suspected_fake_dlr_count INTEGER NOT NULL DEFAULT 0,
  -- Cost
  total_cost_microcents_24h BIGINT NOT NULL DEFAULT 0,
  avg_cost_per_segment_microcents INTEGER,
  -- Failover
  failover_triggers_24h INTEGER NOT NULL DEFAULT 0,
  consecutive_failures INTEGER NOT NULL DEFAULT 0,
  last_failure_at TIMESTAMPTZ,
  last_success_at TIMESTAMPTZ,
  -- Throughput
  current_tps INTEGER NOT NULL DEFAULT 0,
  max_tps INTEGER NOT NULL,
  -- Per-country breakdown
  country_stats JSONB NOT NULL DEFAULT '{}',
  -- Example country_stats:
  -- {
  --   "US": {"sent": 5000, "delivered": 4850, "rate": 0.97, "avg_latency_ms": 120},
  --   "IN": {"sent": 3000, "delivered": 2700, "rate": 0.90, "avg_latency_ms": 450}
  -- }
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (tenant_id, carrier_id)
);

CREATE INDEX idx_rm_carrier_health_status ON rm_carrier_health(health_status);
```

### rm_opt_out_list

Pre-send opt-out enforcement list.

```sql
CREATE TABLE rm_opt_out_list (
  tenant_id UUID NOT NULL,
  phone_number TEXT NOT NULL,
  message_type TEXT,
  source TEXT NOT NULL,
  opted_out_at TIMESTAMPTZ NOT NULL,
  is_active BOOLEAN NOT NULL DEFAULT true,
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(tenant_id, phone_number, message_type)
);

CREATE INDEX idx_rm_opt_out_lookup ON rm_opt_out_list(tenant_id, phone_number, is_active)
  WHERE is_active = true;
```

### rm_delivery_analytics

Time-bucketed delivery analytics for dashboards and reporting.

```sql
CREATE TABLE rm_delivery_analytics (
  tenant_id UUID NOT NULL,
  bucket TIMESTAMPTZ NOT NULL,
  granularity TEXT NOT NULL DEFAULT '1h'
    CHECK (granularity IN ('1m','5m','1h','1d')),
  -- Aggregate counts
  messages_queued INTEGER NOT NULL DEFAULT 0,
  messages_sent INTEGER NOT NULL DEFAULT 0,
  messages_delivered INTEGER NOT NULL DEFAULT 0,
  messages_failed INTEGER NOT NULL DEFAULT 0,
  messages_rejected INTEGER NOT NULL DEFAULT 0,
  messages_expired INTEGER NOT NULL DEFAULT 0,
  -- Rates
  delivery_rate NUMERIC(5,4),
  failure_rate NUMERIC(5,4),
  -- DLR quality
  handset_dlr_count INTEGER NOT NULL DEFAULT 0,
  network_dlr_count INTEGER NOT NULL DEFAULT 0,
  handset_dlr_rate NUMERIC(5,4),
  -- Latency
  avg_delivery_latency_ms INTEGER,
  p50_delivery_latency_ms INTEGER,
  p95_delivery_latency_ms INTEGER,
  p99_delivery_latency_ms INTEGER,
  -- Cost
  total_cost_microcents BIGINT NOT NULL DEFAULT 0,
  total_segments INTEGER NOT NULL DEFAULT 0,
  -- Per-message-type breakdown
  type_stats JSONB NOT NULL DEFAULT '{}',
  -- Example type_stats:
  -- {
  --   "otp": {"sent": 500, "delivered": 495, "avg_latency_ms": 850, "p95_latency_ms": 1800},
  --   "transactional": {"sent": 2000, "delivered": 1900, "avg_latency_ms": 2500},
  --   "promotional": {"sent": 5000, "delivered": 4200, "avg_latency_ms": 8000}
  -- }
  -- Per-carrier breakdown
  carrier_stats JSONB NOT NULL DEFAULT '{}',
  -- Per-country breakdown
  country_stats JSONB NOT NULL DEFAULT '{}',
  -- Failover stats
  failover_count INTEGER NOT NULL DEFAULT 0,
  failover_success_count INTEGER NOT NULL DEFAULT 0,
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (tenant_id, bucket, granularity)
);

CREATE INDEX idx_rm_analytics_tenant ON rm_delivery_analytics(tenant_id, bucket DESC);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Infrastructure | 3 | event_store (partitioned), stream_snapshots, projection_checkpoints |
| Read Models | 5 | rm_routing_config, rm_message_status, rm_carrier_health, rm_opt_out_list, rm_delivery_analytics |
| **Total** | **8** | 1 partitioned table (event_store) |

---

## Key Design Decisions

1. **Message lifecycle as a rich event stream** — A single message generates 5–10 events (queued → opt_out_checked → compliance_checked → hlr_lookup → routing_decided → submitted → sent → delivered). This granularity is what makes the event-sourced model valuable for SMS: every step in the dispatch pipeline is recorded, enabling post-hoc analysis of why a message took a specific route, how long each step took, and where failures occurred. This directly addresses the "transparent carrier routing" gap identified in the market research.

2. **Carrier health derived from events, not polled** — Carrier health status is projected from carrier.health_check_completed and message.submission_failed events into `rm_carrier_health`. This means health changes are traceable — you can replay the event stream to see exactly when a carrier went from healthy to degraded and which failures triggered the change. The read model includes rolling-window metrics (1h delivery rate, p95 latency) computed from message events.

3. **DLR accuracy tracking as first-class events** — Every DLR is an event with an explicit dlr_type (handset/network/intermediate/synthetic). The `rm_carrier_health` read model computes handset vs. network DLR rates per carrier. AI events flag suspected fake DLRs. This makes DLR accuracy auditable in a way that no incumbent currently offers.

4. **Opt-out as an immutable compliance stream** — Opt-out events (created, resubscribed, bulk_imported, carrier_feedback_received) form an immutable timeline that serves as TCPA/GDPR compliance proof. The `rm_opt_out_list` read model is the enforcement mechanism — checked before every send. If the read model is ever suspected of being inconsistent, it can be rebuilt from the opt_out stream.

5. **Compliance registration lifecycle as events** — 10DLC brand/campaign and India DLT entity/template registrations each generate a stream of events (submitted → approved/rejected → expired). This enables temporal compliance queries: "was this campaign registered when message X was sent?" — critical for regulatory audits where proving compliance at the time of sending matters.

6. **Conversation as a stream, not a table** — Two-way conversations are modelled as event streams (started → message_added → inbound_received → auto_reply_sent → webhook_delivered → closed). The conversation stream captures the full interaction lifecycle including webhook delivery attempts and failures, keyword detection, and auto-replies. No separate conversations table — the stream IS the conversation.

7. **Routing config as a pre-computed read model** — The dispatch hot path reads `rm_routing_config` — a single row per tenant containing the full carrier fleet, routing rules, phone numbers, and compliance state. This is projected from tenant, carrier, routing_rule, phone_number, and compliance stream events. The config_hash field enables cache invalidation: the dispatch engine caches the routing config and only re-reads when the hash changes.

8. **Actor types include infrastructure components** — Beyond user/system/ai, actor types include smpp_gateway (for SMPP-specific events), hlr_service (for HLR lookup events), keyword_processor (for STOP/START/HELP detection), and webhook_engine (for webhook delivery tracking). This fine-grained attribution enables per-component debugging and performance analysis.

9. **Delivery analytics with OTP-specific latency tracking** — The `rm_delivery_analytics` read model breaks down metrics by message type, with OTP latency percentiles (p50, p95, p99) as a first-class concern. This supports the latency SLA commitment identified in the project README — "OTP delivery 95th percentile under 2 seconds" — by making the metric directly queryable.

10. **12 stream types covering the full SMS domain** — tenant, user, carrier, phone_number, sender_id, routing_rule, compliance, message, opt_out, conversation, ai, and config. Each stream type has a well-defined event catalogue with specific field schemas. New stream types (e.g., rcs_message for future RCS support) can be added without modifying existing streams or read models.
