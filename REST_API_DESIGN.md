# HadatHub Regional Event & Ticketing API Design

**Author:** Levis Ishimwe  
**Email:** i.levis@alustudent.com  
**Date:** January 2026

---

## Executive Summary

This API architecture models events, venues, users, and tickets as first-class resources with strict capacity and ownership rules, ensuring the backend consistently enforces real-world venue constraints. Endpoints follow REST conventions using plural nouns and standard HTTP semantics, enabling web and mobile clients to discover resources, filter by region and date, and execute domain-specific operations like check-in and event cancellation. Responses use consistent JSON schemas with explicit error structures and include pagination and filtering to maintain performance as the East African event ecosystem scales.

## Domain Analysis

### Business Context
HadatHub operates across East African cities (Nairobi, Kigali, Dar es Salaam, Kampala, Addis Ababa) hosting diverse events: tech conferences, music festivals, meetups, and sports tournaments. The transition from manual Excel-based ticketing to a digital platform requires a robust architecture that:

- **Enforces capacity integrity**: Prevents overbooking by validating ticket sales against venue maximum capacity and event-specific capacity overrides.
- **Manages scheduling complexity**: Allows or prevents concurrent events at the same venue (configurable per venue) to reflect real-world constraints.
- **Supports multi-currency operations**: Events in different cities operate in different currencies (KES, TZS, RWF, UGX) with consistent pricing models.
- **Tracks organizational workflows**: From event draft → published → attendee sales → check-in, with clear role-based access (organizers, attendees, staff).
- **Enables discovery and search**: Attendees must easily find events by location, date range, category, and price—critical for a regional platform.

### Primary Entities

**Venue**: A physical location with fixed capacity, timezone, and administrative contact. The `allows_overlap` flag permits venue operators to control whether multiple events can occur simultaneously (e.g., a large convention center vs. a single-room theater). Critical for scheduling logic and capacity constraints.

**Event**: A time-bound occurrence at a specific venue, hosted by an organizer, with a defined price and sales window. The `capacity_override` lets organizers reduce effective capacity below the venue's maximum for ticketing purposes. Status transitions (draft → published → cancelled) enforce business rules.

**User**: A multi-role actor (organizer, attendee, or staff) who can create events, purchase tickets, or perform check-ins. Role-based access controls ensure only organizers publish events and only staff scan attendees.

**Ticket**: The primary transaction record linking a user (attendee) to an event. Captures the price paid (in regional currency), payment status, and QR code for check-in verification. Status transitions ensure tickets flow from reserved → paid → checked_in, with cancellation allowed at specific points.

**Check-in**: An audit log entry recording when staff scanned a ticket at the venue. Separate from tickets to maintain an immutable history and enable reports on actual attendance vs. sales.

### Key Relationships

| Relationship | Cardinality | Business Rule |
|---|---|---|
| Venue → Events | 1:N | A venue hosts many events; overlap rules apply per venue |
| Organizer → Events | 1:N | One user creates and owns multiple events |
| Event → Tickets | 1:N | One event can issue many tickets up to capacity |
| Attendee → Tickets | 1:N | One user can purchase multiple tickets across events |
| Ticket → Check-ins | 1:N | One ticket can be scanned once per session (1:1 typical, 1:N for multi-gate events) |

### Core Operations

| Operation | Actor | Example |
|---|---|---|
| Publish Event | Organizer | Organizer creates a conference, sets capacity, price, and sales window, then publishes |
| List Events | Attendee | Attendee filters "Concerts" in "Kampala" from "2026-03-01" to "2026-03-31" |
| Purchase Ticket | Attendee | Attendee buys ticket, payment is verified (e.g., M-Pesa), ticket status → paid |
| Check-in Attendee | Staff | Staff scans QR code at gate; system verifies ticket is paid and marks checked_in |
| Cancel Event | Organizer | Organizer cancels event (e.g., venue issue); tickets revert to reserved or are refunded |
| Refund Ticket | Attendee/Organizer | Attendee requests refund; ticket status → cancelled |

### Data and Currency Strategy

Events span Nairobi (KES), Kigali (RWF), Dar es Salaam (TZS), and Kampala (UGX). Each event specifies its native currency; prices are stored as decimals to avoid rounding errors. Exchange rate conversion is handled at the application layer, not in the API response, to keep pricing transparent per region.

---

## Resource Specifications

### Event Resource
**Purpose**: Represents a time-bound occurrence at a venue, hosted by an organizer. Core to the platform; events are discovered by attendees and managed by organizers.

**Schema**:
| Field | Type | Constraints |
|-------|------|-------------|
| `id` | UUID | Primary key, auto-generated |
| `name` | String | 1–140 characters, required |
| `description` | String | Optional; max 2000 characters |
| `category` | Enum | concert \| conference \| meetup \| festival \| sports; required |
| `venue_id` | UUID | FK to Venue; required |
| `organizer_id` | UUID | FK to User (role=organizer); required |
| `start_at` | ISO 8601 Datetime | UTC; must be in future; required |
| `end_at` | ISO 8601 Datetime | UTC; must be > start_at; required |
| `status` | Enum | draft \| published \| cancelled; default draft |
| `capacity_override` | Integer | Optional; if set, ≤ venue.capacity; used for ticketing |
| `base_price` | Decimal | Required; >= 0.00; supports cents (e.g., 2500.50) |
| `currency` | Enum | KES \| TZS \| RWF \| UGX \| USD; required |
| `sales_start_at` | ISO 8601 Datetime | When ticket sales open; required |
| `sales_end_at` | ISO 8601 Datetime | When ticket sales close; must be <= start_at; required |
| `created_at` | Timestamp | Auto-set at creation |
| `updated_at` | Timestamp | Auto-updated on modify |

**Business Rules**:
- Cannot publish unless: status=draft, start_at is in the future, sales_start_at <= now <= sales_end_at.
- If `venue.allows_overlap=false`, no two published events at the same venue can have overlapping [start_at, end_at] intervals.
- `effective_capacity` = capacity_override if set, else venue.capacity.
- Cannot delete if tickets already sold; must cancel instead.
- Organizer must have permission to modify or cancel.

---

### Venue Resource
**Purpose**: A physical location hosting events. Encapsulates capacity, timezone, and scheduling rules to enforce real-world constraints.

**Schema**:
| Field | Type | Constraints |
|-------|------|-------------|
| `id` | UUID | Primary key, auto-generated |
| `name` | String | 1–100 characters, required |
| `address` | String | Full street address; required |
| `city` | String | City name (Nairobi, Kigali, etc.); required |
| `country` | String | Country code or name; required |
| `latitude` | Decimal | Range [-90, 90]; required if longitude set |
| `longitude` | Decimal | Range [-180, 180]; required if latitude set |
| `capacity` | Integer | > 0; required |
| `timezone` | String | IANA timezone (e.g., Africa/Nairobi); required |
| `allows_overlap` | Boolean | If false, no concurrent published events; default false |
| `contact_email` | String | Valid email; optional |
| `contact_phone` | String | E.164 format (+254...); optional |
| `created_at` | Timestamp | Auto-set |
| `updated_at` | Timestamp | Auto-updated |

**Business Rules**:
- Capacity must be positive and reflects physical venue limits.
- Cannot decrease capacity below current ticket sales for active events.
- Timezone ensures local time scheduling (e.g., "2 PM local" vs. UTC).
- Cannot delete if future published events exist.

---

### User Resource
**Purpose**: Represents any actor in the system: event organizers, ticket purchasers (attendees), or venue staff performing check-ins.

**Schema**:
| Field | Type | Constraints |
|-------|------|-------------|
| `id` | UUID | Primary key, auto-generated |
| `full_name` | String | 1–100 characters; required |
| `email` | String | Valid email; unique across system; required |
| `phone` | String | E.164 format (+254...) if provided; unique; optional |
| `role` | Enum | organizer \| attendee \| staff; required |
| `verified_email` | Boolean | Confirms email ownership; default false |
| `verified_phone` | Boolean | Confirms SMS/OTP; default false |
| `created_at` | Timestamp | Auto-set |
| `updated_at` | Timestamp | Auto-updated |

**Business Rules**:
- Email is unique and must be verified before some operations (e.g., payment).
- Role determines permissions: organizers can create/publish events; staff can perform check-ins.
- Cannot delete if user is organizer of active published events or has unchecked-in paid tickets.
- Phone verification may be required for regional payment processors (M-Pesa, etc.).

---

### Ticket Resource
**Purpose**: The core transaction record linking an attendee to an event. Captures purchase details and tracks state through payment to check-in.

**Schema**:
| Field | Type | Constraints |
|-------|------|-------------|
| `id` | UUID | Primary key, auto-generated |
| `event_id` | UUID | FK to Event; required |
| `user_id` | UUID | FK to User (attendee); required |
| `status` | Enum | reserved \| paid \| checked_in \| cancelled; default reserved |
| `price_paid` | Decimal | Amount in specified currency; >= 0.00; required |
| `currency` | String | KES \| TZS \| RWF \| UGX \| USD; required |
| `qr_code` | String | Unique per ticket; random; used for check-in scanning |
| `payment_ref` | String | External payment reference (M-Pesa txn ID, card auth code); optional |
| `purchased_at` | Timestamp | When payment completed; set when status → paid |
| `checked_in_at` | Timestamp | When ticket scanned at gate; set when status → checked_in |
| `created_at` | Timestamp | Auto-set (initial reserve) |
| `updated_at` | Timestamp | Auto-updated |

**Business Rules**:
- QR code must be unique across all tickets and all time.
- Total tickets for an event cannot exceed event.effective_capacity.
- Status transitions allowed: reserved→paid, paid→checked_in, paid→cancelled, reserved→cancelled.
- checked_in is a terminal state (no further transitions).
- Cannot create ticket if event.status=cancelled or sales window closed.
- User can only check in if ticket status=paid.

---

### Check-in Resource
**Purpose**: An audit log entry recording attendance verification. Separate from tickets to maintain immutable history and enable reporting.

**Schema**:
| Field | Type | Constraints |
|-------|------|-------------|
| `id` | UUID | Primary key, auto-generated |
| `ticket_id` | UUID | FK to Ticket; required |
| `scanned_by` | UUID | FK to User (role=staff); required |
| `gate` | String | Gate identifier (e.g., "A", "Main Entrance"); optional |
| `scanned_at` | Timestamp | When QR scanned; auto-set to now |
| `created_at` | Timestamp | Auto-set |

**Business Rules**:
- Ticket must be in paid status; cannot check in reserved or cancelled tickets.
- One successful check-in per ticket (idempotent: re-scanning same QR returns existing record, 409 conflict if already checked in).
- Scanned by must have role=staff.
- Enables audit trail: reports can count check-ins vs. ticket sales for actual attendance metrics.

---

## Endpoint Documentation

**Base URL**: `https://api.hadathub.com/v1`

All endpoints listed below are relative to this base URL. For example:
- Full URL for listing events: `https://api.hadathub.com/v1/events`
- Full URL for creating a ticket: `https://api.hadathub.com/v1/tickets`

### HTTP Status Codes Reference

| Code | Meaning | Use Case |
|------|---------|----------|
| **200** | OK | Successful GET, PUT on existing resource |
| **201** | Created | Successful POST creating new resource |
| **204** | No Content | Successful DELETE; no response body |
| **400** | Bad Request | Invalid query parameters, malformed JSON, validation failure |
| **401** | Unauthorized | Missing or invalid authentication token |
| **403** | Forbidden | Valid auth, but insufficient permissions (e.g., attendee trying to publish event) |
| **404** | Not Found | Resource (event, user, ticket) does not exist |
| **409** | Conflict | Constraint violation: sold out, overlapping events, already checked-in, email duplicate |
| **422** | Unprocessable Entity | Request semantically correct but cannot process (e.g., end_at before start_at) |
| **429** | Too Many Requests | Rate limit exceeded |
| **500** | Internal Server Error | Unexpected server error |

---

### Event Endpoints

| Resource | Action | Method | URI | Request Body Example | Success Response (Example) | Error Responses |
|----------|--------|--------|-----|----------------------|---------------------------|-----------------|
| Event | Create | POST | /events | `{"name":"Nairobi DevFest 2026","description":"Annual tech conference","venue_id":"550e8400-e29b-41d4-a716-446655440000","organizer_id":"660e8400-e29b-41d4-a716-446655440001","start_at":"2026-02-20T09:00:00Z","end_at":"2026-02-20T18:00:00Z","category":"conference","base_price":2500.00,"currency":"KES","sales_start_at":"2026-01-20T00:00:00Z","sales_end_at":"2026-02-20T08:00:00Z"}` | **201 Created**: `{"id":"770e8400-...","name":"Nairobi DevFest 2026","status":"draft","organizer_id":"660e8400-...","venue_id":"550e8400-...","start_at":"2026-02-20T09:00:00Z","end_at":"2026-02-20T18:00:00Z","base_price":2500.00,"currency":"KES","created_at":"2026-01-13T10:30:00Z","updated_at":"2026-01-13T10:30:00Z"}` | **400**: `{"error":"invalid_dates","message":"end_at must be after start_at"}` **401**: `{"error":"unauthorized","message":"Missing auth token"}` **403**: `{"error":"forbidden","message":"Only organizers can create events"}` **409**: `{"error":"overlap_conflict","message":"Event overlaps with existing event at this venue"}` **422**: `{"error":"validation_failed","message":"base_price must be >= 0"}` |
| Event | List | GET | /events?city=Nairobi&category=conference&from=2026-02-01&to=2026-02-28&status=published&price_lte=5000&page=1&limit=20&sort=start_at:asc | n/a | **200 OK**: `{"data":[{"id":"770e8400-...","name":"Nairobi DevFest","city":"Nairobi","start_at":"2026-02-20T09:00:00Z","base_price":2500.00,...},{...}],"pagination":{"page":1,"limit":20,"total":42,"pages":3}}` | **400**: `{"error":"bad_query","message":"Invalid date format; use ISO 8601"}` |
| Event | Retrieve | GET | /events/{event_id} | n/a | **200 OK**: `{"id":"770e8400-...","name":"Nairobi DevFest 2026","description":"...","category":"conference",...}` | **404**: `{"error":"not_found","message":"Event not found"}` |
| Event | Update | PUT | /events/{event_id} | `{"name":"Updated: Nairobi DevFest 2026","status":"published"}` | **200 OK**: `{"id":"770e8400-...","name":"Updated: Nairobi DevFest 2026","status":"published",...}` | **400**: `{"error":"invalid_state_transition","message":"Cannot publish without a future start_at"}` **404**: `{"error":"not_found","message":"Event not found"}` **409**: `{"error":"overlap_conflict","message":"Publishing creates overlap"}` |
| Event | Delete | DELETE | /events/{event_id} | n/a | **204 No Content** | **404**: `{"error":"not_found","message":"Event not found"}` **409**: `{"error":"has_sales","message":"Cannot delete; tickets already sold. Use cancel instead."}` |

---

### Venue Endpoints

| Resource | Action | Method | URI | Request Body Example | Success Response (Example) | Error Responses |
|----------|--------|--------|-----|----------------------|---------------------------|-----------------|
| Venue | Create | POST | /venues | `{"name":"Kigali Convention Centre","address":"KCC Building, Kigali","city":"Kigali","country":"Rwanda","latitude":-1.9536,"longitude":30.0605,"capacity":5000,"timezone":"Africa/Kigali","allows_overlap":false,"contact_email":"info@kcc.rw"}` | **201 Created**: `{"id":"550e8400-...","name":"Kigali Convention Centre","city":"Kigali","capacity":5000,"timezone":"Africa/Kigali",...}` | **400**: `{"error":"invalid_coords","message":"latitude and longitude must both be present"}` **422**: `{"error":"validation_failed","message":"capacity must be > 0"}` |
| Venue | List | GET | /venues?city=Kigali&page=1&limit=20 | n/a | **200 OK**: `{"data":[{"id":"550e8400-...","name":"Kigali Convention Centre","city":"Kigali",...}],"pagination":{"page":1,"limit":20,"total":8,"pages":1}}` | **400**: `{"error":"bad_query","message":"Invalid query parameters"}` |
| Venue | Retrieve | GET | /venues/{venue_id} | n/a | **200 OK**: `{"id":"550e8400-...","name":"Kigali Convention Centre",...}` | **404**: `{"error":"not_found","message":"Venue not found"}` |
| Venue | Update | PUT | /venues/{venue_id} | `{"capacity":5500,"contact_phone":"+250788123456"}` | **200 OK**: `{"id":"550e8400-...","capacity":5500,...}` | **400**: `{"error":"invalid_capacity","message":"Cannot reduce capacity below current ticket sales"}` **404**: `{"error":"not_found","message":"Venue not found"}` |
| Venue | Delete | DELETE | /venues/{venue_id} | n/a | **204 No Content** | **404**: `{"error":"not_found","message":"Venue not found"}` **409**: `{"error":"has_future_events","message":"Venue has future published events"}` |

---

### User Endpoints

| Resource | Action | Method | URI | Request Body Example | Success Response (Example) | Error Responses |
|----------|--------|--------|-----|----------------------|---------------------------|-----------------|
| User | Create | POST | /users | `{"full_name":"Amina Musa","email":"amina@example.com","phone":"+254712345678","role":"attendee"}` | **201 Created**: `{"id":"660e8400-...","full_name":"Amina Musa","email":"amina@example.com","role":"attendee","verified_email":false,...}` | **409**: `{"error":"email_exists","message":"Email already registered"}` **422**: `{"error":"validation_failed","message":"Invalid email format"}` |
| User | List | GET | /users?role=organizer&page=1&limit=20 | n/a | **200 OK**: `{"data":[{"id":"660e8400-...","full_name":"John Doe","role":"organizer",...}],"pagination":{"page":1,"limit":20,"total":15,"pages":1}}` | **400**: `{"error":"bad_query","message":"Invalid role value"}` |
| User | Retrieve | GET | /users/{user_id} | n/a | **200 OK**: `{"id":"660e8400-...","full_name":"Amina Musa","email":"amina@example.com",...}` | **404**: `{"error":"not_found","message":"User not found"}` |
| User | Update | PUT | /users/{user_id} | `{"full_name":"Amina M.","verified_phone":true}` | **200 OK**: `{"id":"660e8400-...","full_name":"Amina M.",...}` | **404**: `{"error":"not_found","message":"User not found"}` **422**: `{"error":"validation_failed","message":"Invalid phone format"}` |
| User | Delete | DELETE | /users/{user_id} | n/a | **204 No Content** | **404**: `{"error":"not_found","message":"User not found"}` **409**: `{"error":"has_active_events","message":"User is organizer of published events"}` |

---

### Ticket Endpoints

| Resource | Action | Method | URI | Request Body Example | Success Response (Example) | Error Responses |
|----------|--------|--------|-----|----------------------|---------------------------|-----------------|
| Ticket | Create (Purchase) | POST | /tickets | `{"event_id":"770e8400-...","user_id":"660e8400-...","price_paid":2500.00,"currency":"KES","payment_ref":"mpesa_txn_ABC123"}` | **201 Created**: `{"id":"880e8400-...","event_id":"770e8400-...","user_id":"660e8400-...","status":"paid","price_paid":2500.00,"currency":"KES","qr_code":"QR_ABC123XYZ","purchased_at":"2026-01-13T10:35:00Z",...}` | **400**: `{"error":"sales_closed","message":"Event ticket sales are closed"}` **404**: `{"error":"not_found","message":"Event or User not found"}` **409**: `{"error":"sold_out","message":"Event is sold out; capacity reached"}` **402**: `{"error":"payment_pending","message":"Payment verification failed"}` |
| Ticket | List | GET | /tickets?event_id=770e8400-...&status=paid&page=1&limit=50 | n/a | **200 OK**: `{"data":[{"id":"880e8400-...","event_id":"770e8400-...","user_id":"660e8400-...","status":"paid",...}],"pagination":{"page":1,"limit":50,"total":234,"pages":5}}` | **400**: `{"error":"bad_query","message":"Invalid status value"}` |
| Ticket | Retrieve | GET | /tickets/{ticket_id} | n/a | **200 OK**: `{"id":"880e8400-...","event_id":"770e8400-...","status":"paid",...}` | **404**: `{"error":"not_found","message":"Ticket not found"}` |
| Ticket | Update | PUT | /tickets/{ticket_id} | `{"status":"cancelled"}` | **200 OK**: `{"id":"880e8400-...","status":"cancelled",...}` | **400**: `{"error":"invalid_transition","message":"Cannot cancel checked-in ticket"}` **404**: `{"error":"not_found","message":"Ticket not found"}` |
| Ticket | Delete | DELETE | /tickets/{ticket_id} | n/a | **204 No Content** | **404**: `{"error":"not_found","message":"Ticket not found"}` **409**: `{"error":"checked_in","message":"Cannot delete checked-in ticket"}` |

---

### Check-in Endpoints

| Resource | Action | Method | URI | Request Body Example | Success Response (Example) | Error Responses |
|----------|--------|--------|-----|----------------------|---------------------------|-----------------|
| Check-in | Create (Scan) | POST | /check-ins | `{"ticket_id":"880e8400-...","gate":"Gate A"}` | **201 Created**: `{"id":"990e8400-...","ticket_id":"880e8400-...","scanned_by":"staff_uuid","gate":"Gate A","scanned_at":"2026-02-20T09:15:00Z"}` (Note: ticket status updated to checked_in) | **404**: `{"error":"not_found","message":"Ticket not found"}` **409**: `{"error":"already_checked_in","message":"Ticket already checked in"}` **409**: `{"error":"ticket_not_paid","message":"Ticket must be paid to check in"}` |
| Check-in | List by Event | GET | /check-ins?event_id=770e8400-...&page=1&limit=50 | n/a | **200 OK**: `{"data":[{"id":"990e8400-...","ticket_id":"880e8400-...","gate":"Gate A","scanned_at":"2026-02-20T09:15:00Z",...}],"pagination":{"page":1,"limit":50,"total":450,"pages":9}}` | **400**: `{"error":"bad_query","message":"Missing required event_id"}` |

---

## Advanced Features & Specialized Endpoints

Beyond basic CRUD, the HadatHub API supports complex business workflows:

### 1. Resource Associations (One-to-Many Queries)

**Get all events at a specific venue**
```
GET /venues/{venue_id}/events?status=published&page=1&limit=20

Response (200 OK):
{
  "data": [
    {
      "id": "event_uuid",
      "name": "Nairobi Tech Summit",
      "start_at": "2026-02-20T09:00:00Z",
      "ticket_count": 450,
      "available_capacity": 50
    }
  ],
  "pagination": {"page": 1, "limit": 20, "total": 8}
}
```

**Get all tickets for a specific event**
```
GET /events/{event_id}/tickets?status=paid&page=1&limit=50

Response (200 OK):
{
  "data": [
    {
      "id": "ticket_uuid",
      "user_id": "user_uuid",
      "status": "paid",
      "price_paid": 2500.00,
      "currency": "KES",
      "purchased_at": "2026-02-10T14:30:00Z"
    }
  ],
  "pagination": {"page": 1, "limit": 50, "total": 234}
}
```

**Get all tickets owned by a specific user (attendee)**
```
GET /users/{user_id}/tickets?status=paid&sort=purchased_at:desc

Response (200 OK):
{
  "data": [
    {
      "id": "ticket_uuid",
      "event_id": "event_uuid",
      "event_name": "Nairobi Tech Summit",
      "status": "paid",
      "checked_in_at": "2026-02-20T09:15:00Z"
    }
  ]
}
```

**Get all attendees for a specific event**
```
GET /events/{event_id}/attendees?checked_in=true

Response (200 OK):
{
  "data": [
    {
      "user_id": "user_uuid",
      "full_name": "Amina Musa",
      "email": "amina@example.com",
      "checked_in": true,
      "checked_in_at": "2026-02-20T09:15:00Z"
    }
  ],
  "pagination": {"page": 1, "limit": 100, "total": 415}
}
```

---

### 2. Advanced Search & Filtering

**Search events by multiple criteria**
```
GET /events/search?city=Kampala&category=concert&from_date=2026-03-01&to_date=2026-03-31&price_min=5000&price_max=50000&currency=UGX&status=published&q=music&page=1&limit=20&sort=start_at:asc

Response (200 OK):
{
  "data": [
    {
      "id": "event_uuid",
      "name": "Kampala Music Fest 2026",
      "category": "concert",
      "venue_city": "Kampala",
      "start_at": "2026-03-15T18:00:00Z",
      "base_price": 25000.00,
      "currency": "UGX",
      "available_tickets": 200
    }
  ],
  "pagination": {"page": 1, "limit": 20, "total": 45}
}

Error (400 Bad Request if invalid date range):
{
  "error": "invalid_date_range",
  "message": "from_date must be before to_date"
}
```

**Search venues by location and capacity**
```
GET /venues/search?city=Dar%20es%20Salaam&min_capacity=1000&max_capacity=5000&country=Tanzania

Response (200 OK):
{
  "data": [
    {
      "id": "venue_uuid",
      "name": "Dar Convention Hall",
      "city": "Dar es Salaam",
      "capacity": 2500,
      "allows_overlap": false
    }
  ]
}
```

---

### 3. Domain-Specific Actions

**Publish an event (status transition from draft)**
```
POST /events/{event_id}/publish

Request Body:
{
  "notify_subscribers": true
}

Response (200 OK):
{
  "id": "event_uuid",
  "status": "published",
  "published_at": "2026-01-13T10:45:00Z",
  "message": "Event published successfully; tickets now on sale"
}

Error (400 Bad Request):
{
  "error": "invalid_state",
  "message": "Cannot publish: start_at is in the past or sales window is closed"
}
```

**Cancel an event (status transition with cascading effects)**
```
POST /events/{event_id}/cancel

Request Body:
{
  "reason": "Venue has a scheduling conflict",
  "refund_tickets": true
}

Response (200 OK):
{
  "id": "event_uuid",
  "status": "cancelled",
  "cancelled_at": "2026-02-10T15:00:00Z",
  "tickets_refunded": 234,
  "refund_amount_total": 585000.00,
  "currency": "KES",
  "message": "Event cancelled; all ticket holders notified and refunded"
}

Error (409 Conflict):
{
  "error": "already_cancelled",
  "message": "Event is already cancelled"
}
```

**Check-in multiple attendees (bulk operation for staff)**
```
POST /check-ins/bulk

Request Body:
{
  "ticket_ids": ["ticket_uuid_1", "ticket_uuid_2", "ticket_uuid_3"],
  "gate": "Main Entrance"
}

Response (200 OK):
{
  "success_count": 3,
  "failure_count": 0,
  "checked_in": [
    {
      "ticket_id": "ticket_uuid_1",
      "user": "Amina Musa",
      "checked_in_at": "2026-02-20T09:15:00Z"
    }
  ],
  "errors": []
}
```

**Refund a ticket**
```
POST /tickets/{ticket_id}/refund

Request Body:
{
  "reason": "Attendee requested cancellation"
}

Response (200 OK):
{
  "id": "ticket_uuid",
  "status": "cancelled",
  "refund_amount": 2500.00,
  "currency": "KES",
  "refunded_at": "2026-02-15T11:00:00Z",
  "message": "Refund processed; attendee will receive KES 2500 within 24 hours"
}

Error (409 Conflict):
{
  "error": "already_checked_in",
  "message": "Cannot refund already checked-in tickets"
}
```

---

### 4. Reporting & Analytics Endpoints

**Event sales report**
```
GET /reports/events/{event_id}/sales

Response (200 OK):
{
  "event": {
    "id": "event_uuid",
    "name": "Nairobi DevFest",
    "start_at": "2026-02-20T09:00:00Z"
  },
  "sales_summary": {
    "capacity": 1000,
    "tickets_sold": 850,
    "tickets_reserved": 50,
    "tickets_cancelled": 100,
    "revenue_gross": 2125000.00,
    "revenue_net": 1980000.00,
    "currency": "KES"
  },
  "attendance": {
    "checked_in": 780,
    "no_show": 70
  },
  "sales_by_day": [
    {"date": "2026-01-20", "tickets": 10, "revenue": 25000.00},
    {"date": "2026-01-21", "tickets": 45, "revenue": 112500.00}
  ]
}
```

**Venue occupancy report**
```
GET /reports/venues/{venue_id}/events?from=2026-01-01&to=2026-12-31

Response (200 OK):
{
  "venue": {
    "id": "venue_uuid",
    "name": "Kigali Convention Centre",
    "capacity": 5000
  },
  "period": {"from": "2026-01-01", "to": "2026-12-31"},
  "events_hosted": 12,
  "total_attendees": 4250,
  "avg_occupancy": 0.71,
  "peak_event": {
    "name": "Rwanda Tech Summit",
    "attendees": 850
  }
}
```

---

### 5. Error Response Format (Consistent Across All Endpoints)

All error responses follow this standard JSON structure:

```json
{
  "error": "error_code_snake_case",
  "message": "Human-readable error description",
  "status": 400,
  "timestamp": "2026-01-13T10:50:00Z",
  "details": {
    "field_or_context": "Additional error details if applicable"
  }
}
```

**Examples**:

**Invalid request (400)**:
```json
{
  "error": "validation_failed",
  "message": "Request validation failed",
  "status": 400,
  "details": {
    "base_price": "Must be a positive decimal",
    "sales_end_at": "Must be before or equal to event start_at"
  }
}
```

**Unauthorized (401)**:
```json
{
  "error": "unauthorized",
  "message": "Missing or invalid authentication token",
  "status": 401
}
```

**Forbidden (403)**:
```json
{
  "error": "forbidden",
  "message": "Insufficient permissions for this operation",
  "status": 403,
  "details": {
    "required_role": "organizer",
    "user_role": "attendee"
  }
}
```

**Conflict (409)**:
```json
{
  "error": "sold_out",
  "message": "Event is sold out; no tickets available",
  "status": 409,
  "details": {
    "capacity": 1000,
    "tickets_sold": 1000,
    "event_id": "event_uuid"
  }
}
```

---

## Design Rationale

### Why These Five Core Resources?

**Event, Venue, User, Ticket, Check-in** map directly to the business workflow:

1. **Venue** anchors all scheduling; capacity limits are venue-driven and immutable constraints.
2. **Event** is discoverable; users search by city (via venue), date, and category—essential for attendee experience.
3. **User** unifies three roles (organizer, attendee, staff) under one identity model, reducing schema complexity and enabling future role evolution (e.g., promoters, discounters).
4. **Ticket** is the transaction record; separating it from users allows one person to buy multiple tickets and supports future features (gift tickets, transfers).
5. **Check-in** is a separate audit entity, not a ticket status alone, enabling immutable attendance logs and performance reports without re-scanning the same ticket.

### REST Compliance Decisions

**Plural Resource Names** (`/events`, `/venues`, `/users`, `/tickets`, `/check-ins`): Easier for developers to predict URI structure and aligns with RESTful conventions.

**Stateless State Transitions**: Event status flows (draft → published → cancelled) are not implicit; they require explicit POST to endpoints like `/events/{event_id}/publish`. This prevents accidental state corruption and makes audit logs clear.

**HTTP Method Semantics**:
- **POST /events**: Create new event (idempotent if we return existing with 200/409).
- **GET /events**: List with filtering and pagination; no side effects.
- **PUT /events/{event_id}**: Replace/update event attributes.
- **DELETE /events/{event_id}**: Only allowed if no tickets sold; otherwise, use POST `.../cancel` action.
- **POST /check-ins**: Create check-in record (scan operation is a write, not a read).

**Status Codes**:
- **201 Created**: For POST creating new resources.
- **200 OK**: For GET, PUT, PATCH returning data.
- **204 No Content**: For DELETE with no response body needed.
- **400 Bad Request**: Malformed input or validation failure (invalid dates, negative price).
- **401 Unauthorized**: Missing token or invalid signature.
- **403 Forbidden**: Valid token, but user role insufficient (e.g., attendee trying to publish).
- **404 Not Found**: Resource doesn't exist.
- **409 Conflict**: Constraint violation (e.g., sold out, overlapping events, duplicate email, already checked-in).
- **422 Unprocessable Entity**: Semantically valid but cannot process (e.g., circular or illogical data).

### Capacity & Scheduling Constraints

**Venue-Level `allows_overlap` Flag**: 
- Single-purpose venues (e.g., theater, small hall) set `allows_overlap=false`; multi-purpose venues (e.g., convention center) set `true`.
- Checked at event publish time, not at list time, to avoid expensive queries.

**`capacity_override` on Events**:
- Organizers may rope off part of a venue (e.g., 500 of 5000 seats for a VIP section).
- Prevents creating separate event records for partial venue usage.

**Ticket Count ≤ Effective Capacity**:
- Enforced at ticket creation time; returning 409 Conflict if sold out avoids double-selling.
- Future enhancement: Implement optimistic locking or distributed locks for high-concurrency scenarios (e.g., flash sales).

### Currency Strategy

**Multi-Currency Support**:
- Each event and ticket specifies its currency (KES, TZS, RWF, UGX, USD).
- Prices stored as decimals (e.g., 2500.50 KES) to avoid floating-point rounding errors.
- Exchange rates handled at the application layer, not in the API—keeps API responses transparent and region-specific.

**Why Not a Single Global Currency?**
- East African platforms operate in local currencies; forcing USD conversion adds latency and trust issues.
- Attendees expect prices in their local currency.

### Why Separate Check-in from Ticket Status?

**Audit Immutability**:
- A ticket's `checked_in` status is set once by a single check-in record.
- Separate check-in records allow venues to run reports: "How many unique attendees checked in?" vs. "How many tickets were sold?"

**Idempotency for QR Scanning**:
- Scanning the same QR code twice returns the same check-in record (409 conflict on second attempt).
- Staff at gates don't need to worry about double-scanning crashing the system.

### Pagination & Filtering Strategy

**Default Pagination** (`limit=20`):
- Lists grow unbounded; pagination prevents timeout/memory exhaustion on city-wide queries.
- Clients specify `page` and `limit`; cursor-based pagination (returning next/prev links) is left to v2 for simplicity.

**Filters as Query Parameters**:
- `?city=Nairobi&category=conference&from=2026-02-01`: Familiar to developers; cacheable by CDNs.
- Filters are optional; missing filters return all results (paginated).

**Sorting** (`sort=start_at:asc`):
- Allows users to sort by date, price, relevance (future feature with elasticsearch).
- Defaults to creation order if not specified.

### Authentication & Authorization (Not Detailed in This Design Doc)

- Token-based (JWT or OAuth2); passed in Authorization header.
- Roles determine permissions:
  - **Organizer**: Can create/publish/cancel their own events; read reports.
  - **Attendee**: Can list events, purchase tickets, cancel reservations.
  - **Staff**: Can check-in attendees.
- Implemented via middleware; not exposed in API contracts.

### Why No Explicit User Registration Endpoint?

- `POST /users` creates a user record (implicit registration).
- Email + password verification is an auth concern, not an API concern—handled by login/signup flow, not documented here.
- This design assumes auth is a separate service (Auth0, Firebase, custom).

---
## AI Tool Usage Statement

Then AI assistance (Claude 4.5 code max by Anthropic) was used for:
-  Formatting; content reviewed and edited by hand.

All core API design decisions, resource modeling, endpoint specifications, business logic, constraint definitions, and architectural rationale were independently developed based on the assignment requirements and my understanding of RESTful API principles and the East African event management ecosystem.

## Document Metadata

- **Document Version**: 1.0
- **Last Updated**: January 13, 2026
- **Assignment**: Week 1 Project: REST API Design Fundamentals (HadatHub)
- **Course**: Advanced Python Programming (ALU)
- **Author**: Levis Ishimwe
- **Email**: i.levis@alustudent.com

---
