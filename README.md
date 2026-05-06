# SMS & Messaging Gateway

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source SMS gateway with multi-provider routing, number management, and built-in compliance for global A2P messaging.

The SMS & Messaging Gateway aggregates connections to multiple carriers and routes messages through the optimal path based on cost, delivery speed, and reliability. It is intended for teams building SMS-dependent products — OTP delivery, appointment reminders, fraud alerts, and two-way conversations — who need a provider abstraction that hides carrier complexity while offering transparent routing quality, delivery receipts, and cross-market compliance tooling.

---

## Why SMS & Messaging Gateway?

- Incumbents like Twilio offer limited transparency on carrier relationships and routing decisions, with pricing that becomes opaque for complex multi-carrier scenarios.
- Carrier-grade platforms such as Infobip deliver HLR intelligence and 800+ direct connections, but pricing models layer per-query lookup costs and SMPP configuration carries a steep learning curve.
- No major gateway clearly exposes why a message took a specific route, making grey-route detection and DLR accuracy auditing impossible for customers to verify independently.
- End-to-end regulatory compliance (10DLC in the US, DLT in India, GDPR in the EU) — including consent capture, record-keeping, and audit trails — remains largely manual across all incumbents.
- No provider commits to latency SLAs per use case (e.g. OTP delivery 95th percentile under 2 seconds) or guarantees webhook delivery order and deduplication across retries.

---

## Key Features

### Unified Messaging API

- REST API for sending and receiving SMS messages
- Delivery receipt (DLR) handling with webhook delivery
- Inbound message handling (SMS MO) with webhook delivery
- Sender ID management with customizable from/sender field
- Support for OTP, transactional, and promotional message types

### Multi-Carrier Routing

- Multi-provider failover with automatic carrier switching on failure
- HLR (Home Location Register) lookup for number intelligence and Mobile Number Portability resolution
- Batch SMS API for bulk delivery to multiple recipients in a single request
- SMPP protocol support for enterprise telecom integration
- Detailed DLR accuracy metrics distinguishing handset versus network receipts

### Compliance & Number Management

- 10DLC campaign registration automation for US A2P traffic
- DLT registration support for India, including the post-2025 message category suffix system
- GDPR consent record tracking and opt-out handling
- Number provisioning across long codes, short codes, toll-free numbers, and virtual mobile numbers
- Geographic coverage in primary markets (US, EU, APAC)

### Two-Way & Conversational Messaging

- Inbound routing with webhook-based conversation handling
- Webhook reliability with guaranteed delivery and deduplication
- Preferred channel routing with fallback to WhatsApp, Viber, and other channels by geography
- Custom routing rules for per-segment or per-user carrier selection
- Offline message queueing for delivery on service restoration

### Analytics & Observability

- Basic delivery analytics covering sent, delivered, and failed counts
- Granular carrier performance analytics and route transparency
- Cost prediction and budgeting for multi-carrier failover scenarios
- Transparent per-message pricing

---

## AI-Native Advantage

AI models can replace rule-based carrier selection with predictions of delivery success per carrier and region, and can flag fake or unreliable DLRs before they distort analytics. ML-driven compliance inference auto-detects message category (transactional versus marketing) and applies the correct regional rules, while fraud detection identifies suspicious sender IDs, numbers, and content patterns. Latency prediction and congestion-window detection optimise sends for time-critical OTP delivery, and content moderation flags messages likely to trigger carrier filtering before they leave the queue.

---

## Tech Stack & Deployment

The gateway is designed for both REST and SMPP protocol access, supporting modern API-first integrations alongside enterprise telecom infrastructure. Webhooks deliver delivery receipts and inbound messages. Deployment targets include self-hosted SMPP gateway servers (in line with TeleOSS-style enterprise deployments) and cloud-hosted multi-tenant operation. Carrier connectivity is established through direct relationships or SMPP aggregators, with a routing intelligence layer using HLR lookups and Mobile Number Portability data to determine the serving network before dispatch.

---

## Market Context

The global SMS market is projected to reach USD 12.6 billion in 2026. Incumbent providers — Twilio, Infobip, Vonage (Ericsson), Sinch, Prelude, and AWS SNS — are all proprietary commercial services with no open-source component. Primary buyers are engineering teams at product companies needing OTP, transactional, and conversational messaging, plus enterprise telecom integrators seeking SMPP-native multi-carrier routing.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
