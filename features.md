# SMS & Messaging Gateway — Feature & Functionality Survey

> Candidate #387 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Twilio | Commercial SaaS | Proprietary | https://www.twilio.com/en-us/messaging/channels/sms |
| Infobip | Commercial SaaS | Proprietary | https://www.infobip.com/ |
| Vonage (Nexmo) | Commercial SaaS | Proprietary | https://www.vonage.com/communications-apis/sms/ |
| Sinch | Commercial SaaS | Proprietary | https://sinch.com/ |
| Prelude | Commercial SaaS | Proprietary | https://prelude.so/ |
| AWS SNS | Commercial SaaS | Proprietary (AWS) | https://aws.amazon.com/sns/ |

## Feature Analysis by Solution

### Twilio

**Core features**
- Unified REST API for SMS, MMS, and WhatsApp messaging
- Message Delivery Service: message routing, sending, and receipt handling
- Messaging Services feature: grouping of global senders with intelligent sender selection
- Delivery tracking and status callbacks via webhooks
- Multi-channel fallback: MMS/RCS automatically falls back to SMS if delivery fails
- Message scheduling and long message handling
- Out-of-the-box messaging insights dashboards for troubleshooting
- Global coverage: 193+ billion messages annually at 99.95% uptime

**Differentiating features**
- AI-powered real-time routing with latency optimization and outage avoidance
- Messaging Services abstraction manages compliance regulations automatically
- Comprehensive integration across Twilio's broader communications platform (voice, video, email)
- Market-leading API maturity and developer ecosystem

**UX patterns**
- REST API-first design
- Programmable fallback logic (MMS → SMS routing)
- Dashboard-based campaign creation for simple use cases
- Webhook-driven delivery status updates

**Integration points**
- REST API for sending/receiving messages
- Webhooks for delivery receipts and inbound messages
- SDKs for JavaScript, Python, Java, Ruby, Go, C#
- Integration with Twilio Studio (visual workflow builder)
- Messaging Insights dashboards

**Known gaps**
- Vendor-specific APIs (not SMPP protocol-native)
- Less transparent pricing for complex routing scenarios
- Limited transparency on carrier relationships and routing decisions

**Licence / IP notes**
- Proprietary commercial service; no open source component

---

### Infobip

**Core features**
- REST API and SMPP protocol support for SMS and MMS delivery
- HLR (Home Location Register) lookup service for mobile number intelligence
- Intelligent routing: real-time route selection optimized for cost and delivery speed
- Mobile Number Portability (MNP) data integration
- Global compliance engine: automatic compliance with country-specific regulations
- SMS MT (Mobile Terminated) and MO (Mobile Originated) support
- Delivery receipt handling with DLR accuracy tracking
- Direct connections to 800+ mobile networks

**Differentiating features**
- HLR lookup service for improved delivery rates and MNP resolution
- Least Cost Routing: uses HLR results to optimize routing and reduce costs
- 800+ direct carrier connections for carrier-grade reliability
- Built-in global compliance configuration for multiple jurisdictions
- SMPP native support (not just REST)
- Carrier-grade platform with enterprise SLAs

**UX patterns**
- Dual protocol support: REST and SMPP
- HLR intelligence integrated into routing decisions
- Compliance-as-a-service (automatic regional rules)
- Number lookup and fraud prevention tools

**Integration points**
- REST API for message sending and delivery status
- SMPP protocol interface (with HLR system_type option)
- Number Lookup API (for HLR validation)
- Webhooks for delivery receipts and inbound messages
- Integration APIs for fraud detection and compliance

**Known gaps**
- Complex pricing model (HLR lookups add per-query costs)
- Steeper learning curve for SMPP configuration vs. REST-only providers
- Documentation can be dense for non-telecom teams

**Licence / IP notes**
- Proprietary commercial service; no open source component

---

### Vonage (Nexmo)

**Core features**
- SMS API with REST and SMPP protocol support
- Number Insight API: phone number validation, assessment, and enrichment
- Global phone number provisioning (virtual numbers for receiving SMS)
- Campaign analytics: delivery insights by carrier and country
- Verify API: SMS-based two-factor authentication
- Support for SMS, MMS, WhatsApp, and Viber
- Number insight for caller authentication and database cleaning
- Coverage in 160+ countries

**Differentiating features**
- Number Insight API: enrichment data on phone numbers for fraud and compliance
- Step-up authentication: triggers additional verification based on number risk assessment
- Ericsson telecom heritage: deep integration with carrier relationships
- Vonage Protection Suite: comprehensive security and verification features
- NLP-powered number analysis

**UX patterns**
- API-first approach with rich developer documentation
- Separate APIs for different concerns: SMS, Number Insight, Verify
- Dashboard for campaign analytics and monitoring
- Protocol support: REST and SMPP

**Integration points**
- REST API for SMS sending and receiving
- SMPP protocol interface
- Number Insight API for phone number enrichment
- Verify API for OTP delivery
- Webhooks for delivery receipts and inbound messages
- Phone number provisioning API

**Known gaps**
- Number Insight requires separate API calls (not transparent in SMS routing)
- Pricing model adds costs for enrichment services
- Some enterprise customers report slower support response times vs. Twilio

**Licence / IP notes**
- Proprietary commercial service (Ericsson-owned); no open source component

---

### Sinch

**Core features**
- SMS API with support for bulk messaging (batch delivery to multiple numbers)
- A2P (Application-to-Person) messaging with 10DLC, toll-free numbers, virtual long numbers, and short codes
- 600+ direct carrier connections globally
- 10DLC campaign setup: 5-step process with 3-5 day provisioning
- Tier-1, Tier-2, and Tier-3 carrier relationships in the US
- Conversation API for two-way messaging and conversational flows
- IDC MarketScape 2026 Leader in Communications Engagement Platforms

**Differentiating features**
- 10DLC specialist: fastest provisioning pathway (3-5 days) with clear process
- Batch SMS API: single call delivers to multiple recipients
- Direct carrier super-network: 600+ connections globally for reliability
- Conversation API: purpose-built for two-way interactive messaging
- A2P number format expertise: diverse number type support (10DLC, TFN, VLN, short codes)

**UX patterns**
- Carrier-grade platform mindset
- Batch messaging convenience for bulk campaigns
- Conversational API for modern chat-like experiences
- Clear A2P regulatory pathway documentation

**Integration points**
- REST API for SMS sending
- Conversation API for multi-turn messaging
- 10DLC provisioning APIs
- Webhooks for delivery receipts and inbound messages
- Carrier management interfaces
- Number provisioning APIs

**Known gaps**
- Less granular HLR/number intelligence compared to Infobip
- Analytics dashboards are functional but less polished than Twilio
- International routing optimization less transparent than Infobip's HLR-driven approach

**Licence / IP notes**
- Proprietary commercial service; no open source component

---

### Prelude

**Core features**
- SMS OTP verification API specialized for time-sensitive delivery
- Intelligent multi-routing: automatic failover to alternative carriers on congestion/failure
- Preferred channel routing: selects best channel per country (WhatsApp in Brazil, Viber in Ukraine, Zalo in Vietnam, SMS fallback globally)
- AI-driven routing optimization based on carrier performance, fraud signals, and geography
- Real-time carrier performance analysis
- Instant automatic failover to backup routes
- Developer-first, transparent pricing model

**Differentiating features**
- OTP-specialized optimization: routes for speed and reliability, not just cost
- Multi-channel fallback strategy: tries optimal channel first, cascades to SMS
- Real-time carrier performance learning: adapts routing dynamically
- Specialized fraud signal detection for OTP delivery
- Transparent routing analytics: visibility into why messages take certain paths
- Cost optimization through volume aggregation

**UX patterns**
- Developer-friendly API with clear documentation
- Real-time performance dashboards
- Transparent multi-route failover (no hidden rerouting)
- Specialized metrics for OTP use cases (attempt rates, conversion rates)

**Integration points**
- REST API for OTP verification
- Webhook support for delivery status
- SDKs for mobile and web
- Analytics dashboards with per-route visibility
- Multi-channel APIs (WhatsApp, Viber, SMS)

**Known gaps**
- Smaller market presence vs. Twilio/Infobip (less enterprise support)
- Limited number provisioning (focuses on outbound OTP delivery, not two-way)
- Analytics more specialized for OTP than general SMS campaigns

**Licence / IP notes**
- Proprietary commercial service; no open source component

---

### AWS SNS

**Core features**
- Simple SMS messaging via pub/sub interface
- 200+ country coverage from 32 AWS Regions
- SMS message types: promotional and transactional with different QoS settings
- Customizable SMS preferences: cost efficiency vs. reliable delivery trade-offs
- CloudWatch integration for delivery metrics and alerting
- Daily CSV usage reports to S3
- Real-time console dashboards for delivery rate visibility
- Cost-based pricing model aligned with AWS infrastructure

**Differentiating features**
- Native integration with AWS ecosystem: Lambda, DynamoDB, CloudWatch, etc.
- Infrastructure-as-code friendly: SMS configuration through CloudFormation/Terraform
- CloudWatch deep integration: metrics, logs, alarms for SMS delivery
- S3 daily delivery reports for analytics and auditing
- Architectural flexibility: promotional vs. transactional preference tuning

**UX patterns**
- Cloud-native mindset (SNS topic-based subscription model)
- Infrastructure-as-code orientation
- CloudWatch for operational observability
- Minimal configuration required for simple use cases

**Integration points**
- AWS SNS Pub/Sub API for message publishing
- CloudWatch Metrics and Logs
- S3 for daily usage reports
- Lambda for serverless trigger-based SMS
- IAM for access control
- EventBridge for event-driven messaging

**Known gaps**
- Limited number provisioning (no dedicated SMS numbers)
- No HLR or number intelligence features
- Minimal routing optimization or failover support
- Less granular compliance controls compared to dedicated SMS gateways
- No SMPP support (SNS API only)
- Smaller developer ecosystem for SMS use cases vs. Twilio

**Licence / IP notes**
- Proprietary AWS service; no open source component

---

## Cross-Cutting Feature Themes

### Table-Stakes Features

These capabilities must be present in any viable SMS gateway:

- REST API for sending SMS messages
- Delivery receipt handling (DLR) with webhook delivery
- Inbound message handling (SMS MO) with webhook delivery
- Number provisioning or carrier partnerships for message delivery
- Sender ID management (customizable from/sender field)
- Basic delivery and open analytics (delivery rate, sent count)
- Support for multiple message types (OTP, transactional, marketing)
- Integration with major carrier networks
- Geographic coverage in primary markets (US, EU, APAC)
- Cost-transparent pricing per message

### Differentiating Features

Capabilities that provide competitive advantage:

- HLR (Home Location Register) lookup for number intelligence and MNP resolution
- Multi-provider failover: automatic rerouting to backup carriers on primary failure
- Preferred channel optimization: routing to non-SMS channels (WhatsApp, Viber) when appropriate
- AI-driven routing: real-time carrier performance analysis and dynamic route selection
- 10DLC and compliance automation: one-click registration for US/India/EU requirements
- SMPP protocol support: enables enterprise integration with existing telecom infrastructure
- Batch SMS APIs: single request delivery to multiple numbers
- Two-way conversational messaging: Conversation API or dedicated inbound features
- Global compliance engine: automatic enforcement of regional rules (DLT in India, GDPR in EU, etc.)
- Detailed DLR accuracy tracking: transparency into handset vs. network DLRs
- Direct carrier relationships: branded as "600+ carriers" or "800+ networks" for credibility
- Send-time optimization: per-user or per-carrier optimal delivery timing

### Underserved Areas / Opportunities

Gaps that represent genuine opportunities for differentiation:

- Transparent carrier routing decisions: no gateway clearly exposes why a message took a specific route
- Grey-route risk mitigation: no clear tools to detect or prevent grey-route usage across providers
- Conversation state management: two-way messaging lacks standardized session/context handling
- DLR accuracy certification: no independent audit of DLR accuracy; providers self-report
- Regulatory compliance automation: while some providers offer 10DLC/DLT support, full end-to-end compliance (consent capture, record-keeping, audit trails) remains manual
- Latency SLAs per use case: no gateway clearly commits to OTP delivery SLAs (e.g., "95th percentile <2 seconds")
- Cost prediction and budgeting: providers lack transparent cost calculators for multi-carrier fallback scenarios
- Webhook reliability and ordering guarantees: no gateway guarantees webhook delivery order or deduplication across retries
- Custom routing rules: no provider enables per-segment or per-user routing policies without custom code
- Offline message queueing: no provider queues messages during carrier outages for delivery when service restores

### AI-Augmentation Candidates

Manual/rule-based features where AI could provide meaningful improvement:

- Optimal carrier selection: rule-based routing replaced by ML models predicting delivery success per carrier/region
- DLR accuracy prediction: ML models detecting fake/unreliable DLRs before they impact analytics
- Compliance inference: ML models auto-detecting message category (transactional vs. marketing) and applying regional rules
- Fraud detection: ML-driven detection of fraudulent sender IDs, numbers, and message patterns
- Preferred channel prediction: ML models learning user channel preferences beyond geography (e.g., WhatsApp over SMS for tech-savvy cohorts)
- Latency prediction: ML models predicting per-carrier delivery latency to optimize for time-critical use cases
- Cost optimization: ML models learning optimal carrier mix and route fallback order to minimize total cost while meeting delivery SLAs
- Message content moderation: AI-driven content scanning to flag messages that may trigger carrier filtering
- Recipient number validation: AI inference of valid/invalid numbers before sending to reduce wasted traffic
- Congestion detection: ML models predicting carrier congestion windows to optimize send times

---

## Legal & IP Summary

All major SMS gateway providers reviewed (Twilio, Infobip, Vonage, Sinch, Prelude, AWS SNS) are proprietary commercial services with no open source components. No patents were identified as blocking implementation of core SMS messaging functionality.

Critical compliance considerations identified for 2026:
- **US (10DLC)**: As of February 1, 2025, carriers block 100% of unregistered A2P 10DLC traffic; registration is mandatory for any business SMS use.
- **India (DLT)**: February 12, 2025 amendments introduced new message category suffix system (-P, -S, -T, -G), Service Explicit category discontinued, stricter UCC complaint mechanisms, and biometric authentication requirements.
- **EU/UK (GDPR)**: Compliance required across four areas including lawful basis for personal data processing (consent, contract, or legitimate interest).

DLR accuracy concerns emerged as a 2026 issue: distinction between "fake" intermediate DLRs (carrier acceptance only) and true handset delivery reports is increasingly important; no standardized audit or certification exists.

No copyright or licensing conflicts identified. Core SMS protocols (SMPP, carrier APIs) are standardized and open to implement. No material was omitted due to legal uncertainty.

---

## Recommended Feature Scope

Based on the above analysis, here is a prioritised feature scope for the project:

**Must-have (MVP)**
- Unified REST API for SMS sending and receiving
- Delivery receipt (DLR) handling with webhook delivery
- Inbound message handling (SMS MO) with webhook webhooks
- Single or multi-carrier provisioning (at minimum, partnership with one major carrier or SMPP aggregator)
- Sender ID management (customizable from/sender field)
- Basic delivery analytics (sent, delivered, failed counts)
- Support for OTP, transactional, and promotional message types
- Geographic coverage in primary markets (US, EU, APAC)
- Transparent per-message pricing

**Should-have (v1.1)**
- Multi-provider failover: automatic carrier switching on failure
- HLR lookup service for number intelligence and MNP resolution
- 10DLC and DLT compliance automation (US and India registration)
- GDPR consent record tracking and opt-out handling
- Batch SMS API for bulk messaging
- Two-way conversational messaging with inbound routing
- Webhook reliability: guaranteed delivery and deduplication
- Detailed DLR accuracy metrics (handset vs. network DLRs)
- SMPP protocol support for enterprise integration

**Nice-to-have (backlog)**
- Preferred channel routing: fallback to WhatsApp, Viber, etc. by geography
- AI-driven send-time optimization per carrier and region
- Global compliance engine: automatic enforcement of regional rules
- Custom routing rules: per-segment or per-user carrier selection
- Offline message queueing with automatic delivery on service restoration
- Fraud detection and content moderation for message filtering
- Cost prediction and budgeting calculator for failover scenarios
- Webhook ordering guarantees across multiple retry attempts
- Granular carrier performance analytics and route transparency
- SMS template management with dynamic variable substitution
