# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Tour Operator Platform · Created: 2026-05-22

## Philosophy

This model combines PostgreSQL relational tables for transactional CRUD (bookings, payments, availability) with a property graph layer for the inherently relationship-heavy parts of the tour operator domain: itinerary routing, supplier networks, guide-to-destination matching, guest preference graphs, and destination connectivity. The graph layer can be implemented either as PostgreSQL tables (`graph_node` / `graph_edge`) with recursive CTEs, or via Apache AGE (a PostgreSQL extension that adds Cypher query support), or as a separate Neo4j instance for teams that prefer a native graph database.

Tour operations are fundamentally about connecting people (guests, guides) to places (destinations, venues) via paths (routes, transfers, activities) constrained by time (availability windows, seasons) and logistics (capacity, transport). This is a graph problem. A normalised relational model handles individual bookings well but struggles with queries like "find all 5-day itinerary routes from Amsterdam to Rome that avoid cities the guest has already visited, use preferred suppliers, and have a guide who speaks Italian." A graph traversal answers this in milliseconds.

The design is inspired by Neo4j's documented travel itinerary patterns and by property graph approaches used in airline route planning (where airports are nodes and flights are edges). The relational side handles the transactional booking lifecycle, payments, and OCTO API compatibility. The graph side handles route planning, recommendation, supplier relationship analysis, and AI-driven itinerary generation.

**Best for:** Operators offering complex multi-destination tours where itinerary routing, supplier relationship analysis, and AI-powered recommendation are core competitive advantages.

**Trade-offs:**
- Pro: Natural representation of multi-destination routes, supplier networks, and guest-destination relationships
- Pro: Itinerary generation queries (pathfinding, constraint-based routing) are orders of magnitude faster than relational joins
- Pro: Supplier relationship intelligence -- "which suppliers are connected to which destinations?" -- is a direct graph traversal
- Pro: Guest personalisation -- "what destinations/activities are similar to what this guest has enjoyed?" -- maps to collaborative filtering on the graph
- Pro: Guide matching becomes a graph query: find guides connected to the required languages, specialisations, and destinations
- Con: Increased operational complexity -- two query paradigms (SQL + Cypher/recursive CTE) for one system
- Con: Graph query performance requires careful index management and query tuning
- Con: Development team needs graph thinking skills, which are less common than SQL skills
- Con: If using a separate Neo4j instance, data synchronisation between PostgreSQL and Neo4j adds infrastructure burden
- Con: ACID transactions across relational and graph stores require distributed transaction patterns or eventual consistency

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OCTO v1 | Booking lifecycle (Product/Option/Availability/Booking) is fully relational, OCTO-compatible; graph layer is for routing and recommendation, not transactional booking |
| ISO 3166-1 / ISO 3166-2 | Destination nodes carry ISO country and subdivision codes as properties |
| ISO 4217 | Monetary values in relational tables use ISO 4217 currency codes |
| ISO 8601 | Temporal properties on edges (season windows, availability periods) use ISO 8601 |
| RFC 4122 | UUID identifiers shared between relational and graph layers for cross-referencing |
| UN/LOCODE | Destination nodes can carry UN/LOCODE identifiers for transport hub standardisation |
| Schema.org | Graph node types map to Schema.org vocabulary: Destination -> TouristDestination, Activity -> TouristAttraction |
| GSTC / UNWTO | Sustainability certifications as properties on destination and supplier nodes, enabling graph queries like "find eco-certified routes" |

---

## Relational Layer (Transactional CRUD)

### Booking Lifecycle (OCTO-compatible)

```sql
CREATE TABLE operator (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    country_code    CHAR(2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE platform_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    password_hash   VARCHAR(255),
    full_name       VARCHAR(255) NOT NULL,
    operator_id     UUID REFERENCES operator(id),
    role            VARCHAR(30) NOT NULL DEFAULT 'agent',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE product (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operator_id     UUID NOT NULL REFERENCES operator(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    product_type    VARCHAR(30) NOT NULL,
    duration_days   INTEGER,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    graph_node_id   UUID,  -- cross-reference to graph layer
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_product_operator ON product(operator_id);

CREATE TABLE product_option (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES product(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE availability (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES product(id),
    option_id       UUID NOT NULL REFERENCES product_option(id),
    local_date      DATE NOT NULL,
    start_time      TIME,
    total_capacity  INTEGER NOT NULL,
    remaining_capacity INTEGER NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'available',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_avail_product_date ON availability(product_id, local_date);

CREATE TABLE booking (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operator_id     UUID NOT NULL REFERENCES operator(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    option_id       UUID NOT NULL REFERENCES product_option(id),
    availability_id UUID REFERENCES availability(id),
    booking_number  VARCHAR(20) NOT NULL UNIQUE,
    channel_id      UUID,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    contact_name    VARCHAR(255),
    contact_email   VARCHAR(255),
    contact_phone   VARCHAR(50),
    total_amount    NUMERIC(12, 2) NOT NULL,
    amount_paid     NUMERIC(12, 2) NOT NULL DEFAULT 0,
    currency_code   CHAR(3) NOT NULL,
    travel_date     DATE NOT NULL,
    booking_data    JSONB NOT NULL DEFAULT '{}',
    booked_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    confirmed_at    TIMESTAMPTZ,
    cancelled_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_booking_operator ON booking(operator_id);
CREATE INDEX idx_booking_status ON booking(status);
CREATE INDEX idx_booking_travel_date ON booking(travel_date);

CREATE TABLE payment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_id      UUID NOT NULL REFERENCES booking(id),
    operator_id     UUID NOT NULL REFERENCES operator(id),
    amount          NUMERIC(12, 2) NOT NULL,
    currency_code   CHAR(3) NOT NULL,
    payment_type    VARCHAR(20) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    gateway_data    JSONB NOT NULL DEFAULT '{}',
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_payment_booking ON payment(booking_id);

CREATE TABLE guest (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operator_id     UUID NOT NULL REFERENCES operator(id),
    email           VARCHAR(255),
    full_name       VARCHAR(255) NOT NULL,
    graph_node_id   UUID,  -- cross-reference to graph layer
    profile         JSONB NOT NULL DEFAULT '{}',
    consent_marketing BOOLEAN NOT NULL DEFAULT FALSE,
    consent_data_processing BOOLEAN NOT NULL DEFAULT TRUE,
    data_retention_until DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_guest_operator ON guest(operator_id);
CREATE INDEX idx_guest_email ON guest(operator_id, email);

CREATE TABLE booking_guest (
    booking_id      UUID NOT NULL REFERENCES booking(id) ON DELETE CASCADE,
    guest_id        UUID NOT NULL REFERENCES guest(id),
    is_lead         BOOLEAN NOT NULL DEFAULT FALSE,
    PRIMARY KEY (booking_id, guest_id)
);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operator_id     UUID NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(20) NOT NULL,
    changes         JSONB NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_operator ON audit_log(operator_id, created_at);
```

---

## Graph Layer (PostgreSQL Implementation)

The graph is implemented using two PostgreSQL tables -- `graph_node` and `graph_edge` -- with JSONB properties. This keeps everything in one database and avoids cross-store synchronisation. For teams wanting native graph syntax, Apache AGE can be layered on top of these same tables.

### Graph Schema

```sql
-- Graph nodes represent entities that participate in relationships
CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operator_id     UUID NOT NULL REFERENCES operator(id),
    node_type       VARCHAR(30) NOT NULL,
    -- Node types:
    --   destination    (a city, region, or specific venue)
    --   activity       (a bookable activity at a destination)
    --   accommodation  (a hotel, lodge, camp)
    --   supplier       (a company providing services)
    --   guide          (a tour guide)
    --   guest          (a traveller)
    --   transport_hub  (airport, train station, port)
    --   tag            (interest tag: "wildlife", "history", "food")

    label           VARCHAR(255) NOT NULL,  -- human-readable label
    external_id     UUID,                   -- FK to relational table (product.id, guest.id, etc.)
    properties      JSONB NOT NULL DEFAULT '{}',
    /*
    Example properties for a destination node:
    {
      "countryCode": "IT",
      "subdivisionCode": "IT-VE",
      "city": "Venice",
      "latitude": 45.4408,
      "longitude": 12.3155,
      "unLocode": "ITVCE",
      "timezone": "Europe/Rome",
      "peakSeason": ["Jun", "Jul", "Aug", "Sep"],
      "sustainabilityCertified": true,
      "avgDailyBudget": {"amount": 180, "currency": "EUR"}
    }

    Example properties for a guide node:
    {
      "languages": ["en", "it", "de"],
      "specialisations": ["history", "art", "food"],
      "dailyRate": 250.00,
      "currency": "EUR",
      "rating": 4.9,
      "totalTours": 412,
      "basedIn": "Venice"
    }

    Example properties for an activity node:
    {
      "activityType": "tour",
      "duration": "PT3H",
      "difficulty": "easy",
      "minAge": 6,
      "maxParticipants": 12,
      "retailPrice": 65.00,
      "currency": "EUR",
      "sustainability": ["walking_only", "local_guide"],
      "schemaOrgType": "TouristAttraction"
    }
    */

    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_node_operator ON graph_node(operator_id);
CREATE INDEX idx_node_type ON graph_node(node_type);
CREATE INDEX idx_node_external ON graph_node(external_id);
CREATE INDEX idx_node_label ON graph_node(operator_id, label);
CREATE INDEX idx_node_properties ON graph_node USING GIN (properties);

-- Graph edges represent typed relationships between nodes
CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operator_id     UUID NOT NULL REFERENCES operator(id),
    source_node_id  UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    edge_type       VARCHAR(50) NOT NULL,
    -- Edge types:
    --   CONNECTS_TO        (destination -> destination: travel route)
    --   HAS_ACTIVITY       (destination -> activity)
    --   HAS_ACCOMMODATION  (destination -> accommodation)
    --   SUPPLIED_BY        (activity/accommodation -> supplier)
    --   GUIDES_AT          (guide -> destination)
    --   SPEAKS             (guide -> language tag)
    --   SPECIALISES_IN     (guide -> interest tag)
    --   VISITED            (guest -> destination)
    --   BOOKED             (guest -> activity)
    --   RATED              (guest -> activity/accommodation: with rating)
    --   INTERESTED_IN      (guest -> interest tag)
    --   SIMILAR_TO         (activity -> activity: computed similarity)
    --   SERVED_BY          (transport_hub -> destination)
    --   TRANSFERS_TO       (transport_hub -> destination: with transport details)

    properties      JSONB NOT NULL DEFAULT '{}',
    /*
    Example properties for CONNECTS_TO edge (destination -> destination):
    {
      "distanceKm": 260,
      "travelTimeHours": 2.5,
      "transportModes": ["train", "bus", "car"],
      "preferredMode": "train",
      "trainOperator": "Trenitalia",
      "scenicRating": 4.5,
      "cost": {"amount": 45, "currency": "EUR", "mode": "train"},
      "frequency": "hourly",
      "seasonAvailable": ["all_year"]
    }

    Example properties for RATED edge (guest -> activity):
    {
      "rating": 5,
      "review": "Incredible walking tour of the historic centre",
      "travelDate": "2025-09-18",
      "bookingId": "uuid-of-booking",
      "wouldRecommend": true
    }

    Example properties for GUIDES_AT edge (guide -> destination):
    {
      "experienceYears": 8,
      "toursCompleted": 156,
      "avgRating": 4.8,
      "lastTourDate": "2026-05-15",
      "certified": true
    }
    */

    weight          DECIMAL(10, 4) DEFAULT 1.0,  -- for pathfinding algorithms
    valid_from      DATE,           -- seasonal availability
    valid_to        DATE,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_edge_operator ON graph_edge(operator_id);
CREATE INDEX idx_edge_source ON graph_edge(source_node_id);
CREATE INDEX idx_edge_target ON graph_edge(target_node_id);
CREATE INDEX idx_edge_type ON graph_edge(edge_type);
CREATE INDEX idx_edge_source_type ON graph_edge(source_node_id, edge_type);
CREATE INDEX idx_edge_target_type ON graph_edge(target_node_id, edge_type);
CREATE INDEX idx_edge_properties ON graph_edge USING GIN (properties);
```

---

## Key Graph Query Patterns

### 1. Multi-Destination Itinerary Route Planning

```sql
-- Find all 3-hop routes from Amsterdam to Rome via intermediate destinations
-- that have at least one activity and one accommodation
WITH RECURSIVE route AS (
    -- Start from Amsterdam
    SELECT
        n.id AS current_node,
        n.label AS current_city,
        ARRAY[n.id] AS path,
        ARRAY[n.label] AS city_names,
        0 AS hops,
        0::numeric AS total_distance,
        0::numeric AS total_travel_time
    FROM graph_node n
    WHERE n.operator_id = 'operator-uuid'
      AND n.node_type = 'destination'
      AND n.label = 'Amsterdam'

    UNION ALL

    -- Traverse CONNECTS_TO edges
    SELECT
        e.target_node_id,
        tn.label,
        r.path || tn.id,
        r.city_names || tn.label,
        r.hops + 1,
        r.total_distance + COALESCE((e.properties->>'distanceKm')::numeric, 0),
        r.total_travel_time + COALESCE((e.properties->>'travelTimeHours')::numeric, 0)
    FROM route r
    JOIN graph_edge e ON e.source_node_id = r.current_node
        AND e.edge_type = 'CONNECTS_TO'
        AND e.is_active = TRUE
    JOIN graph_node tn ON tn.id = e.target_node_id
    WHERE r.hops < 4
      AND tn.id != ALL(r.path)  -- no cycles
)
SELECT
    city_names,
    hops AS stops,
    total_distance AS distance_km,
    total_travel_time AS travel_hours
FROM route
WHERE current_node = (
    SELECT id FROM graph_node
    WHERE operator_id = 'operator-uuid'
      AND node_type = 'destination'
      AND label = 'Rome'
)
ORDER BY total_travel_time;
```

### 2. AI Guide Matching via Graph Traversal

```sql
-- Find guides who: speak Italian, specialise in "history" or "art",
-- have guided at "Florence", and have rating >= 4.5
SELECT
    g.id AS guide_node_id,
    g.label AS guide_name,
    g.properties->>'rating' AS rating,
    g.properties->>'dailyRate' AS daily_rate,
    lang_edge.properties AS language_details,
    dest_edge.properties AS destination_experience
FROM graph_node g
-- Guide speaks Italian
JOIN graph_edge lang_edge ON lang_edge.source_node_id = g.id
    AND lang_edge.edge_type = 'SPEAKS'
JOIN graph_node lang_node ON lang_node.id = lang_edge.target_node_id
    AND lang_node.label = 'Italian'
-- Guide specialises in history or art
JOIN graph_edge spec_edge ON spec_edge.source_node_id = g.id
    AND spec_edge.edge_type = 'SPECIALISES_IN'
JOIN graph_node spec_node ON spec_node.id = spec_edge.target_node_id
    AND spec_node.label IN ('history', 'art')
-- Guide has guided at Florence
JOIN graph_edge dest_edge ON dest_edge.source_node_id = g.id
    AND dest_edge.edge_type = 'GUIDES_AT'
JOIN graph_node dest_node ON dest_node.id = dest_edge.target_node_id
    AND dest_node.label = 'Florence'
WHERE g.node_type = 'guide'
  AND g.operator_id = 'operator-uuid'
  AND (g.properties->>'rating')::numeric >= 4.5
ORDER BY (g.properties->>'rating')::numeric DESC;
```

### 3. Guest Personalisation: Recommend Activities Based on Past Preferences

```sql
-- Find activities similar to what this guest has rated highly,
-- at destinations they haven't visited yet
WITH guest_preferences AS (
    -- Activities this guest rated 4+
    SELECT e.target_node_id AS liked_activity_id
    FROM graph_edge e
    WHERE e.source_node_id = 'guest-node-uuid'
      AND e.edge_type = 'RATED'
      AND (e.properties->>'rating')::int >= 4
),
visited_destinations AS (
    -- Destinations this guest has already visited
    SELECT e.target_node_id AS destination_id
    FROM graph_edge e
    WHERE e.source_node_id = 'guest-node-uuid'
      AND e.edge_type = 'VISITED'
),
similar_activities AS (
    -- Activities similar to liked ones (via SIMILAR_TO edges)
    SELECT
        e.target_node_id AS activity_id,
        e.weight AS similarity_score
    FROM graph_edge e
    JOIN guest_preferences gp ON gp.liked_activity_id = e.source_node_id
    WHERE e.edge_type = 'SIMILAR_TO'
      AND e.weight >= 0.7  -- similarity threshold
)
SELECT
    a.label AS activity_name,
    a.properties->>'activityType' AS type,
    a.properties->>'retailPrice' AS price,
    d.label AS destination,
    sa.similarity_score
FROM similar_activities sa
JOIN graph_node a ON a.id = sa.activity_id
-- Find the destination this activity belongs to
JOIN graph_edge da ON da.target_node_id = a.id AND da.edge_type = 'HAS_ACTIVITY'
JOIN graph_node d ON d.id = da.source_node_id AND d.node_type = 'destination'
-- Exclude already-visited destinations
WHERE d.id NOT IN (SELECT destination_id FROM visited_destinations)
ORDER BY sa.similarity_score DESC
LIMIT 10;
```

### 4. Supplier Network Analysis

```sql
-- Find suppliers who serve multiple destinations, ranked by coverage
SELECT
    s.label AS supplier_name,
    s.properties->>'supplierType' AS type,
    COUNT(DISTINCT dest.id) AS destinations_served,
    array_agg(DISTINCT dest.label) AS destination_list,
    AVG((rating_edges.properties->>'rating')::numeric) AS avg_guest_rating
FROM graph_node s
-- Supplier provides activities or accommodation
JOIN graph_edge supply_edge ON supply_edge.target_node_id = s.id
    AND supply_edge.edge_type = 'SUPPLIED_BY'
JOIN graph_node item ON item.id = supply_edge.source_node_id
-- Item is at a destination
JOIN graph_edge dest_edge ON dest_edge.target_node_id = item.id
    AND dest_edge.edge_type IN ('HAS_ACTIVITY', 'HAS_ACCOMMODATION')
JOIN graph_node dest ON dest.id = dest_edge.source_node_id
    AND dest.node_type = 'destination'
-- Guest ratings of supplier's items
LEFT JOIN graph_edge rating_edges ON rating_edges.target_node_id = item.id
    AND rating_edges.edge_type = 'RATED'
WHERE s.node_type = 'supplier'
  AND s.operator_id = 'operator-uuid'
GROUP BY s.id, s.label, s.properties
HAVING COUNT(DISTINCT dest.id) >= 2
ORDER BY destinations_served DESC, avg_guest_rating DESC;
```

### 5. Eco-Certified Route Finding

```sql
-- Find routes where every destination and supplier is sustainability-certified
WITH RECURSIVE eco_route AS (
    SELECT
        n.id AS current_node,
        n.label AS current_city,
        ARRAY[n.id] AS path,
        ARRAY[n.label] AS city_names,
        0 AS hops
    FROM graph_node n
    WHERE n.operator_id = 'operator-uuid'
      AND n.node_type = 'destination'
      AND n.label = 'Amsterdam'
      AND (n.properties->>'sustainabilityCertified')::boolean = TRUE

    UNION ALL

    SELECT
        e.target_node_id,
        tn.label,
        r.path || tn.id,
        r.city_names || tn.label,
        r.hops + 1
    FROM eco_route r
    JOIN graph_edge e ON e.source_node_id = r.current_node
        AND e.edge_type = 'CONNECTS_TO'
        AND e.is_active = TRUE
    JOIN graph_node tn ON tn.id = e.target_node_id
        AND tn.node_type = 'destination'
        AND (tn.properties->>'sustainabilityCertified')::boolean = TRUE
    WHERE r.hops < 5
      AND tn.id != ALL(r.path)
)
SELECT city_names, hops
FROM eco_route
WHERE hops >= 2
ORDER BY hops;
```

---

## Itinerary Storage (Relational + Graph Reference)

```sql
-- Itineraries reference the graph for routing but store operational data relationally
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
    -- The graph route this itinerary follows (array of destination node IDs)
    route_node_ids  UUID[] NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_itinerary_operator ON itinerary(operator_id);
CREATE INDEX idx_itinerary_booking ON itinerary(booking_id);

CREATE TABLE itinerary_day (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    itinerary_id    UUID NOT NULL REFERENCES itinerary(id) ON DELETE CASCADE,
    day_number      INTEGER NOT NULL,
    destination_node_id UUID REFERENCES graph_node(id),  -- links to graph destination
    title           VARCHAR(255),
    accommodation_node_id UUID REFERENCES graph_node(id),
    meal_plan       VARCHAR(30),
    guide_node_id   UUID REFERENCES graph_node(id),  -- links to graph guide
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_itin_day ON itinerary_day(itinerary_id);

CREATE TABLE itinerary_activity (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    itinerary_day_id UUID NOT NULL REFERENCES itinerary_day(id) ON DELETE CASCADE,
    activity_node_id UUID REFERENCES graph_node(id),  -- links to graph activity
    title           VARCHAR(255) NOT NULL,
    start_time      TIME,
    end_time        TIME,
    activity_type   VARCHAR(30),
    cost            NUMERIC(12, 2),
    currency_code   CHAR(3),
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_itin_activity ON itinerary_activity(itinerary_day_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Identity | 2 | operator, platform_user |
| Product Catalogue | 2 | product, product_option |
| Availability | 1 | availability |
| Bookings | 1 | booking |
| Payments | 1 | payment |
| Guests | 2 | guest, booking_guest |
| Itinerary | 3 | itinerary, itinerary_day, itinerary_activity |
| Graph Layer | 2 | graph_node, graph_edge |
| Audit | 1 | audit_log |
| **Total** | **15** | Plus graph nodes/edges which represent destinations, activities, suppliers, guides, guests, tags |

---

## Key Design Decisions

1. **Graph layer in PostgreSQL, not a separate database**: Using `graph_node` / `graph_edge` tables keeps everything in one PostgreSQL instance, avoiding cross-store synchronisation complexity. Apache AGE can be added later for Cypher syntax if the team prefers it, without changing the underlying storage.

2. **Relational for transactions, graph for relationships**: Bookings, payments, and availability are relational because they need ACID transactions, foreign keys, and index-based lookups. Routes, supplier networks, and guest preferences are graph because they need traversal, pathfinding, and relationship-based queries.

3. **Cross-reference via `graph_node_id` columns**: Relational tables (product, guest, guide) carry an optional `graph_node_id` that links to their corresponding graph node. This enables joining relational data with graph query results without duplicating data.

4. **Edge types as a vocabulary**: The fixed set of edge types (CONNECTS_TO, HAS_ACTIVITY, SUPPLIED_BY, GUIDES_AT, VISITED, RATED, SIMILAR_TO, etc.) creates a controlled vocabulary for relationships. New relationship types can be added without schema changes.

5. **Weight column for pathfinding**: The `graph_edge.weight` column enables weighted pathfinding algorithms (Dijkstra, A*) implemented via recursive CTEs. Weights can represent distance, travel time, cost, or a composite score.

6. **Seasonal validity on edges**: `valid_from` / `valid_to` on edges allow seasonal routes, activities, and supplier availability to be modeled as time-bounded relationships. Queries filter edges by the travel date window.

7. **SIMILAR_TO edges for recommendation**: Pre-computed similarity edges between activities enable collaborative filtering recommendations without real-time computation. These edges are recalculated periodically by a background job or ML pipeline.

8. **Itinerary as a graph route + relational detail**: The `itinerary.route_node_ids` array captures the sequence of destination nodes the itinerary follows. Individual days link back to destination, accommodation, and guide nodes. This means itinerary generation can be a graph pathfinding query, with the result materialised into relational day/activity rows for operational use.

9. **Supplier network analysis as a first-class capability**: By modelling suppliers as nodes connected to activities and destinations via edges, supplier relationship intelligence (coverage, performance, pricing) becomes a set of graph aggregation queries rather than complex multi-join SQL.

10. **Guest preference graph enables AI personalisation**: Guest nodes connected to destinations (VISITED), activities (RATED), and interest tags (INTERESTED_IN) create a rich preference graph. AI recommendation models can traverse this graph to suggest personalised itineraries based on similar guests' patterns (collaborative filtering) or content similarity (content-based filtering).
