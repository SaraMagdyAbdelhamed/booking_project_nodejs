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
16. [Testing Suite Setup](#16-testing-suite-setup)

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
- **Separate Entity Bookings:** Distinct booking creation pipelines for flights (`flight_bookings`) and hotels (`hotel_bookings`).
- **Payment Integration:** Multiple payment methods (card, wallet, bank transfer) via multiple payment providers/gateways (e.g. Stripe, PayMob), webhook verification, and receipt generation.
- **Refunds:** Cancel a paid booking and trigger refund back through the original payment provider, with status tracking (requested/processing/completed).
- **Post-Payment Change:** Change flight date/time on an already-paid booking — reprice against provider, charge/refund the difference, reissue confirmation.
- **Notifications:** Asynchronous email/SMS booking receipts and promotional offer alerts.
- **Reporting:** System-wide dashboard metrics tracking daily/monthly booking volume and cancellations.
- **Customer Support:** Pre-booking (search/pricing questions) and post-booking (refund status, reschedule help, complaints) support via ticketing, live chat, or agent-assisted channel, linked to the customer's booking history.
- **Favorites:** Registered customer saves flights, hotels, or price watches (route + target price) for quick access later; price-watch favorites can trigger a notification when the price drops.

## 5. Non-Functional Requirements (NFRs)

- **Performance:** Scatter-gather aggregator search response under 2,500ms (enforced hard cutoff time).
- **Scalability:** Horizontal scaling for high read-to-write ratio during search queries.
- **Security:** PCI-DSS compliant payment processing (tokenized via external providers), JWT authentication, rate limiting.
- **Availability:** 99.9% uptime with circuit breakers on third-party API adapters.

## 6. Entity Relationship Diagram (ERD)

Approach: separate flight and hotel bookings into independent entity tables, without a common base entity table.

```mermaid
erDiagram
    USERS ||--o{ FLIGHT_BOOKINGS : places
    USERS ||--o{ HOTEL_BOOKINGS : places
    USERS ||--o{ NOTIFICATIONS : receives
    USERS ||--o{ FAVORITES : saves
    FLIGHT_BOOKINGS ||--o| PAYMENTS : "paid via"
    HOTEL_BOOKINGS ||--o| PAYMENTS : "paid via"

    USERS {
        int id PK
        string email
        string password_hash
        datetime created_at
    }

    FLIGHT_BOOKINGS {
        int id PK
        int user_id FK
        string airline_code
        string flight_number
        string departure_airport
        string arrival_airport
        decimal total_amount
        string status
    }

    HOTEL_BOOKINGS {
        int id PK
        int user_id FK
        string hotel_id
        string hotel_name
        date check_in_date
        decimal total_amount
        string status
    }

    PAYMENTS {
        int id PK
        string booking_type FK
        int booking_id
        decimal amount
        string status
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
        string ref_id
        decimal target_price
        string snapshot
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

## 8. System Flow: Search Scatter-Gather

```mermaid
flowchart TD
    A[Client: Request Search /search/flights] --> B{Check Redis Cache?}
    B -->|Yes| C[Return Cached Data]
    B -->|No| D[Fan-out Parallel Async Requests<br/>Amadeus, Duffel, Tequila]
    D --> E[Wait for Results OR 2.5s Timeout]
    E --> F[Deduplicate & Normalize]
    F --> G[Save to Redis & Return Response]
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
    Note over D: Create Record in flight_bookings
    A-->>U: Booking Confirm
```

## 10. System Assumptions

- Suppliers return standardized ISO currency rates, or require local client-side handling.
- Payment holds are finalized synchronously before confirming supplier bookings.
- Redis is deployed with persistence enabled to survive restarts during cached search spikes.

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
│   │   ├── flight-booking/   # Flight Bookings Pipeline
│   │   ├── hotel-booking/    # Hotel Bookings Pipeline
│   │   ├── payment/          # Payment Webhooks & Auditing
│   │   ├── notification/     # Queue Workers for Email/SMS
│   │   ├── support/          # Customer Support Tickets & Chat
│   │   └── favorites/        # Saved Flights, Hotels & Price Watches
│   ├── common/               # Filters, Interceptors, Guards
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
  "refId": "flg_982341",
  "targetPrice": 300.00
}
```

Response:

```json
{
  "favoriteId": "fav_77120",
  "type": "flight",
  "refId": "flg_982341",
  "targetPrice": 300.00,
  "createdAt": "2026-07-28T10:00:00Z"
}
```

### 10. List Favorites

`GET /api/v1/favorites?userId=usr_12345`

Response:

```json
[
  { "favoriteId": "fav_77120", "type": "flight", "refId": "flg_982341", "targetPrice": 300.00 },
  { "favoriteId": "fav_77121", "type": "hotel", "refId": "htl_8821", "targetPrice": null }
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

## 16. Testing Suite Setup

### Unit & Integration Testing (Jest)

To test the parallel aggregator without hitting live APIs, mock supplier responses with Jest:

```

### Running Tests

```bash
# Run Unit Tests
npm run test

# Run End-to-End Tests
npm run test:e2e
```
