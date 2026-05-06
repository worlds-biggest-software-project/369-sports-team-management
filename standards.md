# Standards & API Reference

> Project: Sports Team Management · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards
- **ISO 20614:2017** — Data exchange protocol for interoperability and preservation. https://www.iso.org/standard/68562.html — Useful framing for long-lived player/season records that may need export to NGB archives.
- **ISO 8601** — Date and time representation. https://www.iso.org/iso-8601-date-and-time-format.html — All schedules, RSVPs, and stat events MUST use ISO 8601 timestamps with timezone.
- **ISO/IEC 27001:2022** — Information security management. https://www.iso.org/standard/27001 — Baseline ISMS reference for handling minor and medical data.
- **ISO/IEC 27701:2019** — Privacy information management extension to 27001. https://www.iso.org/standard/71670.html — Aligns with GDPR for player PII.
- **ISO 20488:2018** — Online consumer reviews (relevant for coach/club rating features). https://www.iso.org/standard/68193.html

### W3C & IETF Standards
- **RFC 5545 — iCalendar** https://datatracker.ietf.org/doc/html/rfc5545 — Schedule export to Google/Apple/Outlook.
- **RFC 7986** — New properties for iCalendar. https://datatracker.ietf.org/doc/html/rfc7986
- **RFC 6321 — xCal** XML format for iCalendar. https://datatracker.ietf.org/doc/html/rfc6321
- **RFC 7265 — jCal** JSON format for iCalendar. https://datatracker.ietf.org/doc/html/rfc7265
- **RFC 4791 — CalDAV** https://datatracker.ietf.org/doc/html/rfc4791 — Two-way calendar sync.
- **RFC 6749 — OAuth 2.0** https://datatracker.ietf.org/doc/html/rfc6749 — Auth framework for third-party integrations.
- **RFC 7519 — JWT** https://datatracker.ietf.org/doc/html/rfc7519 — Session and API tokens.
- **RFC 7231 / 9110 — HTTP semantics** https://datatracker.ietf.org/doc/html/rfc9110
- **RFC 8288 — Web Linking** https://datatracker.ietf.org/doc/html/rfc8288 — Pagination headers for list endpoints.
- **RFC 6902 — JSON Patch** https://datatracker.ietf.org/doc/html/rfc6902 — Partial roster/lineup updates.
- **RFC 7807 / 9457 — Problem Details for HTTP APIs** https://datatracker.ietf.org/doc/html/rfc9457 — Standard error envelope.
- **W3C WCAG 2.2** https://www.w3.org/TR/WCAG22/ — Accessibility for parent/player apps.
- **W3C WebRTC** https://www.w3.org/TR/webrtc/ — For optional live streaming features.
- **W3C Service Workers** https://www.w3.org/TR/service-workers/ — Underpins offline match-day mode.

### Data Model & API Specifications
- **OpenAPI 3.1.0** https://spec.openapis.org/oas/v3.1.0 — Recommended for the project's REST API definition.
- **JSON Schema 2020-12** https://json-schema.org/specification — Validation for sport-specific stat templates.
- **GraphQL (October 2021 spec)** https://spec.graphql.org/October2021/ — Optional read API for client dashboards.
- **AsyncAPI 3.0** https://www.asyncapi.com/docs/reference/specification/v3.0.0 — For live-scoring event streams.
- **Server-Sent Events (HTML Living Standard)** https://html.spec.whatwg.org/multipage/server-sent-events.html — Push live score updates.
- **CloudEvents 1.0 (CNCF)** https://github.com/cloudevents/spec — Event envelope for webhooks (registration, RSVP, score events).
- **SportsML-G2 (IPTC)** https://iptc.org/standards/sportsml-g2/ — XML standard for sports data interchange (rosters, schedules, results).
- **SDMX (statistics data exchange)** https://sdmx.org/ — Useful for season-level statistical exports.
- **OPF (Open Performance Framework)** community efforts around athlete monitoring data — emerging; reference https://www.openperformanceframework.org/ where available.

### Security & Authentication Standards
- **OAuth 2.1 (draft)** https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1 — Successor consolidating OAuth 2.0 best practice.
- **OpenID Connect Core 1.0** https://openid.net/specs/openid-connect-core-1_0.html — SSO for clubs using Google/Apple/Microsoft.
- **PKCE — RFC 7636** https://datatracker.ietf.org/doc/html/rfc7636 — Mobile app auth.
- **WebAuthn Level 3** https://www.w3.org/TR/webauthn-3/ — Passkey auth for coaches/admins.
- **OWASP ASVS 4.0.3** https://owasp.org/www-project-application-security-verification-standard/ — Application security verification.
- **OWASP API Security Top 10 (2023)** https://owasp.org/API-Security/ — API hardening reference.
- **NIST SP 800-63B** https://pages.nist.gov/800-63-3/sp800-63b.html — Authentication/lifecycle guidance.
- **PCI DSS v4.0** https://www.pcisecuritystandards.org/ — If processing card payments directly (delegate to Stripe to minimise scope).
- **GDPR (EU 2016/679)** https://gdpr.eu/ — Mandatory for EU player data.
- **COPPA (15 USC §6501)** https://www.ftc.gov/legal-library/browse/rules/childrens-online-privacy-protection-rule-coppa — US under-13 player data.
- **HIPAA (US)** https://www.hhs.gov/hipaa/ — Triggered if integrating with athletic-trainer EHR data.
- **SafeSport Code** https://uscenterforsafesport.org/ — Required workflow for US youth sport organisations.

### MCP Server Specifications
- **Model Context Protocol** https://modelcontextprotocol.io/ — Reference spec for exposing team data, schedules, and stats to LLM coaching assistants.
- **MCP TypeScript SDK** https://github.com/modelcontextprotocol/typescript-sdk
- **MCP Python SDK** https://github.com/modelcontextprotocol/python-sdk
- An MCP server for this project could expose tools like `get_roster`, `query_stats`, `draft_match_recap`, `find_schedule_conflicts`.

## Similar Products — Developer Documentation & APIs

### TeamSnap
- **Description:** Youth/amateur team management with rosters, schedules, messaging, and payments.
- **API Documentation:** https://www.teamsnap.com/documentation/apiv3 (Partner API, request access)
- **SDKs/Libraries:** Community Ruby/JS clients; no first-party SDKs published.
- **Developer Guide:** https://www.teamsnap.com/documentation/apiv3/getting-started
- **Standards:** REST/JSON, JSON:API style; collection+JSON variants
- **Authentication:** OAuth 2.0

### SportsEngine
- **Description:** League/club governance, registration, compliance.
- **API Documentation:** https://developer.sportsengine.com/ (partner-gated)
- **SDKs/Libraries:** None publicly published
- **Developer Guide:** https://developer.sportsengine.com/docs
- **Standards:** REST/JSON
- **Authentication:** OAuth 2.0 / API key for partners

### LeagueApps
- **Description:** Youth-league registration and operations.
- **API Documentation:** https://developer.leagueapps.com/
- **SDKs/Libraries:** REST clients in any language; Zapier connectors
- **Developer Guide:** https://developer.leagueapps.com/docs
- **Standards:** REST/JSON, webhooks
- **Authentication:** OAuth 2.0 client credentials

### Hudl
- **Description:** Video analysis and performance.
- **API Documentation:** https://www.hudl.com/api (partner programme)
- **SDKs/Libraries:** Partner-only
- **Developer Guide:** Partner portal
- **Standards:** REST/JSON; chunked video upload
- **Authentication:** OAuth 2.0

### Stripe Connect (payments)
- **Description:** Payment processor for registration fees and dues.
- **API Documentation:** https://docs.stripe.com/api
- **SDKs/Libraries:** Node, Python, Ruby, PHP, Java, Go, .NET, iOS, Android
- **Developer Guide:** https://docs.stripe.com/connect
- **Standards:** REST/JSON, idempotency keys, webhooks (CloudEvents-like)
- **Authentication:** API key + OAuth for connected accounts

### Opta / Stats Perform
- **Description:** Professional sports data feeds (soccer, basketball, etc.).
- **API Documentation:** https://www.statsperform.com/opta-api/ (commercial)
- **SDKs/Libraries:** Partner-issued
- **Developer Guide:** Commercial portal
- **Standards:** REST/JSON, XML, SportsML-G2
- **Authentication:** API key / OAuth

### Wyscout (Hudl)
- **Description:** Scouting platform with video and event data.
- **API Documentation:** https://apidocs.wyscout.com/
- **SDKs/Libraries:** None published
- **Developer Guide:** https://apidocs.wyscout.com/
- **Standards:** REST/JSON
- **Authentication:** Basic auth + token

### STATSports
- **Description:** GPS wearables for athlete load monitoring.
- **API Documentation:** https://www.statsports.com/api (partner)
- **SDKs/Libraries:** Partner SDK
- **Developer Guide:** Partner portal
- **Standards:** REST/JSON
- **Authentication:** OAuth 2.0

### Catapult Sports
- **Description:** Athlete monitoring wearables and analytics.
- **API Documentation:** https://www.catapult.com/connect (partner programme)
- **SDKs/Libraries:** Partner-issued
- **Standards:** REST/JSON, CSV exports
- **Authentication:** OAuth 2.0

### Google Calendar API
- **Description:** Calendar sync for schedules.
- **API Documentation:** https://developers.google.com/calendar/api
- **SDKs/Libraries:** Official SDKs in Java, Python, Node.js, Go, .NET, PHP, Ruby
- **Developer Guide:** https://developers.google.com/calendar/api/guides/overview
- **Standards:** REST/JSON, iCalendar import/export
- **Authentication:** OAuth 2.0

### Microsoft Graph (Calendar / Teams)
- **Description:** Outlook calendar and Teams messaging integration.
- **API Documentation:** https://learn.microsoft.com/en-us/graph/api/overview
- **SDKs/Libraries:** Official SDKs in C#, Java, JS/TS, Python, PHP, Go, Ruby
- **Developer Guide:** https://learn.microsoft.com/en-us/graph/use-the-api
- **Standards:** REST/JSON, OData v4
- **Authentication:** OAuth 2.0 / OpenID Connect

### Twilio (SMS / push)
- **Description:** SMS, voice, and push notifications for game-day reminders.
- **API Documentation:** https://www.twilio.com/docs/api
- **SDKs/Libraries:** Official SDKs in 8+ languages
- **Standards:** REST/JSON, webhooks
- **Authentication:** API key (Account SID + Auth Token)

## Notes

- **Open data interoperability is weak.** SportsML-G2 from IPTC is the closest open data-interchange standard but adoption outside news/media organisations is limited; most platforms use proprietary schemas. An open-source project has an opportunity to publish a JSON Schema-based exchange format and adapters for the major incumbents.
- **Athlete monitoring data lacks an open standard.** The Open Performance Framework is community-driven but not yet widely adopted; until it stabilises, vendor-specific GPS/wearable schemas must be normalised internally.
- **Privacy compliance for minors is the highest-risk area.** GDPR + COPPA + SafeSport overlap means parental consent workflows, audit logging, and right-to-erasure tooling must be first-class — not afterthoughts.
- **MCP exposure is an emerging differentiator.** Few incumbents expose AI-friendly access to team/stat data; offering an MCP server lets coaches and analysts query the team via Claude/ChatGPT-style assistants.
- **Live-scoring transport.** Server-Sent Events and AsyncAPI-described WebSocket channels are preferred over polling; CloudEvents envelopes keep webhook integrations consistent.
