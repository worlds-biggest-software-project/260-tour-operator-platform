# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Tour Operator Platform · Created: 2026-05-22

## Philosophy

This model follows classical third-normal-form (3NF) relational design. Every real-world concept -- operators, products, options, availability slots, bookings, itineraries, guests, guides, suppliers, payments, channels -- gets its own table with well-defined foreign keys. Reference data (countries, currencies, languages, compliance regimes) is modeled as lookup tables aligned with ISO and industry standards. Junction tables handle many-to-many relationships (itinerary-to-product, booking-to-guest, guide-to-language).

The approach mirrors how mature booking platforms like Rezdy and Checkfront structure their backend: operational CRUD against normalized tables, with reporting views or materialised views layered on top. It also aligns naturally with the OCTO specification's object hierarchy (Product -> Option -> Availability -> Booking -> UnitItem), making API mapping straightforward.

Normalised design is the default choice for regulated environments where data integrity, referential consistency, and auditable change trails matter. Tour operators subject to the EU Package Travel Directive, ATOL bonding, or IATA BSP settlement need to produce evidence-grade reports -- normalized schemas make this reliable because every fact is stored once.

**Best for:** Operators who prioritise data integrity, regulatory compliance, and complex cross-entity reporting over schema flexibility.

**Trade-offs:**
- Pro: Strong referential integrity prevents orphaned bookings, duplicate payments, or inconsistent inventory
- Pro: Natural fit for OCTO and OTA standard object hierarchies
- Pro: Straightforward SQL reporting for bonding scheme audits and financial reconciliation
- Pro: Well-understood by most development teams; extensive PostgreSQL ecosystem support
- Con: High table count (80+) increases migration complexity and onboarding time
- Con: Schema changes require migrations; adding jurisdiction-specific fields means altering tables
- Con: Many-to-many junction tables add query complexity for common operations
- Con: Read-heavy dashboards may need materialised views or denormalised reporting tables

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OCTO v1 (Open Connectivity for Tours & Activities) | Product/Option/Availability/Booking hierarchy directly maps to table structure; OCTO field names used as column names where possible |
| ISO 4217 | `currency_code CHAR(3)` on all monetary tables references the ISO 4217 currency lookup |
| ISO 3166-1 / ISO 3166-2 | Country and subdivision codes for operator jurisdictions, guest nationalities, destination regions |
| ISO 8601 | All temporal columns use `TIMESTAMPTZ`; date-only fields use `DATE`; duration stored as `INTERVAL` |
| RFC 4122 | UUID v7 primary keys on all tables |
| PCI DSS v4.0 | No raw card data stored; `payment_method_token` references tokenised payment via Stripe/Adyen |
| GDPR / CCPA | `guest` table includes `consent_marketing`, `consent_data_processing`, `data_retention_until` fields |
| EU Package Travel Directive 2015/2302 | `compliance_profile` table models per-jurisdiction obligations; `segregated_fund_ledger` tracks client money |
| Schema.org | `product` table includes `schema_org_type` for SEO JSON-LD emission (TouristTrip, TouristAttraction) |
| GSTC / UNWTO | `sustainability_attribute` table for tagging products with sustainability certifications |

---

## Core Platform Tables

### Tenant & Identity

```sql
-- Multi-tenant operator accounts
CREATE TABLE operator (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    legal_name      VARCHAR(255),
    tax_id          VARCHAR(50),
    country_code    CHAR(2) NOT NULL REFERENCES country(code),  -- ISO 3166-1
    subdivision_code VARCHAR(6) REFERENCES subdivision(code),   -- ISO 3166-2
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD' REFERENCES currency(code), -- ISO 4217
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    logo_url        TEXT,
    website_url     TEXT,
    iata_number     VARCHAR(20),         -- IATA agency number if applicable
    atol_number     VARCHAR(20),         -- UK ATOL licence number
    abta_number     VARCHAR(20),         -- ABTA membership number
    compliance_profile_id UUID REFERENCES compliance_profile(id),
    subscription_tier VARCHAR(20) NOT NULL DEFAULT 'foundation',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_operator_country ON operator(country_code);

-- Users who can log into the platform (agents, admins, guides)
CREATE TABLE platform_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    password_hash   VARCHAR(255),
    full_name       VARCHAR(255) NOT NULL,
    phone           VARCHAR(50),
    avatar_url      TEXT,
    auth_provider   VARCHAR(20) DEFAULT 'local',  -- local, google, microsoft
    auth_provider_id VARCHAR(255),
    email_verified  BOOLEAN NOT NULL DEFAULT FALSE,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Links users to operators with roles (multi-tenant RBAC)
CREATE TABLE operator_membership (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operator_id     UUID NOT NULL REFERENCES operator(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES platform_user(id) ON DELETE CASCADE,
    role            VARCHAR(30) NOT NULL DEFAULT 'agent',  -- owner, admin, agent, guide, viewer
    permissions     VARCHAR(30)[] DEFAULT '{}',  -- granular overrides: booking.create, itinerary.edit, etc.
    invited_at      TIMESTAMPTZ,
    accepted_at     TIMESTAMPTZ,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(operator_id, user_id)
);
CREATE INDEX idx_membership_operator ON operator_membership(operator_id);
CREATE INDEX idx_membership_user ON operator_membership(user_id);
```

### Reference Data

```sql
CREATE TABLE country (
    code            CHAR(2) PRIMARY KEY,  -- ISO 3166-1 alpha-2
    name            VARCHAR(100) NOT NULL,
    alpha3          CHAR(3) NOT NULL,     -- ISO 3166-1 alpha-3
    numeric_code    CHAR(3)               -- ISO 3166-1 numeric
);

CREATE TABLE subdivision (
    code            VARCHAR(6) PRIMARY KEY,  -- ISO 3166-2 e.g. US-CA
    country_code    CHAR(2) NOT NULL REFERENCES country(code),
    name            VARCHAR(200) NOT NULL,
    type            VARCHAR(50)  -- state, province, region, territory
);
CREATE INDEX idx_subdivision_country ON subdivision(country_code);

CREATE TABLE currency (
    code            CHAR(3) PRIMARY KEY,  -- ISO 4217
    name            VARCHAR(100) NOT NULL,
    symbol          VARCHAR(5),
    minor_units     SMALLINT NOT NULL DEFAULT 2
);

CREATE TABLE language (
    code            VARCHAR(10) PRIMARY KEY,  -- IETF BCP 47 e.g. en-US
    name            VARCHAR(100) NOT NULL
);

CREATE TABLE compliance_profile (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(100) NOT NULL,  -- e.g. "EU Package Travel", "UK ATOL", "US Standard"
    jurisdiction_code CHAR(2) NOT NULL REFERENCES country(code),
    requires_bonding BOOLEAN NOT NULL DEFAULT FALSE,
    requires_fund_segregation BOOLEAN NOT NULL DEFAULT FALSE,
    cancellation_policy_template JSONB,
    pre_booking_info_template JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Product Catalogue (OCTO-aligned)

```sql
-- Tour products (maps to OCTO Product)
CREATE TABLE product (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operator_id     UUID NOT NULL REFERENCES operator(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(150),
    description     TEXT,
    short_description VARCHAR(500),
    product_type    VARCHAR(30) NOT NULL,  -- day_tour, multi_day, activity, transfer, package
    duration        INTERVAL,
    duration_days   INTEGER,               -- for multi-day tours
    country_code    CHAR(2) REFERENCES country(code),
    destination     VARCHAR(255),
    meeting_point   TEXT,
    latitude        DECIMAL(10, 7),
    longitude       DECIMAL(10, 7),
    min_participants INTEGER DEFAULT 1,
    max_participants INTEGER,
    difficulty_level VARCHAR(20),          -- easy, moderate, challenging, extreme
    minimum_age     INTEGER,
    schema_org_type VARCHAR(50) DEFAULT 'TouristTrip',  -- Schema.org type for SEO
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',  -- draft, active, archived
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_product_operator ON product(operator_id);
CREATE INDEX idx_product_type ON product(product_type);
CREATE INDEX idx_product_status ON product(status);
CREATE INDEX idx_product_destination ON product(country_code, destination);

-- Product options (maps to OCTO Option -- e.g. "Morning departure", "With lunch")
CREATE TABLE product_option (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES product(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    duration        INTERVAL,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    is_default      BOOLEAN NOT NULL DEFAULT FALSE,
    required_contact_fields VARCHAR(50)[] DEFAULT '{}',  -- OCTO: fullName, emailAddress, phoneNumber
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_option_product ON product_option(product_id);

-- Unit types within an option (maps to OCTO Unit -- e.g. "Adult", "Child", "Senior")
CREATE TABLE unit_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    option_id       UUID NOT NULL REFERENCES product_option(id) ON DELETE CASCADE,
    name            VARCHAR(100) NOT NULL,  -- adult, child, infant, senior, family
    reference       VARCHAR(50),           -- OCTO unit reference
    min_age         INTEGER,
    max_age         INTEGER,
    is_accompanied  BOOLEAN DEFAULT FALSE, -- must be accompanied by adult
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_unit_option ON unit_type(option_id);

-- Product images
CREATE TABLE product_image (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES product(id) ON DELETE CASCADE,
    url             TEXT NOT NULL,
    alt_text        VARCHAR(255),
    sort_order      INTEGER NOT NULL DEFAULT 0,
    is_primary      BOOLEAN NOT NULL DEFAULT FALSE
);
CREATE INDEX idx_image_product ON product_image(product_id);

-- Product sustainability attributes (GSTC alignment)
CREATE TABLE product_sustainability (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES product(id) ON DELETE CASCADE,
    attribute_type  VARCHAR(50) NOT NULL,  -- carbon_offset, eco_certified, local_community, wildlife_safe
    certification   VARCHAR(100),          -- e.g. "Green Globe", "Travelife", "EarthCheck"
    details         TEXT,
    verified        BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sustainability_product ON product_sustainability(product_id);
```

### Pricing

```sql
-- Pricing for each unit type
CREATE TABLE pricing (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    unit_type_id    UUID NOT NULL REFERENCES unit_type(id) ON DELETE CASCADE,
    currency_code   CHAR(3) NOT NULL REFERENCES currency(code),
    retail_price    NUMERIC(12, 2) NOT NULL,
    net_price       NUMERIC(12, 2),  -- cost to operator (supplier net rate)
    commission_pct  DECIMAL(5, 2),
    tax_included    BOOLEAN NOT NULL DEFAULT TRUE,
    tax_rate        DECIMAL(5, 4),
    valid_from      DATE NOT NULL,
    valid_to        DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_pricing_unit ON pricing(unit_type_id);
CREATE INDEX idx_pricing_validity ON pricing(valid_from, valid_to);

-- Season-based pricing overrides
CREATE TABLE pricing_season (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES product(id) ON DELETE CASCADE,
    name            VARCHAR(100) NOT NULL,  -- "High Season", "Christmas Peak"
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    price_modifier  DECIMAL(5, 2) NOT NULL,  -- multiplier e.g. 1.25 for 25% surcharge
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_season_product ON pricing_season(product_id);
```

### Availability & Inventory

```sql
-- Availability slots (maps to OCTO Availability)
CREATE TABLE availability (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES product(id) ON DELETE CASCADE,
    option_id       UUID NOT NULL REFERENCES product_option(id),
    local_date      DATE NOT NULL,
    start_time      TIME,
    end_time        TIME,
    total_capacity  INTEGER NOT NULL,
    remaining_capacity INTEGER NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'available',  -- available, limited, sold_out, closed
    cutoff_minutes  INTEGER DEFAULT 60,  -- booking cutoff before start
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_availability_product_date ON availability(product_id, local_date);
CREATE INDEX idx_availability_option_date ON availability(option_id, local_date);
CREATE INDEX idx_availability_status ON availability(status);

-- Availability rules (recurring schedules)
CREATE TABLE availability_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES product(id) ON DELETE CASCADE,
    option_id       UUID NOT NULL REFERENCES product_option(id),
    day_of_week     SMALLINT[],  -- 0=Sun, 1=Mon, ... 6=Sat
    start_time      TIME NOT NULL,
    end_time        TIME,
    capacity        INTEGER NOT NULL,
    valid_from      DATE NOT NULL,
    valid_to        DATE,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rule_product ON availability_rule(product_id);
```

### Bookings

```sql
-- Booking (maps to OCTO Booking)
CREATE TABLE booking (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operator_id     UUID NOT NULL REFERENCES operator(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    option_id       UUID NOT NULL REFERENCES product_option(id),
    availability_id UUID REFERENCES availability(id),
    booking_number  VARCHAR(20) NOT NULL UNIQUE,  -- human-readable: OP-2026-001234
    reseller_reference VARCHAR(100),              -- OCTO resellerReference
    channel_id      UUID REFERENCES distribution_channel(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',  -- pending, confirmed, cancelled, completed, no_show
    contact_name    VARCHAR(255),
    contact_email   VARCHAR(255),
    contact_phone   VARCHAR(50),
    contact_country CHAR(2) REFERENCES country(code),
    contact_locale  VARCHAR(10),
    notes           TEXT,
    internal_notes  TEXT,
    total_amount    NUMERIC(12, 2) NOT NULL,
    currency_code   CHAR(3) NOT NULL REFERENCES currency(code),
    amount_paid     NUMERIC(12, 2) NOT NULL DEFAULT 0,
    tax_amount      NUMERIC(12, 2) DEFAULT 0,
    booked_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    confirmed_at    TIMESTAMPTZ,
    cancelled_at    TIMESTAMPTZ,
    cancellation_reason TEXT,
    travel_date     DATE NOT NULL,
    created_by      UUID REFERENCES platform_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_booking_operator ON booking(operator_id);
CREATE INDEX idx_booking_product ON booking(product_id);
CREATE INDEX idx_booking_status ON booking(status);
CREATE INDEX idx_booking_travel_date ON booking(travel_date);
CREATE INDEX idx_booking_number ON booking(booking_number);
CREATE INDEX idx_booking_channel ON booking(channel_id);

-- Individual participants in a booking (maps to OCTO UnitItem)
CREATE TABLE booking_unit_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_id      UUID NOT NULL REFERENCES booking(id) ON DELETE CASCADE,
    unit_type_id    UUID NOT NULL REFERENCES unit_type(id),
    full_name       VARCHAR(255),
    age             INTEGER,
    dietary_requirements TEXT,
    special_requests TEXT,
    unit_price      NUMERIC(12, 2) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_unit_item_booking ON booking_unit_item(booking_id);

-- Booking form field responses (custom questions per product)
CREATE TABLE booking_field_response (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_id      UUID NOT NULL REFERENCES booking(id) ON DELETE CASCADE,
    unit_item_id    UUID REFERENCES booking_unit_item(id),  -- NULL = booking-level, non-NULL = guest-level
    field_name      VARCHAR(100) NOT NULL,
    field_value     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_field_response_booking ON booking_field_response(booking_id);
```

### Payments & Financial

```sql
CREATE TABLE payment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_id      UUID NOT NULL REFERENCES booking(id),
    operator_id     UUID NOT NULL REFERENCES operator(id),
    amount          NUMERIC(12, 2) NOT NULL,
    currency_code   CHAR(3) NOT NULL REFERENCES currency(code),
    payment_type    VARCHAR(20) NOT NULL,  -- deposit, balance, refund
    payment_method  VARCHAR(20) NOT NULL,  -- card, bank_transfer, cash, voucher
    payment_method_token VARCHAR(255),     -- Stripe/Adyen token (PCI DSS compliant)
    gateway         VARCHAR(30),           -- stripe, adyen, paypal
    gateway_reference VARCHAR(255),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',  -- pending, completed, failed, refunded
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_payment_booking ON payment(booking_id);
CREATE INDEX idx_payment_operator ON payment(operator_id);
CREATE INDEX idx_payment_status ON payment(status);

-- Segregated client fund tracking (ATOL/ABTA compliance)
CREATE TABLE segregated_fund_ledger (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operator_id     UUID NOT NULL REFERENCES operator(id),
    booking_id      UUID REFERENCES booking(id),
    payment_id      UUID REFERENCES payment(id),
    entry_type      VARCHAR(20) NOT NULL,  -- deposit_in, balance_in, supplier_payout, refund_out, commission_release
    amount          NUMERIC(12, 2) NOT NULL,  -- positive = money in, negative = money out
    currency_code   CHAR(3) NOT NULL REFERENCES currency(code),
    running_balance NUMERIC(14, 2) NOT NULL,
    description     TEXT,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_fund_operator ON segregated_fund_ledger(operator_id);
CREATE INDEX idx_fund_booking ON segregated_fund_ledger(booking_id);
```

### Itinerary Management

```sql
-- Multi-day itinerary (linked to a product or a custom booking)
CREATE TABLE itinerary (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operator_id     UUID NOT NULL REFERENCES operator(id),
    product_id      UUID REFERENCES product(id),  -- NULL for custom/bespoke itineraries
    booking_id      UUID REFERENCES booking(id),  -- NULL for template itineraries
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    total_days      INTEGER NOT NULL,
    start_date      DATE,
    end_date        DATE,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',  -- draft, published, active, completed
    version         INTEGER NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_itinerary_operator ON itinerary(operator_id);
CREATE INDEX idx_itinerary_product ON itinerary(product_id);
CREATE INDEX idx_itinerary_booking ON itinerary(booking_id);

-- Individual day within an itinerary
CREATE TABLE itinerary_day (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    itinerary_id    UUID NOT NULL REFERENCES itinerary(id) ON DELETE CASCADE,
    day_number      INTEGER NOT NULL,
    title           VARCHAR(255),
    description     TEXT,
    destination     VARCHAR(255),
    country_code    CHAR(2) REFERENCES country(code),
    accommodation_id UUID REFERENCES supplier(id),
    meal_plan       VARCHAR(30),  -- none, breakfast, half_board, full_board, all_inclusive
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_itin_day_itinerary ON itinerary_day(itinerary_id);

-- Activities/events within an itinerary day
CREATE TABLE itinerary_activity (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    itinerary_day_id UUID NOT NULL REFERENCES itinerary_day(id) ON DELETE CASCADE,
    product_id      UUID REFERENCES product(id),       -- internal product or NULL
    supplier_id     UUID REFERENCES supplier(id),       -- external supplier
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    start_time      TIME,
    end_time        TIME,
    duration        INTERVAL,
    activity_type   VARCHAR(30),  -- tour, transfer, meal, free_time, accommodation, flight
    location        VARCHAR(255),
    latitude        DECIMAL(10, 7),
    longitude       DECIMAL(10, 7),
    cost            NUMERIC(12, 2),
    currency_code   CHAR(3) REFERENCES currency(code),
    sort_order      INTEGER NOT NULL DEFAULT 0,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_itin_activity_day ON itinerary_activity(itinerary_day_id);
```

### Suppliers & Guides

```sql
CREATE TABLE supplier (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operator_id     UUID NOT NULL REFERENCES operator(id),
    name            VARCHAR(255) NOT NULL,
    supplier_type   VARCHAR(30) NOT NULL,  -- accommodation, transport, restaurant, activity_provider, airline, guide_company
    contact_name    VARCHAR(255),
    contact_email   VARCHAR(255),
    contact_phone   VARCHAR(50),
    country_code    CHAR(2) REFERENCES country(code),
    address         TEXT,
    website_url     TEXT,
    payment_terms   VARCHAR(50),  -- prepaid, net_7, net_14, net_30, on_arrival
    commission_pct  DECIMAL(5, 2),
    rating          DECIMAL(3, 2),  -- internal performance rating 0.00-5.00
    is_preferred    BOOLEAN NOT NULL DEFAULT FALSE,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_supplier_operator ON supplier(operator_id);
CREATE INDEX idx_supplier_type ON supplier(supplier_type);

-- Guide profiles
CREATE TABLE guide (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operator_id     UUID NOT NULL REFERENCES operator(id),
    user_id         UUID REFERENCES platform_user(id),  -- may also be a platform user
    full_name       VARCHAR(255) NOT NULL,
    email           VARCHAR(255),
    phone           VARCHAR(50),
    bio             TEXT,
    photo_url       TEXT,
    certifications  TEXT[],         -- first_aid, wilderness, diving_master
    specialisations TEXT[],        -- history, wildlife, adventure, culinary
    daily_rate      NUMERIC(10, 2),
    currency_code   CHAR(3) REFERENCES currency(code),
    rating          DECIMAL(3, 2),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_guide_operator ON guide(operator_id);

-- Guide languages
CREATE TABLE guide_language (
    guide_id        UUID NOT NULL REFERENCES guide(id) ON DELETE CASCADE,
    language_code   VARCHAR(10) NOT NULL REFERENCES language(code),
    proficiency     VARCHAR(20) NOT NULL DEFAULT 'fluent',  -- native, fluent, conversational, basic
    PRIMARY KEY (guide_id, language_code)
);

-- Guide assignments to bookings/itinerary days
CREATE TABLE guide_assignment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    guide_id        UUID NOT NULL REFERENCES guide(id),
    booking_id      UUID REFERENCES booking(id),
    itinerary_day_id UUID REFERENCES itinerary_day(id),
    assignment_date DATE NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'tentative',  -- tentative, confirmed, declined, completed
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_assignment_guide ON guide_assignment(guide_id);
CREATE INDEX idx_assignment_booking ON guide_assignment(booking_id);
CREATE INDEX idx_assignment_date ON guide_assignment(assignment_date);
```

### Guest & CRM

```sql
CREATE TABLE guest (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operator_id     UUID NOT NULL REFERENCES operator(id),
    email           VARCHAR(255),
    full_name       VARCHAR(255) NOT NULL,
    phone           VARCHAR(50),
    nationality     CHAR(2) REFERENCES country(code),
    date_of_birth   DATE,
    passport_number VARCHAR(50),  -- encrypted at application level
    dietary_requirements TEXT,
    medical_notes   TEXT,
    preferred_language VARCHAR(10) REFERENCES language(code),
    consent_marketing BOOLEAN NOT NULL DEFAULT FALSE,
    consent_data_processing BOOLEAN NOT NULL DEFAULT TRUE,
    data_retention_until DATE,   -- GDPR: auto-purge after this date
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_guest_operator ON guest(operator_id);
CREATE INDEX idx_guest_email ON guest(operator_id, email);

-- Links guests to bookings
CREATE TABLE booking_guest (
    booking_id      UUID NOT NULL REFERENCES booking(id) ON DELETE CASCADE,
    guest_id        UUID NOT NULL REFERENCES guest(id),
    is_lead         BOOLEAN NOT NULL DEFAULT FALSE,  -- lead guest / primary contact
    PRIMARY KEY (booking_id, guest_id)
);
```

### Distribution Channels

```sql
CREATE TABLE distribution_channel (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operator_id     UUID NOT NULL REFERENCES operator(id),
    name            VARCHAR(100) NOT NULL,  -- "Viator", "GetYourGuide", "Direct Website", "Agent Portal"
    channel_type    VARCHAR(30) NOT NULL,  -- ota, direct, agent, api
    protocol        VARCHAR(20),           -- octo, ota_xml, custom_api, manual
    api_endpoint    TEXT,
    api_key_hash    VARCHAR(255),
    commission_pct  DECIMAL(5, 2),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_channel_operator ON distribution_channel(operator_id);

-- Maps which products are distributed on which channels
CREATE TABLE channel_product (
    channel_id      UUID NOT NULL REFERENCES distribution_channel(id) ON DELETE CASCADE,
    product_id      UUID NOT NULL REFERENCES product(id) ON DELETE CASCADE,
    external_product_id VARCHAR(255),  -- product ID on the OTA side
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    last_synced_at  TIMESTAMPTZ,
    PRIMARY KEY (channel_id, product_id)
);
```

### Communications & Notifications

```sql
CREATE TABLE communication (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operator_id     UUID NOT NULL REFERENCES operator(id),
    booking_id      UUID REFERENCES booking(id),
    guest_id        UUID REFERENCES guest(id),
    channel         VARCHAR(20) NOT NULL,  -- email, sms, whatsapp, in_app
    direction       VARCHAR(10) NOT NULL,  -- outbound, inbound
    template_name   VARCHAR(100),
    subject         VARCHAR(255),
    body            TEXT,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',  -- pending, sent, delivered, failed, opened
    sent_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_comm_booking ON communication(booking_id);
CREATE INDEX idx_comm_guest ON communication(guest_id);
```

### Audit Trail

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operator_id     UUID NOT NULL REFERENCES operator(id),
    user_id         UUID REFERENCES platform_user(id),
    entity_type     VARCHAR(50) NOT NULL,  -- booking, product, itinerary, payment, guide_assignment
    entity_id       UUID NOT NULL,
    action          VARCHAR(20) NOT NULL,  -- create, update, delete, status_change
    old_values      JSONB,
    new_values      JSONB,
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_operator ON audit_log(operator_id);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Identity | 3 | operator, platform_user, operator_membership |
| Reference Data | 4 | country, subdivision, currency, language |
| Compliance | 1 | compliance_profile |
| Product Catalogue | 5 | product, product_option, unit_type, product_image, product_sustainability |
| Pricing | 2 | pricing, pricing_season |
| Availability | 2 | availability, availability_rule |
| Bookings | 3 | booking, booking_unit_item, booking_field_response |
| Payments & Financial | 2 | payment, segregated_fund_ledger |
| Itinerary | 3 | itinerary, itinerary_day, itinerary_activity |
| Suppliers & Guides | 4 | supplier, guide, guide_language, guide_assignment |
| Guest & CRM | 2 | guest, booking_guest |
| Distribution | 2 | distribution_channel, channel_product |
| Communications | 1 | communication |
| Audit | 1 | audit_log |
| **Total** | **35** | Core schema; additional tables for AI features, reviews, and reporting views expected |

---

## Key Design Decisions

1. **OCTO-native object hierarchy**: The Product -> Option -> UnitType -> Availability -> Booking -> UnitItem chain directly mirrors the OCTO specification, making API implementation a thin mapping layer rather than a transformation.

2. **Multi-tenant via `operator_id` foreign keys and RBAC membership table**: Every business-data table carries an `operator_id` foreign key. PostgreSQL Row-Level Security policies can enforce tenant isolation at the database level.

3. **ISO-standard reference data tables**: Countries (ISO 3166-1), subdivisions (ISO 3166-2), currencies (ISO 4217), and languages (BCP 47) as lookup tables rather than inline strings, ensuring consistency and enabling jurisdictional filtering.

4. **Segregated fund ledger for bonding compliance**: A dedicated ledger table tracks client money movements separately from operational accounting, producing audit-ready reports for ATOL/ABTA/ASTA bonding scheme inspections.

5. **Itinerary as a first-class entity with day/activity decomposition**: Itineraries are not embedded in bookings but exist independently, supporting both template itineraries (linked to products) and bespoke itineraries (linked to bookings). Day-level granularity enables guide assignment and accommodation tracking.

6. **JSONB for audit log diffs only**: The schema avoids JSONB for business data, using it only in `audit_log.old_values`/`new_values` where the shape of changed fields is inherently variable.

7. **No raw payment card data**: PCI DSS compliance is achieved by storing only tokenised references (`payment_method_token`) from Stripe/Adyen. The platform never touches card numbers.

8. **GDPR-ready guest table**: Explicit consent columns and a `data_retention_until` date enable automated data purge workflows and right-to-erasure compliance.

9. **Distribution channel protocol awareness**: The `distribution_channel` table records whether a channel speaks OCTO, OTA XML, or a custom API, enabling the integration layer to route messages through the correct adapter.

10. **Guide assignment with language and specialisation matching**: The `guide`, `guide_language`, and `guide_assignment` tables support the AI guide-matching feature by providing structured data for ML-based assignment optimisation.
