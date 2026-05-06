# Standards & API Reference

> Project: Video Conferencing Platform · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

- **ISO/IEC 27001:2022** — Information security management systems; video conferencing platforms handling confidential business meetings must address A.8.3 (Information access restriction), A.8.11 (Data masking for recordings), and A.8.24 (Cryptography) for encrypted media streams. URL: https://www.iso.org/standard/82875.html

- **ISO/IEC 27018:2019 — Protection of PII in Public Clouds** — Governs processing of participant personal data (video/audio recordings, transcripts, biometric voiceprints) in cloud-hosted video conferencing services; requires retention policy transparency and data minimisation. URL: https://www.iso.org/standard/76559.html

### W3C & IETF Standards

- **RFC 3261 — SIP: Session Initiation Protocol** — IETF standard for initiating, managing, and terminating multimedia communication sessions; the dominant signalling protocol for video conferencing interoperability between enterprise systems; ASCII text-based, HTTP-inspired. URL: https://datatracker.ietf.org/doc/html/rfc3261

- **RFC 4566 — SDP: Session Description Protocol** — Defines the session description format used within SIP to negotiate media codecs, transport protocols, encryption keys, and bandwidth for audio/video sessions. URL: https://datatracker.ietf.org/doc/html/rfc4566

- **RFC 8829 — JavaScript Session Establishment Protocol (JSEP)** — Defines how WebRTC applications negotiate sessions using SDP offer/answer exchanges in the browser; the standard API specification for WebRTC session management. URL: https://datatracker.ietf.org/doc/html/rfc8829

- **RFC 8825 — Overview: Real-Time Protocols for Browser-Based Applications (WebRTC)** — Provides the architectural overview of WebRTC's protocol stack; the foundational IETF RFC for the WebRTC standards suite. URL: https://datatracker.ietf.org/doc/html/rfc8825

- **RFC 8831 — WebRTC Data Channels** — Defines the SCTP-based data channel mechanism for non-media data exchange in WebRTC sessions; used for chat, file transfer, and application signalling within video calls. URL: https://datatracker.ietf.org/doc/html/rfc8831

- **RFC 3550 — RTP: Real-Time Transport Protocol** — Defines the end-to-end network transport functions for real-time audio/video media; used in all video conferencing implementations. URL: https://datatracker.ietf.org/doc/html/rfc3550

- **RFC 3711 — SRTP: Secure Real-time Transport Protocol** — Provides encryption, message authentication, and integrity protection for RTP streams; mandatory for secure video conferencing implementations. URL: https://datatracker.ietf.org/doc/html/rfc3711

- **ITU-T H.323** — ITU Telecommunication Standardization standard for audio-visual communication over packet networks; the legacy enterprise video conferencing protocol (1996) still in use in on-premises systems; being replaced by SIP/WebRTC in modern deployments. URL: https://www.itu.int/rec/T-REC-H.323

- **W3C WebRTC API** — W3C specification defining the browser JavaScript API for real-time audio/video communication (getUserMedia, RTCPeerConnection, RTCDataChannel); the foundation for browser-based video conferencing. URL: https://www.w3.org/TR/webrtc/

- **WCAG 2.2 — Web Content Accessibility Guidelines** — Governs accessibility of video conferencing web clients; requires captioning (1.2.4 Captions Live), keyboard navigation, sufficient colour contrast, and screen-reader compatibility; referenced in GDPR DPA guidance for equal access obligations. URL: https://www.w3.org/TR/WCAG22/

- **RFC 6455 — WebSocket Protocol** — Used for signalling channel communication between video conferencing clients and servers (SFU control, presence, chat messages). URL: https://datatracker.ietf.org/doc/html/rfc6455

### Data Model & API Specifications

- **OpenAPI 3.1** — Zoom, Daily.co, 100ms, and LiveKit Cloud use OpenAPI 3.1 to document their REST management APIs for meeting scheduling, recording management, and participant administration. URL: https://spec.openapis.org/oas/latest.html

- **WebRTC SFU Architecture** — Selective Forwarding Unit (SFU) is the dominant server architecture for scalable video conferencing; used by LiveKit, Jitsi Videobridge, Janus, mediasoup; routes media streams without transcoding. No single standard document but implemented per RFC 8825 principles.

- **Simulcast / SVC** — Scalable Video Coding and simulcast techniques (standardised within WebRTC specifications) enable adaptive quality based on bandwidth; used by all modern video conferencing SFUs for quality optimisation.

- **OAuth 2.0 (RFC 6749)** — All major video conferencing APIs (Zoom, Google Meet, Microsoft Teams) use OAuth 2.0 for authorising third-party app integrations and calendar-connected meeting scheduling. URL: https://datatracker.ietf.org/doc/html/rfc6749

### Security & Authentication Standards

- **GDPR Articles 6, 13, 32 — Lawful Processing, Transparency, and Security** — Video conferencing recording requires explicit opt-in consent (2026: "implied consent is dead" per EDPB); platforms must implement E2EE option, configurable retention, data residency in EU, and transparent AI feature disclosures. URL: https://gdpr-info.eu/art-32-gdpr/

- **GDPR Recital 51 / Article 9 — Biometric Data** — March 2026 legal analysis: AI speaker recognition features create voiceprints classified as biometric data under GDPR Article 9; requires explicit documented opt-in and DPA agreements covering biometric processing. URL: https://gdpr-info.eu/art-9-gdpr/

- **EU AI Act (2024/2847)** — AI transcription, automated summarisation, and speaker recognition features in video conferencing platforms are subject to GPAI model transparency requirements (enforcement from August 2025); platforms must disclose AI use and provide opt-out mechanisms. URL: https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX%3A32024R2847

- **HIPAA — §164.312 — Technical Safeguards** — Video conferencing platforms used for telehealth must implement access controls, audit controls, integrity controls, and transmission security; requires Business Associate Agreements (BAAs) with platform providers. URL: https://www.hhs.gov/hipaa/for-professionals/security/

- **SOC 2 Type II** — De facto enterprise compliance certification for SaaS video conferencing platforms; required by healthcare and financial service customers alongside HIPAA/GDPR DPAs. URL: https://www.aicpa-cima.com/topic/audit-assurance/audit-and-assurance-greater-than-soc-2

- **DTLS-SRTP** — IETF standard (RFC 5764) for key agreement in WebRTC media encryption; mandatory in all WebRTC implementations; provides both transport security (DTLS) and media encryption (SRTP). URL: https://datatracker.ietf.org/doc/html/rfc5764

- **OWASP API Security Top 10 (2023)** — Governs security of video conferencing REST APIs; API2 (Broken Authentication) and API5 (Broken Function Level Authorization) are critical for meeting access control and recording management. URL: https://owasp.org/API-Security/

### MCP Server Specifications

Video conferencing platforms are integrating with the MCP ecosystem for AI-assisted meeting management:

- **Zoom MCP Server** — Community MCP server enabling AI agents to schedule and manage Zoom meetings; requires Zoom API credentials. URL: https://mcpservers.org/servers/JavaProgrammerLB/zoom-mcp-server

- **Zoom MCP (echelon-ai-labs)** — Open-source MCP server implementation for Zoom meeting management via AI agents. URL: https://github.com/echelon-ai-labs/zoom-mcp

- **LiveKit Agents Framework** — LiveKit's built-in AI agent framework enables voice-based AI agents to join video rooms as participants, with speech-to-text, LLM processing, and text-to-speech pipeline; a form of native MCP-compatible agentic integration. URL: https://github.com/livekit/livekit

---

## Similar Products — Developer Documentation & APIs

### Zoom (REST API + Meeting SDK)

- **Description:** Largest commercial video conferencing platform globally; REST API for meeting management, recordings, participants, and reports; Meeting SDK for embedding Zoom meetings in applications; Zoom Phone, Zoom Events, and AI Companion APIs extending the platform.
- **API Documentation:** https://developers.zoom.us/docs/api/
- **Meeting SDK:** https://developers.zoom.us/docs/meeting-sdk/
- **Meetings API Reference:** https://developers.zoom.us/docs/api/meetings/
- **SDKs/Libraries:** JavaScript Meeting SDK; iOS SDK; Android SDK; REST API SDKs (Python, Node.js)
- **Developer Guide:** https://developers.zoom.us/docs/
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0, WebRTC (Meeting SDK), WebSockets (real-time events)
- **Authentication:** OAuth 2.0 (user-managed or server-to-server JWT); Zoom App credentials

### Microsoft Teams (Microsoft Graph API)

- **Description:** Enterprise video conferencing integrated into Microsoft 365; meetings accessed via Microsoft Graph API; Teams Live Events for large-scale broadcast; Teams calling via PSTN/SIP; expanding with Teams AI Library for conversational bots.
- **API Documentation:** https://learn.microsoft.com/en-us/graph/api/resources/teams-api-overview and https://learn.microsoft.com/en-us/graph/teams-messaging-overview
- **SDKs/Libraries:** Microsoft Graph SDKs (Python, .NET, Go, Java, JavaScript); Teams Bot Framework SDK; Teams Toolkit
- **Developer Guide:** https://developer.microsoft.com/en-us/graph
- **Standards:** REST/JSON, OpenAPI 3.1, OAuth 2.0 (Azure AD), WebSocket (real-time events), SIP (PSTN integration)
- **Authentication:** Azure AD OAuth 2.0 (delegated or application permissions)

### Google Meet (REST API + Media API)

- **Description:** Google's enterprise video conferencing platform integrated with Google Workspace; REST API for meeting creation and management; Google Meet Media API (2024) provides real-time audio/video stream access for bots and recording integrations.
- **API Documentation:** https://developers.google.com/meet/api/reference/rest
- **Google Meet Media API:** https://developers.google.com/meet/media-api
- **SDKs/Libraries:** Google API client libraries (Python, Go, Java, .NET, Node.js); OAuth 2.0 client libraries
- **Developer Guide:** https://developers.google.com/meet
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0, WebRTC (Media API)
- **Authentication:** Google OAuth 2.0 (sensitive scopes requiring verification); service accounts for server-to-server

### Jitsi Meet (Open Source)

- **Description:** Open-source (Apache 2.0) WebRTC video conferencing platform maintained by 8x8; self-hostable; components include Jitsi Videobridge (SFU), Jicofo (focus), Jibri (recording), Jigasi (SIP gateway); IFrame API for embedding; free public server at meet.jit.si (up to 100 participants).
- **API Documentation:** https://jitsi.github.io/handbook/docs/dev-guide/dev-guide-iframe
- **IFrame API:** Embedded via `<script src="https://meet.jit.si/external_api.js">`
- **SDKs/Libraries:** Jitsi Meet IFrame API (JavaScript); React Native SDK; Jitsi Android/iOS SDKs; jitsi-meet (npm)
- **Developer Guide:** https://jitsi.github.io/handbook/
- **Standards:** WebRTC, SIP (Jigasi), XMPP (Jicofo signalling), Colibri (JVB control protocol)
- **Authentication:** JWT tokens for room authentication (optional); open rooms by default on public server

### LiveKit (Open Source + Cloud)

- **Description:** Open-source (Apache 2.0) WebRTC SFU platform; self-hostable or managed cloud (LiveKit Cloud with free tier 5,000 participant-minutes/month); notable for built-in AI Agents framework (speech-to-text → LLM → text-to-speech pipeline) as of 2025; global edge nodes for managed tier.
- **API Documentation:** https://docs.livekit.io/
- **SDKs/Libraries:** livekit-client (JavaScript); Python SDK; Go SDK; Swift iOS SDK; Android Kotlin SDK; Unity SDK; Rust SDK; LiveKit Agents (Python)
- **Developer Guide:** https://docs.livekit.io/
- **Standards:** WebRTC, OpenAPI, WebSocket (signalling), gRPC (server SDK), OAuth 2.0 / JWT for room access
- **Authentication:** JWT access tokens signed with API key/secret; room-level permissions encoded in token claims

### Daily.co

- **Description:** Developer-focused video API with pre-built UI components and pure WebRTC infrastructure; iframe embed or custom UI via call object; 1-minute quickstart to working video call; strong in healthcare, EdTech, and telehealth use cases.
- **API Documentation:** https://docs.daily.co/reference
- **SDKs/Libraries:** daily-js (JavaScript); React daily-react; iOS Daily Swift; Android daily-android; Python REST client
- **Developer Guide:** https://docs.daily.co/
- **Standards:** WebRTC, REST/JSON, OpenAPI, OAuth 2.0
- **Authentication:** API key (daily-auth header for REST); meeting tokens (JWT) for per-participant room access control

### 100ms

- **Description:** Fully managed WebRTC video/audio conferencing platform with pre-built UI kits; targets education, healthcare, and entertainment; provides recording, live streaming (RTMP), and interactive live streaming (ILS); no self-hosting required.
- **API Documentation:** https://www.100ms.live/docs/api-reference/
- **SDKs/Libraries:** hms-video-store (JavaScript/React); iOS Swift SDK; Android Kotlin SDK; Flutter SDK; React Native SDK; Unity SDK
- **Developer Guide:** https://www.100ms.live/docs/
- **Standards:** WebRTC, REST/JSON, OpenAPI, OAuth 2.0
- **Authentication:** API key + secret for REST management; room tokens (JWT) for participant access

### Recall.ai

- **Description:** Cross-platform meeting bot API providing recordings, transcripts, and metadata from Zoom, Google Meet, Microsoft Teams, and Webex without platform-specific SDK complexity; desktop recording SDK and mobile recording SDK also available.
- **API Documentation:** https://docs.recall.ai/
- **SDKs/Libraries:** REST API; Python SDK; webhook events
- **Developer Guide:** https://docs.recall.ai/
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0, WebSocket (real-time transcript streaming)
- **Authentication:** API key (Authorization header)

---

## Notes

- **SFU dominance in 2025-2026**: Selective Forwarding Units (LiveKit, Jitsi Videobridge, mediasoup) have displaced older MCU (Multipoint Control Unit) architectures for scalable video conferencing; SFU routes streams without transcoding, dramatically reducing server compute costs.

- **WebRTC AI agent integration**: LiveKit's Agents framework (2025) represents a significant evolution — AI agents can join video rooms as first-class participants with STT/LLM/TTS pipelines, enabling real-time AI meeting assistance, coaching, and interpretation. This is the primary architecture for MCP-integrated video conferencing.

- **GDPR recording consent (2026)**: The EDPB's 2026 Coordinated Enforcement Framework focus on transparency (Articles 12-14) means all video conferencing AI features (transcription, summarisation, speaker ID) must be explicitly disclosed with opt-in consent; "implied consent" is no longer sufficient under current DPA guidance.

- **Biometric voiceprints (March 2026)**: Legal analysis confirms that AI speaker recognition features create biometric identifiers requiring GDPR Article 9 explicit consent — a significant compliance risk for platforms with automatic AI transcription features.

- **Open-source landscape**: Jitsi Meet (Apache 2.0) and LiveKit (Apache 2.0) are the dominant production-ready open-source options; LiveKit's AI agents framework makes it particularly compelling for AI-native video conferencing applications.

- **H.323 legacy**: H.323 (ITU-T, 1996) remains in use in legacy enterprise telephony infrastructure; modern platforms provide SIP/H.323 gateways (Jigasi for Jitsi) for backward compatibility while moving to WebRTC for browser clients.
