# Standards & API Reference

> Project: Insurance Policy Management · Generated: 2026-05-03

## Industry Standards & Specifications

### ACORD Data Standards

**ACORD P&C Data Standards (AL3 & XML)**
- **URL:** https://www.acord.org/standards-architecture/acord-data-standards/Property_Casualty_Data_Standards
- The foundational interchange standard for property and casualty insurance transactions. AL3 is a batch communication format for policy and commission data; XML supports real-time request/response transactions. Mandatory for EDI between carriers, agents, and reinsurers in most US markets.

**ACORD Life & Annuity Data Standards**
- **URL:** https://www.acord.org/standards-architecture/acord-data-standards/Life_Annuity_Data_Standards
- XML and JSON standards covering claims, placing, and accounting & settlement for life insurance and annuity products. Relevant for any system supporting life, health, or annuity policy lines.

**ACORD Global Reinsurance & Large Commercial Data Standards**
- **URL:** https://www.acord.org/standards-architecture/acord-data-standards/Global_Reinsurance_Data_Standards
- XML and JSON standards for reinsurance and large commercial placements. Covers treaty and facultative reinsurance data interchange between cedants and reinsurers.

**ACORD Next-Generation Digital Standards (JSON/YAML — Microservices & RESTful APIs)**
- **URL:** https://www.acord.org/standards-architecture/acord-data-standards/next-generation-digital-standards
- A suite of JSON and YAML data-interchange formats designed for microservices architectures and RESTful APIs. Written for lightweight, technology-agnostic fine-grained transactions across mobile apps, IoT, distributed ledgers, and cloud-native services. Covers policy, claims, and reinsurance transactions. Available to ACORD member organisations for P&C, L&A, and Global Re/Large Commercial lines.

### ISO Standards

**ISO 9001 — Quality Management Systems**
- **URL:** https://www.iso.org/standard/62085.html
- Establishes a quality management framework that insurance carriers use to ensure consistent service delivery, customer satisfaction, and operational continuous improvement across policy administration processes.

**ISO 27001 — Information Security Management Systems**
- **URL:** https://www.iso.org/standard/27001
- The primary information security standard. Insurers processing large volumes of sensitive policyholder data (PII, health records, financial data) are increasingly required or expected by regulators to hold ISO 27001 certification.

**ISO 22301 — Business Continuity Management**
- **URL:** https://www.iso.org/standard/75106.html
- Ensures insurance platforms define and maintain resilience plans for unplanned disruptions, a regulatory expectation in most jurisdictions given the essential-service nature of insurance.

**ISO 20022 — Universal Financial Industry Message Scheme**
- **URL:** https://www.iso20022.org/iso-20022
- The global standard for electronic financial messaging, increasingly adopted for reinsurance settlement payments and premium fund transfers. Provides richer, structured data compared with older SWIFT message formats. Adopted by over 70 countries for payments harmonisation.

### W3C & IETF Standards

**OpenAPI Specification 3.2 (OAS 3.2)**
- **URL:** https://www.openapis.org/
- The vendor-neutral, Linux Foundation-governed specification for describing RESTful APIs (YAML or JSON). Current stable version is 3.2.0 (September 2025), adding streaming media type support (SSE, JSON Lines), native QUERY HTTP method, and OAuth 2.0 Device Authorization Flow. All major insurance platform APIs (Guidewire, BriteCore, Duck Creek) publish OpenAPI/Swagger specifications.

**RFC 7231 — HTTP/1.1 Semantics and Content**
- **URL:** https://datatracker.ietf.org/doc/html/rfc7231
- Defines the semantics of HTTP methods (GET, POST, PUT, DELETE, PATCH) used by all REST-based insurance policy APIs for CRUD operations on policy, claims, and billing resources.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- **URL:** https://datatracker.ietf.org/doc/html/rfc6749
- Standard for delegated authorisation used by all major insurance platforms to secure API access for agents, brokers, policyholders, and third-party integrations.

**OpenID Connect Core 1.0**
- **URL:** https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer built on OAuth 2.0 that adds authentication capabilities. Used for SSO across insurer portals and partner integrations, and for GDPR-aligned consent management with explicit data access scopes.

### Data Model & API Specifications

**JSON:API Specification**
- **URL:** https://jsonapi.org/
- A standardised specification for building APIs using JSON, covering resource relationships, pagination, sorting, filtering, and sparse fieldsets. Adopted by Insurity's Digital Platform APIs for consistent presentation across all product API endpoints.

**JSON Schema**
- **URL:** https://json-schema.org/
- Vocabulary for annotating and validating JSON documents. Used in OpenAPI specifications and ACORD Next-Gen Digital Standards to define and validate insurance data models (policy, coverage, endorsement, claim resources).

**HL7 FHIR R5 — Fast Healthcare Interoperability Resources**
- **URL:** https://www.hl7.org/fhir/
- Health data exchange standard with dedicated Claim and Coverage resources covering insurance eligibility verification, prior authorisation, and claims adjudication for health and dental insurance lines. CMS mandates FHIR-based payer-to-provider APIs with full compliance deadlines from January 2027.

### Security & Compliance Standards

**PCI DSS v4.0.1**
- **URL:** https://www.pcisecuritystandards.org/standards/
- Payment Card Industry Data Security Standard. Mandatory for any insurance platform that stores, processes, or transmits premium payment card data. PCI DSS v4.0.1 explicitly addresses API security requirements for payment processing APIs, requiring documented API inventories, encryption of PANs in transit, and segregation of duties.

**NAIC Model Laws & Regulatory Framework**
- **URL:** https://content.naic.org/model-laws
- The US National Association of Insurance Commissioners publishes model laws and regulations that states adopt to govern rate approval, policy filing, consumer protection, and financial reporting. Policy administration systems must support state-specific filing workflows and regulatory data submission formats (Annual Statement, quarterly financials).

**Solvency II / EIOPA Supervisory Reporting (EU)**
- **URL:** https://www.eiopa.europa.eu/browse/regulation-and-policy/solvency-ii_en
- The European risk-based prudential framework for insurers (Directive 2009/138/EC), revised in 2025 (Directive (EU) 2025/2), requiring transposition by January 2027. EIOPA's Data Point Model (DPM) Standard defines structured templates for supervisory reporting. Relevant for any insurance policy system operating in EU markets or serving EU-regulated entities.

**GDPR — General Data Protection Regulation**
- **URL:** https://gdpr.eu/
- Governs processing of EU policyholder personal data. Insurance platforms must implement data minimisation, consent management, right-to-erasure workflows, and data portability APIs. OpenID Connect scopes provide a technical mechanism for GDPR-aligned consent at the API layer.

**OWASP API Security Top 10**
- **URL:** https://owasp.org/www-project-api-security/
- Industry reference for API security vulnerabilities. Insurance policy APIs handle highly sensitive PII and financial data; adherence to the OWASP API Security Top 10 (broken object-level authorisation, authentication failures, excessive data exposure, etc.) is considered baseline practice.

---

## Similar Products — Developer Documentation & APIs

### Guidewire PolicyCenter Cloud APIs
- **Description:** Industry-leading P&C policy administration system serving mid-to-large carriers. Cloud APIs expose policy, quote, account, and document resources via REST.
- **API Documentation:** https://docs.guidewire.com/cloud/pc/202503/cloudapica/
- **SDKs/Libraries:** Jutro Digital Platform SDK (JavaScript/TypeScript) for consuming Cloud APIs in front-end applications; REST API Client library: https://www.guidewire.com/developers/apis/rest-api-client
- **Developer Guide:** https://docs.guidewire.com/cloud/pc/202407/cloudapica/pdf/CloudAPI-Developer.pdf
- **Standards:** OpenAPI 3.x (Swagger UI bundled), REST/JSON
- **Authentication:** OAuth 2.0

### BriteCore Platform APIs
- **Description:** Cloud-native P&C core platform for mid-market carriers. Exposes 1,000+ public-facing REST API endpoints covering policy, billing, claims, documents, and account management.
- **API Documentation:** https://help.britecore.com/hc/en-us/articles/8723669712915-API-overview
- **SDKs/Libraries:** Amazon API Gateway-backed; no dedicated SDK; standard HTTP client libraries.
- **Developer Guide:** https://api.britecore.com/specifications/BriteClaims%20API/v1/index.html (BriteClaims API v1)
- **Standards:** OpenAPI 3.0; REST/JSON; hard-versioned APIs for backward compatibility
- **Authentication:** OAuth 2.0 via Amazon API Gateway

### Duck Creek Anywhere APIs
- **Description:** SaaS P&C suite offering a rich set of RESTful APIs for policy, billing, and claims, plus a Content Exchange marketplace with 180+ pre-built integrations.
- **API Documentation:** https://www.duckcreek.com/product/anywhere-integration/
- **SDKs/Libraries:** Duck Creek Anywhere API Extension SDK: https://www.duckcreek.com/content-exchange/anywhere_api_extension_sdk/
- **Developer Guide:** https://policyapp.azurewebsites.net/Help
- **Standards:** REST/JSON (OpenAPI); ACORD data standard support
- **Authentication:** OAuth 2.0

### Majesco API Management Platform
- **Description:** CloudInsurer suite with a centralised API Management platform (APIM) exposing thousands of APIs for policy, billing, claims, and analytics across P&C and L&H lines.
- **API Documentation:** https://www.majesco.com/ecosystem-insurance-solutions/api-management/
- **SDKs/Libraries:** Not publicly documented; accessible via Majesco Product Portal for customers.
- **Developer Guide:** https://www.majesco.com/blog/an-eye-on-the-apis-a-cloud-platforms-role-in-api-documentation-administration-and-governance/
- **Standards:** REST and SOAP; ACORD XML/JSON compliance; OpenAPI specification format
- **Authentication:** API key and OAuth 2.0

### Salesforce Financial Services Cloud — Insurance APIs
- **Description:** CRM and policy management platform with dedicated InsurancePolicy, InsurancePolicyCoverage, and InsurancePolicyAsset objects. Used by carriers and agencies for policyholder relationship management.
- **API Documentation:** https://developer.salesforce.com/docs/atlas.en-us.financial_services_cloud_object_reference.meta/financial_services_cloud_object_reference/fsc_api_access_and_usage.htm
- **SDKs/Libraries:** Salesforce SDKs (JavaScript, Java, Python, .NET, Go): https://developer.salesforce.com/developer-centers/integration-apis
- **Developer Guide:** https://resources.docs.salesforce.com/latest/latest/en-us/sfdc/pdf/fsc_dev_guide.pdf
- **Standards:** REST/JSON; SOAP; Bulk API 2.0; Streaming API (Pub/Sub)
- **Authentication:** OAuth 2.0 (JWT Bearer, Web Server, Device, Username-Password flows)

### Insurity Digital Platform APIs
- **Description:** End-to-end SaaS platform for 500+ P&C carriers, MGAs, and brokers. REST APIs follow the JSON:API specification with pagination, sorting, filtering, and correlation IDs.
- **API Documentation:** https://go.insurity.com/rs/527-XVY-326/images/Insurity_Digital_Platform_Overview.pdf
- **SDKs/Libraries:** No public SDK; accessed via API Gateway.
- **Developer Guide:** Contact Insurity directly for customer portal access.
- **Standards:** REST/JSON; JSON:API specification; OpenAPI
- **Authentication:** OAuth 2.0 via API Gateway

### Boost Insurance API (Embedded Insurance Infrastructure)
- **Description:** Full-cycle insurance API platform enabling end-to-end digital policy lifecycle management (rating, quoting, issuing, endorsement, renewal, cancellation, claims) for P&C programs in all 50 US states via a white-label SaaS model.
- **API Documentation:** https://learn.boostinsurance.com/docs/boost-api-guide
- **SDKs/Libraries:** Available via developer portal: https://boostinsurance.com/developer/
- **Developer Guide:** https://boostinsurance.com/developer/
- **Standards:** REST/JSON; OpenAPI
- **Authentication:** API Key + OAuth 2.0

### Canopy Connect Insurance Data API
- **Description:** "Plaid for insurance" — enables applications to retrieve fully structured P&C insurance data (policy details, coverage, claims history, driver data) directly from 400+ carriers covering 95%+ of the US market in real time.
- **API Documentation:** https://docs.usecanopy.com/reference/getting-started
- **SDKs/Libraries:** JavaScript/TypeScript SDK; white-label widget SDK
- **Developer Guide:** https://www.usecanopy.com/api
- **Standards:** REST/JSON; OpenAPI; webhook-based real-time updates
- **Authentication:** OAuth 2.0 (carrier consent flow); API Key for platform access

### Coalition Active Insurance API (Cyber)
- **Description:** API for embedding cyber and management liability insurance into broker platforms and SaaS products. Supports rating, quoting, binding, and policy management with sub-2-second quote response times.
- **API Documentation:** https://www.coalitioninc.com/brokers/api
- **SDKs/Libraries:** No public SDK; direct REST integration.
- **Developer Guide:** https://www.coalitioninc.com/blog/coalition-apis-helping-power-the-future-of-insurance-distribution
- **Standards:** REST/JSON
- **Authentication:** API Key + OAuth 2.0

---

## Notes

- **ACORD standards fragmentation:** The insurance industry is mid-transition from legacy AL3/XML EDI messaging to ACORD Next-Gen Digital Standards (JSON/YAML + REST). Many carriers still require AL3 support for agency management system (AMS) integrations, making dual-format support an implementation requirement for the foreseeable future.

- **Emerging MCP / AI agent standards:** No insurance-specific Model Context Protocol (MCP) server specifications exist yet. However, Guidewire, BriteCore, and Boost all expose REST APIs that are straightforwardly wrappable as MCP tools for AI agent workflows (policy lookup, claims intake, endorsement processing).

- **Regulatory data model divergence:** US (NAIC) and EU (Solvency II / EIOPA DPM) regulatory reporting requirements use different data models and submission formats. An open-source platform targeting international carriers would need pluggable regulatory adapters for each jurisdiction.

- **HL7 FHIR mandates upcoming:** CMS's Interoperability and Prior Authorization Final Rule requires payers (health insurers) to implement FHIR R4/R5-based APIs for payer-to-provider data sharing by January 2027, creating a significant integration requirement for health insurance policy systems in the US.

- **ISO 20022 payments adoption:** Premium settlement, reinsurance payments, and refund processing are migrating to ISO 20022 messaging as central banks and payment networks (SWIFT, FedNow, CHAPS) roll out the standard. Insurance platforms should plan for ISO 20022-compatible payment instruction generation.
