# SkyBooking OTA Engine

Software Architecture Document

## Table of Contents

1. [Overview](#1-overview)
2. [Vision & Mission](#2-vision--mission)
3. [Actors of the System](#3-actors-of-the-system)
4. [Functional Requirements (FRs)](#4-functional-requirements-frs)
5. [Non-Functional Requirements (NFRs)](#5-non-functional-requirements-nfrs)
6. [Entity Relationship Diagram (ERD)](#6-entity-relationship-diagram-erd)
7. [Core User Stories](#7-core-user-stories)
8. [System Flow: Search Scatter-Gather](#8-system-flow-search-scatter-gather)
9. [Sequence Diagram: Flight Booking Pipeline](#9-sequence-diagram-flight-booking-pipeline)
10. [System Assumptions](#10-system-assumptions)
11. [Tech Stack](#11-tech-stack)
12. [Project Structure](#12-project-structure)
13. [Applied Design Patterns](#13-applied-design-patterns)
14. [Setup & Installation Guide](#14-setup--installation-guide)
15. [API Documentation](#15-api-documentation)
16. [Feature Flows & Pseudocode](#16-feature-flows--pseudocode)
17. [Testing Suite Setup](#17-testing-suite-setup)

## 1. Overview

SkyBooking Engine is a high-concurrency Online Travel Agency (OTA) and travel search aggregator. It queries third-party travel providers (e.g. Amadeus, Duffel) in parallel, caches normalized results, and facilitates flight and hotel reservations with payment processing.

## 2. Vision & Mission

- **Vision:** Deliver a zero-friction, ultra-low latency travel search and booking experience for flights and hotels worldwide.
- **Mission:** Provide a resilient, scalable micro-modular system capable of aggregating, filtering, and executing real-time bookings across disparate legacy and modern travel APIs.

## 3. Actors of the System

| Actor | Description |
|---|---|
| Guest User | Unauthenticated user searching for flights/hotels and viewing public rates. |
| Registered Customer | Authenticated user with saved profile data, booking history, and active reservations. |
| Admin / Operations User | Internal manager auditing transactions, customer accounts, and daily revenue reporting. |
| Support Agent | Internal user handling customer inquiries, complaints, refund/reschedule assistance, before and after booking. |
| External Provider (System Actor) | External GDS / API integrations (e.g. Amadeus, Duffel, Stripe). |

## 4. Functional Requirements (FRs)

- **Auth & Profiles:** User registration, OAuth2 social login (Google/Keycloak), JWT token management, and profile updates.
- **Multi-Provider Search:** Parallel (Scatter-Gather) execution querying flight and hotel APIs concurrently across multiple providers (Amadeus, Duffel, Tequila, etc).
- **AI Natural-Language Search:** User types free-text wish (e.g. "cheap flight to Dubai next weekend, direct only") instead of filling search form. LLM parses intent into structured search params (origin, destination, dates, filters), then runs normal scatter-gather search.
- **Fixed-Date Price Search:** User picks exact travel date, gets price comparison across all matching flights from all providers, picks the suitable one.
- **Whole-Month Price View:** Skyscanner-style calendar showing cheapest price per day for an entire month, for a given route/airline, so user can pick the cheapest day to fly.
- **Filtering & Sorting:** Filter/sort results returned from providers by price, cabin class, stops, ratings, and departure times.
- **Separate Entity Bookings:** Distinct booking creation pipelines and tables for flights (`flight_bookings`) and hotels (`hotel_bookings`), each storing the provider offer snapshot alongside the booking.
- **Payment Integration:** Multiple payment methods (card, wallet, bank transfer) via multiple payment providers/gateways (e.g. Stripe, PayMob), webhook verification, and receipt generation.
- **Refunds:** Cancel a paid booking and trigger refund back through the original payment provider, with status tracking (requested/processing/completed).
- **Post-Payment Change:** Change flight date/time on an already-paid booking — reprice against provider, charge/refund the difference, reissue confirmation.
- **Notifications:** Asynchronous email/SMS booking receipts and promotional offer alerts.
- **Reporting:** System-wide dashboard metrics tracking daily/monthly booking volume and cancellations.
- **Customer Support:** Pre-booking (search/pricing questions) and post-booking (refund status, reschedule help, complaints) support via ticketing, live chat, or agent-assisted channel, linked to the customer's booking history.
- **Favorites:** Registered customer saves flights, hotels, or price watches (route + target price) for quick access later; price-watch favorites can trigger a notification when the price drops.
- **Auditing & Logging:** Every state-changing action (booking, payment, refund, reschedule, admin/support action) is written to an immutable audit log (who, what, when, before/after values); structured request/error logs shipped to a central log store for debugging and monitoring.

## 5. Non-Functional Requirements (NFRs)

- **Performance:** Scatter-gather aggregator search response under 2,500ms (enforced hard cutoff time).
- **Scalability:** Horizontal scaling for high read-to-write ratio during search queries.
- **Security:** PCI-DSS compliant payment processing (tokenized via external providers), JWT authentication, rate limiting.
- **Availability:** 99.9% uptime with circuit breakers on third-party API adapters.
- **Auditability & Observability:** All financial and admin/support actions traceable via audit log; structured, correlation-ID-tagged logs across services for tracing a request end-to-end.

## 6. Entity Relationship Diagram (ERD)

Approach: separate `FLIGHT_BOOKINGS` and `HOTEL_BOOKINGS` tables, no common base table. Each table holds the booking itself *and* a snapshot of the provider's offer details (airline/flight or hotel/room info) directly as columns — there's no separate `FLIGHTS`/`HOTELS` catalog table to join through. The provider snapshot is captured at booking time (from the live search/Redis result), so the booking row is self-contained and immune to the provider later changing or removing that offer.

`PAYMENT_PROVIDERS` holds per-provider config (Stripe, PayMob, etc) — credentials are stored as vault/secret-manager references inside `config`, never raw keys in the DB. `PAYMENTS` is the one logical payment per booking (aggregate status, which provider), linked polymorphically via `booking_type` + `booking_id` since a payment can belong to either booking table. `TRANSACTIONS` is the detail ledger under a payment — one row per attempt/event (authorize, capture, refund, retry), each carrying the provider's own transaction id and raw response for reconciliation. A refund is a new transaction row against the existing payment, not a new payment.

```mermaid
erDiagram
    USERS ||--o{ FLIGHT_BOOKINGS : places
    USERS ||--o{ HOTEL_BOOKINGS : places
    USERS ||--o{ NOTIFICATIONS : receives
    USERS ||--o{ FAVORITES : saves
    USERS ||--o{ AUDIT_LOGS : "acts (actor)"
    FLIGHT_BOOKINGS ||--o| PAYMENTS : "paid via"
    HOTEL_BOOKINGS ||--o| PAYMENTS : "paid via"
    PAYMENT_PROVIDERS ||--o{ PAYMENTS : processes
    PAYMENTS ||--o{ TRANSACTIONS : has
    PAYMENT_PROVIDERS ||--o{ TRANSACTIONS : via

    USERS {
        int id PK
        string email
        string password_hash
        datetime created_at
    }

    FLIGHT_BOOKINGS {
        int id PK
        int user_id FK
        string provider
        string provider_offer_id
        string airline_code
        string flight_number
        string departure_airport
        string arrival_airport
        datetime departure_time
        datetime arrival_time
        string cabin_class
        string seat_number
        decimal total_amount
        string currency
        string status
        datetime created_at
    }

    HOTEL_BOOKINGS {
        int id PK
        int user_id FK
        string provider
        string provider_offer_id
        string hotel_name
        string room_type
        int stars
        boolean free_cancellation
        date check_in_date
        date check_out_date
        decimal total_amount
        string currency
        string status
        datetime created_at
    }

    PAYMENTS {
        int id PK
        string booking_type
        int booking_id
        int provider_id FK
        decimal amount
        string currency
        string status
        datetime created_at
    }

    PAYMENT_PROVIDERS {
        int id PK
        string name
        string environment
        boolean is_active
        json config
        datetime created_at
    }

    TRANSACTIONS {
        int id PK
        int payment_id FK
        int provider_id FK
        string type
        string method
        decimal amount
        string currency
        string status
        string provider_transaction_id
        json raw_response
        datetime created_at
    }

    NOTIFICATIONS {
        int id PK
        int user_id FK
        string channel
        string recipient
        string payload
    }

    FAVORITES {
        int id PK
        int user_id FK
        string type
        string provider_offer_id
        decimal target_price
        json snapshot
    }

    AUDIT_LOGS {
        int id PK
        int actor_user_id FK
        string action
        string entity_type
        string entity_id
        string before_value
        string after_value
        string ip_address
        datetime created_at
    }
```

## 7. Core User Stories

| ID | As a... | I want to... | So that... |
|---|---|---|---|
| US-01 | Guest | Search flights by origin, destination, and date | I can find available flights across suppliers. |
| US-02 | Customer | Book a flight and pay via credit card | I can secure my seat and get an instant confirmation receipt. |
| US-03 | Customer | View my separate flight and hotel booking histories | I can manage my travel itineraries. |
| US-04 | Admin | View daily/monthly cancellation and sales reports | I can analyze system metrics. |
| US-05 | Guest | See the cheapest price per day across a whole month for a route | I can pick the cheapest day to fly. |
| US-06 | Customer | Filter and sort flight/hotel results by price, stops, rating, etc | I can narrow results to what suits me. |
| US-07 | Customer | Pay using different payment methods and providers | I can use whichever option is convenient. |
| US-08 | Customer | Request a refund on a paid booking | I get my money back if I cancel. |
| US-09 | Customer | Change the date/time of a flight after payment | I can adjust my travel plans without rebooking from scratch. |
| US-10 | Customer | Contact support before or after booking | I can get help with questions, refunds, or rescheduling. |
| US-11 | Support Agent | View a customer's booking, payment, and refund history | I can resolve their inquiry without asking them to repeat details. |
| US-12 | Guest | Type what I want in plain language instead of filling a search form | I get matching flights without learning form fields/filters. |
| US-13 | Customer | Save a flight, hotel, or price watch to favorites | I can find it again without re-searching. |
| US-14 | Customer | Get notified when a favorited route's price drops | I can book at the best time. |
| US-15 | Admin | View an audit trail of who changed/refunded/rescheduled a booking and when | I can investigate disputes and detect abuse. |

## 8. System Flow: Search Scatter-Gather

```mermaid
flowchart TD
    A[Client: Request Search /search/flights] --> B{Check Redis Cache?}
    B -->|Yes| C[Return Cached Data]
    B -->|No| D[Fan-out Parallel Async Requests<br/>Amadeus, Duffel, Tequila]
    D --> E[Wait for Results OR 2.5s Timeout]
    E --> F[Deduplicate & Normalize]
    F --> G[Save to Redis & Return Response<br/>id = provider_offer_id, not a DB row]
```

## 9. Sequence Diagram: Flight Booking Pipeline

```mermaid
sequenceDiagram
    participant U as User/Next.js
    participant A as NestJS API
    participant P as Payment Provider
    participant D as PostgreSQL

    U->>A: Book Flight (Payload)
    A->>P: Initiate Tx
    P-->>A: Tx Secret
    A-->>U: Render Payment
    U->>P: Complete Payment
    P-->>A: Webhook Success
    A->>D: Save Booking Data
    Note over D: Create Record in bookings (type=flight)
    A-->>U: Booking Confirm
```

## 10. System Assumptions

- Suppliers return standardized ISO currency rates, or require local client-side handling.
- Payment holds are finalized synchronously before confirming supplier bookings.
- Redis is deployed with persistence enabled to survive restarts during cached search spikes.
- Search results are not persisted to a DB catalog table — only Redis-cached until a booking is made. On booking, the full provider offer snapshot (airline/flight or hotel/room fields) is copied directly into the `flight_bookings`/`hotel_bookings` row.
- Provider price/availability is re-validated at booking time (Redis-cached offers can be stale relative to the live provider).

## 11. Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Next.js (React, TypeScript), TanStack Query (React Query), TailwindCSS, Axios |
| Backend | NestJS (TypeScript), @nestjs/axios, @nestjs/cache-manager |
| Database & ORM | PostgreSQL + Prisma ORM |
| Caching & Jobs | Redis Cluster (search results cache) + BullMQ for event-driven notification dispatching |
| Auth | Keycloak / Passport.js (JWT) |
| API Documentation | Swagger (@nestjs/swagger, OpenAPI) |
| Testing | Jest |
| AI Search | LLM (e.g. Claude/GPT) for parsing free-text queries into structured search params |
| Logging & Auditing | Pino/Winston (structured logs) + NestJS Interceptor for audit trail, shipped to ELK/Loki |

## 12. Project Structure

```
skybooking-backend/
├── src/
│   ├── modules/
│   │   ├── auth/             # User Auth & Keycloak Guards
│   │   ├── search/           # Aggregator & Adapters
│   │   │   ├── adapters/     # External API Integrations
│   │   │   └── search.service.ts
│   │   ├── ai-search/        # Free-Text Query -> Structured Search Params (LLM)
│   │   ├── flight-booking/   # Flight Bookings (booking + provider snapshot)
│   │   ├── hotel-booking/    # Hotel Bookings (booking + provider snapshot)
│   │   ├── payment/          # Payment Webhooks & Auditing
│   │   ├── notification/     # Queue Workers for Email/SMS
│   │   ├── support/          # Customer Support Tickets & Chat
│   │   ├── favorites/        # Saved Flights, Hotels & Price Watches
│   │   └── audit/            # Audit Log Writer & Query API
│   ├── common/               # Filters, Interceptors (Audit, Logging), Guards
│   └── main.ts
└── prisma/
    └── schema.prisma
```

## 13. Applied Design Patterns

- **Scatter-Gather Pattern:** Used in the Search module to fan-out queries to external APIs and aggregate the fastest responses.
- **Adapter Pattern:** Used to translate third-party supplier responses (Amadeus, Duffel) into a normalized internal JSON payload.
- **Repository Pattern:** Provided natively by Prisma ORM for database abstraction.
- **Factory / Module Pattern:** Applied via NestJS modular architecture.

## 14. Setup & Installation Guide

### Prerequisites

- Node.js v20+
- Docker & Docker Compose (PostgreSQL & Redis)

### Step 1: Clone & Install

```bash
git clone https://github.com/SaraMagdyAbdelhamed/booking_project_nodejs.git
cd booking_project_nodejs

# Install backend dependencies
cd skybooking-backend
npm install

# Install frontend dependencies
cd ../skybooking-frontend
npm install
```

### Step 2: Environment Setup (.env)

Create a `.env` file in `skybooking-backend`:

```
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/skybooking?schema=public"
REDIS_HOST="localhost"
REDIS_PORT=6379
JWT_SECRET="super-secret-key"
AMADEUS_CLIENT_ID="your-id"
AMADEUS_CLIENT_SECRET="your-secret"
```

### Step 3: Run Infrastructure & Database

```bash
# Start Postgres & Redis
docker-compose up -d

# Run Prisma Migrations
npx prisma migrate dev --name init
```

### Step 4: Run Applications

```bash
# Start NestJS Backend
npm run start:dev

# In another terminal, start Next.js Frontend
cd ../skybooking-frontend
npm run dev
```

## 15. API Documentation

Swagger UI auto-generated via `@nestjs/swagger`, served at `/api/docs` when the backend runs.

Setup in `main.ts`:

```typescript
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';

const config = new DocumentBuilder()
  .setTitle('SkyBooking API')
  .setDescription('Flight & hotel search, booking, and payment endpoints')
  .setVersion('1.0')
  .addBearerAuth()
  .build();

const document = SwaggerModule.createDocument(app, config);
SwaggerModule.setup('api/docs', app, document);
```

Endpoints below reflect what Swagger documents.

### 1. Search Flights

`GET /api/v1/search/flights`

Query params (mirrors the flights search bar — From/To, Depart/Return, travellers & cabin class, nearby airports, direct only, add a place to stay):

| Param | Example | Notes |
|---|---|---|
| origin | `CAI` | "From" — airport/city code |
| destination | `DXB` | "To" |
| departureDate | `2026-08-01` | "Depart" |
| returnDate | `2026-08-05` | "Return" — omit for one-way |
| passengers | `1` | Travellers count |
| cabinClass | `economy` | economy / premium / business / first |
| nearbyOrigin | `false` | Add nearby airports (origin side) |
| nearbyDestination | `false` | Add nearby airports (destination side) |
| directOnly | `false` | Direct flights only |
| includeStay | `false` | "Add a place to stay" — bundle a hotel search alongside |

Response:

```json
[
  {
    "id": "flg_982341",
    "airline": "MS",
    "flightNumber": "MS777",
    "departureAirport": "CAI",
    "arrivalAirport": "DXB",
    "departureTime": "2026-09-01T08:00:00Z",
    "arrivalTime": "2026-09-01T12:30:00Z",
    "price": 320.00,
    "currency": "USD",
    "provider": "Amadeus"
  }
]
```

### 2. Search Hotels

`GET /api/v1/search/hotels`

Query params (mirrors the stays search bar — destination, check-in/out, guests & rooms, free cancellation, star rating):

| Param | Example | Notes |
|---|---|---|
| destination | `Dubai` | City or hotel name |
| checkInDate | `2026-08-04` | |
| checkOutDate | `2026-08-05` | |
| adults | `2` | |
| rooms | `1` | |
| freeCancellation | `false` | Filter: free cancellation only |
| minStars | `4` | Filter: minimum star rating |

Response:

```json
[
  {
    "id": "htl_8821",
    "hotelName": "Grand Nile Tower",
    "stars": 5,
    "freeCancellation": true,
    "pricePerNight": 130.00,
    "currency": "USD",
    "provider": "Duffel"
  }
]
```

### 3. Create Flight Booking

`POST /api/v1/flight-bookings`

Body:

```json
{
  "userId": "usr_12345",
  "airlineCode": "MS",
  "flightNumber": "MS777",
  "departureAirport": "CAI",
  "arrivalAirport": "DXB",
  "departureTime": "2026-09-01T08:00:00Z",
  "arrivalTime": "2026-09-01T12:30:00Z",
  "seatNumber": "12A",
  "totalAmount": 320.00
}
```

### 4. Create Hotel Booking

`POST /api/v1/hotel-bookings`

Body:

```json
{
  "userId": "usr_12345",
  "hotelId": "htl_8821",
  "hotelName": "Grand Nile Tower",
  "checkInDate": "2026-09-01",
  "checkOutDate": "2026-09-05",
  "roomType": "Deluxe Suite",
  "totalAmount": 650.00
}
```

### 5. AI Natural-Language Search

`POST /api/v1/search/ai`

Body:

```json
{
  "query": "cheap flight to Dubai next weekend, direct only"
}
```

Response (parsed params + results, same shape as `/search/flights`):

```json
{
  "parsed": {
    "origin": "CAI",
    "destination": "DXB",
    "departureDate": "2026-08-01",
    "returnDate": "2026-08-03",
    "stops": 0,
    "sort": "price_asc"
  },
  "results": [
    {
      "id": "flg_982341",
      "airline": "MS",
      "flightNumber": "MS777",
      "price": 320.00,
      "currency": "USD",
      "provider": "Amadeus"
    }
  ]
}
```

### 6. Search Flights: Whole Month Price View

`GET /api/v1/search/flights/month`

Query params: `origin=CAI&destination=DXB&month=2026-09&airline=MS`

Response:

```json
[
  { "date": "2026-09-01", "lowestPrice": 320.00, "currency": "USD" },
  { "date": "2026-09-02", "lowestPrice": 298.50, "currency": "USD" }
]
```

### 7. Request Refund

`POST /api/v1/flight-bookings/{id}/refund`

Body:

```json
{
  "reason": "Change of plans"
}
```

Response:

```json
{
  "refundId": "rfd_55210",
  "bookingId": "flg_982341",
  "status": "processing",
  "amount": 320.00
}
```

### 8. Change Flight Date/Time (Post-Payment)

`PATCH /api/v1/flight-bookings/{id}/reschedule`

Body:

```json
{
  "newDepartureTime": "2026-09-05T08:00:00Z",
  "newArrivalTime": "2026-09-05T12:30:00Z"
}
```

Response:

```json
{
  "bookingId": "flg_982341",
  "status": "confirmed",
  "priceDifference": 45.00,
  "newDepartureTime": "2026-09-05T08:00:00Z"
}
```

### 9. Add Favorite

`POST /api/v1/favorites`

Body:

```json
{
  "userId": "usr_12345",
  "type": "flight",
  "providerOfferId": "flg_982341",
  "targetPrice": 300.00
}
```

Response:

```json
{
  "favoriteId": "fav_77120",
  "type": "flight",
  "providerOfferId": "flg_982341",
  "targetPrice": 300.00,
  "createdAt": "2026-07-28T10:00:00Z"
}
```

### 10. List Favorites

`GET /api/v1/favorites?userId=usr_12345`

Response:

```json
[
  { "favoriteId": "fav_77120", "type": "flight", "providerOfferId": "flg_982341", "targetPrice": 300.00 },
  { "favoriteId": "fav_77121", "type": "hotel", "providerOfferId": "htl_8821", "targetPrice": null }
]
```

### 11. Create Support Ticket

`POST /api/v1/support/tickets`

Body:

```json
{
  "userId": "usr_12345",
  "bookingId": "flg_982341",
  "subject": "Refund not received",
  "message": "Requested refund 5 days ago, no update yet."
}
```

Response:

```json
{
  "ticketId": "tkt_44210",
  "status": "open",
  "createdAt": "2026-07-28T10:00:00Z"
}
```

### 12. Query Audit Logs

`GET /api/v1/audit-logs?entityType=flight_booking&entityId=flg_982341`

Response:

```json
[
  {
    "id": "aud_1001",
    "actorUserId": "usr_12345",
    "action": "reschedule",
    "entityType": "flight_booking",
    "entityId": "flg_982341",
    "beforeValue": { "departureTime": "2026-09-01T08:00:00Z" },
    "afterValue": { "departureTime": "2026-09-05T08:00:00Z" },
    "ipAddress": "203.0.113.10",
    "createdAt": "2026-09-02T09:15:00Z"
  }
]
```

## 16. Feature Flows & Pseudocode

Flowchart, sequence diagram, and pseudocode per feature, in the same order as [§4 Functional Requirements](#4-functional-requirements-frs).

### 16.1 Auth & Profiles

```mermaid
flowchart TD
    A[POST /auth/register or /auth/login] --> B{Local or OAuth?}
    B -->|Local| C[Validate email/password]
    B -->|OAuth Google/Keycloak| D[Redirect to provider consent]
    D --> E[Provider callback with auth code]
    E --> F[Exchange code for provider profile]
    C --> G{User exists?}
    F --> G
    G -->|No| H[Create user record]
    G -->|Yes| I[Load user record]
    H --> J[Issue JWT access + refresh token]
    I --> J
    J --> K[Client stores token, sends on future requests]
```

```mermaid
sequenceDiagram
    participant U as User/Next.js
    participant A as NestJS API
    participant O as OAuth Provider (Google/Keycloak)
    participant D as PostgreSQL

    U->>A: GET /auth/oauth/google
    A-->>U: Redirect to Google consent screen
    U->>O: Approve access
    O-->>A: Callback with auth code
    A->>O: Exchange code for profile + tokens
    O-->>A: Google profile (email, name, sub)
    A->>D: Find or create user by email
    D-->>A: User record
    A-->>U: JWT access token + refresh token (httpOnly cookie)
```

```
function login(email, password):
    user = db.users.findByEmail(email)
    if !user or !bcrypt.compare(password, user.passwordHash):
        throw InvalidCredentialsError
    return issueTokens(user)

function oauthCallback(provider, code):
    profile = oauthClient(provider).exchangeCodeForProfile(code)  // Google/Keycloak
    user = db.users.findByEmail(profile.email)
    if !user:
        user = db.users.create({ email: profile.email, passwordHash: null, oauthProvider: provider })
    return issueTokens(user)

function issueTokens(user):
    accessToken = jwt.sign({ sub: user.id, email: user.email }, ttl='15m')
    refreshToken = jwt.sign({ sub: user.id }, ttl='30d')
    db.refreshTokens.store(user.id, refreshToken)
    return { accessToken, refreshToken }

// NestJS Guard on protected routes
function jwtAuthGuard(request):
    token = extractBearerToken(request)
    payload = jwt.verify(token)  // throws if expired/invalid
    request.user = db.users.find(payload.sub)
    return true

function updateProfile(userId, updates):
    audit.log('profile_update', { userId, before: db.users.find(userId), after: updates })
    return db.users.update(userId, updates)
```

### 16.2 Multi-Provider Search (Scatter-Gather)

See flowchart in [§8](#8-system-flow-search-scatter-gather).

```mermaid
sequenceDiagram
    participant U as User/Next.js
    participant A as NestJS API
    participant R as Redis
    participant Am as Amadeus
    participant Du as Duffel
    participant Te as Tequila

    U->>A: GET /search/flights?origin=CAI&destination=DXB&...
    A->>R: Check cache
    alt cache hit
        R-->>A: Cached results
    else cache miss
        par fan-out
            A->>Am: search(query)
            A->>Du: search(query)
            A->>Te: search(query)
        end
        Am-->>A: offers (or timeout/error, isolated)
        Du-->>A: offers
        Te-->>A: offers
        A->>A: Dedupe + normalize
        A->>R: Cache normalized results (ttl 5min)
    end
    A-->>U: Flight offers
```

```
function searchFlights(query):
    cacheKey = hash(query)
    cached = redis.get(cacheKey)
    if cached exists:
        return cached

    providers = [Amadeus, Duffel, Tequila]
    promises = []
    for provider in providers:
        promises.push(provider.search(query).catch(() => []))  // isolate provider failures

    results = awaitAllWithTimeout(promises, 2500ms)  // partial results on timeout
    merged = flatten(results)
    deduped = deduplicateByFlightSignature(merged)
    normalized = normalizeToInternalSchema(deduped)  // id = provider_offer_id, nothing written to DB yet

    redis.set(cacheKey, normalized, ttl=5min)
    return normalized
```

### 16.3 AI Natural-Language Search

```mermaid
flowchart TD
    A[User types free-text query] --> B[POST /search/ai]
    B --> C[LLM: extract origin, destination, dates, filters]
    C --> D{Parse confident enough?}
    D -->|No| E[Ask user to clarify / show raw form]
    D -->|Yes| F[Build structured search params]
    F --> G[Run normal Scatter-Gather search]
    G --> H[Return parsed params + results]
```

```mermaid
sequenceDiagram
    participant U as User/Next.js
    participant A as NestJS API
    participant L as LLM
    participant S as Search Service

    U->>A: POST /search/ai { query: "cheap flight to Dubai next weekend, direct only" }
    A->>L: Extract structured params from free text
    L-->>A: { origin, destination, dates, stops, confidence }
    alt confidence too low
        A-->>U: Clarifying questions
    else confident
        A->>S: searchFlights(parsed params)
        S-->>A: Results
        A-->>U: { parsed, results }
    end
```

```
function aiSearch(freeTextQuery):
    prompt = buildExtractionPrompt(freeTextQuery)
    parsed = llm.extract(prompt)  // { origin, destination, departureDate, returnDate, stops, cabinClass, sort }

    if parsed.confidence < THRESHOLD:
        return { needsClarification: true, questions: parsed.missingFields }

    results = searchFlights(parsed)
    return { parsed, results }
```

### 16.4 Fixed-Date Price Search

```mermaid
flowchart TD
    A[User picks exact travel date] --> B[GET /search/flights?departureDate=fixed]
    B --> C[Run Scatter-Gather search for that date]
    C --> D[Sort results by price ascending]
    D --> E[User compares & picks one offer]
    E --> F[Proceed to booking with chosen providerOfferId]
```

```mermaid
sequenceDiagram
    participant U as User/Next.js
    participant A as NestJS API
    participant S as Search Service (Scatter-Gather)

    U->>A: GET /search/flights?departureDate=2026-09-01&...
    A->>S: searchFlights(query)
    S-->>A: Normalized offers from all providers
    A-->>U: Offers sorted by price
    U->>A: Select offer -> proceed to booking
```

```
function fixedDateSearch(origin, destination, departureDate, passengers):
    results = searchFlights({ origin, destination, departureDate, passengers })
    return results.sortBy(price => price, 'asc')
```

### 16.5 Whole-Month Price View

```mermaid
flowchart TD
    A[GET /search/flights/month] --> B{Cached month prices?}
    B -->|Yes| C[Return cached calendar]
    B -->|No| D[For each day in month: fan-out provider search]
    D --> E[Take lowest price per day]
    E --> F[Cache calendar in Redis, longer TTL]
    F --> G[Return day -> lowestPrice list]
```

```mermaid
sequenceDiagram
    participant U as User/Next.js
    participant A as NestJS API
    participant R as Redis
    participant S as Search Service (per-day)

    U->>A: GET /search/flights/month?origin=CAI&destination=DXB&month=2026-09
    A->>R: Check cached calendar
    alt cache hit
        R-->>A: Cached day->price list
    else cache miss
        loop for each day in month
            A->>S: searchFlights({ ...route, departureDate: day })
            S-->>A: Offers for that day
        end
        A->>A: Take lowest price per day
        A->>R: Cache calendar (ttl 1h)
    end
    A-->>U: Calendar of lowest prices
```

```
function monthPriceView(origin, destination, month, airline):
    cacheKey = hash(origin, destination, month, airline)
    cached = redis.get(cacheKey)
    if cached exists:
        return cached

    days = allDaysIn(month)
    results = parallelMap(days, day => searchFlights({ origin, destination, departureDate: day, airline }))
    calendar = days.map(day => ({ date: day, lowestPrice: min(results[day].map(f => f.price)) }))

    redis.set(cacheKey, calendar, ttl=1hour)
    return calendar
```

### 16.6 Filtering & Sorting

```mermaid
flowchart TD
    A[Client sends filter/sort params<br/>with search request or after] --> B[Load raw provider results<br/>from cache/scatter-gather]
    B --> C[Apply filters: price range, cabin class, stops, rating, departure window]
    C --> D[Apply sort: price/duration/rating asc or desc]
    D --> E[Return filtered + sorted list]
```

```mermaid
sequenceDiagram
    participant U as User/Next.js
    participant A as NestJS API
    participant S as Search Service

    U->>A: GET /search/flights?...&maxStops=0&minRating=4&sort=price_asc
    A->>S: searchFlights(query) -> raw normalized offers
    A->>A: filterAndSort(offers, filters, sort)
    A-->>U: Filtered + sorted offers
```

```
function filterAndSort(offers, filters, sort):
    filtered = offers.filter(o =>
        (filters.maxPrice == null or o.price <= filters.maxPrice) and
        (filters.cabinClass == null or o.cabinClass == filters.cabinClass) and
        (filters.maxStops == null or o.stops <= filters.maxStops) and
        (filters.minRating == null or o.rating >= filters.minRating) and
        (filters.departureWindow == null or withinWindow(o.departureTime, filters.departureWindow))
    )

    return filtered.sortBy(sort.field, sort.direction)  // e.g. price/asc, duration/asc, rating/desc
```

### 16.7 Separate Entity Bookings (Flight & Hotel Booking Creation)

```mermaid
flowchart TD
    A[Client picks type: flight or hotel] --> B{Booking type}
    B -->|Flight| C[POST /flight-bookings]
    B -->|Hotel| D[POST /hotel-bookings]
    C --> E[Fetch offer from Redis/live provider]
    D --> E
    E --> F[Snapshot provider details into booking row]
    F --> G[Create row in flight_bookings or hotel_bookings<br/>status=pending_payment]
    G --> H[Proceed to Payment]
```

See sequence diagram in [§9](#9-sequence-diagram-flight-booking-pipeline) (flight case; hotel case is structurally identical against `hotel_bookings`).

```
function createFlightBooking(providerOfferId, userId, seatNumber):
    offer = fetchOffer(providerOfferId)  // Redis cache or live provider re-fetch
    return db.flightBookings.create({
        userId, provider: offer.provider, providerOfferId: offer.id,
        airlineCode: offer.airlineCode, flightNumber: offer.flightNumber,
        departureAirport: offer.departureAirport, arrivalAirport: offer.arrivalAirport,
        departureTime: offer.departureTime, arrivalTime: offer.arrivalTime,
        cabinClass: offer.cabinClass, seatNumber,
        totalAmount: offer.price, currency: offer.currency, status: 'pending_payment',
    })

function createHotelBooking(providerOfferId, userId, checkInDate, checkOutDate):
    offer = fetchOffer(providerOfferId)
    return db.hotelBookings.create({
        userId, provider: offer.provider, providerOfferId: offer.id,
        hotelName: offer.hotelName, roomType: offer.roomType, stars: offer.stars,
        freeCancellation: offer.freeCancellation, checkInDate, checkOutDate,
        totalAmount: offer.pricePerNight * nights(checkInDate, checkOutDate),
        currency: offer.currency, status: 'pending_payment',
    })
```

### 16.8 Payment Integration

```mermaid
flowchart TD
    A[Booking created, status=pending_payment] --> B[Client selects payment method + provider]
    B --> C[Create PAYMENTS row + TRANSACTIONS row type=authorize]
    C --> D[Call chosen provider's initiate-transaction API]
    D --> E[Client completes payment on provider's side]
    E --> F[Provider sends webhook]
    F --> G{Webhook status}
    G -->|success| H[Mark transaction/payment/booking confirmed, send receipt]
    G -->|failure| I[Mark transaction/payment/booking failed]
```

```mermaid
sequenceDiagram
    participant U as User/Next.js
    participant A as NestJS API
    participant P as Payment Provider (Stripe/PayMob/...)
    participant D as PostgreSQL

    U->>A: Choose method (card/wallet/bank) + provider
    A->>D: Load PAYMENT_PROVIDERS config for chosen provider
    A->>D: Create PAYMENTS + TRANSACTIONS (type=authorize)
    A->>P: Initiate transaction
    P-->>A: Client secret / redirect URL
    A-->>U: Render provider's payment UI
    U->>P: Complete payment
    P-->>A: Webhook (success/failure)
    A->>D: Update transaction, payment, booking status
    A-->>U: Booking confirmation
```

```
function initiatePayment(bookingType, bookingId, method, providerName):
    booking = db[bookingType + 'Bookings'].find(bookingId)
    providerConfig = db.paymentProviders.findByName(providerName)  // credentials via vault refs in config

    payment = db.payments.create({
        bookingType, bookingId, providerId: providerConfig.id,
        amount: booking.totalAmount, currency: booking.currency, status: 'pending',
    })
    txn = db.transactions.create({
        paymentId: payment.id, providerId: providerConfig.id,
        type: 'authorize', method, amount: booking.totalAmount, currency: booking.currency, status: 'pending',
    })

    tx = paymentProviderClient(providerConfig).initiateTransaction(booking.totalAmount, method)
    db.transactions.update(txn.id, { providerTransactionId: tx.id })
    return { paymentId: payment.id, txSecret: tx.secret }

function onPaymentWebhook(event):
    txn = db.transactions.findByProviderTransactionId(event.providerTransactionId)
    payment = db.payments.find(txn.paymentId)
    bookingTable = db[payment.bookingType + 'Bookings']

    if event.status == 'success':
        db.transactions.update(txn.id, { status: 'completed', rawResponse: event.raw })
        db.payments.update(payment.id, { status: 'paid' })
        booking = bookingTable.update(payment.bookingId, { status: 'confirmed' })
        audit.log('payment_success', booking)
        notification.sendReceipt(booking)
    else:
        db.transactions.update(txn.id, { status: 'failed', rawResponse: event.raw })
        db.payments.update(payment.id, { status: 'failed' })
        bookingTable.update(payment.bookingId, { status: 'payment_failed' })
```

### 16.9 Refunds

```mermaid
flowchart TD
    A[POST /flight-bookings/:id/refund] --> B{Booking is paid & refundable?}
    B -->|No| C[Reject: not eligible]
    B -->|Yes| D[Create TRANSACTIONS row: type=refund, status=processing]
    D --> E[Call payment provider refund API via stored PAYMENT_PROVIDERS config]
    E --> F{Provider confirms refund?}
    F -->|Yes| G[Update transaction + payment + booking status = refunded]
    F -->|No| H[Mark transaction failed, alert support]
    G --> I[Audit log + notify customer]
```

```mermaid
sequenceDiagram
    participant U as Customer
    participant A as NestJS API
    participant P as Payment Provider
    participant D as PostgreSQL

    U->>A: POST /flight-bookings/:id/refund
    A->>D: Check booking status & refund eligibility
    A->>D: Load PAYMENTS row + PAYMENT_PROVIDERS config
    A->>D: Create TRANSACTIONS row (type=refund, status=processing)
    A->>P: Request refund (using provider config)
    P-->>A: Refund confirmation + provider_transaction_id
    A->>D: Update transaction, payment, booking status
    A-->>U: Refund result
```

```
function requestRefund(bookingId, reason):
    booking = db.flightBookings.find(bookingId)
    if booking.status != 'confirmed' or not isRefundable(booking):
        throw NotEligibleError

    payment = db.payments.findOne({ bookingType: 'flight', bookingId: booking.id })
    provider = db.paymentProviders.find(payment.providerId)
    originalCharge = db.transactions.findOne({ paymentId: payment.id, type: 'authorize', status: 'completed' })

    txn = db.transactions.create({
        paymentId: payment.id, providerId: provider.id,
        type: 'refund', method: originalCharge.method,
        amount: booking.totalAmount, currency: payment.currency, status: 'processing',
    })

    result = paymentProviderClient(provider).refund(originalCharge.providerTransactionId, booking.totalAmount)

    if result.success:
        db.transactions.update(txn.id, { status: 'completed', providerTransactionId: result.id, rawResponse: result.raw })
        db.payments.update(payment.id, { status: 'refunded' })
        db.flightBookings.update(bookingId, { status: 'refunded' })
    else:
        db.transactions.update(txn.id, { status: 'failed', rawResponse: result.raw })
        alertSupportTeam(txn)

    audit.log('refund', booking, { before: booking.status, after: 'refunded' })
    notification.send(booking.userId, 'refund_result', result)
    return txn
```

### 16.10 Post-Payment Change (Reschedule)

```mermaid
flowchart TD
    A[PATCH /flight-bookings/:id/reschedule] --> B{Booking confirmed & changeable?}
    B -->|No| C[Reject: not changeable]
    B -->|Yes| D[Reprice new date/time with provider]
    D --> E{Price difference}
    E -->|Higher| F[Charge difference via payment provider]
    E -->|Lower| G[Refund difference via payment provider]
    E -->|Same| H[No charge]
    F --> I[Update booking, reissue confirmation]
    G --> I
    H --> I
```

```mermaid
sequenceDiagram
    participant U as Customer
    participant A as NestJS API
    participant P as Payment Provider
    participant Ext as External Provider (Amadeus/Duffel)
    participant D as PostgreSQL

    U->>A: PATCH /flight-bookings/:id/reschedule
    A->>D: Load booking, verify changeable
    A->>Ext: Reprice new date/time
    Ext-->>A: New price
    A->>P: Charge or refund price difference
    P-->>A: Payment result
    A->>D: Update booking (new time, status=confirmed)
    A-->>U: Reschedule confirmation + price difference
```

```
function rescheduleBooking(bookingId, newDepartureTime, newArrivalTime):
    booking = db.flightBookings.find(bookingId)
    if booking.status != 'confirmed' or not isChangeable(booking):
        throw NotChangeableError

    newPrice = externalProvider.reprice(booking, newDepartureTime, newArrivalTime)
    diff = newPrice - booking.totalAmount

    payment = db.payments.findOne({ bookingType: 'flight', bookingId: booking.id })
    provider = db.paymentProviders.find(payment.providerId)
    originalCharge = db.transactions.findOne({ paymentId: payment.id, type: 'authorize', status: 'completed' })

    if diff > 0:
        db.transactions.create({ paymentId: payment.id, providerId: provider.id, type: 'charge', amount: diff, currency: booking.currency, status: 'processing' })
        paymentProviderClient(provider).charge(originalCharge.providerTransactionId, diff)
    elif diff < 0:
        db.transactions.create({ paymentId: payment.id, providerId: provider.id, type: 'refund', amount: abs(diff), currency: booking.currency, status: 'processing' })
        paymentProviderClient(provider).refund(originalCharge.providerTransactionId, abs(diff))

    db.flightBookings.update(bookingId, {
        departureTime: newDepartureTime,
        arrivalTime: newArrivalTime,
        totalAmount: newPrice,
        status: 'confirmed',
    })

    audit.log('reschedule', booking, { before: booking.departureTime, after: newDepartureTime })
    notification.send(booking.userId, 'reschedule_confirmed')
    return { bookingId, status: 'confirmed', priceDifference: diff }
```

### 16.11 Notifications

```mermaid
flowchart TD
    A[Domain event: booking confirmed,<br/>refund done, price drop, ticket update, etc] --> B[Enqueue notification job in BullMQ]
    B --> C[Notification Worker picks up job]
    C --> D{Channel}
    D -->|Email| E[Send via email provider]
    D -->|SMS| F[Send via SMS provider]
    E --> G[Write NOTIFICATIONS row: sent/failed]
    F --> G
```

```mermaid
sequenceDiagram
    participant S as Service (booking/payment/refund/etc)
    participant Q as BullMQ Queue
    participant W as Notification Worker
    participant P as Email/SMS Provider
    participant D as PostgreSQL

    S->>Q: enqueue({ userId, channel, template, payload })
    Q->>W: deliver job
    W->>P: send(channel, recipient, renderedMessage)
    P-->>W: delivery result
    W->>D: insert NOTIFICATIONS row (channel, recipient, payload, status)
```

```
function notify(userId, channel, template, payload):
    queue.enqueue('notifications', { userId, channel, template, payload })

// BullMQ worker
function notificationWorker(job):
    user = db.users.find(job.userId)
    recipient = job.channel == 'email' ? user.email : user.phone
    message = renderTemplate(job.template, job.payload)

    result = channelProvider(job.channel).send(recipient, message)

    db.notifications.create({
        userId: job.userId, channel: job.channel, recipient,
        payload: message, status: result.success ? 'sent' : 'failed',
    })
```

### 16.12 Reporting

```mermaid
flowchart TD
    A[Admin opens dashboard] --> B[GET /admin/reports?range=daily|monthly]
    B --> C{Cached for this range?}
    C -->|Yes| D[Return cached metrics]
    C -->|No| E[Aggregate query: bookings, cancellations, revenue]
    E --> F[Cache result, short TTL]
    F --> G[Return metrics]
```

```mermaid
sequenceDiagram
    participant Ad as Admin/Next.js
    participant A as NestJS API
    participant R as Redis
    participant D as PostgreSQL

    Ad->>A: GET /admin/reports?range=monthly
    A->>R: Check cached report
    alt cache hit
        R-->>A: Cached metrics
    else cache miss
        A->>D: Aggregate bookings/cancellations/revenue for range
        D-->>A: Aggregated rows
        A->>R: Cache (ttl=5min)
    end
    A-->>Ad: Dashboard metrics
```

```
function getDashboardMetrics(range):  // range = 'daily' | 'monthly'
    cacheKey = 'report:' + range
    cached = redis.get(cacheKey)
    if cached exists:
        return cached

    bookingsCount = db.flightBookings.count({ createdAt: withinRange(range) })
                  + db.hotelBookings.count({ createdAt: withinRange(range) })
    cancellations = db.flightBookings.count({ status: 'refunded', createdAt: withinRange(range) })
                  + db.hotelBookings.count({ status: 'refunded', createdAt: withinRange(range) })
    revenue = db.payments.sum('amount', { status: 'paid', createdAt: withinRange(range) })

    metrics = { bookingsCount, cancellations, revenue, range }
    redis.set(cacheKey, metrics, ttl=5min)
    return metrics
```

### 16.13 Customer Support

```mermaid
flowchart TD
    A[POST /support/tickets] --> B[Create ticket: status=open, linked to userId/bookingId]
    B --> C[Notify support queue]
    C --> D[Support Agent picks up ticket]
    D --> E{Resolved?}
    E -->|No| F[Agent replies / escalates]
    F --> D
    E -->|Yes| G[Close ticket, notify customer]
```

```mermaid
sequenceDiagram
    participant U as Customer
    participant A as NestJS API
    participant Q as Support Queue
    participant Ag as Support Agent
    participant D as PostgreSQL

    U->>A: POST /support/tickets { bookingId, subject, message }
    A->>D: Create ticket (status=open)
    A->>Q: Notify support queue
    Q-->>Ag: New ticket alert
    Ag->>A: PATCH /support/tickets/:id (reply/resolve)
    A->>D: Update ticket
    A-->>U: Notification: ticket updated/resolved
```

```
function createSupportTicket(userId, bookingId, subject, message):
    ticket = db.tickets.create({ userId, bookingId, subject, message, status: 'open' })
    notification.notifySupportQueue(ticket)
    audit.log('support_ticket_created', ticket)
    return ticket

function resolveTicket(ticketId, agentId, resolution):
    ticket = db.tickets.update(ticketId, { status: 'closed', resolution, resolvedBy: agentId })
    notification.send(ticket.userId, 'ticket_resolved')
    audit.log('support_ticket_resolved', ticket, { before: 'open', after: 'closed' })
```

### 16.14 Favorites & Price-Watch Notification

```mermaid
flowchart TD
    A[POST /favorites] --> B[Save favorite: type, providerOfferId, targetPrice, snapshot]
    B --> C[(Favorites table)]

    D[Scheduled job: every N minutes] --> E[For each price-watch favorite]
    E --> F[Re-run search for saved route]
    F --> G{currentPrice <= targetPrice?}
    G -->|Yes| H[Send price-drop notification]
    G -->|No| I[Skip]
```

```mermaid
sequenceDiagram
    participant U as Customer
    participant A as NestJS API
    participant D as PostgreSQL
    participant Sch as Scheduler (BullMQ repeatable job)
    participant S as Search Service

    U->>A: POST /favorites { type, providerOfferId, targetPrice }
    A->>D: Create FAVORITES row (with snapshot)
    A-->>U: Saved

    loop every N minutes
        Sch->>D: Load favorites with targetPrice set
        Sch->>S: searchFlights(favorite.snapshot.route)
        S-->>Sch: Current lowest price
        alt price <= targetPrice
            Sch->>U: Notify price drop
        end
    end
```

```
function addFavorite(userId, type, providerOfferId, targetPrice):
    snapshot = fetchCurrentSnapshot(type, providerOfferId)  // from live search, not a DB catalog table
    return db.favorites.create({ userId, type, providerOfferId, targetPrice, snapshot })

// Runs on a schedule (BullMQ repeatable job)
function checkPriceWatches():
    watches = db.favorites.findAll({ targetPrice: notNull() })
    for watch in watches:
        currentPrice = searchFlights(watch.snapshot.route).minPrice()
        if currentPrice <= watch.targetPrice:
            notification.send(watch.userId, 'price_drop', { watch, currentPrice })
```

### 16.15 Auditing & Logging

```mermaid
flowchart TD
    A[Any state-changing request hits controller] --> B[Audit Interceptor wraps handler]
    B --> C[Handler executes business logic]
    C --> D{Success?}
    D -->|Yes| E[Write audit_logs row: actor, action, before/after]
    D -->|No| F[Write structured error log with correlation ID]
    E --> G[Ship log line to ELK/Loki]
    F --> G
```

```
// NestJS Interceptor, wraps every write endpoint
function auditInterceptor(context, next):
    request = context.getRequest()
    before = loadEntityStateIfApplicable(request)

    result = next.handle()  // run the actual handler

    result.subscribe(response => {
        audit.log({
            actorUserId: request.user.id,
            action: request.method + ' ' + request.route,
            entityType: inferEntityType(request),
            entityId: response.id,
            beforeValue: before,
            afterValue: response,
            ipAddress: request.ip,
        })
    })

    return result

function log(level, message, context):
    logger.write({
        level, message, ...context,
        correlationId: currentRequestContext.correlationId,
        timestamp: now(),
    })  // shipped async to ELK/Loki
```

## 17. Testing Suite Setup

### Unit & Integration Testing (Jest)

To test the parallel aggregator without hitting live APIs, mock supplier responses with Jest:

```typescript
// src/modules/search/search.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { SearchService } from './search.service';
import { HttpService } from '@nestjs/axios';
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { of } from 'rxjs';

describe('SearchService (Scatter-Gather)', () => {
  let service: SearchService;
  let httpService: HttpService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        SearchService,
        {
          provide: HttpService,
          useValue: {
            get: jest.fn().mockReturnValue(of({ data: [] })),
            post: jest.fn().mockReturnValue(of({ data: [] })),
          },
        },
        {
          provide: CACHE_MANAGER,
          useValue: {
            get: jest.fn().mockResolvedValue(null),
            set: jest.fn().mockResolvedValue(true),
          },
        },
      ],
    }).compile();

    service = module.get<SearchService>(SearchService);
    httpService = module.get<HttpService>(HttpService);
  });

  it('should collect flight offers from providers concurrently', async () => {
    const results = await service.searchFlights({
      origin: 'CAI',
      destination: 'DXB',
      date: '2026-09-01',
    });
    expect(results).toBeDefined();
    expect(Array.isArray(results)).toBe(true);
  });
});
```

### Running Tests

```bash
# Run Unit Tests
npm run test

# Run End-to-End Tests
npm run test:e2e
```
