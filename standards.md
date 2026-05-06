# Standards & API Reference

> Project: SMS Messaging Gateway · Generated: 2026-05-06

## Industry Standards & Specifications

### 3GPP Standards

**3GPP TS 23.040 (GSM 03.40) — Short Message Service — Point to Point (SMS-PP)**
- URL: https://www.3gpp.org/DynaReport/23040.htm
- Defines the format of Transfer Protocol Data Units (TPDU) for the Short Message Transfer Protocol (SM-TP) used in GSM networks. This is the foundational standard for how SMS messages are structured and transported in mobile networks. Supersedes the original GSM 03.40 designation and is maintained by 3GPP.

**3GPP TS 23.038 (GSM 03.38) — Alphabets and Language-Specific Information**
- URL: https://www.3gpp.org/DynaReport/23038.htm
- ETSI reference: https://www.etsi.org/deliver/etsi_ts/123000_123099/123038/10.00.00_60/ts_123038v100000p.pdf
- Defines the GSM 7-bit default alphabet (160 characters per SMS), 8-bit data alphabet, and 16-bit UCS-2 encoding (for Chinese, Korean, Japanese and other non-Latin scripts, limited to 70 characters per message). Also defines National Language Shift Tables (since Release 8, 2008) for extended character support.

**3GPP TS 23.041 — Cell Broadcast Service (CBS)**
- URL: https://www.3gpp.org/DynaReport/23041.htm
- Defines the Cell Broadcast Service standard, which allows SMS-like messages to be broadcast to all mobile devices in a geographic area. Relevant for emergency alerting systems built on SMS infrastructure.

**3GPP TS 29.540 — 5G System; SMS Services**
- URL: https://www.etsi.org/deliver/etsi_ts/129500_129599/129540/15.06.00_60/ts_129540v150600p.pdf
- Specifies how SMS services are provided within 5G System (5GS) architectures, including the SMS Function (SMSF). Both MAP and Diameter protocols are supported for interoperability between the SMSF and legacy SMS Centres (SMSCs).

### IETF Standards

**RFC 3428 — SIP Extension for Instant Messaging (MESSAGE method)**
- URL: https://datatracker.ietf.org/doc/html/rfc3428
- Defines the MESSAGE method as an extension to SIP (Session Initiation Protocol) for transferring short messages over IP networks. Widely used in VoLTE (Voice over LTE) networks for SMS delivery over IP. The primary standard for SMS-like messaging over IP.

**RFC 8591 — SIP-Based Messaging with S/MIME**
- URL: https://datatracker.ietf.org/doc/rfc8591/
- Provides guidance on end-to-end authentication, integrity protection, and confidentiality for SIP-based messaging using S/MIME. Relevant for secure SMS gateway implementations and updates RFCs 3261, 3428, and 4975.

**RFC 5194 — Framework for Real-Time Text over IP Using SIP**
- URL: https://datatracker.ietf.org/doc/html/rfc5194
- Defines the framework for real-time text transmission over IP using SIP, relevant to messaging gateway implementations that support multiple transport mechanisms.

### ITU-T Standards

**ITU-T E.164 — The International Public Telecommunication Numbering Plan**
- URL: https://www.itu.int/rec/T-REC-E.164/
- Defines the international numbering plan used for all SMS addressing. E.164 numbers are limited to 15 digits: country code (1–3 digits) plus subscriber number (up to 12 digits), prefixed with '+'. All SMS gateway implementations must handle E.164-formatted numbers for international routing.

### GSMA Standards

**GSMA NG.111 — SMS Evolution**
- URL: https://www.gsma.com/newsroom/wp-content/uploads//NG.111-v3.0.pdf
- Describes the evolution pathway for SMS services across generations of mobile networks. Essential reference for gateway vendors supporting both legacy and modern SMS infrastructure.

**GSMA IR.75 — OPEN Connectivity SMS Hubbing Architecture**
- URL: https://www.gsma.com/newsroom/wp-content/uploads/IR.75-v2.0.pdf
- Specifies the architecture for SMS hubbing — the interconnection of SMS networks between different operators worldwide. Relevant for building a gateway that routes messages across multiple carriers.

**GSMA IR.82 — SS7 Security Network Implementation Guidelines**
- URL: https://www.gsma.com/solutions-and-impact/technologies/security/gsma_resources/ir-82-ss7-security-network-implementation-guidelines-v5-0/
- Outlines security measures for SS7 networks including MAP and CAP signalling with specific SMS security protections. Important for understanding and mitigating SS7-based attacks on SMS gateways.

**GSMA RCS Universal Profile 4.0 (March 2026)**
- URL: https://www.gsma.com/solutions-and-impact/technologies/networks/gsma_resources/rich-communication-service-rcs-february-2026-publications/
- The latest RCS specification, finalised March 2026, introducing native video calls (MIVC), rich text formatting, and streaming video in Rich Cards for business messaging. SMS gateways with RCS support should target this specification for modern deployments.

**GSMA RCS.07 — Specification for Enhanced Messaging**
- URL: https://www.gsma.com/solutions-and-impact/technologies/networks/gsma_resources/rcc-07-advanced-communications-services-and-client-specification/
- Defines the technical specifications for RCS Enhanced Messaging, including chatbots and A2P (application-to-person) business messaging. SMS gateways implementing RCS should reference this specification.

### Data Model & API Specifications

**SMPP 3.4 — Short Message Peer-to-Peer Protocol**
- URL: https://smpp.org/
- The industry-standard open protocol for high-throughput SMS exchange between External Short Message Entities (ESMEs), Message Centres (MCs), and Routing Entities. SMPP 3.4 is the most widely deployed version, adding TLV (Tag-Length-Value) parameters, non-GSM technology support, and transceiver mode (single connection for send and receive). IANA-assigned port: 2775 (TCP). Mandatory for any gateway connecting directly to carrier SMSC infrastructure.

**SMPP 5.0 — Short Message Peer-to-Peer Protocol**
- URL: https://smpp.org/
- The latest version of the SMPP protocol, adding cell broadcasting support and smart flow control. Not yet widely deployed as of 2026 but should be considered for future-proofing gateway implementations.

**OpenAPI Specification 3.x**
- URL: https://spec.openapis.org/oas/latest.html
- Modern SMS gateway REST APIs are built on the OpenAPI 3.x specification. Providers including Twilio, Vonage, Sinch, and Infobip expose OpenAPI-compliant documentation enabling SDK generation, Postman import, and Swagger UI exploration. Any new SMS gateway should publish an OpenAPI 3.x specification.

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749)**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The predominant authentication framework for SMS gateway REST APIs. Providers such as Vonage (Nexmo), Infobip, and Sakari use OAuth 2.0 for token-based API authentication. SMS gateways should implement OAuth 2.0 with scoped access tokens.

**NIST SP 800-63B — Digital Identity Guidelines: Authentication & Lifecycle Management**
- URL: https://pages.nist.gov/800-63-3/sp800-63b.html
- Classifies SMS-based OTP as a "restricted authenticator" in Revision 4, acknowledging risks from SIM-swap attacks, number porting, and SS7 vulnerabilities. Any SMS gateway offering authentication use-cases must communicate these limitations to integrators and offer alternative authentication pathways.

### Regulatory & Compliance Frameworks

**TCPA — Telephone Consumer Protection Act (US)**
- URL: https://www.fcc.gov/consumers/guides/stopping-unwanted-calls-and-texts
- US federal law administered by the FCC governing commercial SMS messaging. Requires explicit opt-in consent before sending A2P messages. Violations incur statutory damages of $500–$1,500 per unsolicited message. Since February 2025, one-to-one consent is required — blanket "partner marketing" consent clauses are no longer valid.

**A2P 10DLC — Application-to-Person 10-Digit Long Code**
- URL: https://www.campaignregistry.com/
- US carrier-mandated registration system for A2P SMS sent via 10-digit long codes, managed through The Campaign Registry (TCR). Registration is mandatory for all A2P SMS in the US as of February 2025. Gateways must support brand and campaign registration workflows to maintain message deliverability.

**GDPR — General Data Protection Regulation (EU)**
- URL: https://gdpr.eu/
- Governs the processing of personal data (including phone numbers) for EU/UK recipients. SMS gateways must support data subject rights, lawful basis documentation, data minimisation, and international transfer mechanisms. Overlaps with A2P 10DLC compliance for international operations.

**SHAKEN/STIR (RFC 8226, RFC 8588)**
- URL: https://datatracker.ietf.org/doc/html/rfc8226
- Caller ID authentication standards increasingly being extended to SMS contexts. Relevant for SMS gateways that need to demonstrate message origin authenticity to carriers.

---

## Similar Products — Developer Documentation & APIs

### Twilio Programmable Messaging

- **Description:** The largest CPaaS provider globally, offering SMS, MMS, RCS, and WhatsApp messaging via a unified REST API with extensive SDKs and carrier relationships.
- **API Documentation:** https://www.twilio.com/docs/messaging/api
- **SDKs/Libraries:** Node.js, Python, Ruby, Java, C#/.NET, PHP, Go — https://www.twilio.com/docs/libraries
- **Developer Guide:** https://www.twilio.com/docs/messaging/quickstart
- **Standards:** REST/JSON, OpenAPI 3.x compatible, HTTP Basic Auth with API credentials
- **Authentication:** HTTP Basic Auth (Account SID + Auth Token); also supports API Keys with secret

### Vonage (Nexmo) SMS API

- **Description:** CPaaS provider offering SMS, MMS, Voice, and WhatsApp through a REST API. Strong presence in Europe. Now part of Ericsson.
- **API Documentation:** https://developer.vonage.com/en/api/sms
- **SDKs/Libraries:** Node.js, Python, Ruby, Java, PHP, .NET — https://developer.vonage.com/en/tools
- **Developer Guide:** https://developer.vonage.com/en/messaging/sms/overview
- **Standards:** REST/JSON; OpenAPI-compliant documentation; JWT and API key authentication
- **Authentication:** API Key + Secret; JWT Bearer Tokens for newer APIs

### Sinch SMS API

- **Description:** Global messaging provider with direct carrier connections in 180+ countries, offering SMS, MMS, RCS, WhatsApp, and voice.
- **API Documentation:** https://developers.sinch.com/docs/sms/api-reference
- **SDKs/Libraries:** Java, .NET, Node.js, Python — https://developers.sinch.com/docs/sms/sdks/
- **Developer Guide:** https://developers.sinch.com/docs/sms/getting-started
- **Standards:** REST/JSON; Bearer Token (Service Plan ID + API Token)
- **Authentication:** Bearer Token using Service Plan ID and API Token issued from Sinch Dashboard

### Infobip SMS API

- **Description:** Enterprise-grade CPaaS platform offering SMS, email, WhatsApp, RCS, and more with global direct carrier network coverage.
- **API Documentation:** https://www.infobip.com/docs/api
- **SDKs/Libraries:** Python (https://github.com/infobip/infobip-api-python-client), Java, PHP, C#, Go, Node.js
- **Developer Guide:** https://www.infobip.com/docs/tutorials/send-your-first-sms-message-using-infobip-api
- **Standards:** REST/JSON; OpenAPI compliant; Postman collection available
- **Authentication:** API Key (`Authorization: App {API_KEY}`); OAuth 2.0 also supported

### MessageBird SMS API

- **Description:** Cloud communications platform offering SMS, voice, WhatsApp, and omnichannel customer engagement APIs with a RESTful interface.
- **API Documentation:** https://developers.messagebird.com/api/sms-messaging/
- **SDKs/Libraries:** Multi-language client SDKs — https://developers.messagebird.com/api/client-sdk
- **Developer Guide:** https://developers.messagebird.com/
- **Standards:** REST/JSON; HTTP verbs with RESTful endpoint structure
- **Authentication:** API Access Key passed in HTTP Authorization header

### Telnyx SMS API

- **Description:** Carrier-grade CPaaS built on owned network infrastructure, offering SMS, voice, fax, and wireless with competitive pricing.
- **API Documentation:** https://developers.telnyx.com/docs/messaging
- **SDKs/Libraries:** Node.js, Python, PHP, Java, Ruby — https://developers.telnyx.com/
- **Developer Guide:** https://devdocs.telnyx.com/docs/v2/messaging/messages/tutorials/quickstarts/
- **Standards:** REST/JSON; OpenAPI-compliant
- **Authentication:** API Keys; Bearer Token authentication

### Plivo SMS API

- **Description:** Developer-focused CPaaS with SMS and voice APIs, known for competitive pricing and straightforward REST API design.
- **API Documentation:** https://www.plivo.com/docs/sms
- **SDKs/Libraries:** Python, Node.js, Ruby, Java, PHP, .NET, Go — https://www.plivo.com/docs/
- **Developer Guide:** https://www.plivo.com/docs/home
- **Standards:** REST/JSON; RESTful API with webhooks for inbound messages and delivery receipts
- **Authentication:** Auth ID + Auth Token (HTTP Basic Auth)

### Amazon SNS (Simple Notification Service) SMS

- **Description:** AWS-native pub/sub messaging service with SMS capabilities for transactional and promotional message delivery worldwide.
- **API Documentation:** https://docs.aws.amazon.com/sns/latest/api/welcome.html
- **SDKs/Libraries:** All AWS SDKs (JavaScript, Python/boto3, Java, PHP, .NET, Go, Ruby) — https://aws.amazon.com/developer/tools/
- **Developer Guide:** https://docs.aws.amazon.com/sns/latest/dg/sms_sending-overview.html
- **Standards:** AWS REST API (Signature Version 4); JSON/XML; integrates with IAM for access control
- **Authentication:** AWS IAM (access key + secret key with SigV4 signing); IAM roles recommended

### SMSGlobal REST API

- **Description:** Australian-based SMS gateway provider offering global SMS delivery via REST API with delivery reporting and two-way messaging.
- **API Documentation:** https://www.smsglobal.com/us/rest-api/
- **SDKs/Libraries:** PHP, Python, Node.js, Ruby, Java
- **Developer Guide:** https://www.smsglobal.com/us/rest-api/
- **Standards:** REST/JSON; HMAC authentication
- **Authentication:** API Key + API Secret with HMAC-SHA256 request signing

---

## Notes

### Emerging Standards

- **RCS Universal Profile 4.0** (finalised March 2026) is rapidly becoming the successor to SMS for rich business messaging. Any SMS gateway targeting the next 3–5 years should plan an RCS migration or parallel support path.
- **WhatsApp Business API** and **Apple Messages for Business** represent parallel proprietary messaging channels that SMS gateways increasingly need to bridge — though neither follows open standards.
- **MCP (Model Context Protocol)** has not yet established specific conventions for SMS gateway integration, but AI agents increasingly need programmatic SMS capabilities; an MCP server wrapping an SMS gateway API is a viable architectural pattern.

### Protocol Gaps

- There is no universally adopted open standard for SMS gateway REST APIs. While most providers use REST/JSON with OpenAPI documentation, each provider has a proprietary API schema. The SMPP protocol remains the only widely adopted open standard at the transport layer.
- SS7/MAP vulnerabilities (location tracking, call interception, SMS interception) remain unresolved at the protocol level — gateways relying on SMPP to carrier networks are inherently exposed to these risks.
- NIST's classification of SMS OTP as a "restricted authenticator" may drive regulatory requirements for gateways to add disclosure and alternative MFA mechanisms to their authentication services.
