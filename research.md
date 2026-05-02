# Video Conferencing Platform

> Candidate #157 · Researched: 2026-05-02

## Existing Products and Software Packages

| Tool | Description | Type | Pricing | Strengths / Weaknesses |
|------|-------------|------|---------|------------------------|
| Zoom | Dominant video conferencing platform; 55.9% market share; USD 4.87B FY2026 revenue; AI Companion bundled | SaaS | Free tier; Pro USD 15.99/month; Business USD 19.99/month | Ubiquitous, reliable, strong AI features; commoditised; heavy client app |
| Microsoft Teams | 32.3% market share; deeply embedded in Microsoft 365 ecosystem; 320M DAU | SaaS | Bundled with M365 from USD 6/user/month | Enterprise-dominant; complex administration; video quality trails Zoom |
| Google Meet | 5.5% market share; browser-native, no download required; integrated with Workspace | SaaS | Bundled with Workspace from USD 7/user/month | Zero-install simplicity; limited advanced features; weak standalone value |
| Cisco Webex | 7.6% market share; enterprise-grade security, on-premises options, AI noise removal | SaaS / on-prem | Webex Free; Starter USD 14.50/user/month | Strong enterprise and government compliance; dated UI; declining consumer mindshare |
| Daily.co | WebRTC infrastructure-as-a-service with recording, transcription, and screen sharing APIs | API/SDK | Pay-as-you-go per minute | Developer-friendly; not an end-user product; requires build effort |
| Agora | Multi-platform WebRTC SDK (iOS, Android, Web, Unity) with cloud recording and AI features | API/SDK | Usage-based pricing | Widest platform footprint of any SDK provider; infrastructure not a finished product |
| Whereby | Embeddable video rooms with simple SDK; no download required for participants | SaaS / API | Free; Pro USD 6.99/month; API pricing varies | Easy embedding; limited scalability for large enterprises |

## Relevant Industry Standards or Protocols

- **WebRTC (Web Real-Time Communication)** — W3C/IETF standard enabling browser-native, plugin-free audio/video and data communication; the foundational technology for all modern conferencing platforms
- **SIP (Session Initiation Protocol) / SDP** — signalling protocol stack used for call setup, renegotiation, and teardown in VoIP and conferencing systems
- **SRTP (Secure Real-time Transport Protocol)** — IETF standard for encrypting audio and video streams in transit
- **DTLS (Datagram Transport Layer Security)** — key exchange protocol used in conjunction with SRTP for WebRTC security
- **WCAG 2.2 / Section 508** — accessibility standards requiring captions, keyboard navigation, and screen-reader compatibility in conferencing tools

## Available Research Materials

1. Future Market Insights (2026). *Web Real-Time Communication Market Forecast 2025 to 2035*. https://www.futuremarketinsights.com/reports/web-real-time-communication-solution-market
2. GetStream (2026). *Top 8 WebRTC Companies in 2026: Features & Pricing Comparison*. https://getstream.io/blog/webrtc-companies/
3. GetStream (2026). *The Future of Video Technology in 2026: 5 Shifts Developers Need to Know*. https://getstream.io/blog/future-video-technology/
4. Meetrix (2026). *Integrating AI into WebRTC Video Conferencing: Transforming User Experience and the Telecommunication Industry*. https://meetrix.io/articles/integrating-ai-into-webrtc-video-conferencing/
5. DemandSage (2026). *Zoom User Statistics 2026 — Market Share & Active Users*. https://www.demandsage.com/zoom-statistics/
6. ModernMeetingStandard (2026). *Online Meeting Software Statistics 2026: Market Size, Teams, Zoom, Webex, Google Meet*. https://modernmeetingstandard.com/online-meeting-software-statistics/

## Market Research

**Market Size:** The global video conferencing market is projected to reach USD 12 billion in 2026, growing at a CAGR of approximately 11.8%. The broader WebRTC market is expected to reach USD 10.89 billion by 2026.

**Funding:** Zoom is publicly traded (NASDAQ: ZM) with a market capitalisation of approximately USD 20 billion in early 2026. Daily.co raised USD 60 million in a 2021 Series B. Agora is publicly traded (NASDAQ: API). Whereby raised USD 13.9 million in 2021.

**Pricing Landscape:** Consumer and SMB tiers are broadly commoditised, with free tiers from Zoom, Teams, and Meet sufficient for most lightweight use cases. Enterprise pricing runs USD 14–20+/user/month. API/SDK infrastructure (Agora, Daily) uses per-minute usage billing, typically USD 0.0001–0.001 per participant-minute. AI-powered features (transcription, summaries, translation) command a premium across all tiers.

**Key Buyer Personas:** IT administrators at enterprises standardising on a video tool; developers embedding video into SaaS products or platforms; healthcare and telehealth providers requiring HIPAA-compliant recording and transcription; education institutions needing scalable webinar and classroom tools.

**Notable Trends:** AI is shifting from post-call processing (transcription, summaries) to real-time in-call processing (live captions, noise suppression, translation, coaching). Every major platform is embedding AI copilots. Zoom holds 55.9% market share despite intense competition. SMBs are consolidating video into their messaging platform (Teams, Slack Huddles) rather than maintaining a separate video tool.

## AI-Native Opportunity

- Real-time meeting transcription with speaker diarisation, searchable across all past meetings within a workspace
- AI meeting summaries and decision logs auto-generated at meeting end and distributed to attendees and relevant stakeholders
- In-call AI coach providing live prompts for active listening, question quality, and meeting agenda adherence
- Automatic action-item extraction with assignment to named participants and direct sync to project management tools
- AI-powered background noise suppression, video enhancement, and lighting correction running client-side for low-bandwidth participants
