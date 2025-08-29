# Technical & Functional Requirements — Backend Features

This document specifies technical and functional requirements for core backend modules. It covers API endpoints, request/response schemas, validation rules, authorization, errors, and performance criteria.

> Base URL: `/api/v1`  
> Auth: JWT in `Authorization: Bearer <token>` unless endpoint is public.

---

## 1) User Authentication & Account Management

### Goals
- Allow users (guest, host, admin) to register, authenticate, and manage their profile securely.
- Enforce RBAC and password best practices.

### Endpoints
| Method | Path | Auth | Roles | Purpose |
|---|---|---|---|---|
| POST | `/auth/register` | Public | — | Create account |
| POST | `/auth/login` | Public | — | Obtain JWT |
| POST | `/auth/refresh` | Public | — | Refresh access token |
| POST | `/auth/forgot-password` | Public | — | Send reset link/code |
| POST | `/auth/reset-password` | Public | — | Reset password via token |
| GET | `/users/me` | Required | Any | Get own profile |
| PATCH | `/users/me` | Required | Any | Update own profile |
| PATCH | `/users/me/password` | Required | Any | Change password |
| GET | `/admin/users` | Required | admin | List users (paginated) |
| PATCH | `/admin/users/:id/status` | Required | admin | Activate/deactivate user |

### Request/Response (examples)
**POST /auth/register**
```json
// Request
{
  "first_name": "Alice",
  "last_name": "Johnson",
  "email": "alice@example.com",
  "password": "P@ssw0rd!",
  "role": "guest",
  "phone_number": "+250788000000"
}
```
```json
// Response 201
{
  "user_id": "UUID",
  "email": "alice@example.com",
  "role": "guest",
  "created_at": "2025-08-29T06:45:00Z"
}
```

**POST /auth/login**
```json
// Request
{ "email": "alice@example.com", "password": "P@ssw0rd!" }
```
```json
// Response 200
{
  "access_token": "jwt-access",
  "refresh_token": "jwt-refresh",
  "expires_in": 3600
}
```

### Validation Rules
- `email`: valid email, unique.
- `password`: min 8 chars, at least 1 uppercase, 1 lowercase, 1 number; hash with Argon2 or bcrypt(12+).
- `role`: one of `guest|host|admin` (admin assignable by existing admin only).
- Rate-limit login/register: e.g., **10 req / 15 min / IP**.
- Reset tokens: single use, expire in **15 minutes**.

### Errors
- `400` invalid input; `401` invalid credentials/token; `403` insufficient role; `409` email exists; `429` rate limited.

### Performance & Security
- Index `email` for O(log n) lookup.
- JWT access TTL 15–60 mins; refresh 7–30 days; support token revocation (allowlist/rotation).
- Audit log: login success/failure, password changes.
- Expected p95 latency: **<150 ms** for login/refresh; **<200 ms** for profile ops.


---

## 2) Property Management

### Goals
- Hosts can CRUD properties; guests can search and view.
- Support filters & pagination for efficient discovery.

### Endpoints
| Method | Path | Auth | Roles | Purpose |
|---|---|---|---|---|
| POST | `/properties` | Required | host, admin | Create property |
| GET | `/properties/:id` | Optional | any | Get property detail |
| PATCH | `/properties/:id` | Required | owner(host), admin | Update property |
| DELETE | `/properties/:id` | Required | owner(host), admin | Soft-delete property |
| GET | `/properties` | Optional | any | Search & list properties |
| GET | `/hosts/me/properties` | Required | host | List my properties |

### Request/Response (examples)
**POST /properties**
```json
// Request
{
  "name": "Cozy Apartment",
  "description": "2-bedroom in city center",
  "location": "Kigali, Rwanda",
  "price_per_night": 120.0,
  "amenities": ["wifi","kitchen","parking"],
  "photos": ["https://.../1.jpg","https://.../2.jpg"]
}
```
```json
// Response 201
{
  "property_id": "UUID",
  "host_id": "UUID",
  "name": "Cozy Apartment",
  "location": "Kigali, Rwanda",
  "price_per_night": 120.0,
  "created_at": "2025-08-29T06:45:00Z"
}
```

**GET /properties** — Query Params
```
?page=1&limit=20&location=Kigali&min_price=50&max_price=200&amenities=wifi,kitchen&start=2025-09-01&end=2025-09-05&sort=price_asc
```

### Validation Rules
- `name` 3–150 chars; `description` 10–5000 chars.
- `location` non-empty; `price_per_night` >= 0.
- `amenities` as controlled list; `photos` valid URLs.
- Ownership check: only property owner or admin can update/delete.

### Errors
- `400` bad payload; `403` not owner; `404` not found; `409` duplicate name per host (optional).

### Performance
- Indexes: `(location)`, `(price_per_night)`, `(host_id)`, GIN/JSON path for amenities (if JSONB).
- Pagination: limit <= 100; default 20.
- Search endpoint p95 latency **<300 ms** for typical filters (with indexes).

---

## 3) Booking System

### Goals
- Allow guests to create and manage bookings; prevent date overlaps; support lifecycle statuses.

### Endpoints
| Method | Path | Auth | Roles | Purpose |
|---|---|---|---|---|
| POST | `/bookings` | Required | guest, admin | Create booking (pending) |
| GET | `/bookings/:id` | Required | owner(guest, host of property), admin | Get booking |
| GET | `/me/bookings` | Required | guest | List my bookings |
| GET | `/hosts/me/bookings` | Required | host | Bookings on my properties |
| PATCH | `/bookings/:id/status` | Required | host, admin | Confirm/Cancel |
| DELETE | `/bookings/:id` | Required | guest | Cancel (rules apply) |

### Request/Response (examples)
**POST /bookings**
```json
// Request
{
  "property_id": "UUID",
  "start_date": "2025-09-01",
  "end_date": "2025-09-05",
  "guests": 2
}
```
```json
// Response 201
{
  "booking_id": "UUID",
  "status": "pending",
  "total_price": 480.0,
  "currency": "USD",
  "created_at": "2025-08-29T06:45:00Z"
}
```

**PATCH /bookings/:id/status**
```json
// Request
{ "action": "confirm" } // or "cancel"
```
```json
// Response 200
{ "booking_id": "UUID", "status": "confirmed" }
```

### Business/Validation Rules
- `start_date < end_date`; min stay = 1 night; max stay configurable (e.g., 30).
- **No overlap** for same `property_id` in `[start_date, end_date)`. Use transactional check with unique constraint on date ranges or serializable logic.
- `total_price = nights × property.price_per_night` (+fees/taxes if applicable).
- Cancellation policy: guest can cancel until `start_date - X days` (configurable); otherwise host/admin only.
- Status flow: `pending → confirmed → completed` or `pending/confirmed → canceled`.

### Errors
- `400` invalid dates; `403` unauthorized (not owner/host); `404` unknown booking; `409` date conflict.

### Performance
- Index `(property_id, start_date, end_date)` for fast overlap queries.
- Use **row-level lock** on property or calendar table during create.
- p95 latency: create booking **<350 ms**, list endpoints **<250 ms** with pagination.

---

## 4) Payments (Bonus)

### Goals
- Record payments for confirmed bookings; support multiple methods; prevent duplicates.

### Endpoints
| Method | Path | Auth | Roles | Purpose |
|---|---|---|---|---|
| POST | `/payments` | Required | guest | Pay for confirmed booking |
| GET | `/payments/:id` | Required | owner(guest, host of booking), admin | Get payment |
| GET | `/me/payments` | Required | guest | My payments |
| GET | `/admin/payments` | Required | admin | All payments (paginated) |

### Request/Response
**POST /payments**
```json
// Request
{
  "booking_id": "UUID",
  "method": "stripe",
  "amount": 480.0,
  "currency": "USD",
  "payment_token": "gateway-token-or-intent-id"
}
```
```json
// Response 201
{
  "payment_id": "UUID",
  "booking_id": "UUID",
  "status": "completed",
  "processed_at": "2025-08-29T06:50:00Z",
  "provider": "stripe",
  "provider_ref": "pi_12345"
}
```

### Validation Rules
- Booking must exist and be `confirmed`.
- Amount must equal server-side computed `total_price` to prevent tampering.
- Idempotency-Key header required to avoid duplicate charges.

### Errors
- `400` invalid token/amount; `402` payment required/failed; `403` not owner; `409` duplicate payment; `422` booking not payable.

### Performance & Security
- Call external gateway asynchronously with webhooks; persist pending → completed transitions.
- Store only non-PCI data; tokenize cards via provider.
- p95 latency **<600 ms** (excluding async webhook), queue for retries on provider errors.

---

## Common Non-Functional Requirements
- **Observability**: structured logs (correlation IDs), metrics (latency, error rate), tracing.
- **Backups & DR**: daily snapshots; RPO ≤ 24h, RTO ≤ 4h.
- **Internationalization**: store timestamps in UTC, prices with currency code and 2‑decimal precision.
- **Testing**: unit/integration tests; contract tests for APIs; seed data for staging.
- **Documentation**: OpenAPI/Swagger spec auto-generated from routes.
