# Video Conferencing Platform — Feature & Functionality Survey

> Candidate #157 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Zoom | Commercial SaaS | Proprietary; per-seat and free tiers | https://zoom.com |
| Microsoft Teams | Commercial SaaS | Bundled with Microsoft 365 | https://teams.microsoft.com |
| Google Meet | Commercial SaaS | Bundled with Google Workspace | https://meet.google.com |
| Cisco Webex | Commercial SaaS / on-premises | Proprietary | https://www.cisco.com/products/webex |
| Daily.co | WebRTC API/SaaS | Proprietary; pay-as-you-go | https://daily.co |
| Agora | WebRTC SDK | Proprietary; usage-based | https://www.agora.io |
| Whereby | Embedded API/SaaS | Proprietary; usage-based | https://whereby.com |

## Feature Analysis by Solution

### Zoom

**Core features**
- HD/UHD video and audio with codec support for low-bandwidth scenarios
- Screen sharing with annotation tools
- Recording to local or cloud storage with automatic transcription
- Live transcription and real-time captions
- Virtual and custom backgrounds with blur options
- Meeting gallery and speaker view
- Breakout rooms for group discussions
- Whiteboard for collaborative drawing and brainstorming
- File sharing and in-meeting chat
- Meeting scheduling with calendar integration
- User presence and participant management

**Differentiating features**
- Zoom AI Companion providing meeting summaries, smart recording with chapters and takeaways
- Live interpretation with simultaneous translation channels allowing multilingual meetings
- Smart Recording organizing cloud recordings into intelligent chapters with key takeaways
- AI Meeting Summary automatically generated at meeting end and distributed to attendees
- Zoom Workplace (AI Companion 3.0) with conversational AI work surface for insights and actions
- AI Docs, AI Sheets, AI Slides enabling creation and editing of documents directly from meeting context
- Dominant market position (55.9% share) with proven reliability and ubiquity
- Granular security and encryption controls

**UX patterns**
- Meeting-first design with clear host and participant controls
- Gallery view for distributed attention
- One-click recording and transcription initiation
- AI-first features in higher-tier plans
- Simple join experience (one URL)

**Integration points**
- Calendar integration (Google, Outlook, iCal)
- Webhooks for custom integrations
- Zoom App Marketplace with 1,500+ apps
- Slack, Microsoft Teams, Salesforce, Jira integrations
- API for custom SDK implementations
- OAuth for authentication

**Known gaps**
- Heavy desktop client download required for full feature access
- AI features bundled only in higher-tier plans
- Recording storage limited or requires separate purchase
- Video quality subjectively trails competing platforms in some enterprise benchmarks

**Licence / IP notes**
- Proprietary SaaS; publicly traded (NASDAQ: ZM). USD 4.87B FY2026 revenue. AI Companion Policy Disclaimer required from all participants when AI features enabled as of January 2026

---

### Microsoft Teams

**Core features**
- HD video conferencing with screen sharing
- Live transcription of meetings with speaker names and timestamps
- Automatic transcription stored with cloud recording
- Call recording to OneDrive with searchable transcripts
- Virtual backgrounds and blur options
- Meeting gallery and speaker view
- Breakout rooms for team discussions
- Whiteboard for collaboration (shared across Microsoft Whiteboard app)
- In-meeting chat and file sharing
- Meeting scheduling with Outlook integration
- Calendar sync and attendee management
- Facilitator Agent providing real-time meeting notes capturing discussion topics, key points, and action items

**Differentiating features**
- Deep Microsoft 365 integration (Outlook, OneDrive, SharePoint, Word, Excel, PowerPoint)
- Live translated transcription (Teams Premium)
- Facilitator Agent automatically capturing discussion topics, key points, and action items during meetings
- Native OneNote integration for meeting notes
- Seamless handoff to Office apps for document creation
- Enterprise-dominant deployment (32.3% market share; 320M DAU)
- On-premises deployment options for government and sensitive workloads

**UX patterns**
- Unified collaboration workspace combining messaging, calling, and productivity
- Transcript and recording automatically stored in meeting organizer's OneDrive
- Facilitator-assistant pattern providing autonomous note-taking
- Enterprise-first permission and compliance model

**Integration points**
- Microsoft 365 ecosystem (all Office apps, Outlook, SharePoint, OneDrive)
- Webhook-based integrations
- Azure AD for identity and provisioning
- SCIM for bulk user management
- Third-party app integrations through Teams app marketplace
- OneNote and Loop components for collaborative notes

**Known gaps**
- Video quality reported as trailing Zoom in some benchmarks
- Complex administration and permission model
- Heavy emphasis on meeting functionality over other collaboration features
- Steep learning curve for non-Microsoft organizations

**Licence / IP notes**
- Proprietary; bundled with Microsoft 365 (USD 6+/user/month). Core business unit generating over USD 8 billion in ecosystem revenue

---

### Google Meet

**Core features**
- Browser-native video conferencing (no download required)
- Screen sharing with annotation
- Recording to Google Drive with automatic transcription
- Live captions in 30+ languages with AI speech recognition
- Noise suppression (AI-powered)
- Virtual backgrounds
- Low-light mode for improved video quality in dark environments
- Meeting grid and spotlight views
- In-meeting chat and file sharing
- Calendar integration with Google Calendar
- Participant attendance tracking

**Differentiating features**
- Zero-install simplicity; works entirely in browser
- Live captions in 30+ languages out-of-the-box
- Low-light mode (AI-powered) improving video quality in poor lighting
- Seamless Google Workspace ecosystem integration (Gmail, Calendar, Drive, Docs, Sheets)
- Simple user experience focused on reliability
- Strong accessibility features with captions and screen reader support

**UX patterns**
- Browser-first, no-install design
- Workspace-native integration
- Simple, streamlined interface with minimal configuration
- Accessibility-first caption support

**Integration points**
- Google Workspace (Calendar, Drive, Gmail, Docs, Sheets, Slides)
- Google Classroom for education deployments
- Calendar sync for scheduling
- Chrome extensions for meeting launching
- Google Meet hardware for dedicated rooms

**Known gaps**
- Limited standalone value outside Google Workspace ecosystem
- Fewer advanced features (breakout rooms limited, whiteboard basic)
- Smaller third-party integration ecosystem
- Recording storage limited by Google Drive quota

**Licence / IP notes**
- Proprietary; bundled with Google Workspace (USD 7+/user/month)

---

### Cisco Webex

**Core features**
- High-definition video with multiple codec support
- Screen and content sharing with annotation
- Cloud or on-premises recording options
- Live transcription and closed captions
- AI-powered background noise removal and speaker focus
- Virtual and custom backgrounds
- Breakout rooms for team discussions
- Webex Whiteboard for collaborative sessions
- Meeting chat and file sharing
- Calendar integration (Outlook, Google Calendar)
- Security-focused encryption and enterprise controls
- Meeting series and recurring scheduling

**Differentiating features**
- Enterprise and government compliance (FedRAMP certified, HIPAA compliant)
- On-premises deployment options for air-gapped networks
- Strong AI noise suppression and speaker enhancement
- Secure meeting room infrastructure with government-grade encryption
- Deep integration with Cisco infrastructure and security products
- Extensive automation and API capabilities

**UX patterns**
- Enterprise-focused security and control
- Advanced administrative permissions and audit trails
- On-premises deployment option for sensitive workloads

**Integration points**
- Outlook and Google Calendar for scheduling
- Slack, Microsoft Teams, Salesforce, Jira connectors
- On-premises infrastructure (Cisco infrastructure)
- Webhooks and REST APIs
- SCIM for identity provisioning

**Known gaps**
- Declining consumer mindshare and dated UI compared to Zoom/Teams/Meet
- Higher complexity and learning curve
- Premium pricing
- Smaller ecosystem compared to Zoom

**Licence / IP notes**
- Proprietary SaaS and on-premises; 7.6% market share. Enterprise-grade encryption and security certifications

---

### Daily.co

**Core features**
- WebRTC-based video API with REST and JavaScript SDK
- Prebuilt UI components for rapid integration
- Cloud recording with automatic transcription
- Screen sharing with whiteboard
- Custom room features and dynamic room creation
- Real-time transcription API
- Meeting access controls and permissions
- Participant roster and presence management
- GDPR and HIPAA compliance
- Simple pricing model (pay-as-you-go per participant-minute)
- Low-latency, globally distributed edge network

**Differentiating features**
- Developer-friendly API and clean documentation
- Fast time-to-integration with prebuilt components
- Flexible custom build options alongside prebuilt UI
- Transparent pricing model with no per-seat surcharges
- Global edge network for low-latency delivery
- Comprehensive API for custom workflows and automation

**UX patterns**
- API-first design allowing both prebuilt and custom UIs
- Headless video option for programmatic control
- Developer-centric tooling and documentation

**Integration points**
- REST API for room creation, participant management, recordings
- JavaScript SDK for browser integration
- Webhooks for event notifications
- Third-party SIP/PSTN gateways

**Known gaps**
- Not an end-user product; requires significant development effort
- Smaller ecosystem compared to end-user platforms
- Limited out-of-the-box features (no AI assistant, no meeting scheduling)
- Requires own infrastructure for features like transcription

**Licence / IP notes**
- Proprietary API/SaaS model; raised USD 60 million Series B in 2021

---

### Agora

**Core features**
- Multi-platform WebRTC SDK (iOS, Android, Web, Unity, Windows, macOS)
- Real-time voice and video with HD/FHD/UHD support
- Screen sharing and whiteboard
- Cloud recording and cloud playback
- AI Noise Suppression for high-quality audio
- Real-Time Speech to Text transcription
- Low-latency streaming with audience interaction
- Global SD-RTN (Software Defined Real-Time Network) for reliability
- Chat and messaging within video context
- Presence and participant management
- Analytics and quality monitoring dashboard

**Differentiating features**
- Widest platform footprint of any SDK provider (web, iOS, Android, desktop, gaming engines)
- Integrated real-time transcription API
- AI-powered noise suppression native to platform
- Lowest-cost option for high-volume use cases
- Global edge network with automatic failover
- Interactive live streaming combining video conferencing with broadcast audience

**UX patterns**
- SDK-first approach requiring significant development
- Programmatic control via APIs
- Infrastructure-centric, not end-user focused

**Integration points**
- Multi-platform SDKs (iOS, Android, Web, Windows, macOS, Unity)
- REST APIs for server-side operations
- Webhooks for events
- Cloud recording and playback APIs
- Agora App Builder for no-code rapid deployment

**Known gaps**
- Not a finished end-user product; requires build effort
- Limited out-of-the-box features
- Smaller app ecosystem
- Requires technical expertise to implement

**Licence / IP notes**
- Proprietary SDK; publicly traded (NASDAQ: API). Usage-based pricing model

---

### Whereby

**Core features**
- Embeddable video API with simple SDK integration
- No download required for participants (browser-native)
- Prebuilt and customizable room interfaces
- Live transcription support
- Virtual whiteboard for collaboration
- Participant management and access controls
- GDPR and HIPAA compliance
- ISO 27001 certification
- Dynamic room creation via API
- Pay-as-you-go pricing

**Differentiating features**
- Easy embedding into SaaS products and websites
- Simple, white-label customization
- No download required for end users
- Compliance-first design (GDPR, HIPAA, ISO 27001)
- Scalable infrastructure for embedded use cases

**UX patterns**
- Embedded widget design
- API-driven room management
- White-label customization for partners

**Integration points**
- JavaScript SDK for embedding
- REST API for room management
- Webhook support for events
- OAuth for authentication

**Known gaps**
- Limited scalability for very large enterprises
- Smaller ecosystem and fewer integrations
- Limited AI features
- Smaller feature set compared to full platforms

**Licence / IP notes**
- Proprietary SaaS/API model; raised USD 13.9 million in 2021. ISO 27001, GDPR, HIPAA compliant

---

## Cross-Cutting Feature Themes

### Table-Stakes Features

- HD video and audio codec support with automatic quality adaptation
- Screen sharing with annotation tools
- Cloud recording and local recording options
- Live captions and transcription
- Virtual background and blur options
- Meeting scheduling with calendar integration
- Breakout rooms or small group features
- Chat within meeting context
- Participant roster and management
- Access controls and security (encryption, passwords, waiting rooms)
- Mobile clients (iOS, Android)
- WebRTC or equivalent real-time protocol
- Meeting analytics (duration, participant count, etc.)

### Differentiating Features

- AI-powered meeting summaries and action item extraction (Zoom, Teams, Google, Daily)
- AI noise suppression and audio enhancement (Zoom, Teams, Google, Cisco, Agora)
- Real-time transcription in 30+ languages (Google Meet, Zoom)
- Whiteboard and collaborative drawing (all platforms)
- On-premises deployment options (Teams, Cisco Webex)
- Deep ecosystem integrations (Zoom 1,500+ apps; Teams Microsoft 365; Meet Google Workspace)
- Browser-native zero-install experience (Google Meet, Whereby)
- Facilitator/Assistant agent capturing notes automatically (Teams)
- API/SDK for custom integration (Daily, Agora, Whereby)
- Live interpretation and translation channels (Zoom)

### Underserved Areas / Opportunities

- **AI-powered real-time coaching**: No platform yet provides live guidance for meeting participation quality (active listening, question quality, agenda adherence)
- **Multi-party transcription with speaker diarization**: Most platforms transcribe but struggle with accurate speaker identification in large groups
- **Meeting effectiveness analytics**: Limited analysis of meeting patterns, decision velocity, participation equity
- **Searchable meeting knowledge base**: Transcripts and recordings exist but are rarely indexed and searchable across the organization
- **Accessibility beyond captions**: Limited support for sign language interpretation, live translation beyond a few languages, accessibility for cognitive differences
- **Energy-efficient video delivery**: No platform optimizes for bandwidth and energy usage at scale
- **Cross-platform federation**: Cannot easily interoperate with other video platforms or legacy systems
- **Offline-first and resilient**: Heavy reliance on cloud connectivity; limited offline capability
- **Asynchronous video feedback**: Limited tools for recorded message exchanges with timestamp-based feedback

### AI-Augmentation Candidates

- **Real-time meeting transcription with speaker diarization**: LLM-enhanced transcription with accurate speaker identification and context awareness
- **Automated action item extraction**: Identifying and assigning action items mentioned during meeting with deadline extraction
- **Meeting summaries and decision logs**: Generating executive summaries, decision logs, and follow-up task lists automatically
- **Real-time meeting coaching**: LLM-powered suggestions for question quality, active listening cues, and agenda adherence
- **Sentiment and engagement analysis**: Detecting participation patterns, dominance, and disengagement signals
- **Smart participant matching**: Recommending relevant participants for future meetings based on past attendance and expertise
- **Automated note-taking and transcription**: Capturing key points, decisions, and follow-ups without manual intervention
- **Multi-language real-time translation**: Neural machine translation for live captions in participant's preferred language
- **Speaker identification and context tagging**: Automatically identifying speakers and tagging topics mentioned
- **Searchable meeting knowledge base**: Indexing recordings and transcripts with semantic search across all past meetings

---

## Legal & IP Summary

Zoom, Microsoft Teams, Google Meet, Cisco Webex, Daily.co, Whereby operate as proprietary SaaS offerings. Agora is proprietary SDK with public markets listing. WebRTC, SIP, SRTP, DTLS are all open standards with no known IP encumbrances. WCAG 2.2 and Section 508 accessibility standards are public. AI-powered transcription, noise suppression, and real-time translation may involve patents; recommend legal review if implementing from scratch (not via vendor APIs). Recording and transcription consent requirements vary by jurisdiction (GDPR in EU, CCPA in California, state consent laws in US); ensure legal review before deploying transcription features. No copyright conflicts identified.

---

## Recommended Feature Scope

**Must-have (MVP)**
- HD video and audio with WebRTC transport layer
- Screen sharing with annotation tools
- Basic meeting scheduling with calendar integration
- Recording (cloud or local) with search capability
- Live captions (at least English)
- Virtual backgrounds
- Breakout rooms for small groups
- In-meeting chat and file sharing
- Participant roster with presence indicators
- Access controls (passwords, waiting rooms, host controls)
- Mobile-responsive web client
- Meeting analytics (duration, participation)

**Should-have (v1.1)**
- AI-powered meeting transcription with speaker diarization
- Automated action item extraction from transcripts
- AI meeting summaries and decision logs
- Real-time transcription across multiple languages
- AI noise suppression and audio enhancement
- Whiteboard for collaborative drawing
- Calendar integrations (Google, Outlook, iCal)
- Integration ecosystem (Slack, Jira, Salesforce, etc.)
- Participant attendance tracking
- Meeting series and recurring scheduling
- Custom backgrounds and virtual room themes
- Advanced security (end-to-end encryption option, SSO, 2FA)

**Nice-to-have (backlog)**
- Real-time meeting coaching and guidance
- Sentiment and engagement analysis
- Live interpretation with simultaneous translation channels
- On-premises deployment option
- AI Facilitator Agent capturing notes autonomously
- Asynchronous video feedback and messaging
- Meeting effectiveness analytics
- Searchable knowledge base of past meetings
- Integration with CRM and sales workflows
- Custom whitelabel deployment option
- Browser extension for quick meeting launching
- AI-powered speaker focus and eye contact correction
