# Standards & API Reference

> Project: Partner Relationship Management · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards

- **ISO/IEC 27001:2022 — Information Security Management Systems**
  URL: https://www.iso.org/standard/27001
  PRM platforms store sensitive partner contact data, deal pipeline information, and co-marketing funds. ISO 27001 certification is a market expectation for enterprise PRM buyers and provides the framework for access control, risk management, and incident response relevant to multi-tenant partner portals.

- **ISO/IEC 27017:2015 — Code of Practice for Information Security Controls for Cloud Services**
  URL: https://www.iso.org/standard/43757.html
  Extends ISO 27001 with cloud-specific controls. Directly relevant to SaaS PRM deployments where partner data is hosted in shared cloud infrastructure; addresses responsibilities between cloud service providers and customers.

- **ISO/IEC 27018:2019 — Protection of Personally Identifiable Information in Public Clouds**
  URL: https://www.iso.org/standard/76559.html
  Applies to PII held in cloud environments. Critical for PRM tools that aggregate partner contact records, prospect lists, and co-sell opportunity data across multiple jurisdictions.

- **ISO 9001:2015 — Quality Management Systems**
  URL: https://www.iso.org/standard/62085.html
  Applied in channel program management contexts to standardise partner onboarding workflows, escalation procedures, and MDF claim validation. Some enterprise PRM deployments reference ISO 9001 process models when documenting partner program governance.

### W3C & IETF Standards

- **RFC 6749 — The OAuth 2.0 Authorization Framework**
  URL: https://datatracker.ietf.org/doc/html/rfc6749
  The foundational standard for delegated API authorisation. All major PRM platforms (Salesforce, Crossbeam, PartnerStack, Kiflo) use OAuth 2.0 for third-party integrations and API access delegation.

- **RFC 9700 — Best Current Practice for OAuth 2.0 Security**
  URL: https://datatracker.ietf.org/doc/rfc9700/
  Published in 2025, this BCP consolidates OAuth 2.0 security guidance including PKCE requirements, token binding, and protection against common attack patterns. Any PRM API exposing OAuth endpoints should implement these recommendations.

- **RFC 8414 — OAuth 2.0 Authorization Server Metadata**
  URL: https://datatracker.ietf.org/doc/html/rfc8414
  Defines the discovery endpoint (`.well-known/oauth-authorization-server`) that allows API clients to dynamically discover OAuth endpoints. Useful for PRM integrations that must connect to multiple partner organisation identity providers.

- **RFC 9728 — OAuth 2.0 Protected Resource Metadata (PRM)**
  URL: https://datatracker.ietf.org/doc/rfc9728/
  Extends OAuth discovery to resource servers, allowing API clients to discover which authorisation server protects a given resource endpoint. Note the PRM acronym collision with Partner Relationship Management — this is a coincidence in naming only.

- **OpenID Connect Core 1.0**
  URL: https://openid.net/specs/openid-connect-core-1_0.html
  Builds on OAuth 2.0 to provide identity assertions. Used by enterprise PRM portals to federate partner employee identities from corporate identity providers (Okta, Azure AD, Google Workspace) without requiring partners to maintain separate credentials.

- **SAML 2.0 — Security Assertion Markup Language**
  URL: https://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-tech-overview-2.0.html
  OASIS standard ratified March 2005; the de-facto SSO protocol for enterprise partner portals. Channeltivity, Salesforce PRM, Impartner, and virtually all enterprise PRMs support SAML 2.0 for federated single sign-on. Covers assertions, bindings, profiles, and metadata exchange.

- **RFC 8030 — Generic Event Delivery Using HTTP Push**
  URL: https://tools.ietf.org/html/rfc8030
  Defines HTTP/2 push for real-time event delivery. Relevant background standard for PRM webhook and real-time notification systems (partner deal status changes, MDF approval events, co-sell opportunity updates).

- **Standard Webhooks Specification**
  URL: https://github.com/standard-webhooks/standard-webhooks/blob/main/spec/standard-webhooks.md
  An emerging community specification (2024) for standardising webhook payload format, signatures, and retry semantics across SaaS platforms. Relevant for PRM-to-CRM event push integrations.

- **RFC 6350 — vCard Format Specification**
  URL: https://www.rfc-editor.org/rfc/rfc6350.html
  Defines the vCard 4.0 data format for representing and exchanging contact information. Applicable to PRM partner contact import/export, directory integrations, and CRM sync where partner profiles must be serialised portably.

- **RFC 8288 — Web Linking**
  URL: https://datatracker.ietf.org/doc/html/rfc8288
  Specifies the `Link` header and relation types for hypermedia APIs. Relevant to pagination in REST PRM APIs (next/prev/first/last link relations).

- **RFC 7807 — Problem Details for HTTP APIs**
  URL: https://datatracker.ietf.org/doc/html/rfc7807
  Standardises error response bodies for HTTP APIs using `application/problem+json`. Recommended for PRM API error handling to enable consistent client-side error parsing across integrations.

- **RFC 8446 — TLS 1.3**
  URL: https://datatracker.ietf.org/doc/html/rfc8446
  Mandatory transport security for all PRM API traffic. TLS 1.3 eliminates legacy cipher suites and reduces handshake latency; required for SOC 2 and ISO 27001 compliance in SaaS deployments.

### Data Model & API Specifications

- **OpenAPI Specification 3.1.1**
  URL: https://spec.openapis.org/oas/v3.1.1.html
  The industry-standard format for describing REST APIs. PRM API documentation should be published as OpenAPI 3.1 specs to enable SDK auto-generation, integration testing, and third-party connector development. OpenAPI 3.1 is a full superset of JSON Schema Draft 2020-12.

- **JSON Schema Draft 2020-12**
  URL: https://json-schema.org/specification
  The data validation standard underpinning OpenAPI 3.1 Schema Objects. Used to define and validate PRM data models (partner profiles, deal objects, MDF claims, commission records).

- **OData v4 (ISO/IEC 20802)**
  URL: https://docs.oasis-open.org/odata/odata/v4.01/odata-v4.01-part1-protocol.html
  Microsoft Dynamics 365 Customer Engagement Web API implements OData v4 for CRUD operations on partner and opportunity objects. PRM tools integrating with Dynamics must understand OData query syntax ($filter, $expand, $select).

- **GraphQL June 2018 Specification**
  URL: https://spec.graphql.org/June2018/
  Some ecosystem platforms expose GraphQL endpoints for flexible partner data queries. Less common than REST in PRM, but relevant for flexible reporting and analytics integrations.

- **JSON:API 1.1**
  URL: https://jsonapi.org/format/
  A specification for building APIs with JSON that defines resource document structure, relationships, pagination, filtering, and sparse fieldsets. Some PRM platforms (including emerging open-source PRMs) adopt JSON:API for consistent API design.

### Security & Authentication Standards

- **OAuth 2.0 with PKCE (RFC 7636)**
  URL: https://datatracker.ietf.org/doc/html/rfc7636
  Proof Key for Code Exchange — mandatory for public clients (browser-based partner portals, mobile apps) using OAuth 2.0. RFC 9700 requires PKCE support for all new OAuth deployments.

- **SOC 2 Type II (AICPA)**
  URL: https://www.aicpa-cima.com/topic/audit-assurance/audit-and-assurance-excellence/system-and-organization-controls-soc-suite-of-services
  The dominant security attestation standard for US-market SaaS. Enterprise PRM buyers (particularly large SaaS companies) require vendors to hold SOC 2 Type II. Covers availability, confidentiality, security, processing integrity, and privacy trust service criteria. Unifyr, ScalePad, and most established PRM vendors maintain SOC 2 Type II.

- **GDPR — General Data Protection Regulation (EU 2016/679)**
  URL: https://gdpr-info.eu/
  Applies to any PRM storing or processing partner contact data for EU-based organisations. Requires data subject rights (access, erasure, portability), lawful basis for processing, data processor agreements with sub-processors, and breach notification within 72 hours.

- **CCPA — California Consumer Privacy Act**
  URL: https://oag.ca.gov/privacy/ccpa
  US equivalent for California residents; governs partner contact data, prospect lists, and analytics data. PRM tools serving US enterprise customers must implement opt-out mechanisms and data deletion workflows.

- **OWASP API Security Top 10 (2023)**
  URL: https://owasp.org/API-Security/editions/2023/en/0x00-header/
  The reference checklist for securing REST APIs. Covers broken object-level authorisation, authentication failures, excessive data exposure, and injection — all directly applicable to PRM API design, especially for multi-tenant environments where partner orgs must be strictly isolated.

### MCP Server Specifications

- **Model Context Protocol (MCP) — 2025-11-25 Specification**
  URL: https://modelcontextprotocol.io/specification/2025-11-25
  An open protocol published by Anthropic (November 2024) and broadly adopted by the industry in 2025. MCP defines a JSON-RPC 2.0-based protocol for connecting LLM agents to external tools and data sources via Resources, Prompts, and Tools primitives. An AI-native PRM should expose an MCP server so AI agents (sales assistants, co-sell bots) can query partner overlap data, retrieve deal status, submit lead registrations, and trigger MDF workflows without custom integration code.
  GitHub schema: https://github.com/modelcontextprotocol/specification/blob/main/schema/2025-11-25/schema.ts

- **Salesforce MCP Support**
  URL: https://developer.salesforce.com/blogs/2025/06/introducing-mcp-support-across-salesforce
  Salesforce introduced native MCP client support in Agentforce (July 2025), enabling Salesforce-based PRM portals to connect to any MCP-compliant server. PRM tools that publish MCP servers gain automatic compatibility with Salesforce Agentforce agents.

---

## Similar Products — Developer Documentation & APIs

### Salesforce PRM (Experience Cloud + Sales Cloud)

- **Description:** Native Salesforce partner relationship management built on Experience Cloud (partner portals) and Sales Cloud (opportunity and lead objects). The de-facto enterprise PRM for Salesforce shops with ~150,000+ active partner users globally.
- **API Documentation:** https://developer.salesforce.com/docs/apis
- **Experience Cloud Developer Guide:** https://developer.salesforce.com/docs/atlas.en-us.communities_dev.meta/communities_dev/communities_dev_intro_before.htm
- **Connect REST API (includes Experience Cloud):** https://developer.salesforce.com/docs/atlas.en-us.chatterapi.meta/chatterapi/features_communities.htm
- **Partner Object Reference:** https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_partner.htm
- **SDKs/Libraries:** Salesforce DX CLI, Apex, JavaScript (LWC), Python/Java/Node via REST; official SDKs at https://developer.salesforce.com/tools/salesforcecli
- **Developer Guide (Trailhead):** https://trailhead.salesforce.com/content/learn/modules/partner-relationship-management/get-to-know-sales-cloud-prm
- **Standards:** REST/JSON, SOQL, SOSL, Metadata API, Bulk API 2.0, OData (via Salesforce Connect)
- **Authentication:** OAuth 2.0 (Connected App), SAML 2.0 SSO for Experience Cloud, Named Credentials for server-to-server

### PartnerStack

- **Description:** B2B SaaS partner program platform specialising in affiliate, referral, and reseller partner management. Acquired by AppDirect (April 2026). Network of 138,000+ B2B partners.
- **API Documentation:** https://docs.partnerstack.com/docs/partnerstack-api
- **Vendor API Reference:** https://docs.partnerstack.com/reference
- **Partner API Reference:** https://docs.partnerstack.com/reference/partner-api-authentication
- **SDKs/Libraries:** Language-specific client libraries available (see docs); JSON REST responses
- **Developer Guide:** https://docs.partnerstack.com/
- **Standards:** REST/JSON; resource-oriented URLs; separate test and production API keys
- **Authentication:** Basic Auth (public key as username, private key as password); Bearer token for Partner API

### Crossbeam (merged with Reveal)

- **Description:** Ecosystem-Led Growth (ELG) platform for partner overlap mapping and co-selling intelligence. Merged with Reveal in 2024 to form the leading partner account mapping platform.
- **API Documentation:** https://developers.crossbeam.com/
- **REST API Help:** https://help.crossbeam.com/en/articles/4677142-rest-api
- **Signals API (real-time partner events):** https://help.crossbeam.com/en/articles/12732223-getting-started-with-signals-how-to-access-real-time-partner-data-via-api-and-webhooks
- **SDKs/Libraries:** No official SDK listed; standard REST/JSON with OAuth tokens
- **Developer Guide:** https://help.crossbeam.com/en/collections/2111215-integrations-and-api
- **Standards:** REST/JSON; enterprise REST API
- **Authentication:** OAuth 2.0; Authorization URL: `https://auth.crossbeam.com/authorize?audience=https://api.getcrossbeam.com`; Token URL: `https://auth.crossbeam.com/oauth/token`

### Impartner PRM

- **Description:** Enterprise-grade PRM platform designed for large global channel programs. Native iPaaS integration layer for CRM sync. AppExchange-approved Salesforce integration maintained for over a decade.
- **API Documentation:** https://apitracker.io/a/impartner (community-maintained tracker)
- **Base URL:** `https://prod.impartner.live`
- **Integrations Overview:** https://impartner.com/integrations/
- **SDKs/Libraries:** iPaaS integration layer for workflow automation; Salesforce managed package
- **Developer Guide:** Requires Impartner partner account — available via portal; Tray.io connector docs at https://docs.tray.ai/connectors/service/impartner
- **Standards:** REST/JSON; Salesforce Managed Package (Apex/SOQL)
- **Authentication:** OAuth tokens; Salesforce Connected App for CRM sync

### Kiflo PRM

- **Description:** SMB-focused PRM with transparent pricing (~$499/month) and faster deployment than enterprise alternatives. REST API and JavaScript SDK for custom integrations.
- **API Documentation:** https://docs.kiflo.com/article/28-api
- **API Reference:** https://docs.kiflo.com/category/18-api
- **JS SDK Reference:** https://docs.kiflo.com/category/26-js-sdk
- **Developer Hub:** https://docs.kiflo.com/collection/14-developers
- **SDKs/Libraries:** JavaScript SDK (browser/Node); REST API
- **Developer Guide:** https://docs.kiflo.com/
- **Standards:** REST/JSON; resource-oriented URLs
- **Authentication:** API Access Tokens (Bearer)

### Channeltivity

- **Description:** PRM focused on tech companies; deal registration, MDF, channel lead management, and CRM integration via open API and plug-and-play CRM connectors.
- **API Documentation:** https://www.channeltivity.com/prm-integrations/
- **SDKs/Libraries:** Open API (custom integration); Zapier connector
- **Developer Guide:** https://www.channeltivity.com/prm-integrations/
- **Standards:** REST/JSON (open API); SAML 2.0 SSO
- **Authentication:** SAML 2.0 for SSO; API token for REST access
- **CRM Integrations:** HubSpot, Salesforce, Zoho, Dynamics 365 (point-and-click wizard setup)

### HubSpot (Partner Clients API)

- **Description:** HubSpot CRM includes a dedicated Partner Clients API object for HubSpot Solutions Partners, enabling partner-client relationship tracking alongside standard CRM objects.
- **API Documentation:** https://developers.hubspot.com/docs
- **Partner Clients API Reference:** https://developers.hubspot.com/docs/reference/api/crm/objects/partner-clients
- **Developer Portal:** https://developers.hubspot.com/
- **SDKs/Libraries:** Node.js, TypeScript, PHP, .NET, Python, Java, Go — all official; https://developers.hubspot.com/docs/guides/api
- **Developer Guide:** https://developers.hubspot.com/docs/guides/api
- **Standards:** REST/JSON; OpenAPI-compatible; resource-oriented
- **Authentication:** OAuth 2.0 (private apps use API keys); HubSpot App OAuth for public integrations

### Microsoft Dynamics 365 Customer Engagement

- **Description:** Enterprise CRM with partner management objects (Partner, Opportunity, Lead). Implements OData v4 Web API. Used extensively in manufacturing, distribution, and technology channel programs.
- **API Documentation:** https://learn.microsoft.com/en-us/dynamics365/customerengagement/on-premises/developer/use-microsoft-dynamics-365-web-api
- **Web API Samples:** https://learn.microsoft.com/en-us/dynamics365/customerengagement/on-premises/developer/webapi/samples
- **Business Central API v2.0:** https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/api-reference/v2.0/
- **SDKs/Libraries:** NuGet SDK packages; Power Apps / Dataverse SDK; community SDKs for Python, Java
- **Developer Guide:** https://learn.microsoft.com/en-us/dynamics365/customerengagement/on-premises/developer/programming-reference
- **Standards:** OData v4 (ISO/IEC 20802); REST/JSON; SOAP (legacy)
- **Authentication:** OAuth 2.0 via Azure Active Directory; OpenID Connect

### Zapier (Workflow Automation / Integration Hub)

- **Description:** No-code integration platform with 8,000+ app connectors. Central to PRM integration ecosystems — virtually every PRM (Channeltivity, Kiflo, PartnerStack, Impartner) provides a Zapier connector for workflow automation without custom code.
- **Workflow API (Partner Solutions):** https://zapier.com/developer-platform/workflow-api
- **Webhooks Documentation:** https://zapier.com/apps/webhook/integrations
- **Partner Directory:** https://zapier.com/partnerdirectory
- **SDKs/Libraries:** Zapier CLI for building custom app integrations
- **Developer Guide:** https://platform.zapier.com/
- **Standards:** Webhooks (HTTP POST); REST/JSON; trigger/action/search model
- **Authentication:** API key or OAuth 2.0 per connected app

---

## Notes

### Emerging Standards and Gaps

- **No domain-specific PRM data standard exists.** Unlike CRM (which has some OData/Salesforce object model convergence) or ERP (EDI/ANSI X12), the PRM space has no ISO or IETF standard for partner data exchange formats (deal registrations, MDF claims, partner tiers). Each vendor uses proprietary schemas, making cross-platform data portability difficult.

- **MCP as the AI integration layer.** The Model Context Protocol (MCP) is emerging as the de-facto way to expose PRM capabilities to AI agents. An AI-native PRM should publish an MCP server from day one, enabling sales AI assistants, co-sell bots, and partner success agents to interact with the PRM without bespoke integrations.

- **Co-sell APIs from cloud hyperscalers are a separate category.** AWS Partner Network (APN via ACE), Microsoft Partner Center, and Google Cloud Partner Advantage all expose co-sell deal registration APIs that PRM tools must integrate with. These are vendor-specific and not covered by any cross-platform standard.

- **Webhook standardisation is still evolving.** The Standard Webhooks specification (community initiative, 2024) is gaining traction but is not yet an IETF RFC. PRM platforms should track this as it matures toward a de-facto standard for event delivery.

- **Data residency and sovereignty.** EU partner data must not leave EU data centres under GDPR. Emerging national data sovereignty requirements (India DPDP Act 2023, China PIPL) add further constraints for global PRM deployments. No unified cross-jurisdictional standard exists; platforms must implement per-region data partitioning.
