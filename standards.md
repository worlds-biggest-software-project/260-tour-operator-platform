# Standards & API Reference

> Project: Tour Operator Platform · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO/TC 228 — Tourism and Related Services**
- URL: https://www.iso.org/committee/375396.html
- The ISO Technical Committee 228 publishes international standards covering terminology, service quality, safety, and sustainability for tourism service providers. Relevant published standards include ISO 21101 (adventure tourism safety management), ISO 21103 (adventure tourism — information for participants), and ISO 13811 (hunting tourism). Any tour operator platform serving regulated activity or adventure operators should be aware of the certification requirements these standards impose on their customers.

**ISO 9001 — Quality Management Systems**
- URL: https://www.iso.org/standard/62085.html
- The general-purpose quality management systems standard. Tour operators seeking certification use ISO 9001 as the baseline; the platform's reporting and audit-trail functionality should produce documentation compatible with ISO 9001 evidence requirements.

**ISO 27001 — Information Security Management**
- URL: https://www.iso.org/standard/27001
- Governs information security management systems. Relevant to how the platform stores traveller PII, booking records, and supplier contracts. Operators selling to corporate buyers or large travel management companies will increasingly require suppliers to hold ISO 27001 accreditation.

**ISO 13009:2015 — Tourism and Related Services: Beach Operation**
- URL: https://www.iso.org/standard/52329.html
- Narrowly applicable to beach-based operators but illustrative of the ISO/TC 228 approach: defining service delivery requirements that a platform's compliance module may need to surface or verify for activity-category operators.

---

### W3C & IETF Standards

**RFC 7231 — HTTP/1.1 Semantics and Content**
- URL: https://datatracker.ietf.org/doc/html/rfc7231
- Defines the HTTP methods (GET, POST, PUT, PATCH, DELETE), status codes, and content negotiation that underpin every REST API in the tour operator ecosystem. All platform APIs should conform to this RFC.

**RFC 8288 — Web Linking**
- URL: https://datatracker.ietf.org/doc/html/rfc8288
- Defines the `Link` header and `rel` attribute for hypermedia navigation, relevant for HATEOAS-style API design in booking flows.

**RFC 6749 / RFC 6750 — OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The foundational standard for delegated API authorisation used by all major tour booking platforms (Rezdy, Checkfront, Bókun). A tour operator platform must implement OAuth 2.0 for third-party integrations and channel manager connectivity.

**OpenID Connect Core 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Adds an identity layer on top of OAuth 2.0, enabling secure single sign-on for agents, operators, and guests. Relevant for multi-tenant platforms where agents log in on behalf of operators.

**RFC 4122 — UUID Standard**
- URL: https://datatracker.ietf.org/doc/html/rfc4122
- Used universally in booking APIs (Peek/OCTO, Bókun, Rezdy) for booking, product, and availability identifiers. Platform APIs must generate and accept UUIDs.

**Schema.org — TouristAttraction, TouristTrip, TouristDestination Types**
- URL: https://schema.org/TouristAttraction
- W3C-recognised vocabulary for structured data markup. Tour operator platforms should emit JSON-LD using these schema types on product listing pages to improve search engine discoverability. Key types: `TouristAttraction`, `TouristTrip`, `TouristDestination`, `Event`.

---

### Data Model & API Specifications

**OpenTravel Alliance (OTA) XML Specifications**
- URL: https://opentravel.org/
- The industry's longest-standing XML message standard for travel data exchange, covering tours, activities, accommodation, and transport. OTA schemas underpin many legacy GDS integrations and wholesale channel connections. Supports W3C, IATA, and ISO standards. Relevant for operators connecting to traditional wholesale distribution networks.

**OCTO — Open Connectivity Standard for Tours, Activities & Attractions (v1)**
- URL: https://octo.travel/ · Docs: https://docs.octo.travel/ · GitHub: https://github.com/octotravel
- The modern open-source REST/JSON API specification for in-destination experiences. Defines standardised endpoints and field names for products, availability, pricing, booking creation, confirmation, and cancellation. 114+ trading partners as of 2026. Any new tour operator platform should implement OCTO as its primary channel manager interface — it is rapidly becoming the de-facto standard, adopted by Peek Pro, Ventrata, Rezdy (RezdyConnect), and others.

**IATA NDC (New Distribution Capability)**
- URL: https://developer.iata.org/en/ndc/
- IATA's XML/JSON standard replacing EDIFACT for airline content distribution. Relevant to tour operators packaging flights: NDC-compliant APIs allow direct access to airline ancillaries and dynamic pricing. Operators offering air-inclusive packages need NDC connectivity via a GDS or airline direct connection.

**OpenAPI 3.1 / Swagger**
- URL: https://spec.openapis.org/oas/v3.1.0
- The industry-standard specification language for describing REST APIs. All new platform APIs should be specified in OpenAPI 3.1, enabling auto-generated SDKs, interactive documentation, and contract testing.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/
- Used extensively within OpenAPI 3.1 and OCTO to define request/response payload schemas. Essential for validating booking data, itinerary structures, and availability windows.

---

### Security & Authentication Standards

**PCI DSS v4.0 — Payment Card Industry Data Security Standard**
- URL: https://www.pcisecuritystandards.org/
- Mandatory for any platform processing card payments directly. Governs network segmentation, encryption, access control, and monitoring. Tour operator platforms typically achieve compliance via Stripe, Adyen, or PayPal tokenisation, limiting scope to SAQ A or SAQ A-EP. Booking deposit and instalment payment workflows must be designed with PCI scope in mind.

**GDPR (EU) 2016/679 — General Data Protection Regulation**
- URL: https://gdpr-info.eu/
- Governs collection, storage, and processing of EU residents' personal data (traveller profiles, passport details, dietary requirements). The platform must implement lawful basis for processing, right-to-erasure workflows, data portability exports, and consent management for marketing.

**CCPA — California Consumer Privacy Act**
- URL: https://oag.ca.gov/privacy/ccpa
- US-equivalent privacy legislation for California residents. Relevant to platforms with North American customers.

**OWASP API Security Top 10**
- URL: https://owasp.org/www-project-api-security/
- The canonical checklist for API security vulnerabilities including broken object-level authorisation (BOLA), excessive data exposure, and mass assignment. All platform APIs should be audited against this list before public release.

---

### Regulatory & Compliance Frameworks

**EU Package Travel Directive — Directive (EU) 2015/2302**
- URL: https://eur-lex.europa.eu/eli/dir/2015/2302/oj/eng
- Applicable from 1 July 2018 across all EU member states. Requires tour operators to provide clear pre-contractual information, financial insolvency protection (bonding), defined cancellation and refund rights, and assistance obligations. Any platform serving European tour operators must produce documents and workflows that evidence compliance: pre-booking information sheets, confirmation documents, and fund-segregation audit trails.

**ATOL (UK) / ABTA (UK) / ASTA (US) Bonding Schemes**
- ATOL: https://www.caa.co.uk/atol-protection/ · ABTA: https://www.abta.com/ · ASTA: https://www.asta.org/
- Financial protection bonding schemes requiring tour operators to hold consumer funds in trust or provide bonds. The platform's accounting module should support segregated client fund tracking and produce reports compatible with bonding scheme audits.

**IATA Accreditation (BSP)**
- URL: https://www.iata.org/en/services/accreditation/
- Required for operators selling airline tickets directly. Relevant to tour operators packaging flights; the platform must support IATA agency number management and BSP settlement reporting.

**UNWTO Sustainable Tourism Criteria / GSTC**
- URL: https://www.gstc.org/
- The Global Sustainable Tourism Council criteria are increasingly required by corporate buyers and used as the basis for national sustainability certifications. The platform's product management module should support sustainability attribute tagging and documentation to help operators evidence GSTC alignment.

---

## Similar Products — Developer Documentation & APIs

### Rezdy
- **Description:** Online booking and channel management platform for tours, activities, and experiences. Provides supplier and reseller APIs plus a channel manager (RezdyConnect) with REST/JSON architecture.
- **API Documentation:** https://developers.rezdy.com/
- **Supplier API:** https://developers.rezdy.com/rezdyapi/index-supplier.html
- **Reseller API:** https://developers.rezdy.com/rezdyapi/index-reseller.html
- **Channel Manager (RezdyConnect):** https://developers.rezdy.com/rezdyconnect/index.html
- **Webhooks:** https://developers.rezdy.com/rezdywebhooks/index.html
- **Standards:** REST/JSON, HTTPS only, no CORS; webhooks via POST
- **Authentication:** API key

### Bókun (TripAdvisor)
- **Description:** Booking management and channel distribution platform owned by TripAdvisor, with direct Viator integration and 2,600+ OTA reseller connections.
- **API Documentation:** https://bokun.dev/booking-api-rest/
- **Authentication Guide:** https://bokun.dev/booking-api-rest/vU6sCfxwYdJWd1QAcLt12i/configuring-the-platform-for-api-usage-and-authentication/sFiGRpo4detkmrZPcWtQPj
- **APIs Page:** https://www.bokun.io/grow/apis
- **Standards:** REST/JSON; booking channels architecture; HMAC signature header authentication
- **Authentication:** Access key + secret key (HMAC signature)

### Peek Pro (OCTO-compliant)
- **Description:** Online booking software for activity and tour operators; implements the OCTO open standard API for reseller connectivity.
- **API Documentation (Reseller):** https://octodocs.peek.com/
- **OCTO Availability:** https://octodocs.peek.com/booking-flow/availability
- **OCTO Booking:** https://octodocs.peek.com/booking-flow/booking
- **Standards:** OCTO v1 (open standard), REST/JSON, OpenAPI-documented endpoints
- **Authentication:** API key per booking channel

### Checkfront
- **Description:** Booking management system with REST API v3.0, supporting availability queries, booking creation, and item management via OAuth2 or token authentication.
- **API Documentation:** http://api.checkfront.com/
- **GitHub Reference:** https://github.com/Checkfront/API
- **How-To Guides:** https://api.checkfront.com/guide/how_to.html
- **Standards:** REST/JSON (v3.0), OAuth 2.0 or token auth
- **Authentication:** OAuth 2.0 or authentication token

### TrekkSoft
- **Description:** Online booking and channel management for day-tour operators; provides TrekkConnect API for OTA distribution including GetYourGuide and Musement.
- **API Documentation:** https://developer.trekksoft.com/
- **Channel Manager Info:** https://www.trekksoft.com/en/blog/api-connections-for-tour-operators
- **Standards:** REST; unified channel manager API (TrekkConnect)
- **Authentication:** API key

### Ventrata (OCTO)
- **Description:** Enterprise-grade ticketing and tour operator platform implementing the full OCTO specification for high-volume attraction and activity operators.
- **API Documentation:** https://docs.ventrata.com/
- **Standards:** OCTO v1, REST/JSON, digital voucher and QR-code ticket delivery
- **Authentication:** API key per reseller integration

### Viator Partner API (TripAdvisor)
- **Description:** Distribution API providing access to 300,000+ tours and activities for OTAs and resellers; supports full booking lifecycle including hold, confirm, and cancel.
- **API Documentation:** https://docs.viator.com/partner-api/
- **Technical Guide:** https://partnerresources.viator.com/travel-commerce/technical-guide/
- **Supplier (Reservation) API:** https://docs.viator.com/supplier-api/technical/
- **Standards:** REST/JSON, OpenAPI-described
- **Authentication:** API key; partner-type permissions (affiliate vs. merchant)

### GetYourGuide Supplier API
- **Description:** Supplier-side connectivity API allowing tour operators and booking systems to push inventory and receive bookings from GetYourGuide's marketplace.
- **API Documentation:** https://integrator.getyourguide.com/documentation/overview
- **Supplier Endpoints:** https://integrator.getyourguide.com/documentation/supplier_endpoints
- **Partner API Reference:** https://code.getyourguide.com/partner-api-spec/
- **Standards:** REST/JSON
- **Authentication:** OAuth 2.0

### Amadeus for Developers — Tours and Activities API
- **Description:** Provides search and deep-link access to 300,000+ activities globally via REST/JSON; built in partnership with MyLittleAdventure aggregating Viator, GetYourGuide, Klook, and Musement inventory.
- **API Documentation:** https://developers.amadeus.com/self-service/category/destination-experiences/api-doc/tours-and-activities
- **API Reference:** https://developers.amadeus.com/self-service/category/destination-experiences/api-doc/tours-and-activities/api-reference
- **Developer Portal:** https://developers.amadeus.com/
- **Node SDK:** https://github.com/amadeus4dev/amadeus-node
- **Standards:** REST/JSON, OpenAPI-described, self-service sandbox environment
- **Authentication:** OAuth 2.0 (client credentials)

### Sabre Developer APIs
- **Description:** GDS API suite providing access to flights, hotels, car rentals, and tour packages via REST and SOAP; transitioning from legacy EDIFACT/SOAP to REST/JSON across product lines.
- **Developer Portal:** https://developer.sabre.com/
- **Standards:** REST/JSON (new APIs), SOAP/XML (legacy), NDC-compatible
- **Authentication:** OAuth 2.0

---

## Notes

**OCTO as the emerging backbone standard:** The OCTO specification is achieving rapid critical mass among modern tour operator platforms (Peek, Ventrata, Rezdy via RezdyConnect). Building OCTO-native connectivity as the primary channel manager interface will allow a new platform to onboard the majority of technically capable suppliers without bespoke integrations.

**OTA XML legacy coexistence:** Despite OCTO's momentum, a significant portion of wholesale and GDS distribution still runs over OpenTravel Alliance XML. A complete platform will need both an OCTO layer (modern direct OTAs) and an OTA/GDS bridge (traditional wholesale channels and flight-packaging operators).

**NDC for air-inclusive packages:** Tour operators assembling fly-drive or air-inclusive packages increasingly require NDC connectivity. Partnering with an NDC aggregator (e.g. Duffel, Verteil) rather than building direct airline connections is the practical path for a new platform.

**Regulatory compliance by region:** The EU Package Travel Directive (2015/2302), UK ATOL/ABTA, and IATA BSP requirements create distinct compliance surfaces. The platform should model these as configurable compliance profiles so operators in different markets see the appropriate document templates, fund-tracking workflows, and reporting outputs.
