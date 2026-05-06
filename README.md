# Video Conferencing Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, WebRTC-based video conferencing platform with built-in recording, transcription, and meeting summaries.

A browser-first video conferencing platform built on open WebRTC standards, designed for teams who want first-class AI meeting intelligence without being locked into a single productivity suite. Targets developers embedding video into their products, as well as organisations seeking an open alternative to Zoom, Teams, Meet, and Webex.

---

## Why Video Conferencing Platform?

- The market is dominated by proprietary SaaS: Zoom holds 55.9% share, Teams 32.3%, Webex 7.6%, and Meet 5.5%, leaving no credible open-source, AI-native alternative.
- Enterprise pricing runs USD 14–20+/user/month, and the most useful AI features (summaries, transcription, translation) are gated behind higher-tier plans.
- Incumbents tie their best features to their own ecosystems (Teams to Microsoft 365, Meet to Google Workspace), making cross-platform adoption painful.
- Developer-focused infrastructure (Daily.co, Agora, Whereby) is powerful but requires significant build effort and offers limited out-of-the-box AI or end-user features.
- AI is shifting from post-call processing to real-time in-call intelligence — an area where every incumbent is still iterating, leaving room for an open challenger.

---

## Key Features

### Real-Time Meetings

- HD video and audio over WebRTC with automatic quality adaptation
- Screen sharing with annotation tools
- Virtual backgrounds and blur options
- Breakout rooms for small-group discussions
- In-meeting chat and file sharing
- Participant roster with presence indicators

### Recording, Transcription, and Captions

- Cloud and local recording with searchable transcripts
- Live captions in English at MVP, with multi-language support on the roadmap
- AI-powered transcription with speaker diarisation
- Searchable archive of past meetings within a workspace

### AI Meeting Intelligence

- Auto-generated meeting summaries and decision logs distributed at meeting end
- Automated action-item extraction with assignment to named participants
- AI noise suppression and audio enhancement
- In-call AI coach prompting on active listening, question quality, and agenda adherence

### Scheduling and Access Control

- Calendar integration with Google, Outlook, and iCal
- Meeting series and recurring scheduling
- Passwords, waiting rooms, and host controls
- SSO and 2FA on the roadmap

### Developer and Integration Surface

- WebRTC-based stack with REST APIs and webhooks
- Mobile-responsive web client; mobile apps planned
- Integrations with Slack, Jira, Salesforce, and project management tools for action-item sync

---

## AI-Native Advantage

The platform treats AI as a first-class meeting participant rather than a paid add-on. Real-time transcription with reliable speaker diarisation, automated action-item extraction with sync to project management tools, and an in-call coach for meeting quality are all baseline capabilities. Combined with a searchable knowledge base across every recorded meeting, the platform turns conversations into queryable organisational memory.

---

## Tech Stack & Deployment

Built on open standards: WebRTC for media, SIP/SDP for signalling, SRTP and DTLS for encryption. Browser-first delivery removes the need for a heavy desktop client. Self-hosted and cloud deployment modes are both expected, with on-premises options on the roadmap for compliance-sensitive workloads. Accessibility aligns with WCAG 2.2 and Section 508.

---

## Market Context

The global video conferencing market is projected to reach USD 12 billion in 2026 at an ~11.8% CAGR, with the broader WebRTC market reaching USD 10.89 billion. Incumbent enterprise pricing runs USD 14–20+/user/month, and infrastructure providers charge USD 0.0001–0.001 per participant-minute. Primary buyers are IT administrators standardising enterprise video, developers embedding video into SaaS, and HIPAA-bound healthcare and education providers.

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
