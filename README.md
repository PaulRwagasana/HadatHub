# HadatHub Regional Event & Ticketing System - API Architecture Design

## Executive Summary
This REST API design provides a scalable, intuitive foundation for HadatHub's regional event management platform. The architecture follows resource-oriented design principles with clear separation of concerns between events, venues, tickets, and users. Special attention is given to East African business contexts including multi-venue scheduling, flexible pricing models, and mobile-first consumption patterns.

## Domain Analysis

### Business Entities and Relationships
**Primary Entities:**
1. **Events** - Core business object representing concerts, conferences, festivals
2. **Venues** - Physical locations hosting events with capacity constraints
3. **Users** - System participants (attendees, organizers, administrators)
4. **Tickets** - Purchased access rights to specific events
5. **Organizers** - Event creators and managers (subset of Users with special permissions)

**Critical Business Relationships:**
- One Venue can host multiple Events, but not simultaneously (scheduling constraint)
- One Event occurs at exactly one Venue (with potential for virtual events extension)
- One User can purchase multiple Tickets for different Events
- One Ticket grants access to exactly one Event for one User
- Organizers create and manage Events they own

**Key Business Operations:**
1. Event discovery and filtering by location, date, category
2. Ticket purchase flow with inventory management
3. Attendee check-in at event venue
4. Event creation and management by organizers
5. Venue capacity tracking and conflict prevention

**Data Points Rationale:**
- **Events** require date/time, pricing tiers, category tags, and organizer details to support diverse East African events from tech conferences to music festivals
- **Venues** need precise location data, amenities, and capacity limits critical for regional compliance and attendee planning
- **Tickets** must include purchase timestamp, price paid, and validation status for audit trails and fraud prevention
- **Users** require contact information and preference settings optimized for mobile usage patterns in East Africa

## Resource Specifications

### 1. Event Resource
**Purpose:** Represents any scheduled occurrence requiring ticketed access

**Schema:**
```json
{
  "id": "uuid",
  "title": "string",
  "description": "text",
  "category": "enum['concert', 'conference', 'festival', 'workshop']",
  "start_datetime": "ISO 8601",
  "end_datetime": "ISO 8601",
  "venue_id": "uuid (foreign key)",
  "organizer_id": "uuid (foreign key)",
  "base_price": "decimal",
  "currency": "string (ISO 4217)",
  "max_attendees": "integer",
  "current_attendee_count": "integer",
  "status": "enum['draft', 'published', 'sold_out', 'cancelled', 'completed']",
  "image_url": "string (URL)",
  "tags": "array[string]"
}
```

**Constraints:**
- `start_datetime` must be after current time when creating
- `max_attendees` cannot exceed linked venue capacity
- Event dates must not conflict with other events at same venue
- Only published events can accept ticket purchases

### 2. Venue Resource
**Purpose:** Physical location where events are hosted

**Schema:**
```json
{
  "id": "uuid",
  "name": "string",
  "address": "string",
  "city": "string",
  "country": "string",
  "coordinates": {
    "latitude": "decimal",
    "longitude": "decimal"
  },
  "capacity": "integer",
  "amenities": "array[enum['parking', 'wifi', 'catering', 'accessible']]",
  "contact_phone": "string",
  "contact_email": "string",
  "operating_hours": "text",
  "status": "enum['active', 'inactive', 'renovating']"
}
```

**Constraints:**
- `capacity` must be positive integer
- `coordinates` required for location-based search
- Venue cannot be deleted if future events are scheduled

### 3. User Resource
**Purpose:** Anyone interacting with the system

**Schema:**
```json
{
  "id": "uuid",
  "email": "string (unique)",
  "phone": "string",
  "full_name": "string",
  "role": "enum['attendee', 'organizer', 'admin']",
  "preferences": {
    "notifications": "boolean",
    "currency_preference": "string"
  },
  "created_at": "ISO 8601",
  "last_login": "ISO 8601"
}
```

**Constraints:**
- Email must be unique and validated
- Phone format must match regional patterns
- Role changes require administrative approval

### 4. Ticket Resource
**Purpose:** Proof of purchase and event access

**Schema:**
```json
{
  "id": "uuid",
  "event_id": "uuid (foreign key)",
  "user_id": "uuid (foreign key)",
  "purchase_date": "ISO 8601",
  "price_paid": "decimal",
  "currency": "string",
  "status": "enum['active', 'checked_in', 'cancelled', 'refunded']",
  "qr_code": "string",
  "ticket_type": "enum['standard', 'vip', 'early_bird']",
  "seat_number": "string (optional)"
}
```

**Constraints:**
- Ticket creation triggers event attendee count increment
- Price paid cannot exceed event base price (except for admin overrides)
- Only active tickets can be checked in
- Tickets become invalid if event is cancelled

## Endpoint Documentation

### Events Endpoints

| Resource | Action | Method | URI | Request Example | Success Response | Error Response |
|----------|--------|--------|-----|-----------------|------------------|----------------|
| Events | List | GET | `/api/v1/events` | Query params: `?city=nairobi&category=concert` | `200 OK` with paginated events array | `400` Invalid query params |
| Events | Create | POST | `/api/v1/events` | `{"title": "Nairobi Tech Summit", "venue_id": "123"}` | `201 Created` with event object | `422` Validation error |
| Events | Retrieve | GET | `/api/v1/events/{id}` | N/A | `200 OK` with event object | `404` Event not found |
| Events | Update | PUT | `/api/v1/events/{id}` | `{"status": "published"}` | `200 OK` updated event | `403` Not authorized |
| Events | Delete | DELETE | `/api/v1/events/{id}` | N/A | `204 No Content` | `409` Cannot delete published event |
| Events | Publish | POST | `/api/v1/events/{id}/publish` | N/A | `200 OK` with published event | `422` Missing required fields |

### Venues Endpoints

| Resource | Action | Method | URI | Request Example | Success Response | Error Response |
|----------|--------|--------|-----|-----------------|------------------|----------------|
| Venues | List | GET | `/api/v1/venues` | `?capacity=gt:500&city=kigali` | `200 OK` with venues array | `400` Invalid filter |
| Venues | Create | POST | `/api/v1/venues` | Full venue schema | `201 Created` with venue object | `422` Validation error |
| Venues | Retrieve | GET | `/api/v1/venues/{id}` | N/A | `200 OK` with venue object | `404` Not found |
| Venues | Update | PUT | `/api/v1/venues/{id}` | Partial update | `200 OK` updated venue | `403` Not authorized |
| Venues | Delete | DELETE | `/api/v1/venues/{id}` | N/A | `204 No Content` | `409` Has future events |

### Users Endpoints

| Resource | Action | Method | URI | Request Example | Success Response | Error Response |
|----------|--------|--------|-----|-----------------|------------------|----------------|
| Users | Register | POST | `/api/v1/users/register` | User schema without ID | `201 Created` with user object | `409` Email exists |
| Users | Login | POST | `/api/v1/users/login` | `{"email": "", "password": ""}` | `200 OK` with auth token | `401` Invalid credentials |
| Users | Profile | GET | `/api/v1/users/{id}` | N/A | `200 OK` user object | `404` Not found |
| Users | Update | PUT | `/api/v1/users/{id}` | Partial update | `200 OK` updated user | `403` Not owner |
| Users | Upgrade | POST | `/api/v1/users/{id}/upgrade-organizer` | N/A | `200 OK` with new role | `400` Insufficient data |

### Tickets Endpoints

| Resource | Action | Method | URI | Request Example | Success Response | Error Response |
|----------|--------|--------|-----|-----------------|------------------|----------------|
| Tickets | Purchase | POST | `/api/v1/tickets` | `{"event_id": "abc", "user_id": "xyz"}` | `201 Created` with ticket | `409` Event sold out |
| Tickets | List by User | GET | `/api/v1/users/{id}/tickets` | N/A | `200 OK` tickets array | `404` User not found |
| Tickets | Retrieve | GET | `/api/v1/tickets/{id}` | N/A | `200 OK` ticket object | `404` Not found |
| Tickets | Cancel | POST | `/api/v1/tickets/{id}/cancel` | N/A | `200 OK` cancelled ticket | `400` Past event |
| Tickets | Check-in | POST | `/api/v1/tickets/{id}/check-in` | N/A | `200 OK` checked-in ticket | `409` Already checked in |

## Advanced Features

### Resource Associations

**Get User's Tickets:**
```
GET /api/v1/users/{user_id}/tickets
Response: Array of ticket objects with expanded event details
```

**Get Event Attendees:**
```
GET /api/v1/events/{event_id}/attendees
Response: Array of user objects (only attending users)
```

**Get Venue's Upcoming Events:**
```
GET /api/v1/venues/{venue_id}/events?status=published&start_date=gt:2026-02-01
Response: Paginated events at this venue
```

### Search and Filtering

**Comprehensive Event Search:**
```
GET /api/v1/events/search
Query Parameters:
- q: free text search (title, description, tags)
- city: filter by city
- category: filter by event type
- start_date / end_date: date range filter
- price_min / price_max: price range
- radius: kilometers from coordinates
- sort: relevance, date, price_asc, price_desc
```

**Example Search:**
```
GET /api/v1/events/search?city=Kampala&category=concert&start_date=gt:2026-03-01&price_max=50000&sort=date
```

### Domain-Specific Operations

**Bulk Ticket Purchase:**
```
POST /api/v1/events/{event_id}/bulk-tickets
Request: {"user_ids": ["id1", "id2", "id3"], "ticket_type": "vip"}
Response: Array of created tickets with individual status
Status: 207 Multi-Status for partial successes
```

**Event Check-in Management:**
```
POST /api/v1/events/{event_id}/check-in-bulk
Request: {"ticket_ids": ["t1", "t2"]}
Response: Summary of check-in results
```

**Event Cancellation with Refund:**
```
POST /api/v1/events/{event_id}/cancel
Request: {"reason": "Weather emergency", "issue_refunds": true}
Response: Cancellation confirmation with affected ticket count
```

**Venue Availability Check:**
```
GET /api/v1/venues/{venue_id}/availability
Query: date=2026-03-15
Response: {"available": true, "conflicting_events": []}
```

### Reporting Endpoints

**Sales Report for Organizer:**
```
GET /api/v1/organizers/{id}/reports/sales
Query: start_date, end_date, event_id (optional)
Response: Summary with total revenue, ticket counts by type
```

**Attendance Analytics:**
```
GET /api/v1/events/{id}/analytics/attendance
Response: Check-in rates, peak hours, demographic summary
```

## Design Rationale

### Key Decisions and Justifications

1. **Resource Separation of Users and Organizers**
   - Decision: Organizer as a role within User resource rather than separate resource
   - Rationale: Reduces duplication, simplifies authentication, allows users to transition between roles

2. **Embedded vs. Expanded Relationships**
   - Decision: Return foreign keys in basic responses, offer expansion via query parameters
   - Example: `GET /api/v1/events/{id}?expand=venue,organizer`
   - Rationale: Balances payload size with frontend convenience, follows JSON:API patterns

3. **Status-Based Event Lifecycle**
   - Decision: Explicit status field controlling valid operations
   - Rationale: Prevents invalid operations (e.g., selling tickets to cancelled events), provides clear state machine

4. **Check-in as Ticket Operation**
   - Decision: Check-in modifies ticket status rather than creating separate resource
   - Rationale: Simplifies data model, maintains ticket history, supports mobile check-in scenarios

5. **Regional Considerations**
   - Currency field at both event and ticket levels supports multi-currency events
   - Phone number as first-class field accommodates East African SMS-based workflows
   - Location coordinates enable offline-first mobile experiences in areas with spotty connectivity

6. **Pagination Strategy**
   - Decision: Cursor-based pagination for large result sets
   - Example: `GET /api/v1/events?cursor=abc123&limit=20`
   - Rationale: Better performance for infinite scroll on mobile clients, consistent ordering

This design provides a robust foundation for HadatHub's growth while maintaining simplicity for initial implementation. The API follows REST principles while accommodating real-world business constraints specific to the East African event management market.



**AI Usage Note:** I used an AI tool to assist with formatting this document, specifically for:
1. Structuring the Markdown tables to ensure proper alignment and readability
2. Formatting the JSON schema code blocks for consistency
3. Checking for proper HTTP status code usage and consistency
4. Ensuring consistent heading hierarchy and document organization


All business logic, API design decisions, resource modeling, and endpoint specifications were developed based on my analysis of the HadatHub requirements and REST API design principles.

