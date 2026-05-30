# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Tour Operator Platform · Created: 2026-05-22

## Philosophy

This model treats every state change as an immutable event. Instead of mutating rows in-place, the system appends events to an append-only event store -- `BookingCreated`, `BookingConfirmed`, `ItineraryDayAdded`, `GuideAssigned`, `PaymentReceived`, `DisruptionDetected`, `ItineraryAdapted`. The current state of any entity is derived by replaying its event stream. Read-optimised projections (materialised views / read models) serve the booking engine, dashboard, and API layer via CQRS (Command Query Responsibility Segregation).

This approach is inspired by financial ledger systems and is used in production by event-driven booking platforms. The EU Package Travel Directive, ATOL bonding, and IATA BSP settlement all demand complete audit trails -- event sourcing delivers this by construction rather than as an afterthought. Every change is recorded with who, when, and why.

For a tour operator platform, event sourcing unlocks three powerful capabilities that normalised CRUD cannot match: (1) temporal queries -- "show me the itinerary as it was on 15 March before the weather disruption"; (2) full auditability -- regulators can trace every booking lifecycle change; (3) AI training data -- the complete history of itinerary adaptations, pricing changes, and guest interactions becomes a rich dataset for ML models to learn from.

**Best for:** Operators in heavily regulated markets (EU Package Travel, ATOL) who need complete audit trails, temporal queries, and plan to build AI features on historical booking and itinerary data.

**Trade-offs:**
- Pro: Complete audit trail by construction -- every state change is recorded with actor, timestamp, and payload
- Pro: Temporal queries are trivial -- replay to any point in time to see historical state
- Pro: Rich training data for AI models (itinerary adaptation patterns, pricing decisions, guest behaviour)
- Pro: Natural fit for real-time disruption handling -- disruptions are events that trigger adaptation events
- Con: Increased storage requirements -- every event is stored permanently, not just current state
- Con: Read model consistency is eventually consistent (typically milliseconds, but not guaranteed)
- Con: Event schema evolution requires careful versioning (upcasting old events to new shapes)
- Con: More complex to implement and reason about than traditional CRUD
- Con: Debugging requires understanding event replay, not just inspecting rows

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OCTO v1 | Read models (projections) expose OCTO-compatible Product/Option/Availability/Booking shapes; events map to OCTO booking lifecycle states |
| ISO 4217 | All monetary event payloads include `currencyCode` as ISO 4217 three-letter code |
| ISO 3166-1 / ISO 3166-2 | Jurisdiction and destination fields in events use ISO country/subdivision codes |
| ISO 8601 | All event timestamps in UTC ISO 8601 format; duration fields use ISO 8601 duration notation |
| RFC 4122 | UUID v7 for event IDs (time-ordered for efficient indexing) and aggregate IDs |
| EU Package Travel Directive | Complete event history satisfies pre-contractual information disclosure and change-tracking requirements |
| ATOL / ABTA | Fund segregation events create an immutable ledger that produces bonding-scheme audit reports |
| PCI DSS v4.0 | Payment events store only tokenised references; no raw card data enters the event store |
| GDPR | Crypto-shredding pattern: guest PII is encrypted with a per-guest key; right-to-erasure deletes the key, rendering PII in events unreadable |

---

## Event Store (Core Infrastructure)

```sql
-- The immutable event store -- source of truth for the entire system
CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),  -- UUID v7 (time-ordered)
    aggregate_type  VARCHAR(50) NOT NULL,  -- booking, product, itinerary, guide, supplier, payment, guest
    aggregate_id    UUID NOT NULL,         -- the entity this event belongs to
    operator_id     UUID NOT NULL,         -- tenant isolation
    event_type      VARCHAR(100) NOT NULL, -- e.g. BookingCreated, ItineraryDayAdded, GuideAssigned
    event_version   INTEGER NOT NULL,      -- version of this event's schema (for upcasting)
    sequence_number BIGINT NOT NULL,       -- per-aggregate ordering
    payload         JSONB NOT NULL,        -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',  -- actor_id, ip_address, user_agent, correlation_id, causation_id
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(aggregate_id, sequence_number)
);

-- Primary query path: load all events for an aggregate in order
CREATE INDEX idx_event_aggregate ON event_store(aggregate_id, sequence_number);

-- Tenant-scoped queries
CREATE INDEX idx_event_operator ON event_store(operator_id, created_at);

-- Event type queries for projections and analytics
CREATE INDEX idx_event_type ON event_store(event_type, created_at);

-- Time-range queries for audit and reporting
CREATE INDEX idx_event_created ON event_store(created_at);

-- Snapshot store for performance (avoid replaying entire event history)
CREATE TABLE aggregate_snapshot (
    aggregate_id    UUID NOT NULL,
    aggregate_type  VARCHAR(50) NOT NULL,
    sequence_number BIGINT NOT NULL,       -- sequence number this snapshot was taken at
    state           JSONB NOT NULL,        -- serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_id, sequence_number)
);
```

### Event Payload Examples

```jsonc
// BookingCreated
{
  "bookingNumber": "OP-2026-001234",
  "productId": "550e8400-e29b-41d4-a716-446655440001",
  "optionId": "550e8400-e29b-41d4-a716-446655440002",
  "availabilityId": "550e8400-e29b-41d4-a716-446655440003",
  "travelDate": "2026-07-15",
  "contactName": "Jane Smith",
  "contactEmail": "jane@example.com",
  "contactPhone": "+44 7700 900123",
  "unitItems": [
    {"unitTypeId": "adult-uuid", "fullName": "Jane Smith", "unitPrice": 149.00},
    {"unitTypeId": "child-uuid", "fullName": "Tom Smith", "age": 8, "unitPrice": 99.00}
  ],
  "totalAmount": 248.00,
  "currencyCode": "GBP",
  "channelId": "viator-uuid",
  "resellerReference": "V-98765"
}

// ItineraryAdapted (AI disruption response)
{
  "itineraryId": "550e8400-e29b-41d4-a716-446655440010",
  "dayNumber": 3,
  "reason": "weather_disruption",
  "disruption": {
    "type": "severe_weather",
    "source": "met_office_api",
    "severity": "amber",
    "description": "Heavy rainfall warning for Lake District"
  },
  "removedActivity": {
    "activityId": "uuid-outdoor-hike",
    "title": "Helvellyn Summit Hike"
  },
  "addedActivity": {
    "supplierId": "uuid-indoor-supplier",
    "title": "Lakeland Whisky Distillery Tour",
    "startTime": "10:00",
    "endTime": "12:30",
    "cost": 35.00,
    "currencyCode": "GBP"
  },
  "guestNotified": true,
  "adaptedBy": "ai_disruption_engine"
}

// GuideAssigned
{
  "guideId": "550e8400-e29b-41d4-a716-446655440020",
  "bookingId": "550e8400-e29b-41d4-a716-446655440030",
  "itineraryDayId": "550e8400-e29b-41d4-a716-446655440040",
  "assignmentDate": "2026-07-17",
  "matchScore": 0.92,
  "matchFactors": {
    "languageMatch": true,
    "specialisationMatch": ["wildlife", "photography"],
    "availabilityConfirmed": true,
    "previousGuestRating": 4.8
  }
}

// PaymentReceived
{
  "bookingId": "550e8400-e29b-41d4-a716-446655440030",
  "paymentType": "deposit",
  "amount": 100.00,
  "currencyCode": "GBP",
  "paymentMethod": "card",
  "gateway": "stripe",
  "gatewayReference": "pi_3Nk9ABC2eZvKYlo2C0Tf4321",
  "paymentMethodToken": "pm_1Nk9DEF2eZvKYlo2",
  "segregatedFund": true
}
```

---

## Read Model Projections (Materialised Views)

These tables are rebuilt from the event store. They can be dropped and rebuilt at any time.

### Booking Read Model

```sql
-- Current state of bookings (projected from events)
CREATE TABLE rm_booking (
    id              UUID PRIMARY KEY,
    operator_id     UUID NOT NULL,
    booking_number  VARCHAR(20) NOT NULL,
    product_id      UUID NOT NULL,
    product_name    VARCHAR(255),
    option_id       UUID NOT NULL,
    option_name     VARCHAR(100),
    availability_id UUID,
    channel_name    VARCHAR(100),
    reseller_reference VARCHAR(100),
    status          VARCHAR(20) NOT NULL,  -- pending, confirmed, cancelled, completed, no_show
    contact_name    VARCHAR(255),
    contact_email   VARCHAR(255),
    contact_phone   VARCHAR(50),
    travel_date     DATE NOT NULL,
    total_amount    NUMERIC(12, 2) NOT NULL,
    amount_paid     NUMERIC(12, 2) NOT NULL DEFAULT 0,
    currency_code   CHAR(3) NOT NULL,
    guest_count     INTEGER NOT NULL DEFAULT 0,
    notes           TEXT,
    booked_at       TIMESTAMPTZ NOT NULL,
    confirmed_at    TIMESTAMPTZ,
    cancelled_at    TIMESTAMPTZ,
    last_event_at   TIMESTAMPTZ NOT NULL,
    last_event_seq  BIGINT NOT NULL
);
CREATE INDEX idx_rm_booking_operator ON rm_booking(operator_id);
CREATE INDEX idx_rm_booking_status ON rm_booking(status);
CREATE INDEX idx_rm_booking_travel_date ON rm_booking(travel_date);
CREATE INDEX idx_rm_booking_number ON rm_booking(booking_number);

-- Booking participants (projected)
CREATE TABLE rm_booking_participant (
    id              UUID PRIMARY KEY,
    booking_id      UUID NOT NULL REFERENCES rm_booking(id),
    unit_type       VARCHAR(50) NOT NULL,
    full_name       VARCHAR(255),
    age             INTEGER,
    unit_price      NUMERIC(12, 2) NOT NULL,
    dietary_requirements TEXT,
    special_requests TEXT
);
CREATE INDEX idx_rm_participant_booking ON rm_booking_participant(booking_id);
```

### Product Catalogue Read Model

```sql
CREATE TABLE rm_product (
    id              UUID PRIMARY KEY,
    operator_id     UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(150),
    description     TEXT,
    product_type    VARCHAR(30) NOT NULL,
    duration_days   INTEGER,
    destination     VARCHAR(255),
    country_code    CHAR(2),
    status          VARCHAR(20) NOT NULL,
    min_price       NUMERIC(12, 2),  -- denormalised: cheapest adult unit price
    currency_code   CHAR(3),
    rating_avg      DECIMAL(3, 2),
    review_count    INTEGER DEFAULT 0,
    booking_count   INTEGER DEFAULT 0,
    last_event_at   TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_rm_product_operator ON rm_product(operator_id);
CREATE INDEX idx_rm_product_status ON rm_product(status);
CREATE INDEX idx_rm_product_type ON rm_product(product_type);
```

### Itinerary Read Model

```sql
CREATE TABLE rm_itinerary (
    id              UUID PRIMARY KEY,
    operator_id     UUID NOT NULL,
    product_id      UUID,
    booking_id      UUID,
    name            VARCHAR(255) NOT NULL,
    total_days      INTEGER NOT NULL,
    start_date      DATE,
    end_date        DATE,
    status          VARCHAR(20) NOT NULL,
    version         INTEGER NOT NULL,
    adaptation_count INTEGER DEFAULT 0,  -- how many times AI adapted this itinerary
    last_event_at   TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_rm_itinerary_operator ON rm_itinerary(operator_id);
CREATE INDEX idx_rm_itinerary_booking ON rm_itinerary(booking_id);

CREATE TABLE rm_itinerary_day (
    id              UUID PRIMARY KEY,
    itinerary_id    UUID NOT NULL REFERENCES rm_itinerary(id),
    day_number      INTEGER NOT NULL,
    title           VARCHAR(255),
    destination     VARCHAR(255),
    accommodation   VARCHAR(255),
    meal_plan       VARCHAR(30),
    guide_name      VARCHAR(255),
    guide_id        UUID,
    activities      JSONB NOT NULL DEFAULT '[]'  -- denormalised array of activities for fast rendering
    -- Example: [{"title": "Morning Hike", "startTime": "09:00", "location": "Trail Head", "type": "activity"}, ...]
);
CREATE INDEX idx_rm_itin_day ON rm_itinerary_day(itinerary_id);
```

### Availability Read Model

```sql
CREATE TABLE rm_availability (
    id              UUID PRIMARY KEY,
    product_id      UUID NOT NULL,
    option_id       UUID NOT NULL,
    operator_id     UUID NOT NULL,
    local_date      DATE NOT NULL,
    start_time      TIME,
    total_capacity  INTEGER NOT NULL,
    remaining_capacity INTEGER NOT NULL,
    status          VARCHAR(20) NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_rm_avail_product_date ON rm_availability(product_id, local_date);
CREATE INDEX idx_rm_avail_operator ON rm_availability(operator_id, local_date);
```

### Financial Read Model

```sql
-- Payment summary per booking (projected)
CREATE TABLE rm_payment_summary (
    booking_id      UUID PRIMARY KEY,
    operator_id     UUID NOT NULL,
    total_charged   NUMERIC(12, 2) NOT NULL DEFAULT 0,
    total_refunded  NUMERIC(12, 2) NOT NULL DEFAULT 0,
    net_received    NUMERIC(12, 2) NOT NULL DEFAULT 0,
    currency_code   CHAR(3) NOT NULL,
    payment_count   INTEGER NOT NULL DEFAULT 0,
    last_payment_at TIMESTAMPTZ,
    last_event_at   TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_rm_payment_operator ON rm_payment_summary(operator_id);

-- Segregated fund balance (projected from fund events)
CREATE TABLE rm_fund_balance (
    operator_id     UUID PRIMARY KEY,
    currency_code   CHAR(3) NOT NULL,
    total_deposits  NUMERIC(14, 2) NOT NULL DEFAULT 0,
    total_payouts   NUMERIC(14, 2) NOT NULL DEFAULT 0,
    current_balance NUMERIC(14, 2) NOT NULL DEFAULT 0,
    last_event_at   TIMESTAMPTZ NOT NULL
);
```

### Supplier Performance Read Model

```sql
CREATE TABLE rm_supplier_scorecard (
    supplier_id     UUID PRIMARY KEY,
    operator_id     UUID NOT NULL,
    supplier_name   VARCHAR(255) NOT NULL,
    supplier_type   VARCHAR(30) NOT NULL,
    total_bookings  INTEGER NOT NULL DEFAULT 0,
    cancellation_count INTEGER NOT NULL DEFAULT 0,
    cancellation_rate DECIMAL(5, 4) DEFAULT 0,
    avg_lead_time_days DECIMAL(5, 1),
    avg_guest_rating DECIMAL(3, 2),
    total_revenue   NUMERIC(14, 2) DEFAULT 0,
    currency_code   CHAR(3),
    is_preferred    BOOLEAN NOT NULL DEFAULT FALSE,
    last_event_at   TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_rm_supplier_operator ON rm_supplier_scorecard(operator_id);
```

### Guide Performance Read Model

```sql
CREATE TABLE rm_guide_scorecard (
    guide_id        UUID PRIMARY KEY,
    operator_id     UUID NOT NULL,
    guide_name      VARCHAR(255) NOT NULL,
    languages       VARCHAR(10)[] DEFAULT '{}',
    specialisations TEXT[] DEFAULT '{}',
    total_assignments INTEGER NOT NULL DEFAULT 0,
    completed_assignments INTEGER NOT NULL DEFAULT 0,
    declined_count  INTEGER NOT NULL DEFAULT 0,
    avg_guest_rating DECIMAL(3, 2),
    total_days_worked INTEGER NOT NULL DEFAULT 0,
    last_assignment_date DATE,
    last_event_at   TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_rm_guide_operator ON rm_guide_scorecard(operator_id);
```

---

## Reference Data (Shared, Not Event-Sourced)

```sql
-- These are static reference tables, not event-sourced
CREATE TABLE ref_country (
    code            CHAR(2) PRIMARY KEY,
    name            VARCHAR(100) NOT NULL,
    alpha3          CHAR(3) NOT NULL
);

CREATE TABLE ref_currency (
    code            CHAR(3) PRIMARY KEY,
    name            VARCHAR(100) NOT NULL,
    minor_units     SMALLINT NOT NULL DEFAULT 2
);

CREATE TABLE ref_language (
    code            VARCHAR(10) PRIMARY KEY,
    name            VARCHAR(100) NOT NULL
);

-- Operator table is also maintained as a read model but kept simple for auth
CREATE TABLE ref_operator (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    country_code    CHAR(2) NOT NULL,
    currency_code   CHAR(3) NOT NULL,
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC'
);

-- Platform users for authentication (not event-sourced -- managed by auth service)
CREATE TABLE ref_platform_user (
    id              UUID PRIMARY KEY,
    email           VARCHAR(255) NOT NULL UNIQUE,
    password_hash   VARCHAR(255),
    full_name       VARCHAR(255) NOT NULL,
    operator_id     UUID REFERENCES ref_operator(id),
    role            VARCHAR(30) NOT NULL DEFAULT 'agent'
);
```

---

## Key Query Patterns

### Temporal Query: Itinerary State at a Point in Time

```sql
-- Rebuild itinerary state as of 2026-07-14 10:00 UTC (before disruption)
SELECT payload
FROM event_store
WHERE aggregate_type = 'itinerary'
  AND aggregate_id = '550e8400-e29b-41d4-a716-446655440010'
  AND created_at <= '2026-07-14T10:00:00Z'
ORDER BY sequence_number;
-- Application replays these events to reconstruct the state
```

### Audit Query: Complete Booking Lifecycle

```sql
-- Full audit trail for a booking
SELECT event_type, payload, metadata, created_at
FROM event_store
WHERE aggregate_type = 'booking'
  AND aggregate_id = '550e8400-e29b-41d4-a716-446655440030'
ORDER BY sequence_number;
-- Returns: BookingCreated -> PaymentReceived -> BookingConfirmed -> GuideAssigned -> ...
```

### Analytics Query: AI Disruption Adaptation Patterns

```sql
-- All disruption adaptations in the last 90 days for ML training
SELECT
    payload->>'reason' AS disruption_reason,
    payload->'disruption'->>'type' AS disruption_type,
    payload->'disruption'->>'severity' AS severity,
    payload->'removedActivity'->>'title' AS removed,
    payload->'addedActivity'->>'title' AS replacement,
    created_at
FROM event_store
WHERE event_type = 'ItineraryAdapted'
  AND operator_id = 'operator-uuid'
  AND created_at >= now() - INTERVAL '90 days'
ORDER BY created_at;
```

### Fund Segregation Audit Report

```sql
-- Reconstruct fund movements for bonding scheme audit
SELECT
    event_type,
    payload->>'amount' AS amount,
    payload->>'entryType' AS entry_type,
    payload->>'bookingId' AS booking_id,
    payload->>'runningBalance' AS balance,
    created_at
FROM event_store
WHERE event_type IN ('FundDeposited', 'FundPayoutSent', 'FundRefunded', 'CommissionReleased')
  AND operator_id = 'operator-uuid'
  AND created_at BETWEEN '2026-01-01' AND '2026-12-31'
ORDER BY created_at;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | event_store, aggregate_snapshot |
| Booking Read Models | 2 | rm_booking, rm_booking_participant |
| Product Read Models | 1 | rm_product |
| Itinerary Read Models | 2 | rm_itinerary, rm_itinerary_day |
| Availability Read Models | 1 | rm_availability |
| Financial Read Models | 2 | rm_payment_summary, rm_fund_balance |
| Performance Read Models | 2 | rm_supplier_scorecard, rm_guide_scorecard |
| Reference Data | 5 | ref_country, ref_currency, ref_language, ref_operator, ref_platform_user |
| **Total** | **17** | Read models are disposable projections; the event_store is the source of truth |

---

## Key Design Decisions

1. **Single event_store table as source of truth**: All domain events across all aggregate types flow into one partitioned table. This simplifies backup, replication, and cross-aggregate analytics. The table can be partitioned by `created_at` for time-range pruning.

2. **JSONB payloads with versioned schemas**: Event payloads are JSONB with an `event_version` integer. When the shape of an event changes, old events are upcasted at read time rather than migrated. This avoids destructive schema migrations.

3. **Aggregate snapshots for performance**: Instead of replaying thousands of events for a long-lived aggregate (e.g., a supplier with 5,000 bookings), periodic snapshots capture the current state. Replay starts from the latest snapshot.

4. **Read models are disposable**: Every `rm_*` table can be dropped and rebuilt from the event store. This means schema changes to read models are non-destructive -- add a column, rebuild the projection, and the new column is populated from historical events.

5. **Crypto-shredding for GDPR compliance**: Guest PII in event payloads is encrypted with a per-guest encryption key stored in a separate key table. Right-to-erasure deletes the key, making the PII in historical events cryptographically unrecoverable without modifying the immutable event store.

6. **Correlation and causation IDs in metadata**: Every event carries a `correlation_id` (tying together events from the same user action) and `causation_id` (the event that triggered this event). This enables tracing chains like: `DisruptionDetected` -> `ItineraryAdapted` -> `GuestNotified`.

7. **Natural fit for AI disruption handling**: The event model naturally captures the disruption-adaptation cycle. The AI engine emits `DisruptionDetected` and `ItineraryAdapted` events with full context, creating a labelled training dataset for improving future adaptation decisions.

8. **Read models optimised for different consumers**: The booking read model is optimised for the booking engine and OCTO API. The supplier scorecard is optimised for the supplier intelligence dashboard. Each projection denormalises exactly what its consumer needs.

9. **Segregated fund events create an immutable financial ledger**: Fund movements are events, not mutable rows. This produces an append-only ledger that bonding scheme auditors can verify end-to-end without concern about retroactive edits.

10. **Event store enables event-driven integrations**: Channel managers, notification services, and analytics pipelines can subscribe to the event stream (via CDC / logical replication) rather than polling read models, enabling real-time reactions to booking and itinerary changes.
