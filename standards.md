# Standards & API Reference

> Project: Corporate Governance Platform · Generated: 2026-05-07

## Industry Standards & Specifications

### ISO Standards

**ISO 37000:2021 — Governance of Organizations: Guidance**
- URL: https://www.iso.org/standard/65036.html
- Provides principles and key aspects of practice to guide governing bodies on how to meet their responsibilities. Directly applicable as the conceptual framework underlying a corporate governance platform's workflow design. Distills governance into 11 core principles applicable to all organisations regardless of size, location, or structure.

**ISO/IEC 38500:2015 — Corporate Governance of Information Technology**
- URL: https://www.iso.org/standard/62816.html
- Provides guiding principles for governing bodies on the effective, efficient, and acceptable use of IT. Directly relevant for any platform that boards use to govern their IT estate; also defines a framework for the governance platform itself to meet as a tool serving boards.

**ISO 37301:2021 — Compliance Management Systems: Requirements**
- URL: https://www.iso.org/standard/75080.html
- Specifies requirements for establishing, developing, implementing, evaluating, maintaining, and improving a compliance management system. A corporate governance platform should align workflow and audit log features with ISO 37301 to help organisations demonstrate compliance adherence.

**ISO 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- De facto certification requirement for board portal vendors. Governs how sensitive board materials are stored, processed, and transmitted. Govenda, Board Intelligence, and Sherpany explicitly cite ISO 27001 certification; Diligent's SOC 2 covers similar ground under US frameworks.

**ISO 31000:2018 — Risk Management: Guidelines**
- URL: https://www.iso.org/standard/65694.html
- International framework for risk identification, assessment, and treatment. Relevant to board oversight of risk — a governance platform's risk reporting and committee reporting features should align with ISO 31000 terminology and process flows.

**ISO 17442-1:2020 — Legal Entity Identifier (LEI)**
- URL: https://www.gleif.org/en/organizational-identity/introducing-the-legal-entity-identifier-lei/iso-17442-the-lei-code-structure
- Defines the 20-character alpha-numeric LEI code structure for identifying legal entities. Directly relevant to any entity management module — LEI should be a first-class identifier for all corporate entities tracked in a governance platform, enabling interoperability with financial regulators and counterparties.

**ISO 26000:2010 — Guidance on Social Responsibility**
- URL: https://www.iso.org/standard/42546.html
- Guidance on ESG and social responsibility. Relevant as boards are increasingly accountable for ESG reporting; a governance platform's ESG module should map to ISO 26000 principles.

---

### W3C & IETF Standards

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The foundation for delegated access in board portal integrations. Required for any API that allows third-party tools (document management, HR systems, identity providers) to securely access governance platform data on behalf of users.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- Standard for encoding claims in JSON for secure transmission. Used in OAuth 2.0 / OpenID Connect flows and API authentication. Relevant to all API authentication implementations in a governance platform.

**RFC 7617 — SAML 2.0 (via OASIS)**
- SAML 2.0 is the de facto SSO standard for enterprise software. All board portal vendors support SAML 2.0 for integration with corporate identity providers (Okta, Azure AD, Ping Identity). Any open-source governance platform must support SAML 2.0 to be viable in enterprise contexts.
- SAML 2.0 specification: https://docs.oasis-open.org/security/saml/v2.0/

**OpenID Connect 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Authentication layer on top of OAuth 2.0 using JSON/JWT. Increasingly preferred over SAML for modern cloud-native integrations. A governance platform should support both SAML 2.0 (for legacy enterprise IdPs) and OIDC (for modern IdPs).

**SCIM 2.0 — System for Cross-domain Identity Management (RFC 7644)**
- URL: https://datatracker.ietf.org/doc/html/rfc7644
- Standard protocol for automating user provisioning and deprovisioning between identity providers and SaaS applications. Required for enterprise adoption — automated provisioning/deprovisioning of board members as they join or leave is a compliance-critical workflow.

**W3C WebAuthn (FIDO2)**
- URL: https://www.w3.org/TR/webauthn/
- Standard for strong, phishing-resistant authentication using public key cryptography. Relevant for MFA and passwordless authentication options for directors accessing sensitive board materials.

---

### Data Model & API Specifications

**OpenAPI 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0
- Industry standard for documenting REST APIs. Any governance platform offering a developer API should publish an OpenAPI 3.1 specification to enable integration, SDK generation, and third-party automation. Diligent's HighBond API and Equity API are OpenAPI-compliant.

**JSON API v1.0**
- URL: https://jsonapi.org/
- Specification for building APIs in JSON. Diligent's Third-Party Manager API is built on the JSON API v1.0 specification. A natural choice for governance platform APIs handling structured board data.

**XBRL (eXtensible Business Reporting Language)**
- URL: https://www.xbrl.org/
- XML-based format for financial and regulatory reporting, mandated by the SEC since 2009 for EDGAR filings. A governance platform supporting listed companies should be able to consume XBRL data from regulatory filings and connect it to board oversight workflows.

**iCalendar (RFC 5545)**
- URL: https://datatracker.ietf.org/doc/html/rfc5545
- Standard for calendar data exchange. Required for meeting scheduling integrations with Microsoft Outlook, Google Calendar, and Apple Calendar — all cited as integration points by competing products.

---

### Security & Authentication Standards

**SOC 2 Type II (AICPA)**
- URL: https://www.aicpa.org/resources/landing/system-and-organization-controls-soc-suite-of-services
- The US equivalent of ISO 27001 for SaaS security compliance. Required by enterprise procurement processes. All major board portal vendors (Diligent, OnBoard, Govenda) hold SOC 2 Type II certification. A governance platform must achieve SOC 2 Type II to be taken seriously in enterprise sales.

**NIST Cybersecurity Framework 2.0**
- URL: https://www.nist.gov/cyberframework
- Widely adopted US framework for managing cybersecurity risk. Boards are increasingly responsible for cybersecurity oversight; a governance platform's risk reporting features should map to NIST CSF categories. Also informs platform security architecture design.

**OWASP Top 10**
- URL: https://owasp.org/www-project-top-10/
- The standard reference for web application security vulnerabilities. A governance platform handling sensitive board materials must demonstrably address the OWASP Top 10 — injection, broken authentication, sensitive data exposure, etc.

**GDPR (EU General Data Protection Regulation)**
- URL: https://gdpr.eu/
- Applies to any governance platform processing personal data of EU residents. Director personal data (contact details, interests, shareholdings) is in scope. Platform must support data residency configuration, right-to-erasure workflows, and data processing agreements. Sherpany and Azeus Convene explicitly cite GDPR compliance.

**FINMA (Swiss Financial Market Supervisory Authority) Guidelines**
- URL: https://www.finma.ch/
- Swiss regulatory framework for financial institutions; relevant for governance platforms serving Swiss-regulated entities. Sherpany cites FINMA compliance as a differentiator. Similar considerations apply to FCA (UK), MAS (Singapore), and APRA (Australia) for other jurisdictions.

---

### MCP Server Specifications

The Model Context Protocol (MCP) is directly applicable to a corporate governance platform offering AI capabilities.

**Model Context Protocol (MCP)**
- URL: https://modelcontextprotocol.io/
- An open standard for enabling AI assistants to interact with external tools and data sources. A governance platform could expose an MCP server to allow AI models to:
  - Read board pack documents to generate summaries and minutes drafts
  - Query entity structures for due diligence or regulatory workflows
  - Create agenda items and action items from natural language
  - Access historical decisions and resolutions for context
- MCP is particularly relevant for AI minute-drafting and AI board assistant features — allowing a governance platform to integrate with any MCP-compatible AI assistant rather than locking in a single LLM vendor.

---

## Similar Products — Developer Documentation & APIs

### Diligent One Platform

- **Description:** Enterprise governance platform covering boards, GRC, entity management, insider trading compliance, and ESG. Used by 50% of Fortune 1000.
- **API Documentation:** https://developer.diligent.com/
- **Specific APIs:**
  - HighBond API: https://developer.diligent.com/api/highbond
  - Diligent Workflow API: https://developer.diligent.com/api/workflow
  - Diligent ESG API: https://developer.diligent.com/api/esg
  - Entities Reports API: https://developer.diligent.com/api/entities
- **Standards:** REST/JSON; JSON API v1.0; OAuth 2.0; OpenAPI spec available for some modules
- **Authentication:** OAuth 2.0 Authorization Framework; SAML 2.0 SSO

---

### Nasdaq Boardvantage

- **Description:** Board management portal for listed companies with AI summarisation, virtual meeting integration, and Nasdaq ecosystem connectivity.
- **API Documentation:** Not publicly documented; enterprise integrations via direct vendor engagement
- **Integrations:** Google Drive, Microsoft Outlook, Zoom, Microsoft Teams, DocuSign; SAML 2.0 SSO
- **Standards:** REST/JSON (assumed); SAML 2.0
- **Authentication:** SAML 2.0 / corporate SSO; MFA

---

### OnBoard

- **Description:** Mid-market board management software with AI minutes, skills tracking, and high usability ratings.
- **API Documentation:** Limited public documentation; Zapier integration available
- **SDKs/Libraries:** Zapier integration (https://zapier.com/apps/onboard-board-management-software/integrations)
- **Standards:** REST/JSON
- **Authentication:** SSO / SAML 2.0; MFA (Microsoft Azure hosted)

---

### Sherpany

- **Description:** European board and executive meeting management platform with strong GDPR / FINMA compliance and Microsoft Teams integration.
- **API Documentation:** API available; documentation provided on request
- **Integrations:** Microsoft Teams, Outlook, Google Calendar, Office suite, Auth0, IFTTT, Miro, Figma
- **Standards:** REST/JSON; GDPR-compliant data residency
- **Authentication:** Auth0 / NextGen SSO; SAML 2.0; end-to-end encryption

---

### SEC EDGAR API

- **Description:** US Securities and Exchange Commission's public REST API for company filings, XBRL financial data, and insider trading disclosures (Forms 3, 4, 5). Critical integration point for governance platforms serving listed US companies.
- **API Documentation:** https://www.sec.gov/search-filings/edgar-application-programming-interfaces
- **Developer Resources:** https://www.sec.gov/about/developer-resources
- **Data Access:** https://data.sec.gov/
- **Standards:** REST/JSON; XBRL-JSON; no authentication required for public data
- **Authentication:** None required (public data); EDGAR Next requires authentication for filing submission

---

### GLEIF Global LEI System

- **Description:** Global Legal Entity Identifier Foundation's open API for looking up and verifying legal entity identities worldwide. Essential for an entity management module that needs to uniquely identify subsidiary companies across jurisdictions.
- **API Documentation:** https://www.gleif.org/en/lei-data/access-and-use-lei-data
- **Standards:** REST/JSON; ISO 17442 LEI code structure
- **Authentication:** None required for public LEI lookup

---

### Govenda

- **Description:** Board management platform for SMBs and nonprofits with AI meeting assistant Gabii, SOC 2 Type II and ISO 27001 certification.
- **API Documentation:** API access available; limited public documentation
- **Standards:** REST/JSON; SOC 2 Type II; ISO 27001
- **Authentication:** SSO; built-in e-signature (no external dependency)

---

## Notes

- **API documentation gap**: Most board portal vendors provide minimal public API documentation; developer access typically requires an enterprise sales engagement. This represents an opportunity for an open-source platform to compete on developer experience.
- **Identity federation**: SAML 2.0 and OIDC are both required for enterprise SSO compatibility; SCIM 2.0 is increasingly required for automated provisioning. Any new platform should support all three from the outset.
- **Regulatory filing integration**: SEC EDGAR APIs (for US listed companies) and equivalent APIs in other jurisdictions (UK Companies House, ASIC, etc.) represent an underexplored integration opportunity. No current board portal natively bridges the gap between board decision and regulatory filing.
- **XBRL adoption**: XBRL is mandatory for SEC filings but not natively consumed by any board portal reviewed. Integrating XBRL data into board oversight workflows (e.g., automatically pulling audited financial data into board pack templates) would be a meaningful differentiator.
- **MCP server opportunity**: Publishing an MCP server for a governance platform would enable AI-native workflows across a broad ecosystem of AI assistants, without locking into a single AI vendor — a significant advantage over Diligent's proprietary AI Board Member.
