# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Tour Operator Platform · Created: 2026-05-22

## Philosophy

This model uses a traditional relational backbone for core entities (operators, bookings, payments, users) but stores variable, jurisdiction-specific, and rapidly evolving data in JSONB columns. The approach recognises that a tour operator platform spans many jurisdictions (EU, UK, US, APAC), each with different compliance requirements, document templates, and custom fields -- and that the product catalogue varies enormously between a cycling tour operator in the Netherlands and a safari DMC in Kenya.

Rather than adding columns for every jurisdiction-specific field or every supplier's unique metadata, the hybrid model defines a stable relational schema for the 80% of fields that are universal, and uses typed JSONB columns (with JSON Schema validation at the application layer) for the 20% that varies. This is the approach used by modern SaaS platforms like Stripe (where payment method details vary by type) and Shopify (where product metafields store merchant-specific attributes).

The practical benefit is speed: new compliance requirements, custom booking fields, or supplier-specific attributes can be added without database migrations. The cost is that JSONB columns are harder to enforce at the database level and require application-layer validation. PostgreSQL's JSONB operators and GIN indexes mitigate the query performance concern.

**Best for:** Rapid MVP development, multi-region operators with jurisdiction-specific requirements, and teams that want to iterate on the data model without frequent schema migrations.

**Trade-offs:**
- Pro: No migrations needed for jurisdiction-specific or custom fields -- just update the JSON schema
- Pro: Flexible product metadata handles wildly different tour types (cycling, safari, cruise, food tour) without schema changes
- Pro: Fewer tables than a fully normalised model (approximately 25 vs. 35+)
- Pro: JSONB columns with GIN indexes still support efficient queries for common access patterns
- Pro: Natural fit for storing OCTO API request/response payloads and OTA XML mappings
- Con: JSONB fields lack database-enforced referential integrity and type constraints
- Con: Requires disciplined application-layer validation (JSON Schema) to prevent data quality drift
- Con: Complex JSONB queries can be harder to optimise than simple column lookups
- Con: Reporting across JSONB fields requires JSONB path extraction, which is less intuitive for SQL analysts
- Con: Schema-on-read means broken data can enter the system if validation is bypassed

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OCTO v1 | Product/Option/Booking structure is relational; OCTO-specific fields that vary by capability (content, pricing, pickups) stored in `octo_data JSONB` |
| ISO 4217 | `currency_code CHAR(3)` on monetary tables; currencies in JSONB payloads validated against ISO 4217 |
| ISO 3166-1 / ISO 3166-2 | Country codes relational; subdivision and jurisdiction-specific compliance data in `compliance_data JSONB` |
| ISO 8601 | Timestamps as `TIMESTAMPTZ`; durations in JSONB as ISO 8601 duration strings (e.g. `"PT3H30M"`) |
| RFC 4122 | UUID primary keys throughout |
| EU Package Travel Directive | Compliance rules stored as JSONB templates per jurisdiction in `compliance_profile.rules` |
| PCI DSS v4.0 | No raw card data; payment gateway details in `payment.gateway_data JSONB` contain only tokenised references |
| GDPR | Guest PII fields relational with `consent` JSONB for granular consent tracking; `data_retention_policy` in operator settings |
| Schema.org | Product SEO metadata stored in `product.seo_data JSONB` with Schema.org TouristTrip/TouristAttraction fields |
| OpenTravel Alliance | OTA XML mapping rules stored in `channel.integration_config JSONB` per channel |

---

## Core Tables

### Tenant & Identity

```sql
CREATE TABLE operator (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    legal_name      VARCHAR(255),
    country_code    CHAR(2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',

    -- Variable operator settings by jurisdiction / preference
    settings        JSONB NOT NULL DEFAULT '{}',
    /*
    Example settings JSONB:
    {
      "branding": {"logoUrl": "https://...", "primaryColor": "#2563EB"},
      "compliance": {
        "regime": "eu_package_travel",
        "atolNumber": "T1234",
        "requiresPreBookingInfo": true,
        "fundSegregationEnabled": true,
        "cancellationPolicyTemplate": "eu_14day_cooling"
      },
      "notifications": {
        "bookingConfirmEmail": true,
        "guestReminder48h": true,
        "guideAssignmentSms": false
      },
      "iata": {"agencyNumber": "12345678", "bspParticipant": true},
      "sustainability": {"gstcAligned": true, "carbonTrackingEnabled": true}
    }
    */

    subscription_tier VARCHAR(20) NOT NULL DEFAULT 'foundation',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE platform_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    password_hash   VARCHAR(255),
    full_name       VARCHAR(255) NOT NULL,
    phone           VARCHAR(50),
    operator_id     UUID REFERENCES operator(id),
    role            VARCHAR(30) NOT NULL DEFAULT 'agent',
    permissions     JSONB NOT NULL DEFAULT '[]',
    /*
    Example permissions JSONB:
    ["booking.create", "booking.cancel", "itinerary.edit", "guide.assign", "report.view"]
    */
    profile         JSONB NOT NULL DEFAULT '{}',
    /*
    Example profile JSONB:
    {"avatarUrl": "https://...", "locale": "en-GB", "authProvider": "google", "lastLoginAt": "2026-05-20T14:30:00Z"}
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_user_operator ON platform_user(operator_id);
```

### Product Catalogue

```sql
CREATE TABLE product (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operator_id     UUID NOT NULL REFERENCES operator(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(150),
    description     TEXT,
    product_type    VARCHAR(30) NOT NULL,  -- day_tour, multi_day, activity, transfer, package
    duration_days   INTEGER,
    destination     VARCHAR(255),
    country_code    CHAR(2),
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',

    -- Structured metadata for the variable parts of a product
    logistics       JSONB NOT NULL DEFAULT '{}',
    /*
    Example logistics JSONB:
    {
      "meetingPoint": {"address": "123 Main St", "lat": 51.5074, "lng": -0.1278, "instructions": "Meet at the red door"},
      "endPoint": {"address": "456 Park Ave", "lat": 51.5155, "lng": -0.1420},
      "minParticipants": 2,
      "maxParticipants": 16,
      "difficulty": "moderate",
      "minimumAge": 12,
      "fitnessLevel": "average",
      "equipmentProvided": ["helmet", "bike", "water_bottle"],
      "equipmentRequired": ["hiking_boots", "rain_jacket"],
      "inclusions": ["lunch", "hotel_pickup", "entrance_fees"],
      "exclusions": ["travel_insurance", "tips"]
    }
    */

    -- SEO and content metadata
    seo_data        JSONB NOT NULL DEFAULT '{}',
    /*
    Example seo_data JSONB:
    {
      "schemaOrgType": "TouristTrip",
      "keywords": ["cycling tour", "amsterdam", "windmills"],
      "images": [
        {"url": "https://...", "altText": "Cyclists on dike", "isPrimary": true},
        {"url": "https://...", "altText": "Windmill at sunset", "isPrimary": false}
      ],
      "highlights": ["Small group (max 12)", "Local guide", "Lunch at farm"]
    }
    */

    -- Sustainability attributes
    sustainability  JSONB NOT NULL DEFAULT '{}',
    /*
    Example sustainability JSONB:
    {
      "certifications": ["Green Globe", "Travelife Gold"],
      "carbonOffsetIncluded": true,
      "estimatedCo2Kg": 12.5,
      "localCommunityBenefit": "Partners with 3 local farms",
      "wildlifeSafe": true
    }
    */

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_product_operator ON product(operator_id);
CREATE INDEX idx_product_type ON product(product_type);
CREATE INDEX idx_product_status ON product(status);
CREATE INDEX idx_product_destination ON product(country_code, destination);
CREATE INDEX idx_product_sustainability ON product USING GIN (sustainability);

-- Product options (OCTO Option)
CREATE TABLE product_option (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES product(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    is_default      BOOLEAN NOT NULL DEFAULT FALSE,
    sort_order      INTEGER NOT NULL DEFAULT 0,

    -- Variable option configuration
    config          JSONB NOT NULL DEFAULT '{}',
    /*
    Example config JSONB:
    {
      "duration": "PT4H",
      "requiredContactFields": ["fullName", "emailAddress", "phoneNumber"],
      "units": [
        {"id": "adult", "name": "Adult", "minAge": 18, "maxAge": null, "sortOrder": 0},
        {"id": "child", "name": "Child (8-17)", "minAge": 8, "maxAge": 17, "sortOrder": 1},
        {"id": "senior", "name": "Senior (65+)", "minAge": 65, "maxAge": null, "sortOrder": 2}
      ],
      "pricing": {
        "adult": {"retail": 149.00, "net": 120.00, "currency": "EUR"},
        "child": {"retail": 99.00, "net": 80.00, "currency": "EUR"},
        "senior": {"retail": 129.00, "net": 104.00, "currency": "EUR"}
      },
      "seasonalPricing": [
        {"name": "Peak Summer", "startDate": "2026-07-01", "endDate": "2026-08-31", "modifier": 1.25}
      ]
    }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_option_product ON product_option(product_id);
```

### Availability

```sql
CREATE TABLE availability (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES product(id) ON DELETE CASCADE,
    option_id       UUID NOT NULL REFERENCES product_option(id),
    local_date      DATE NOT NULL,
    start_time      TIME,
    total_capacity  INTEGER NOT NULL,
    remaining_capacity INTEGER NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'available',
    cutoff_minutes  INTEGER DEFAULT 60,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_avail_product_date ON availability(product_id, local_date);
CREATE INDEX idx_avail_option_date ON availability(option_id, local_date);
```

### Bookings

```sql
CREATE TABLE booking (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operator_id     UUID NOT NULL REFERENCES operator(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    option_id       UUID NOT NULL REFERENCES product_option(id),
    availability_id UUID REFERENCES availability(id),
    booking_number  VARCHAR(20) NOT NULL UNIQUE,
    channel_id      UUID REFERENCES distribution_channel(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    travel_date     DATE NOT NULL,

    -- Contact info (relational for indexing/search)
    contact_name    VARCHAR(255),
    contact_email   VARCHAR(255),
    contact_phone   VARCHAR(50),

    -- Financials (relational for reporting)
    total_amount    NUMERIC(12, 2) NOT NULL,
    amount_paid     NUMERIC(12, 2) NOT NULL DEFAULT 0,
    currency_code   CHAR(3) NOT NULL,

    -- Booking-specific variable data
    booking_data    JSONB NOT NULL DEFAULT '{}',
    /*
    Example booking_data JSONB:
    {
      "resellerReference": "V-98765",
      "contactLocale": "en-GB",
      "contactCountry": "GB",
      "unitItems": [
        {
          "unitId": "adult",
          "fullName": "Jane Smith",
          "age": 34,
          "unitPrice": 149.00,
          "dietaryRequirements": "vegetarian",
          "specialRequests": "Window seat on bus"
        },
        {
          "unitId": "child",
          "fullName": "Tom Smith",
          "age": 8,
          "unitPrice": 99.00,
          "dietaryRequirements": null,
          "specialRequests": null
        }
      ],
      "customFieldResponses": [
        {"field": "hotel_pickup_address", "value": "Hilton Amsterdam, Apollolaan 138"},
        {"field": "shoe_size", "value": "EU 42"}
      ],
      "internalNotes": "VIP guest - returning customer, 4th booking",
      "cancellationReason": null,
      "complianceDocsSent": {
        "preBookingInfo": "2026-05-01T10:30:00Z",
        "confirmationDoc": "2026-05-01T10:35:00Z"
      }
    }
    */

    booked_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    confirmed_at    TIMESTAMPTZ,
    cancelled_at    TIMESTAMPTZ,
    created_by      UUID REFERENCES platform_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_booking_operator ON booking(operator_id);
CREATE INDEX idx_booking_status ON booking(status);
CREATE INDEX idx_booking_travel_date ON booking(travel_date);
CREATE INDEX idx_booking_number ON booking(booking_number);
CREATE INDEX idx_booking_channel ON booking(channel_id);
CREATE INDEX idx_booking_contact_email ON booking(contact_email);
CREATE INDEX idx_booking_data ON booking USING GIN (booking_data);
```

### Payments

```sql
CREATE TABLE payment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_id      UUID NOT NULL REFERENCES booking(id),
    operator_id     UUID NOT NULL REFERENCES operator(id),
    amount          NUMERIC(12, 2) NOT NULL,
    currency_code   CHAR(3) NOT NULL,
    payment_type    VARCHAR(20) NOT NULL,  -- deposit, balance, refund
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',

    -- Gateway-specific data varies by provider (Stripe, Adyen, PayPal)
    gateway_data    JSONB NOT NULL DEFAULT '{}',
    /*
    Example gateway_data JSONB (Stripe):
    {
      "gateway": "stripe",
      "paymentMethod": "card",
      "paymentMethodToken": "pm_1Nk9DEF2eZvKYlo2",
      "paymentIntentId": "pi_3Nk9ABC2eZvKYlo2C0Tf4321",
      "cardBrand": "visa",
      "cardLast4": "4242",
      "receiptUrl": "https://pay.stripe.com/receipts/..."
    }

    Example gateway_data JSONB (bank transfer):
    {
      "gateway": "manual",
      "paymentMethod": "bank_transfer",
      "bankReference": "FP-2026-0501-001",
      "receivedBy": "finance_team"
    }
    */

    -- Segregated fund tracking (ATOL/ABTA compliance)
    fund_data       JSONB,
    /*
    Example fund_data JSONB:
    {
      "segregated": true,
      "entryType": "deposit_in",
      "runningBalance": 12450.00,
      "trustAccountRef": "TRUST-001"
    }
    */

    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_payment_booking ON payment(booking_id);
CREATE INDEX idx_payment_operator ON payment(operator_id);
CREATE INDEX idx_payment_status ON payment(status);
```

### Itineraries

```sql
CREATE TABLE itinerary (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operator_id     UUID NOT NULL REFERENCES operator(id),
    product_id      UUID REFERENCES product(id),
    booking_id      UUID REFERENCES booking(id),
    name            VARCHAR(255) NOT NULL,
    total_days      INTEGER NOT NULL,
    start_date      DATE,
    end_date        DATE,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    version         INTEGER NOT NULL DEFAULT 1,

    -- Day-by-day structure stored as structured JSONB
    days            JSONB NOT NULL DEFAULT '[]',
    /*
    Example days JSONB:
    [
      {
        "dayNumber": 1,
        "title": "Arrival in Amsterdam",
        "destination": "Amsterdam",
        "countryCode": "NL",
        "accommodation": {"supplierId": "uuid", "name": "Hotel V Nesplein", "mealPlan": "breakfast"},
        "activities": [
          {
            "id": "uuid",
            "title": "Canal Cruise Welcome Tour",
            "type": "activity",
            "startTime": "14:00",
            "endTime": "16:00",
            "duration": "PT2H",
            "location": {"name": "Central Station Pier", "lat": 52.3791, "lng": 4.9003},
            "supplierId": "uuid",
            "cost": 25.00,
            "currency": "EUR",
            "notes": "Included in package"
          },
          {
            "id": "uuid",
            "title": "Welcome Dinner",
            "type": "meal",
            "startTime": "19:00",
            "location": {"name": "Restaurant De Kas"},
            "supplierId": "uuid",
            "cost": 65.00,
            "currency": "EUR"
          }
        ],
        "transfers": [
          {
            "type": "airport_pickup",
            "from": "Schiphol Airport",
            "to": "Hotel V Nesplein",
            "scheduledTime": "11:30",
            "supplierId": "uuid"
          }
        ],
        "notes": "Check-in from 15:00. Early luggage drop available."
      },
      {
        "dayNumber": 2,
        "title": "Cycling the Windmill Route",
        "destination": "Zaanse Schans",
        "countryCode": "NL",
        "accommodation": {"supplierId": "uuid", "name": "Hotel V Nesplein", "mealPlan": "half_board"},
        "activities": [
          {
            "id": "uuid",
            "title": "Guided Cycling Tour: Amsterdam to Zaanse Schans",
            "type": "tour",
            "startTime": "09:00",
            "endTime": "16:00",
            "duration": "PT7H",
            "guideId": "uuid",
            "productId": "uuid",
            "difficulty": "moderate",
            "distance": "45km"
          }
        ]
      }
    ]
    */

    -- AI adaptation history
    adaptation_log  JSONB NOT NULL DEFAULT '[]',
    /*
    Example adaptation_log JSONB:
    [
      {
        "timestamp": "2026-07-14T08:30:00Z",
        "reason": "weather_disruption",
        "severity": "amber",
        "dayNumber": 2,
        "removed": {"title": "Outdoor Cycling Tour"},
        "added": {"title": "Indoor Museum Visit + Afternoon Canal Cruise"},
        "approvedBy": "ai_auto",
        "guestNotified": true
      }
    ]
    */

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_itinerary_operator ON itinerary(operator_id);
CREATE INDEX idx_itinerary_product ON itinerary(product_id);
CREATE INDEX idx_itinerary_booking ON itinerary(booking_id);
CREATE INDEX idx_itinerary_status ON itinerary(status);
CREATE INDEX idx_itinerary_days ON itinerary USING GIN (days);
```

### Suppliers & Guides

```sql
CREATE TABLE supplier (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operator_id     UUID NOT NULL REFERENCES operator(id),
    name            VARCHAR(255) NOT NULL,
    supplier_type   VARCHAR(30) NOT NULL,
    contact_email   VARCHAR(255),
    contact_phone   VARCHAR(50),
    country_code    CHAR(2),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,

    -- Variable supplier details
    details         JSONB NOT NULL DEFAULT '{}',
    /*
    Example details JSONB:
    {
      "contactName": "Johan van der Berg",
      "address": "Keizersgracht 123, 1015 Amsterdam",
      "website": "https://www.example-hotel.nl",
      "paymentTerms": "net_30",
      "commissionPct": 15.0,
      "bankDetails": {"iban": "NL91ABNA0417164300", "bic": "ABNANL2A"},
      "categories": ["accommodation", "restaurant"],
      "starRating": 4,
      "roomTypes": ["standard_double", "deluxe_suite", "family_room"],
      "amenities": ["wifi", "pool", "gym", "restaurant"]
    }
    */

    -- Performance metrics (updated periodically)
    performance     JSONB NOT NULL DEFAULT '{}',
    /*
    Example performance JSONB:
    {
      "totalBookings": 142,
      "cancellationRate": 0.03,
      "avgLeadTimeDays": 14.2,
      "avgGuestRating": 4.6,
      "totalRevenue": 45200.00,
      "lastBookingDate": "2026-05-18",
      "isPreferred": true,
      "flags": []
    }
    */

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_supplier_operator ON supplier(operator_id);
CREATE INDEX idx_supplier_type ON supplier(supplier_type);
CREATE INDEX idx_supplier_performance ON supplier USING GIN (performance);

CREATE TABLE guide (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operator_id     UUID NOT NULL REFERENCES operator(id),
    user_id         UUID REFERENCES platform_user(id),
    full_name       VARCHAR(255) NOT NULL,
    email           VARCHAR(255),
    phone           VARCHAR(50),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,

    -- Variable guide profile
    profile         JSONB NOT NULL DEFAULT '{}',
    /*
    Example profile JSONB:
    {
      "bio": "Expert wildlife guide with 15 years experience in East Africa...",
      "photoUrl": "https://...",
      "languages": [
        {"code": "en", "proficiency": "native"},
        {"code": "sw", "proficiency": "fluent"},
        {"code": "de", "proficiency": "conversational"}
      ],
      "certifications": ["first_aid", "wilderness_rescue", "bronze_cross"],
      "specialisations": ["wildlife", "photography", "birding"],
      "dailyRate": 200.00,
      "currency": "USD",
      "rating": 4.8,
      "totalAssignments": 234,
      "availability": {
        "defaultDays": [1, 2, 3, 4, 5],
        "blockedDates": ["2026-08-10", "2026-08-11"]
      }
    }
    */

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_guide_operator ON guide(operator_id);
CREATE INDEX idx_guide_profile ON guide USING GIN (profile);
```

### Guests

```sql
CREATE TABLE guest (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operator_id     UUID NOT NULL REFERENCES operator(id),
    email           VARCHAR(255),
    full_name       VARCHAR(255) NOT NULL,
    phone           VARCHAR(50),

    -- Consent and privacy (relational for compliance queries)
    consent_marketing BOOLEAN NOT NULL DEFAULT FALSE,
    consent_data_processing BOOLEAN NOT NULL DEFAULT TRUE,
    data_retention_until DATE,

    -- Variable guest profile and preferences
    profile         JSONB NOT NULL DEFAULT '{}',
    /*
    Example profile JSONB:
    {
      "nationality": "GB",
      "dateOfBirth": "1988-03-15",
      "preferredLanguage": "en",
      "dietaryRequirements": "vegetarian",
      "medicalNotes": "Mild asthma - carries inhaler",
      "passportNumber": "ENCRYPTED:aes256:...",
      "preferences": {
        "accommodationType": "boutique_hotel",
        "activityLevel": "moderate",
        "interests": ["history", "food", "wine"],
        "groupSizePreference": "small"
      },
      "bookingHistory": {
        "totalBookings": 3,
        "totalSpend": 2450.00,
        "lastTravelDate": "2025-09-20",
        "destinations": ["Italy", "France", "Netherlands"],
        "averageRating": 4.7
      }
    }
    */

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_guest_operator ON guest(operator_id);
CREATE INDEX idx_guest_email ON guest(operator_id, email);
CREATE INDEX idx_guest_profile ON guest USING GIN (profile);

-- Links guests to bookings
CREATE TABLE booking_guest (
    booking_id      UUID NOT NULL REFERENCES booking(id) ON DELETE CASCADE,
    guest_id        UUID NOT NULL REFERENCES guest(id),
    is_lead         BOOLEAN NOT NULL DEFAULT FALSE,
    PRIMARY KEY (booking_id, guest_id)
);
```

### Distribution Channels

```sql
CREATE TABLE distribution_channel (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operator_id     UUID NOT NULL REFERENCES operator(id),
    name            VARCHAR(100) NOT NULL,
    channel_type    VARCHAR(30) NOT NULL,  -- ota, direct, agent, api
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,

    -- Channel-specific integration configuration
    integration_config JSONB NOT NULL DEFAULT '{}',
    /*
    Example integration_config JSONB (OCTO channel):
    {
      "protocol": "octo",
      "apiEndpoint": "https://api.viator.com/octo/v1",
      "apiKey": "ENCRYPTED:aes256:...",
      "capabilities": ["content", "pricing", "pickups"],
      "commissionPct": 20.0,
      "mappings": {
        "product-uuid-1": {"externalId": "VIATOR-12345", "syncEnabled": true, "lastSynced": "2026-05-20T12:00:00Z"},
        "product-uuid-2": {"externalId": "VIATOR-67890", "syncEnabled": true, "lastSynced": "2026-05-20T12:00:00Z"}
      }
    }

    Example integration_config JSONB (OTA XML channel):
    {
      "protocol": "ota_xml",
      "wsdlEndpoint": "https://api.wholesaler.com/ota/v2",
      "credentials": {"username": "ENCRYPTED:...", "password": "ENCRYPTED:..."},
      "messageTypes": ["OTA_TourActivityAvailRQ", "OTA_TourActivityBookRQ"],
      "fieldMappings": {"productId": "TourCode", "optionId": "DepartureRef"}
    }
    */

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_channel_operator ON distribution_channel(operator_id);
```

### Audit & Communications

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operator_id     UUID NOT NULL,
    user_id         UUID,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(20) NOT NULL,
    changes         JSONB NOT NULL,  -- {field: {old: ..., new: ...}}
    metadata        JSONB NOT NULL DEFAULT '{}',  -- ip_address, user_agent, etc.
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_operator ON audit_log(operator_id, created_at);

CREATE TABLE communication (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operator_id     UUID NOT NULL REFERENCES operator(id),
    booking_id      UUID REFERENCES booking(id),
    guest_id        UUID REFERENCES guest(id),
    channel         VARCHAR(20) NOT NULL,  -- email, sms, whatsapp
    direction       VARCHAR(10) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',

    -- Message content and delivery details
    message_data    JSONB NOT NULL DEFAULT '{}',
    /*
    Example message_data JSONB:
    {
      "templateName": "booking_confirmation",
      "subject": "Your Booking OP-2026-001234 is Confirmed",
      "body": "Dear Jane, ...",
      "recipientEmail": "jane@example.com",
      "sendGridMessageId": "msg_abc123",
      "openedAt": "2026-05-01T14:22:00Z",
      "clickedLinks": ["itinerary_pdf", "contact_us"]
    }
    */

    sent_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_comm_booking ON communication(booking_id);
CREATE INDEX idx_comm_operator ON communication(operator_id, created_at);
```

---

## Key Query Patterns

### Querying JSONB: Find Products with Sustainability Certifications

```sql
-- Find all products with Green Globe certification
SELECT id, name, destination, sustainability
FROM product
WHERE operator_id = 'operator-uuid'
  AND sustainability @> '{"certifications": ["Green Globe"]}';
```

### Querying JSONB: Guest Preference Search for AI Personalisation

```sql
-- Find guests interested in "food" and "wine" who have visited Italy
SELECT id, full_name, email, profile
FROM guest
WHERE operator_id = 'operator-uuid'
  AND profile @> '{"preferences": {"interests": ["food"]}}'
  AND profile #>> '{bookingHistory,destinations}' LIKE '%Italy%';
```

### Querying JSONB: Itinerary Day Activities

```sql
-- Extract all activities from day 2 of an itinerary
SELECT
    day_elem->>'title' AS day_title,
    activity->>'title' AS activity_title,
    activity->>'startTime' AS start_time,
    activity->>'type' AS activity_type
FROM itinerary,
     jsonb_array_elements(days) AS day_elem,
     jsonb_array_elements(day_elem->'activities') AS activity
WHERE id = 'itinerary-uuid'
  AND (day_elem->>'dayNumber')::int = 2;
```

### Querying JSONB: Supplier Performance Filtering

```sql
-- Find preferred suppliers with cancellation rate below 5%
SELECT id, name, supplier_type, performance
FROM supplier
WHERE operator_id = 'operator-uuid'
  AND (performance->>'isPreferred')::boolean = true
  AND (performance->>'cancellationRate')::numeric < 0.05
ORDER BY (performance->>'avgGuestRating')::numeric DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Identity | 2 | operator, platform_user |
| Product Catalogue | 2 | product, product_option |
| Availability | 1 | availability |
| Bookings | 1 | booking (unit items, custom fields in JSONB) |
| Payments | 1 | payment (gateway details, fund tracking in JSONB) |
| Itineraries | 1 | itinerary (days, activities, adaptation log in JSONB) |
| Suppliers & Guides | 2 | supplier, guide (profiles, performance in JSONB) |
| Guests | 2 | guest, booking_guest |
| Distribution | 1 | distribution_channel (integration config in JSONB) |
| Audit & Communications | 2 | audit_log, communication |
| **Total** | **15** | Significantly fewer tables than normalised model; complexity moves to JSONB structures |

---

## Key Design Decisions

1. **Relational for the queryable/reportable, JSONB for the variable**: Financial amounts, status fields, dates, and foreign keys are relational columns with proper types and indexes. Jurisdiction-specific fields, custom booking questions, supplier-specific metadata, and integration configurations live in JSONB.

2. **Itinerary days as a JSONB array rather than separate tables**: A multi-day itinerary with activities, transfers, and accommodation is stored as a single JSONB document. This makes reading and rendering an itinerary a single-row fetch rather than a multi-join query, which is the dominant access pattern for guest-facing apps and itinerary PDFs.

3. **GIN indexes on JSONB columns for containment queries**: PostgreSQL GIN indexes support the `@>` (contains) operator, enabling efficient queries like "find products where sustainability certifications include Green Globe" without full table scans.

4. **Operator settings JSONB absorbs jurisdiction-specific configuration**: Instead of separate tables for compliance profiles, notification preferences, and branding, the `operator.settings` JSONB holds all of these. New jurisdictions or compliance regimes can be added without schema changes.

5. **Product option embeds units and pricing in JSONB**: Since units and pricing are always read together with the option, and the structure varies by product type, embedding them in `config JSONB` reduces joins and allows flexible unit definitions per option.

6. **Adaptation log as append-only JSONB array**: The itinerary's `adaptation_log` records every AI-driven change as an array element. This gives temporal visibility without a separate event store, while keeping the itinerary as a self-contained document.

7. **Channel integration config as JSONB**: Different channel protocols (OCTO, OTA XML, custom API) require different configuration fields. Rather than a union of nullable columns or a key-value table, JSONB stores protocol-specific configuration naturally.

8. **Encrypted sensitive fields within JSONB**: Passport numbers and API credentials in JSONB are stored as `ENCRYPTED:aes256:...` strings, with application-layer encryption/decryption. This allows sensitive data to live alongside non-sensitive data in the same JSONB column.

9. **JSON Schema validation at the application layer**: Each JSONB column has a corresponding JSON Schema definition enforced by the API layer before writes. This compensates for the lack of database-level type enforcement on JSONB content.

10. **Fewer migrations, faster iteration**: With 15 tables instead of 35+, the schema is simpler to manage. New fields are added to JSONB structures with backward-compatible defaults, avoiding migration downtime during rapid product iteration.
