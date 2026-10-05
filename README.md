# Monuments Inventory, Booking & QR Verification System

A comprehensive software solution for managing monument inventory, accepting visitor bookings, generating QR tickets, verifying entry at counters, and providing operational dashboards and reports.

**Technology Stack:** ASP.NET Core Web API (C#) with ASP.NET Core MVC for administrative and public interfaces.

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Technology Stack](#technology-stack)
3. [User Roles & Permissions](#user-roles--permissions)
4. [Functional Requirements](#functional-requirements)
5. [System Architecture](#system-architecture)
6. [Core Data Entities](#core-data-entities)
7. [API Functional Groups](#api-functional-groups)
8. [Dashboard & Reports](#dashboard--reports)
9. [Nonfunctional Requirements](#nonfunctional-requirements)
10. [Key Acceptance Criteria](#key-acceptance-criteria)
11. [Development Phases](#development-phases)
12. [Repository Structure](#repository-structure)
13. [Getting Started](#getting-started)

---

## Project Overview

The Monuments Booking System enables administrators and staff to:

- ✅ Add and manage monuments/heritage sites with detailed inventory
- ✅ Configure flexible visiting time slots with recurring schedules
- ✅ Set limited or unlimited visitor capacity per slot
- ✅ Block dates or individual slots for maintenance/closures/special events
- ✅ Auto-publish bookable listings from inventory
- ✅ Accept visitor bookings through web and counter interfaces
- ✅ Generate secure QR tickets for entry verification
- ✅ Verify QR tickets at entry counters with atomic check-in
- ✅ Monitor bookings, attendance, capacity, and revenue through dashboards and reports

### MVP Assumptions

- Visitors book one monument and one time slot per booking
- Each booking can include multiple visitors
- Capacity is measured in visitors, not bookings
- QR verification requires internet connection
- One QR ticket per booking admits the entire group
- Payments are optional (supports free and paid entry)
- Each monument has a configured local time zone

---

## Technology Stack

### Backend Architecture

| Layer | Technology | Purpose |
|---|---|---|
| **API Server** | ASP.NET Core Web API (C#) | RESTful API for inventory, bookings, availability, verification, reports |
| **Admin UI** | ASP.NET Core MVC with Razor | Monument management, staff administration, inventory control |
| **Public UI** | ASP.NET Core MVC with Razor | Visitor booking, listing display, ticket retrieval, account management |
| **Data Access** | Entity Framework Core (EF Core) | ORM for database operations, migrations, query optimization |
| **Database** | To be confirmed | Relational database for inventory, bookings, payments, check-ins, audit logs |
| **Background Tasks** | ASP.NET Core Hosted Services | Notifications, hold expiry, report generation, cleanup tasks |
| **Caching** | In-Memory Cache / Redis (optional) | Session state, availability caching, rate limiting |
| **Authentication** | ASP.NET Core Identity / Custom | Staff authentication, role-based authorization, session management |
| **File Storage** | Local disk / Cloud storage | Monument images, generated reports, tickets |

### Architecture Principles

- **Single API, Multiple UI Surfaces:** One shared ASP.NET Core Web API handles all business logic (inventory, availability, bookings, verification)
- **Separation of Concerns:** API enforces authorization, capacity rules, and transaction safety; MVC UIs delegate to API
- **Admin MVC Interface:** Separate pages/area for monument managers to manage inventory, slots, closures, users, and reports
- **Public MVC Interface:** Separate pages/area for visitors to browse monuments, book tickets, and manage their bookings
- **Authorization Boundaries:**
  - Public endpoints expose only published monuments and available slots
  - Admin endpoints enforce staff authentication and monument-level authorization
  - Inventory modifications always route through API, never direct database writes from UI
- **Database as Source of Truth:** All capacity reservations, check-ins, and financial transactions are atomic in the database; UI layers do not hold exclusive locks or duplicate business logic

### Implementation Options

- **Single ASP.NET Core Project:** One MVC project with Areas (e.g., `/Areas/Admin`, `/Areas/Public`)
- **Multiple Projects:** Separate MVC projects for Admin and Public, both consuming the shared API
- **Deployment:** Can deploy as single instance or independently scale API and MVC services

---

## User Roles & Permissions

| Role | Main Permissions |
|---|---|
| **System Administrator** | Manage all monuments, users, settings, bookings, reports, audit logs |
| **Monument Manager** | Create and manage inventory for assigned monuments; view reports |
| **Booking Counter Operator** | Create walk-in bookings, retrieve existing bookings, issue tickets |
| **Entry Verification Operator** | Scan QR tickets, check eligibility, record admissions |
| **Visitor** | Browse published monuments, create bookings, retrieve tickets, manage bookings |
| **Auditor / Report Viewer** | View permitted reports and audit records (read-only) |

### Access Control

- Staff authentication required for all administrative and verification features
- Monument-specific permissions enforced at API layer
- Verification staff see only necessary visitor information for entry
- All changes to inventory, bookings, and check-ins are audited at the API level
- Public endpoints validate that monuments are published before returning data

---

## Functional Requirements

### 3.1 Monument Inventory Management

Administrators can create, edit, publish, unpublish, and archive monuments through the Admin MVC interface, which consumes the API.

#### Monument Fields

| Field | Requirement |
|---|---|
| Monument ID | System-generated unique identifier |
| Name | Required |
| Description | Visitor-facing information |
| Location | Address, city, state, country |
| Time zone | Required for booking and verification rules |
| Images | Main image + optional gallery (stored on file system or cloud) |
| Opening schedule | Operating days and hours |
| Ticket categories | Adult, child, student, or custom |
| Ticket prices | Required for paid entry |
| Capacity mode | Limited or unlimited |
| Booking window | How far in advance visitors may book (days) |
| Booking cutoff | Minimum time before slot starts that booking remains open |
| Maximum visitors per booking | Configurable |
| Entry instructions | Visitor-facing guidance |
| Status | Draft, published, inactive, archived |

#### Inventory Rules

- Only published monuments appear in public listings
- API endpoint `/api/monuments/public` returns only published monuments
- Existing bookings remain accessible after a monument is unpublished
- A monument with bookings cannot be permanently deleted
- Price changes do not alter existing booking records (prices are snapshots at booking time)
- Capacity cannot be reduced below confirmed visitors + active holds
- Admin UI shows warnings when operating hours or slots affect existing bookings

---

### 3.2 Time Slot Management

Each monument has multiple visiting slots configured through the Admin MVC interface.

#### Example Schedule

| Time Slot | Capacity Mode | Visitor Limit |
|---|---|---:|
| 09:00–10:00 | Limited | 100 |
| 10:00–11:00 | Limited | 150 |
| 11:00–12:00 | Unlimited | N/A |

#### Required Features

- ✅ Create recurring schedules by weekday
- ✅ Define slot start and end times
- ✅ Set capacity for each slot
- ✅ Create date-specific overrides
- ✅ Disable individual slots
- ✅ Prevent duplicate/overlapping slots
- ✅ Support configurable booking cutoff and entry grace periods
- ✅ Admin interface shows remaining capacity in real-time

#### Capacity Calculation

**For limited-capacity slots:**
```
Available capacity = Slot capacity 
                   − Confirmed visitors 
                   − Visitors in active reservation holds
```

**For unlimited-capacity slots:**
- No slot capacity limit
- Booking quantity limits and cutoffs still apply

---

### 3.3 Blocked Dates & Slots

Managers can block dates/slots through the Admin MVC interface by calling the API.

#### Block Record Fields

| Field | Description |
|---|---|
| Monument | Affected monument |
| Date or date range | Closure period |
| Slot | Optional; if omitted, entire day is blocked |
| Reason | Maintenance, holiday, private event, emergency |
| Public message | Optional visitor-facing notice |
| Audit info | Created by / created at |

#### Business Rules

- Blocked dates/slots reject new bookings at the API level
- If existing bookings are affected, Admin UI shows count before save
- Requires explicit decision: retain or cancel affected bookings
- Cancellations trigger refund processing and visitor notifications
- Emergency closures require documented authorization

---

### 3.4 Public Listings & Availability

Public listings are generated from published monument inventory via the API endpoint `/api/monuments/available`.

#### Listing Information

- Monument name and image
- Location and description
- Opening hours
- Ticket price or "Free entry"
- Available dates and slots for next 30/60/90 days (configurable)
- Remaining capacity for limited slots
- "Available" for unlimited slots
- Entry instructions and closure notices

#### Availability States

| State | Meaning |
|---|---|
| Available | Booking permitted |
| Sold out | Limited capacity exhausted |
| Closed | Date or slot is blocked |
| Booking closed | Booking cutoff passed |
| Unavailable | Monument unpublished or outside booking window |

**⚠️ Important:** Availability is validated again at the API level when confirming a booking. Public listing is not the final authority.

---

### 3.5 Booking System

#### Booking Workflow

1. Visitor selects monument from public MVC interface
2. Visitor selects date and time slot
3. Visitor selects ticket categories and quantities
4. Public MVC calls API endpoint `/api/bookings/calculate` to validate availability and price
5. Visitor provides contact details and accepts terms
6. Public MVC submits to API endpoint `/api/bookings/create`
7. **API validates:**
   - Monument is published
   - Slot is available
   - Capacity allows booking
   - Booking window and cutoff rules satisfied
   - If paid: initiates payment flow
8. API confirms booking and creates QR ticket in single transaction
9. Public MVC displays confirmation and ticket download
10. Visitor receives email/SMS confirmation

#### Booking Information

| Field | Description |
|---|---|
| Booking reference | Unique public reference (e.g., BKG-2026-001234) |
| Monument | Booked location |
| Visit date and slot | Admission period |
| Visitor name | Primary contact |
| Email / mobile | Required for delivery method |
| Ticket categories and quantities | Visitor breakdown |
| Total visitors | Capacity consumed |
| Price breakdown | Snapshot of prices, taxes, fees |
| Booking status | Pending, confirmed, cancelled, expired |
| Payment status | Separate lifecycle (not paid, pending, paid, refunded) |
| Booking channel | Website or counter |
| Audit info | Created at / created by |

#### Booking Statuses

- **Pending** – Awaiting confirmation or payment
- **Confirmed** – Ready for entry
- **Cancelled** – User or admin cancelled
- **Expired** – Hold or booking window expired

Attendance tracked separately:
- Not checked in
- Checked in

"No-show" = confirmed booking not admitted after entry window closes.

#### Capacity & Payment Safeguards at API Level

- ✅ Database transactions prevent overbooking
- ✅ Reservation holds expire after configurable interval (e.g., 15 minutes)
- ✅ Payment callbacks authenticated and processed idempotently
- ✅ Repeated API requests don't create duplicate bookings
- ✅ Cancellations release capacity atomically
- ✅ Payment success never trusted from browser redirect alone

---

### 3.6 QR Ticket Generation

Each confirmed booking receives a secure QR ticket generated by the API.

#### Ticket Display

- Booking reference
- Monument name
- Visit date and slot
- Ticket categories and visitor count
- QR code (PNG/SVG)
- Entry instructions
- Cancellation/entry conditions

#### QR Security

- ✅ Cryptographically random, opaque token
- ✅ Securely signed (JWT or similar)
- ✅ No personal information encoded in QR
- ✅ QR possession doesn't grant access to account details
- ✅ Cancelled tickets invalid immediately
- ✅ Reissued tickets revoke previous tokens
- ✅ Verification checks current server state, not just QR validity

---

### 3.7 Counter QR Verification

Entry verification operators access the verification interface (ASP.NET Core MVC or mobile app) to scan QR codes.

#### Verification Workflow

1. Operator signs in to verification interface
2. Selects authorized monument or gate
3. Scans QR code or enters booking reference
4. Interface calls API endpoint `/api/verification/preview` (read-only)
5. API returns ticket details and eligibility
6. Screen displays visitor count and entry status
7. Operator confirms admission or rejects
8. Interface calls API endpoint `/api/verification/checkin` (atomic write)
9. API validates:
   - Booking is confirmed
   - Correct monument
   - Within entry window (or override authorized)
   - Not previously checked in
   - Cancellation/revocation status
10. API records check-in atomically in database
11. Screen shows admission success or error

#### Verification Outcomes

| Result | Message |
|---|---|
| Eligible | Ready to admit; display visitor count |
| Admission recorded | Entry successfully recorded |
| Already checked in | Display previous entry time |
| Invalid QR | Ticket not recognized |
| Cancelled/revoked | Entry not permitted |
| Wrong monument | Ticket belongs elsewhere |
| Wrong date/outside window | Entry not permitted |
| Not confirmed | Booking ineligible |
| Network/server failure | Verification unavailable |

#### Duplicate-Entry Prevention

- ✅ Check-in must be atomic at database level
- ✅ If two counters admit same ticket simultaneously, only one succeeds
- ✅ Rejected scans don't modify state
- ✅ Repeated requests (retries) don't create duplicates
- ✅ Supervisor overrides require reason and audit record

---

## System Architecture

### High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                  Visitors / Public                          │
│           (Internet-facing, published inventory)            │
└────────────────────────┬────────────────────────────────────┘
                         │
          ┌──────────────▼──────────────┐
          │   Public MVC Interface      │
          │  (Browse, Book, Retrieve)   │
          └──────────────┬──────────────┘
                         │
┌────────────────────────┴──────────────────────────────────────┐
│         ASP.NET Core Web API (C#)                             │
│  (Authentication, Business Logic, Capacity, Verification)    │
│                                                               │
│  ├─ /api/monuments/public (published listings)               │
│  ├─ /api/availability (slots, capacity)                      │
│  ├─ /api/bookings (create, retrieve, cancel)                │
│  ├─ /api/tickets (QR generation, reissue)                    │
│  ├─ /api/verification (check-in, eligibility)               │
│  ├─ /api/reports (bookings, attendance, revenue)            │
│  └─ /api/admin (inventory, users, closures, audit)          │
└────────┬──────────────────────────────────────┬──────────────┘
         │                                      │
    ┌────▼───────────┐              ┌──────────▼─────────┐
    │  Admin MVC     │              │  Entity Framework   │
    │  Interface     │              │  Core (ORM)        │
    │  (Manage       │              └──────────┬─────────┘
    │   inventory,   │                        │
    │   staff,       │              ┌─────────▼────────┐
    │   reports)     │              │  Relational DB   │
    └────────────────┘              │  (Monuments,     │
                                     │   Bookings,      │
                                     │   Check-ins,     │
                                     │   Audit Logs)    │
                                     └──────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                  Entry Points                               │
│           (Staff with authentication)                       │
└────────────────────────┬────────────────────────────────────┘
                         │
          ┌──────────────▼──────────────┐
          │ Verification MVC / Mobile   │
          │  (Scan QR, Check-in)        │
          └──────────────┬──────────────┘
                         │
         ┌───────────────▼────────────┐
         │  API /verification endpoint │
         │  (Atomic check-in)          │
         └─────────────────────────────┘
```

### Technology Details

| Component | Technology | Responsibility |
|---|---|---|
| **Web API** | ASP.NET Core | Implements business logic, authorization, capacity rules, atomic transactions |
| **Admin UI** | ASP.NET Core MVC + Razor | Inventory management, staff administration, report viewing |
| **Public UI** | ASP.NET Core MVC + Razor | Visitor booking, ticket retrieval, account management |
| **ORM** | Entity Framework Core | Database abstraction, migrations, transaction management |
| **Authentication** | ASP.NET Core Identity | User login, password management, role-based authorization |
| **Hosting** | IIS / App Service / Docker | Web server and deployment platform |
| **Background Tasks** | Hosted Services (IHostedService) | Email notifications, hold expiry, report exports, cleanup |
| **Caching** | MemoryCache / Distributed Cache | Performance optimization for listings, availability checks |
| **Database** | *To be confirmed* | Relational database (SQL Server, PostgreSQL, or other) |

### Key Architectural Decisions

1. **Single API, Multiple UIs:** Prevents business logic duplication; all authorization and capacity rules in one place
2. **Database as Authority:** All capacity and state changes are atomic at database layer; UI layers cannot bypass checks
3. **API-First Design:** Admin and Public MVC interfaces are API clients; future mobile apps or integrations use same API
4. **Separation of Admin/Public:** Distinct MVC areas with different authorization ensure staff and visitor experiences don't collide
5. **EF Core for Data Access:** Automatic migrations, LINQ queries, and built-in transaction support
6. **Hosted Services for Background Work:** Asynchronous notifications and cleanup without external job queues (at MVP; can scale to Hangfire/Azure Functions later)

---

## Core Data Entities

### Entity Relationship Overview

| Entity | Purpose | Key Fields |
|---|---|---|
| **User** | Staff authentication and roles | ID, username, email, password hash, role |
| **Role** | Staff permissions (admin, manager, operator, auditor) | ID, name, permissions |
| **Monument** | Bookable location | ID, name, location, timezone, status (draft/published/inactive/archived) |
| **MonumentImage** | Monument gallery | ID, monumentId, imageUrl, displayOrder |
| **TicketCategory** | Visitor type pricing | ID, monumentId, name (Adult/Child/Student), price |
| **Schedule** | Recurring slot template | ID, monumentId, dayOfWeek, startTime, endTime, capacityMode, capacity |
| **SlotInstance** | A dated occurrence of a schedule | ID, scheduleId, date, capacity, remainingCapacity |
| **BlockedPeriod** | Closure date/slot | ID, monumentId, startDate, endDate, slotId (optional), reason, publicMessage |
| **Booking** | Visitor reservation | ID, reference, monumentId, slotInstanceId, visitorName, email, phone, totalVisitors, totalPrice, status, bookingChannel, createdAt |
| **BookingItem** | Quantity breakdown per category | ID, bookingId, ticketCategoryId, quantity, unitPrice, subtotal |
| **Payment** | Financial transaction | ID, bookingId, amount, method, status (pending/paid/failed/refunded), transactionId |
| **Ticket** | QR credential | ID, bookingId, token (opaque/JWT), qrCode, status (active/revoked/expired), createdAt |
| **CheckIn** | Admission record | ID, bookingId, monumentId, operatorId, checkInTime, checkInLocationId, status (success/failed) |
| **VerificationAttempt** | Audit log for verification | ID, ticketId, operatorId, monumentId, attemptTime, outcome (success/invalid/cancelled/etc.) |
| **AuditLog** | Administrative changes | ID, userId, entityType, entityId, action (create/update/delete), oldValues, newValues, timestamp |
| **Notification** | Email/SMS delivery | ID, bookingId, type (confirmation/cancellation/reminder), channel, status, sentAt, retryCount |
| **Report** | Generated export | ID, createdBy, reportType, filters, exportUrl, createdAt |

### Database Integrity Requirements

- ✅ Unique booking references and QR tokens
- ✅ Uniqueness preventing multiple check-ins per booking (MVP)
- ✅ Valid foreign-key relationships with cascade rules
- ✅ Nonnegative quantities and amounts
- ✅ Consistent currency handling (decimal or integer minor units, never float)
- ✅ Indexes on monument/date/slot for fast queries
- ✅ Indexes on booking reference and QR token for verification
- ✅ Timestamps stored consistently with monument timezones for display

---

## API Functional Groups

All API endpoints return JSON and require appropriate authorization.

### Authentication & Authorization

| Endpoint | Method | Purpose | Auth Required |
|---|---|---|---|
| `/api/auth/login` | POST | Staff login | No |
| `/api/auth/logout` | POST | Staff logout | Yes |
| `/api/auth/refresh` | POST | Refresh session | Yes |
| `/api/auth/password-reset` | POST | Reset password | No |

### Monument Management (Admin API)

| Endpoint | Method | Purpose | Auth Required |
|---|---|---|---|
| `/api/monuments` | GET | List all monuments (admin) | Yes (Admin) |
| `/api/monuments` | POST | Create monument | Yes (Admin) |
| `/api/monuments/{id}` | PUT | Update monument | Yes (Admin) |
| `/api/monuments/{id}` | DELETE | Archive monument | Yes (Admin) |
| `/api/monuments/public` | GET | List published monuments (public) | No |

### Availability & Listings (Public API)

| Endpoint | Method | Purpose | Auth Required |
|---|---|---|---|
| `/api/availability/{monumentId}` | GET | Get available dates and slots | No |
| `/api/availability/{monumentId}/{date}` | GET | Get available slots for specific date | No |

### Time Slots (Admin API)

| Endpoint | Method | Purpose | Auth Required |
|---|---|---|---|
| `/api/slots` | GET | List slots for monument | Yes (Admin) |
| `/api/slots` | POST | Create recurring slot | Yes (Admin) |
| `/api/slots/{id}` | PUT | Update slot | Yes (Admin) |
| `/api/slots/{id}` | DELETE | Disable slot | Yes (Admin) |

### Blocked Dates (Admin API)

| Endpoint | Method | Purpose | Auth Required |
|---|---|---|---|
| `/api/closures` | GET | List blocked periods | Yes (Admin) |
| `/api/closures` | POST | Create block | Yes (Admin) |
| `/api/closures/{id}` | DELETE | Remove block | Yes (Admin) |

### Bookings

| Endpoint | Method | Purpose | Auth Required |
|---|---|---|---|
| `/api/bookings/calculate` | POST | Validate availability and price | No |
| `/api/bookings` | POST | Create booking | No (public) / Yes (counter operator) |
| `/api/bookings/{reference}` | GET | Retrieve booking | Visitor or staff |
| `/api/bookings/{reference}/cancel` | POST | Cancel booking | Visitor or staff |
| `/api/bookings` | GET | List bookings (admin) | Yes (Admin) |

### Tickets

| Endpoint | Method | Purpose | Auth Required |
|---|---|---|---|
| `/api/tickets/{reference}` | GET | Get QR ticket | Visitor or staff |
| `/api/tickets/{reference}/download` | GET | Download ticket as PDF/PNG | Visitor or staff |
| `/api/tickets/{reference}/reissue` | POST | Reissue ticket (revoke old) | Staff |

### Verification

| Endpoint | Method | Purpose | Auth Required |
|---|---|---|---|
| `/api/verification/preview` | POST | Preview ticket eligibility (read-only) | Yes (Operator) |
| `/api/verification/checkin` | POST | Record admission (atomic write) | Yes (Operator) |

### Reports

| Endpoint | Method | Purpose | Auth Required |
|---|---|---|---|
| `/api/reports/bookings` | GET | Booking report | Yes (Staff) |
| `/api/reports/attendance` | GET | Visitor attendance report | Yes (Staff) |
| `/api/reports/occupancy` | GET | Slot occupancy report | Yes (Staff) |
| `/api/reports/noshows` | GET | No-show report | Yes (Staff) |
| `/api/reports/revenue` | GET | Revenue report | Yes (Admin/Manager) |
| `/api/reports/export` | GET | Export reports to CSV/Excel | Yes (Staff) |

### Audit

| Endpoint | Method | Purpose | Auth Required |
|---|---|---|---|
| `/api/audit` | GET | Audit log | Yes (Admin/Auditor) |

---

## Dashboard & Reports

### Admin Dashboard

**Filters:**
- Date range
- Monument
- Time slot
- Booking channel (website/counter)
- Ticket category

**Summary Cards:**
- Total confirmed bookings
- Total booked visitors
- Checked-in visitors
- Visitors awaiting entry
- Cancelled bookings
- No-shows
- Remaining capacity
- Gross payments and refunds (if enabled)

**Charts:**
- Bookings by day (line/area chart)
- Visitors by monument (bar chart)
- Slot occupancy (stacked bar per monument)
- Attendance vs. bookings (comparison)
- Revenue trends (if payments enabled)
- Website vs. counter bookings (pie chart)

### Required Reports

| Report | Contents | Export Formats |
|---|---|---|
| **Booking Report** | Reference, monument, date, slot, quantities, status, channel | CSV, Excel, PDF |
| **Visitor Attendance** | Booked visitors, admitted visitors, check-in times | CSV, Excel, PDF |
| **Slot Occupancy** | Capacity, booked visitors, remaining, utilization % | CSV, Excel, PDF |
| **No-Show Report** | Confirmed bookings not admitted after entry window | CSV, Excel, PDF |
| **Revenue Report** | Payments received, refunds, net collections, payment method | CSV, Excel, PDF |
| **Cancellation Report** | Cancellation date, reason, actor, refund state | CSV, Excel, PDF |
| **Counter Activity** | Admissions/rejections by operator and counter | CSV, Excel, PDF |
| **Blocked-Date Report** | Closure dates, affected slots, reasons, booking impacts | CSV, Excel, PDF |
| **Audit Report** | Administrative and operational changes | CSV, Excel, PDF |

### Report Requirements

- ✅ Respect monument-level access restrictions
- ✅ Use monument's time zone for operational dates
- ✅ Include generation time and applied filters in header
- ✅ Restrict and audit exports with personal information
- ✅ Define metrics consistently (bookings vs. visitor counts)
- ✅ Revenue reports distinguish payment date from visit date
- ✅ Exports processed asynchronously for large datasets

---

## Nonfunctional Requirements

### Security

- ✅ HTTPS on all endpoints
- ✅ Secure staff authentication (password hashing with bcrypt/PBKDF2)
- ✅ MFA recommended for admin users
- ✅ Server-side permission checks on all state-changing API calls
- ✅ Rate limiting on public booking and ticket endpoints
- ✅ Input validation (type, length, format) on all API endpoints
- ✅ Secure file uploads with virus scanning and type validation
- ✅ Secrets stored in configuration (not in code)
- ✅ Minimal personal data collection with defined retention policy
- ✅ Sensitive tokens (QR, payment) excluded from logs
- ✅ SQL injection and XSS prevention via parameterized queries and Razor encoding

### Reliability

- ✅ Automated daily backups with tested restoration procedures
- ✅ Monitoring for booking failures, payment failures, verification outages
- ✅ Retryable background jobs with idempotent processing
- ✅ Ticket retrieval available even if notification delivery fails
- ✅ Circuit breakers for external payment service calls
- ✅ Documented entry-counter outage procedure (fallback to manual verification)

### Performance

Initial targets (to be validated through load testing):

- QR eligibility checks: < 2 seconds under expected load
- Public availability requests: < 2 seconds
- Booking confirmation: < 3 seconds
- Admin report generation: Asynchronous; large exports within 30 seconds
- Database queries indexed for monument/date/slot lookups
- Capacity and duplicate-entry guarantees correct under peak concurrent bookings

### Usability

- ✅ Responsive interfaces (desktop, tablet, mobile)
- ✅ Clear visual and textual verification outcomes
- ✅ Manual ticket lookup by booking reference (for damaged QR codes)
- ✅ Accessible forms (WCAG 2.1 Level AA recommended)
- ✅ Keyboard navigation throughout
- ✅ Clear local date, time, and currency display per monument

---

## Key Acceptance Criteria

| Scenario | Expected Result |
|---|---|
| Admin publishes monument with slots | Monument appears in public listings with available dates/slots |
| Slot capacity 100, 95 allocated | Request for 6 more visitors rejected |
| Two users compete for last capacity | Total confirmed + holds never exceeds capacity |
| Unlimited slot created | Bookings accepted subject to booking window and cutoff |
| Date blocked | New bookings for that date rejected; public listing shows "Closed" |
| Block affects existing bookings | Admin UI shows count; requires explicit decision to cancel and notify |
| Booking confirmed | Retrievable QR ticket created and email sent |
| Payment callback repeated | No duplicate booking or ticket created |
| Valid ticket admitted | Exactly one check-in record created; ticket marked as used |
| Same ticket admitted at two counters simultaneously | Only one admission succeeds; other operator sees "Already checked in" |
| Cancelled ticket scanned | Admission rejected; operator sees "Cancelled/revoked" |
| Ticket scanned at wrong monument | Admission rejected; operator sees "Wrong monument" |
| Ticket scanned outside entry window | Admission rejected unless override authorized |
| Report filtered by monument/date | Results and totals match selected scope |
| Unauthorized staff requests other monument data | API returns 403 Forbidden |

---

## Development Phases

### Phase 1: Inventory & Administration (Weeks 1–4)

- ✅ ASP.NET Core project setup with EF Core
- ✅ Authentication and role-based authorization
- ✅ Monument CRUD and publishing
- ✅ Ticket category and pricing
- ✅ Recurring schedule and slot management
- ✅ Capacity configuration
- ✅ Blocked date/slot management
- ✅ Admin MVC interface for above
- ✅ Database schema and migrations

### Phase 2: Listings & Booking (Weeks 5–8)

- ✅ Public API endpoints for availability and listings
- ✅ Public MVC interface for browsing
- ✅ Booking calculation and validation
- ✅ Create booking (atomic transaction)
- ✅ Booking confirmation and QR generation
- ✅ Email notifications
- ✅ Visitor booking cancellation
- ✅ Counter operator booking creation
- ✅ Optional payment integration (Razorpay/Stripe)

### Phase 3: Entry Verification (Weeks 9–10)

- ✅ Verification API endpoints
- ✅ Verification MVC or mobile interface
- ✅ QR scanning and eligibility check
- ✅ Atomic check-in with duplicate prevention
- ✅ Verification audit logs
- ✅ Supervisor override capability

### Phase 4: Reporting & Release (Weeks 11–12)

- ✅ Dashboard and report endpoints
- ✅ Admin dashboard MVC interface
- ✅ Report generation and export (CSV/Excel/PDF)
- ✅ Security and concurrency testing
- ✅ Backup and monitoring setup
- ✅ User acceptance testing
- ✅ Staff training materials
- ✅ Production deployment

---

## Repository Structure

```
monuments-booking-system/
├── README.md                              # This file
├── docs/
│   ├── REQUIREMENTS.md                    # Detailed functional requirements
│   ├── API.md                             # API endpoint documentation
│   ├── DATABASE.md                        # Database schema and relationships
│   ├── DEPLOYMENT.md                      # Deployment and infrastructure
│   └── ARCHITECTURE.md                    # Architecture and design decisions
├── src/
│   ├── MonumentsAPI/                      # ASP.NET Core Web API
│   │   ├── Controllers/                   # API controllers
│   │   ├── Services/                      # Business logic services
│   │   ├── Models/                        # Data models and DTOs
│   │   ├── Data/                          # EF Core DbContext and migrations
│   │   ├── Middleware/                    # Authentication, authorization, error handling
│   │   ├── Program.cs                     # Startup configuration
│   │   └── appsettings.json              # Configuration
│   │
│   ├── MonumentsAdmin/                    # ASP.NET Core MVC (Admin Interface)
│   │   ├── Areas/Admin/                   # Admin area
│   │   │   ├── Controllers/               # Admin controllers
│   │   │   ├── Views/                     # Admin Razor views
│   │   │   └── Models/                    # Admin view models
│   │   ├── Services/                      # API client services
│   │   ├── Middleware/                    # Auth middleware
│   │   ├── Program.cs                     # Startup configuration
│   │   └── appsettings.json              # Configuration
│   │
│   └── MonumentsPublic/                   # ASP.NET Core MVC (Public Website)
│       ├── Areas/Public/                  # Public area
│       │   ├── Controllers/               # Booking, browse, account controllers
│       │   ├── Views/                     # Public Razor views
│       │   └── Models/                    # View models
│       ├── Services/                      # API client services
│       ├── Middleware/                    # Session middleware
│       ├── Program.cs                     # Startup configuration
│       └── appsettings.json              # Configuration
│
├── tests/
│   ├── MonumentsAPI.Tests/                # API unit and integration tests
│   ├── MonumentsAdmin.Tests/              # Admin UI tests
│   └── MonumentsPublic.Tests/             # Public UI tests
│
├── docker-compose.yml                     # Local dev environment
├── Dockerfile                             # Container image definition
├── .github/
│   └── workflows/                         # CI/CD pipelines (GitHub Actions)
│
└── LICENSE                                # License file
```

---

## Getting Started

### Prerequisites

- .NET SDK (version TBD – confirm with team)
- Relational database (SQL Server / PostgreSQL / other – TBD)
- Git
- Visual Studio 2022 / VS Code with C# extension (recommended)

### Local Development Setup

```bash
# Clone repository
git clone https://github.com/mishra-umesh/monuments-booking-system.git
cd monuments-booking-system

# Restore NuGet packages
dotnet restore

# Configure database connection in appsettings.json
# Update: Data:DefaultConnection in both API and MVC projects

# Run EF Core migrations
cd src/MonumentsAPI
dotnet ef database update

# Build all projects
dotnet build

# Run tests
dotnet test

# Start API server (in MonumentsAPI directory)
dotnet run

# In another terminal, start Admin MVC
cd src/MonumentsAdmin
dotnet run

# In another terminal, start Public MVC
cd src/MonumentsPublic
dotnet run

# Access interfaces:
# - Admin: https://localhost:5001/admin
# - Public: https://localhost:5002
# - API: https://localhost:5000/api/
```

### Docker Compose

```bash
docker-compose up -d

# Services will be available at:
# - API: http://localhost:5000
# - Admin MVC: http://localhost:5001
# - Public MVC: http://localhost:5002
```

---

## Development Guidelines

### Code Standards

- Use ASP.NET Core conventions
- Dependency injection for services and repositories
- Async/await for all I/O operations
- Unit tests for business logic; integration tests for API
- Entity Framework Core migrations for schema changes
- Parameterized queries (EF Core prevents SQL injection)

### Git Workflow

1. Create feature branch: `git checkout -b feature/feature-name`
2. Make changes with clear commit messages
3. Push to branch: `git push origin feature/feature-name`
4. Create pull request with description
5. Code review required before merge
6. Delete branch after merge

### Testing

- Unit tests for services and business logic
- Integration tests for API endpoints
- Test coverage target: >80%
- Run tests before committing: `dotnet test`

---

## Key Decisions Required Before Development

1. **Database Choice:** SQL Server, PostgreSQL, MySQL, or other? (Impacts connection strings and specific SQL syntax)
2. **Payment Provider:** Razorpay, Stripe, PayPal, or free-only initially?
3. **Notifications:** Email only, or include SMS? (Provider choice: SendGrid, AWS SES, Twilio, etc.)
4. **Deployment Platform:** On-premises IIS, Azure App Service, AWS EC2, Docker containers?
5. **CDN/Image Storage:** Local file system, Azure Blob Storage, AWS S3, or Cloudinary?
6. **Authentication:** ASP.NET Core Identity, OAuth 2.0, Azure AD, or custom?
7. **Caching Strategy:** In-memory cache for MVP, or Redis for distributed caching?
8. **Verification Interface:** MVC web interface, native mobile app, or both?
9. **Reporting Export:** CSV and Excel only, or include PDF?
10. **Scale Expectations:** Number of monuments, gates, concurrent bookings at launch and in 12 months?

---

## Contributing

Contributions are welcome! Please follow the development guidelines:

1. Fork or create a feature branch
2. Make changes with clear, descriptive commits
3. Add tests for new functionality
4. Ensure all tests pass: `dotnet test`
5. Submit a pull request with description and context

---

## Support & Issues

- 📧 Email: [contact email]
- 🐛 Issues: [GitHub Issues Link](https://github.com/mishra-umesh/monuments-booking-system/issues)
- 📚 Documentation: See `/docs` folder

---

## License

This project is licensed under the MIT License. See `LICENSE` file for details.

---

**Last Updated:** October 5, 2026  
**Version:** 1.0 – Requirements & Technical Specification  
**Tech Stack:** ASP.NET Core Web API (C#) + ASP.NET Core MVC  
**Status:** Ready for Development Phase 1