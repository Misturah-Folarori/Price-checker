#  System Architecture – Price Checker

##  Purpose
This document describes the technical architecture for **Price Checker,** a crowdsourced, location-aware price comparison application. It is intended to guide developers through frontend, backend, database, integrations, deployment and operational concerns for the initial MVP and near-term scaling.


![Price Checker Wireframe](https://whimsical.com/AGHEWFjvtJbacVrUTELdxX)

## High-level overview
- **Frontend (client):** React (Next.js) single-page application (mobile-first).

- **Backend (API):** Node.js + Express (or NestJS) RESTful API.

- **Database:** MongoDB (primary) for flexible, document-based price updates and product records.

- **Cache / Real-time layer:** Redis for caching frequent queries and rate limiting; optional WebSocket service (Socket.IO) for real-time updates.

- **External services:** Google Maps API (geocoding, distance, maps), Cloud storage for uploaded images (AWS S3 / DigitalOcean Spaces).

- **Hosting / Infra:** Vercel for frontend, Render/Heroku/AWS Elastic Beanstalk / Railway for backend; MongoDB Atlas for DB.

- **CI/CD:** GitHub Actions pipeline with automated tests and deploy steps.

- **Monitoring:** Sentry / LogDNA for error logging, Prometheus + Grafana for metrics.

## Components & Responsibilities

###  Frontend (Next.js)

- Responsibilities

  - UI for search, comparison list, product details, and update form.


  - Fetches data from REST endpoints: product search, price list, submit price.

  - Location handling (browser geolocation with fallback to manual entry).

  - Minimal client-side caching and optimistic UI for price submissions.


- Key folders


  - `/pages` — Next.js pages (home, product, store, auth)

  - `/components` — SearchBar, PriceList, PriceRow, UpdateForm, MapView

  - `/lib/api.js` — wrapper for REST calls and error handling


- Considerations

  - Server-side rendering (SSR) for SEO on product pages (Next.js `getServerSideProps`).

  - Accessibility (a11y) and performance: lazy-loading images, code-splitting.



### Backend (Node.js + Express)
- Responsibilities
  - Expose REST API endpoints for search, price retrieval, submission, and user management.

  - Validate, sanitize and persist price submissions.

  - Enforce rate-limits and anti-spam checks.

  - Optional microservice for reputation/scoring of contributors.

- Core modules

  - `app.js / server.js` — Express server + middleware (helmet, cors, body-parser, rate-limit)


  - `/routes` — products, prices, users, auth, admin

  - `/controllers` — implement business logic

  - `/services` — DB access, maps integration, image upload service

  - `/jobs` — background workers for verification, aggregation, cleanup


- Middleware

  - Authentication (JWT) middleware

  - Rate limit middleware (per-IP and per-user)

  - Input validation (Joi or Zod)



###  Database (MongoDB)

- Why MongoDB

  - Flexible schema for price events and different product attributes.

  - Fast writes for high-frequency crowdsourced updates.

  - Good geospatial query support for location-aware results.

- Collections

  - `users` — registered contributors and optional reputation data

  - `products` — canonical product records (name, brand, category, SKU)

  - `stores` — store metadata (name, address, coordinates, verified?)

  - `prices` — price updates (document model shown below)

  - `alerts` — user price alerts and subscriptions


### prices document example
```
{
  "_id": "ObjectId",
  "productId": "ObjectId",      // reference to products
  "storeId": "ObjectId",        // reference to stores
  "price": 3000,                // integer in kobo/cent units recommended
  "currency": "NGN",
  "unit": "50g",                // normalized unit string
  "source": "user",             // 'user' | 'store' | 'partner'
  "userId": "ObjectId",         // who submitted (nullable for partner)
  "photoUrl": "https://.../img.jpg",
  "timestamp": "2025-11-07T14:32:00Z",
  "verified": false,            // boolean
  "meta": {
     "notes": "offer till weekend",
     "confidenceScore": 0.85
  },
  "location": {                 // GeoJSON Point
     "type": "Point",
     "coordinates": [3.425, 6.524]
  }
}
```
### Indexes
  - Compound index: { productId: 1, "location": "2dsphere", timestamp: -1 }

  - Index on storeId, timestamp, and userId for quick queries

  - TTL index on older, irrelevant price events (if needed) or use rolling aggregation

## API Design (core endpoints)
All endpoints should respond with JSON and follow REST conventions. Use HTTP status codes properly.
 Search / Product endpoints
- GET /api/products?query=milo%2050g&location=lat,lng&limit=20

  - Returns product matches and top price aggregates near the location.

  - Response: product summary, bestPrice, nearbyStoresCount.

### Prices / Comparison endpoints
- GET /api/prices?productId={pid}&lat={lat}&lng={lng}&radius=5000&sort=cheapest

  - Returns list of recent prices within radius (meters).

  - Response sample:
```
{
  "productId": "...",
  "results": [
    { "store": {...}, "price": 2900, "timestamp": "...", "photoUrl": "...", "verified": true },
    ...
  ]
}
```
- GET /api/prices/{priceId} — single price details (for history view)


### Price submission
- POST /api/prices

  - Body:

```
{
  "productId": "ObjectId",
  "storeId": "ObjectId",
  "price": 3000,
  "currency": "NGN",
  "unit": "50g",
  "photo": "base64 or presigned-s3-url",
  "timestamp": "ISO date (optional)"
}
```
- Auth: optional; allow guest submissions but require moderation / lower trust weight.

- Response: `201 Created` with created resource.


### User & Auth (optional early-stage)
- `POST /api/users` — register


- `POST /api/auth/login` — returns JWT


- `GET /api/users/{userId}/contributions`


### Admin endpoints
- `/api/admin/prices/{id}/verify` — mark verified


- `/api/admin/stores` — create / verify store




## Data flow & Sequence
### Typical read flow
1. User enters product + location → Frontend calls `GET /api/prices`.

2. Backend checks Redis cache for `prices:productId:lat:lng:radius`.

  - Cache hit → return cached result.

  - Cache miss → query MongoDB: geospatial query to find stores and join latest price per store → aggregate → write to cache → return.

### Typical write flow
1. User submits price via `POST /api/prices`.

2. Backend validates input, stores raw entry in `prices`.

3. If photo provided, backend stores image to S3 and saves `photoUrl`.

4. Optionally enqueue verification job (worker) to:

  - Run heuristics (image OCR, duplicate detection) or

  - Notify moderators/validators or

  - Cross-check with partner feeds.

5. Invalidate or update relevant cache keys.

## Caching strategy
- **Redis** for:

  - Query-level cache for most-searched products within geohash buckets.

  - Rate-limit counters, session store (optional).

- **Cache keys:** `prices:{productId}:{geohash}:{radius}:{sort}`

- Cache TTL: 30s–5min depending on product volatility (fuel shorter, packaged goods longer).

## Verification & Trust model
- **Guest submissions:** accepted but flagged with `source: user` and `verified: false`.

- **Verification flow:**

  - Photo evidence: use simple OCR + heuristics to validate price text on receipts or shelf tags.

  - Reputation weighting: users gain trust score over time; high-score users’ submissions can auto-verify.

  - Manual moderation dashboard for disputed or flagged entries.

- **Confidence scoring:** store `confidenceScore` computed from source reliability, photo analysis, recency.

## Real-time updates (optional)
- Implement Socket.IO or server-sent events:

  - When a new price is submitted and verified, push update to subscribed clients viewing that product in a given radius.

  - Use Redis Pub/Sub to broadcast events across backend instances.


## Scalability & performance
- **Horizontal scaling:** stateless backend instances behind a load balancer.

- **DB scaling:** MongoDB Atlas with sharding for large datasets; use read replicas for heavy read workloads.

- **Query optimization:** pre-aggregate “latest price per store per product” in a materialized view or aggregation collection for frequent reads.

- **Autoscaling:** configure cloud provider to scale backend based on CPU and request latency.

## Security
- **Authentication:** JWT with refresh tokens. Require auth for sensitive endpoints (verify, admin).

- **Input sanitization:** validate all inputs (Joi / Zod) to avoid injection attacks.

- **Secure file uploads:** use pre-signed S3 URLs; scan images for malware if needed.

- **Rate limiting:** per-IP & per-user limits to prevent spam (express-rate-limit + Redis store).

- **Transport:** enforce TLS (HTTPS) for all traffic.

- **Secrets management:** store keys in environment variables / secret manager (e.g., AWS Secrets Manager).



## Observability & Logging
- **App logs:** structured JSON logs sent to LogDNA or CloudWatch.

- **Error tracking:** Sentry for stack traces and user-impacting errors.

- **Metrics:** Prometheus scraping; dashboards in Grafana for request latency, error rate, CPU/memory.

- **Alerts:** set alerts for error rate > X%, high DB CPU, cache miss spike, and queue growth.


## Testing & CI/CD
- **Unit tests:** Jest for backend business logic; React Testing Library for frontend components.

- **Integration tests:** test API endpoints using Supertest or Cypress.

- **E2E tests:** Cypress to simulate user flows (search → view → update).

- **CI Pipeline (GitHub Actions):**

  - Lint → Unit tests → Build → Deploy to staging → run smoke tests → Deploy to production.

- **Rollback strategy:** keep previous release artifacts and DB migration rollbacks.

## Operational considerations
- **Backups:** daily snapshot backups for MongoDB; more frequent for critical collections.

- **Data retention:** decide retention for raw price events vs aggregated latest snapshot.

- **GDPR / Data privacy:** store only necessary PII; allow data deletion on request.

- **Cost management:** monitor S3, DB, and outbound API (Google Maps) usage; cache aggressively to reduce external calls.


## Example infra diagram
```
[User Device / Browser]
   ↕ HTTPS
[Frontend - Next.js (Vercel)]
   ↕ REST (HTTPS)
[API Gateway / Load Balancer]
   ↕ REST (HTTPS)
[Backend - Node.js/Express (multiple instances)]
   ↔ Redis (cache, rate-limit)
   ↔ MongoDB Atlas (prices, products, users)
   ↔ S3 (photo storage)
   ↔ Google Maps API (geocoding, maps)
   ↔ Worker Queue (BullMQ) → Worker instances (image OCR, verification)
   ↔ Monitoring (Prometheus, Grafana) & Logging (Sentry, LogDNA)
```

## Migration & release plan (MVP → v1)
### MVP scope
- Search, compare, and submit price (guest submissions allowed).

- Basic verification: photo optional, admin manual verify.

- Basic map integration.

### v1
- Reputation & auto-verify for trusted users.

- Alerts & subscriptions.

- Partner integrations (store feeds).

- Real-time push notifications.

  







The system uses proven, lightweight web technologies that scale easily with user growth.  
MongoDB’s flexible schema supports frequent updates, while React ensures fast UI performance and real-time refresh capabilities.
