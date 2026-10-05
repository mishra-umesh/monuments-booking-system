# API Documentation

## Overview

The Monuments Booking System API is a RESTful web service built with ASP.NET Core that provides all backend functionality for inventory management, booking, verification, and reporting. The API is consumed by two ASP.NET Core MVC interfaces (Admin and Public) and can be extended to support mobile apps or third-party integrations.

**Base URL:** `https://api.monuments.local/api` (development), `https://api.monuments-prod.com/api` (production)

**Authentication:** Bearer tokens (JWT) for staff endpoints; Session tokens for visitor endpoints (optional)

**Response Format:** JSON

**Versioning:** URL-based (`/api/v1/`, `/api/v2/`) for future compatibility

---

## Table of Contents

1. [Authentication](#authentication)
2. [Monuments](#monuments)
3. [Availability & Listings](#availability--listings)
4. [Time Slots](#time-slots)
5. [Blocked Dates](#blocked-dates)
6. [Bookings](#bookings)
7. [Tickets](#tickets)
8. [Verification](#verification)
9. [Reports](#reports)
10. [Audit Logs](#audit-logs)
11. [Error Handling](#error-handling)
12. [Rate Limiting](#rate-limiting)

---

## Authentication

### Login (Public)

Create a new staff session.

```http
POST /api/auth/login
Content-Type: application/json

{
  "username": "manager@example.com",
  "password": "secure_password"
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expiresIn": 3600,
  "user": {
    "id": "user-001",
    "username": "manager@example.com",
    "email": "manager@example.com",
    "fullName": "John Manager",
    "role": "MonumentManager",
    "assignedMonuments": ["mon-001", "mon-002"]
  }
}
```

**Response (401 Unauthorized):**
```json
{
  "success": false,
  "error": "Invalid username or password"
}
```

### Logout (Protected)

Invalidate current session token.

```http
POST /api/auth/logout
Authorization: Bearer {token}
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Logged out successfully"
}
```

### Refresh Token (Public)

Obtain a new access token using a refresh token.

```http
POST /api/auth/refresh
Content-Type: application/json

{
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expiresIn": 3600
}
```

### Password Reset Request (Public)

Send password reset email to user.

```http
POST /api/auth/password-reset-request
Content-Type: application/json

{
  "email": "user@example.com"
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Password reset link sent to email"
}
```

### Reset Password (Public)

Complete password reset with token from email.

```http
POST /api/auth/password-reset
Content-Type: application/json

{
  "token": "reset-token-from-email",
  "newPassword": "new_secure_password"
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Password updated successfully"
}
```

---

## Monuments

### List All Monuments (Admin)

Retrieve all monuments (draft, published, archived). Requires admin authentication.

```http
GET /api/monuments
Authorization: Bearer {admin_token}
```

**Query Parameters:**
- `status` (optional): `draft` | `published` | `inactive` | `archived`
- `page` (optional): Default 1
- `pageSize` (optional): Default 20, max 100
- `search` (optional): Search by name or location

**Response (200 OK):**
```json
{
  "success": true,
  "data": [
    {
      "id": "mon-001",
      "name": "Taj Mahal",
      "description": "Ivory-white marble mausoleum built in the 17th century",
      "location": {
        "address": "Dharmapuri, Forest Colony",
        "city": "Agra",
        "state": "Uttar Pradesh",
        "country": "India",
        "latitude": 27.1751,
        "longitude": 78.0421
      },
      "timezone": "Asia/Kolkata",
      "status": "published",
      "images": [
        {
          "id": "img-001",
          "url": "/images/taj-mahal-main.jpg",
          "displayOrder": 1,
          "isMainImage": true
        }
      ],
      "openingSchedule": {
        "monday": { "open": "06:00", "close": "19:00" },
        "tuesday": { "open": "06:00", "close": "19:00" },
        "wednesday": null,
        "thursday": { "open": "06:00", "close": "19:00" },
        "friday": { "open": "06:00", "close": "19:00" },
        "saturday": { "open": "06:00", "close": "19:00" },
        "sunday": { "open": "06:00", "close": "19:00" }
      },
      "ticketCategories": [
        {
          "id": "cat-001",
          "name": "Adult",
          "price": 100.00,
          "description": "Ticket for adults (18+)"
        },
        {
          "id": "cat-002",
          "name": "Child",
          "price": 50.00,
          "description": "Ticket for children (5-17)"
        }
      ],
      "capacityMode": "limited",
      "bookingWindowDays": 60,
      "bookingCutoffMinutes": 30,
      "maxVisitorsPerBooking": 10,
      "entryInstructions": "Please arrive 15 minutes before your slot time.",
      "createdAt": "2026-09-01T10:00:00Z",
      "updatedAt": "2026-10-05T14:00:00Z",
      "createdBy": "admin-001"
    }
  ],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "totalRecords": 45,
    "totalPages": 3
  }
}
```

### Get Monument Detail (Public/Admin)

Retrieve a single monument. Public endpoint returns only published monuments; admin can view any.

```http
GET /api/monuments/{monumentId}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "id": "mon-001",
    "name": "Taj Mahal",
    "description": "...",
    "location": {...},
    "timezone": "Asia/Kolkata",
    "status": "published",
    "images": [...],
    "openingSchedule": {...},
    "ticketCategories": [...],
    "capacityMode": "limited",
    "bookingWindowDays": 60,
    "bookingCutoffMinutes": 30,
    "maxVisitorsPerBooking": 10,
    "entryInstructions": "...",
    "createdAt": "2026-09-01T10:00:00Z",
    "updatedAt": "2026-10-05T14:00:00Z"
  }
}
```

**Response (404 Not Found):**
```json
{
  "success": false,
  "error": "Monument not found"
}
```

### List Published Monuments (Public)

Retrieve publicly available monuments.

```http
GET /api/monuments/public
```

**Response (200 OK):**
Same structure as "List All Monuments" but only `status: "published"` monuments.

### Create Monument (Admin)

Create a new monument in draft status.

```http
POST /api/monuments
Authorization: Bearer {admin_token}
Content-Type: application/json

{
  "name": "Taj Mahal",
  "description": "Ivory-white marble mausoleum",
  "location": {
    "address": "Dharmapuri, Forest Colony",
    "city": "Agra",
    "state": "Uttar Pradesh",
    "country": "India",
    "latitude": 27.1751,
    "longitude": 78.0421
  },
  "timezone": "Asia/Kolkata",
  "openingSchedule": {
    "monday": { "open": "06:00", "close": "19:00" },
    "tuesday": { "open": "06:00", "close": "19:00" },
    "wednesday": null,
    "thursday": { "open": "06:00", "close": "19:00" },
    "friday": { "open": "06:00", "close": "19:00" },
    "saturday": { "open": "06:00", "close": "19:00" },
    "sunday": { "open": "06:00", "close": "19:00" }
  },
  "ticketCategories": [
    {
      "name": "Adult",
      "price": 100.00,
      "description": "Ticket for adults (18+)"
    },
    {
      "name": "Child",
      "price": 50.00,
      "description": "Ticket for children (5-17)"
    }
  ],
  "capacityMode": "limited",
  "bookingWindowDays": 60,
  "bookingCutoffMinutes": 30,
  "maxVisitorsPerBooking": 10,
  "entryInstructions": "Please arrive 15 minutes before your slot time."
}
```

**Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "id": "mon-new-001",
    "name": "Taj Mahal",
    "status": "draft",
    ...
  }
}
```

### Update Monument (Admin)

Update an existing monument. Only published status cannot be edited via PUT; use Publish/Unpublish endpoints.

```http
PUT /api/monuments/{monumentId}
Authorization: Bearer {admin_token}
Content-Type: application/json

{
  "name": "Taj Mahal - Updated",
  "description": "Updated description",
  "openingSchedule": {...},
  "ticketCategories": [...]
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {...}
}
```

### Publish Monument (Admin)

Change monument status from draft/inactive to published.

```http
POST /api/monuments/{monumentId}/publish
Authorization: Bearer {admin_token}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "id": "mon-001",
    "status": "published",
    ...
  }
}
```

### Unpublish Monument (Admin)

Change monument status from published to inactive.

```http
POST /api/monuments/{monumentId}/unpublish
Authorization: Bearer {admin_token}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "id": "mon-001",
    "status": "inactive",
    ...
  }
}
```

### Archive Monument (Admin)

Archive a monument (soft delete). Cannot archive if active bookings exist.

```http
DELETE /api/monuments/{monumentId}
Authorization: Bearer {admin_token}
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Monument archived successfully"
}
```

**Response (409 Conflict):**
```json
{
  "success": false,
  "error": "Cannot archive monument with active bookings",
  "data": {
    "activeBookingCount": 5
  }
}
```

---

## Availability & Listings

### Get Available Dates (Public)

Retrieve available dates for a monument in the next N days.

```http
GET /api/availability/{monumentId}?days=60
```

**Query Parameters:**
- `days` (optional): Default 30, max 180
- `excludeBlocked` (optional): Default true

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "monumentId": "mon-001",
    "monumentName": "Taj Mahal",
    "availableDates": [
      {
        "date": "2026-10-10",
        "dayOfWeek": "Friday",
        "isOpen": true,
        "slots": [
          {
            "slotId": "slot-001",
            "startTime": "06:00",
            "endTime": "07:00",
            "capacityMode": "limited",
            "totalCapacity": 100,
            "bookedVisitors": 75,
            "remainingCapacity": 25,
            "status": "available"
          },
          {
            "slotId": "slot-002",
            "startTime": "07:00",
            "endTime": "08:00",
            "capacityMode": "limited",
            "totalCapacity": 150,
            "bookedVisitors": 150,
            "remainingCapacity": 0,
            "status": "sold_out"
          }
        ]
      },
      {
        "date": "2026-10-11",
        "dayOfWeek": "Saturday",
        "isOpen": true,
        "slots": [...]
      }
    ]
  }
}
```

### Get Slots for Specific Date (Public)

Retrieve available slots for a specific monument and date.

```http
GET /api/availability/{monumentId}/{date}
```

**Path Parameters:**
- `monumentId`: Monument ID
- `date`: ISO 8601 date (e.g., `2026-10-10`)

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "monumentId": "mon-001",
    "monumentName": "Taj Mahal",
    "date": "2026-10-10",
    "dayOfWeek": "Friday",
    "isOpen": true,
    "closureReason": null,
    "publicClosureMessage": null,
    "slots": [
      {
        "slotId": "slot-001",
        "startTime": "06:00",
        "endTime": "07:00",
        "capacityMode": "limited",
        "totalCapacity": 100,
        "bookedVisitors": 75,
        "reservationHolds": 10,
        "remainingCapacity": 15,
        "status": "available",
        "bookingCutoffMinutes": 30,
        "entryGraceMinutes": 15
      }
    ]
  }
}
```

---

## Time Slots

### List Slots for Monument (Admin)

Retrieve all slot templates and instances for a monument.

```http
GET /api/slots?monumentId={monumentId}
Authorization: Bearer {admin_token}
```

**Query Parameters:**
- `monumentId` (required): Monument ID
- `fromDate` (optional): ISO 8601 date; default today
- `toDate` (optional): ISO 8601 date; default today + 60 days
- `includeDisabled` (optional): Default false

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "monumentId": "mon-001",
    "schedules": [
      {
        "id": "sched-001",
        "dayOfWeek": "Monday",
        "startTime": "06:00",
        "endTime": "07:00",
        "capacityMode": "limited",
        "capacity": 100,
        "isRecurring": true,
        "isDisabled": false,
        "createdAt": "2026-09-01T10:00:00Z"
      }
    ],
    "instances": [
      {
        "id": "slot-001",
        "scheduleId": "sched-001",
        "date": "2026-10-10",
        "startTime": "06:00",
        "endTime": "07:00",
        "capacity": 100,
        "bookedVisitors": 75,
        "status": "normal"
      }
    ]
  }
}
```

### Create Slot Template (Admin)

Create a recurring slot template.

```http
POST /api/slots
Authorization: Bearer {admin_token}
Content-Type: application/json

{
  "monumentId": "mon-001",
  "daysOfWeek": ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday"],
  "startTime": "06:00",
  "endTime": "07:00",
  "capacityMode": "limited",
  "capacity": 100,
  "bookingCutoffMinutes": 30,
  "entryGraceMinutes": 15
}
```

**Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "id": "sched-001",
    "monumentId": "mon-001",
    "startTime": "06:00",
    "endTime": "07:00",
    "capacityMode": "limited",
    "capacity": 100,
    "createdAt": "2026-10-05T14:00:00Z"
  }
}
```

### Update Slot Template (Admin)

Update a recurring slot template.

```http
PUT /api/slots/{slotTemplateId}
Authorization: Bearer {admin_token}
Content-Type: application/json

{
  "startTime": "06:30",
  "endTime": "07:30",
  "capacity": 120
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {...}
}
```

### Disable Slot (Admin)

Disable a slot template (don't delete; prevents accidental loss of booking history).

```http
DELETE /api/slots/{slotTemplateId}
Authorization: Bearer {admin_token}
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Slot disabled successfully"
}
```

### Create Date Override (Admin)

Create a one-time slot override for a specific date.

```http
POST /api/slots/override
Authorization: Bearer {admin_token}
Content-Type: application/json

{
  "monumentId": "mon-001",
  "date": "2026-10-25",
  "slots": [
    {
      "startTime": "09:00",
      "endTime": "10:00",
      "capacityMode": "limited",
      "capacity": 80
    }
  ]
}
```

**Response (201 Created):**
```json
{
  "success": true,
  "data": {...}
}
```

---

## Blocked Dates

### List Blocked Periods (Admin)

Retrieve all blocked dates and slots.

```http
GET /api/closures?monumentId={monumentId}
Authorization: Bearer {admin_token}
```

**Query Parameters:**
- `monumentId` (required): Monument ID
- `fromDate` (optional): ISO 8601 date
- `toDate` (optional): ISO 8601 date

**Response (200 OK):**
```json
{
  "success": true,
  "data": [
    {
      "id": "closure-001",
      "monumentId": "mon-001",
      "startDate": "2026-10-20",
      "endDate": "2026-10-22",
      "slotId": null,
      "reason": "Maintenance",
      "publicMessage": "Monument closed for maintenance. Reopens October 23.",
      "affectedBookingCount": 3,
      "createdAt": "2026-10-05T14:00:00Z",
      "createdBy": "manager-001"
    }
  ]
}
```

### Create Blocked Period (Admin)

Block a date range or specific slot.

```http
POST /api/closures
Authorization: Bearer {admin_token}
Content-Type: application/json

{
  "monumentId": "mon-001",
  "startDate": "2026-10-20",
  "endDate": "2026-10-22",
  "slotId": null,
  "reason": "Maintenance",
  "publicMessage": "Monument closed for maintenance. Reopens October 23.",
  "affectedBookingAction": "notify_only"
}
```

**Parameters:**
- `affectedBookingAction`: `notify_only` | `cancel_with_refund` | `cancel_no_refund`

**Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "id": "closure-001",
    "monumentId": "mon-001",
    "startDate": "2026-10-20",
    "endDate": "2026-10-22",
    "affectedBookingCount": 3,
    "cancelledBookingCount": 0,
    "refundAmount": 0.00
  }
}
```

### Delete Blocked Period (Admin)

Remove a block.

```http
DELETE /api/closures/{closureId}
Authorization: Bearer {admin_token}
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Blocked period removed"
}
```

---

## Bookings

### Calculate Booking (Public)

Validate availability and calculate price for a booking.

```http
POST /api/bookings/calculate
Content-Type: application/json

{
  "monumentId": "mon-001",
  "slotId": "slot-001",
  "visitDate": "2026-10-10",
  "items": [
    {
      "ticketCategoryId": "cat-001",
      "quantity": 2
    },
    {
      "ticketCategoryId": "cat-002",
      "quantity": 1
    }
  ]
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "monumentId": "mon-001",
    "monumentName": "Taj Mahal",
    "visitDate": "2026-10-10",
    "slotId": "slot-001",
    "slotTime": "06:00 - 07:00",
    "items": [
      {
        "ticketCategoryId": "cat-001",
        "categoryName": "Adult",
        "quantity": 2,
        "unitPrice": 100.00,
        "subtotal": 200.00
      },
      {
        "ticketCategoryId": "cat-002",
        "categoryName": "Child",
        "quantity": 1,
        "unitPrice": 50.00,
        "subtotal": 50.00
      }
    ],
    "totalVisitors": 3,
    "subtotal": 250.00,
    "tax": 25.00,
    "total": 275.00,
    "currency": "INR",
    "remainingCapacity": 22,
    "availabilityStatus": "available"
  }
}
```

**Response (400 Bad Request):**
```json
{
  "success": false,
  "error": "Requested slot is sold out",
  "data": {
    "availabilityStatus": "sold_out"
  }
}
```

### Create Booking (Public)

Create a new visitor booking.

```http
POST /api/bookings
Content-Type: application/json

{
  "monumentId": "mon-001",
  "slotId": "slot-001",
  "visitDate": "2026-10-10",
  "visitorName": "John Doe",
  "email": "john@example.com",
  "phone": "+91-9876543210",
  "items": [
    {
      "ticketCategoryId": "cat-001",
      "quantity": 2
    }
  ],
  "totalVisitors": 2,
  "totalPrice": 200.00,
  "bookingChannel": "website",
  "paymentMethod": "card",
  "paymentReference": null,
  "acceptedTerms": true
}
```

**Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "bookingId": "bkg-001",
    "reference": "BKG-2026-001234",
    "monumentId": "mon-001",
    "monumentName": "Taj Mahal",
    "visitDate": "2026-10-10",
    "slotId": "slot-001",
    "slotTime": "06:00 - 07:00",
    "visitorName": "John Doe",
    "email": "john@example.com",
    "phone": "+91-9876543210",
    "totalVisitors": 2,
    "totalPrice": 200.00,
    "status": "pending",
    "paymentStatus": "pending",
    "bookingChannel": "website",
    "createdAt": "2026-10-05T14:15:00Z",
    "ticketId": "ticket-001",
    "ticketToken": "qr_token_xyz123..."
  }
}
```

### Retrieve Booking (Public/Admin)

Get booking details by reference or ID.

```http
GET /api/bookings/{bookingReference}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "bookingId": "bkg-001",
    "reference": "BKG-2026-001234",
    "monumentId": "mon-001",
    "monumentName": "Taj Mahal",
    "visitDate": "2026-10-10",
    "slotTime": "06:00 - 07:00",
    "visitorName": "John Doe",
    "email": "john@example.com",
    "phone": "+91-9876543210",
    "totalVisitors": 2,
    "items": [
      {
        "categoryName": "Adult",
        "quantity": 2,
        "unitPrice": 100.00,
        "subtotal": 200.00
      }
    ],
    "subtotal": 200.00,
    "tax": 20.00,
    "totalPrice": 220.00,
    "status": "confirmed",
    "paymentStatus": "paid",
    "bookingChannel": "website",
    "createdAt": "2026-10-05T14:15:00Z",
    "confirmedAt": "2026-10-05T14:20:00Z",
    "checkedInAt": null,
    "cancelledAt": null,
    "cancellationReason": null,
    "refundAmount": null
  }
}
```

### List Bookings (Admin)

List bookings with filters.

```http
GET /api/bookings?monumentId={monumentId}&status=confirmed
Authorization: Bearer {admin_token}
```

**Query Parameters:**
- `monumentId` (optional): Filter by monument
- `status` (optional): `pending` | `confirmed` | `cancelled` | `expired`
- `paymentStatus` (optional): `not_paid` | `pending` | `paid` | `failed` | `refunded`
- `fromDate` (optional): ISO 8601 date
- `toDate` (optional): ISO 8601 date
- `bookingChannel` (optional): `website` | `counter`
- `page` (optional): Default 1
- `pageSize` (optional): Default 20

**Response (200 OK):**
```json
{
  "success": true,
  "data": [
    {
      "bookingId": "bkg-001",
      "reference": "BKG-2026-001234",
      "monumentName": "Taj Mahal",
      "visitDate": "2026-10-10",
      "visitorName": "John Doe",
      "totalVisitors": 2,
      "totalPrice": 220.00,
      "status": "confirmed",
      "paymentStatus": "paid",
      "bookingChannel": "website",
      "createdAt": "2026-10-05T14:15:00Z"
    }
  ],
  "pagination": {...}
}
```

### Cancel Booking (Public/Admin)

Cancel an existing booking.

```http
POST /api/bookings/{bookingReference}/cancel
Content-Type: application/json

{
  "cancellationReason": "Visitor is unable to attend",
  "issueRefund": true
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "bookingId": "bkg-001",
    "reference": "BKG-2026-001234",
    "status": "cancelled",
    "cancelledAt": "2026-10-05T14:30:00Z",
    "cancellationReason": "Visitor is unable to attend",
    "refundAmount": 220.00,
    "refundStatus": "processed"
  }
}
```

### Counter Booking (Staff)

Create a walk-in booking at the counter.

```http
POST /api/bookings/counter
Authorization: Bearer {operator_token}
Content-Type: application/json

{
  "monumentId": "mon-001",
  "slotId": "slot-001",
  "visitDate": "2026-10-10",
  "visitorName": "Jane Smith",
  "phone": "+91-8765432109",
  "items": [
    {
      "ticketCategoryId": "cat-001",
      "quantity": 1
    }
  ],
  "totalVisitors": 1,
  "totalPrice": 100.00,
  "paymentMethod": "cash",
  "operatorId": "operator-001"
}
```

**Response (201 Created):**
Same structure as regular booking.

---

## Tickets

### Get Ticket (Public)

Retrieve QR ticket for a booking.

```http
GET /api/tickets/{bookingReference}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "ticketId": "ticket-001",
    "bookingReference": "BKG-2026-001234",
    "monumentName": "Taj Mahal",
    "visitDate": "2026-10-10",
    "slotTime": "06:00 - 07:00",
    "visitorName": "John Doe",
    "totalVisitors": 2,
    "items": [
      {
        "categoryName": "Adult",
        "quantity": 2
      }
    ],
    "qrCode": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAA...",
    "qrToken": "qr_token_xyz123...",
    "entryInstructions": "Please arrive 15 minutes before your slot time.",
    "status": "active",
    "createdAt": "2026-10-05T14:20:00Z",
    "validUntil": "2026-10-10T19:00:00Z"
  }
}
```

### Download Ticket (Public)

Download ticket as PDF or PNG.

```http
GET /api/tickets/{bookingReference}/download?format=pdf
```

**Query Parameters:**
- `format` (optional): `pdf` | `png` | `svg` (default: `pdf`)

**Response:** File download (application/pdf or image/png)

### Reissue Ticket (Admin/Staff)

Revoke current ticket and issue a new one.

```http
POST /api/tickets/{bookingReference}/reissue
Authorization: Bearer {staff_token}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "ticketId": "ticket-002",
    "bookingReference": "BKG-2026-001234",
    "qrCode": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAA...",
    "qrToken": "new_qr_token_abc456...",
    "previousTicketRevoked": true,
    "message": "Old ticket revoked; new ticket issued"
  }
}
```

---

## Verification

### Preview Ticket Eligibility (Staff)

Check ticket eligibility without checking in. Read-only operation.

```http
POST /api/verification/preview
Authorization: Bearer {operator_token}
Content-Type: application/json

{
  "qrToken": "qr_token_xyz123...",
  "monumentId": "mon-001"
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "ticketId": "ticket-001",
    "bookingReference": "BKG-2026-001234",
    "visitorName": "John Doe",
    "totalVisitors": 2,
    "monumentName": "Taj Mahal",
    "monumentId": "mon-001",
    "visitDate": "2026-10-10",
    "slotTime": "06:00 - 07:00",
    "entryWindowStart": "2026-10-10T05:45:00Z",
    "entryWindowEnd": "2026-10-10T07:15:00Z",
    "currentTime": "2026-10-10T06:30:00Z",
    "status": "eligible",
    "eligibilityReasons": [
      "Booking confirmed",
      "Not previously checked in",
      "Within entry window"
    ]
  }
}
```

**Response (400 Bad Request):**
```json
{
  "success": false,
  "error": "Ticket is not eligible for entry",
  "data": {
    "ticketId": "ticket-001",
    "bookingReference": "BKG-2026-001234",
    "status": "cancelled",
    "ineligibilityReasons": [
      "Booking has been cancelled"
    ]
  }
}
```

### Check In (Staff) - Atomic Operation

Record admission for a ticket. This is the final, state-changing operation.

```http
POST /api/verification/checkin
Authorization: Bearer {operator_token}
Content-Type: application/json

{
  "qrToken": "qr_token_xyz123...",
  "monumentId": "mon-001",
  "operatorId": "operator-001",
  "locationId": "gate-001",
  "notes": "Verified ID"
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "ticketId": "ticket-001",
    "bookingReference": "BKG-2026-001234",
    "visitorName": "John Doe",
    "totalVisitors": 2,
    "monumentName": "Taj Mahal",
    "checkedInAt": "2026-10-10T06:30:45Z",
    "operatorId": "operator-001",
    "locationId": "gate-001",
    "status": "admitted",
    "message": "Entry recorded successfully. Welcome!"
  }
}
```

**Response (409 Conflict - Already Checked In):**
```json
{
  "success": false,
  "error": "Ticket already checked in",
  "data": {
    "ticketId": "ticket-001",
    "bookingReference": "BKG-2026-001234",
    "previousCheckInTime": "2026-10-10T06:15:00Z",
    "operatorId": "operator-002"
  }
}
```

**Response (400 Bad Request - Not Eligible):**
```json
{
  "success": false,
  "error": "Ticket is not eligible for entry",
  "data": {
    "status": "cancelled",
    "ineligibilityReasons": ["Booking has been cancelled"]
  }
}
```

### Override Check In (Admin Only)

Force check-in with supervisor authorization.

```http
POST /api/verification/override-checkin
Authorization: Bearer {admin_token}
Content-Type: application/json

{
  "qrToken": "qr_token_xyz123...",
  "monumentId": "mon-001",
  "operatorId": "operator-001",
  "locationId": "gate-001",
  "overrideReason": "Ticket outside window; visitor delayed due to traffic"
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "ticketId": "ticket-001",
    "bookingReference": "BKG-2026-001234",
    "checkedInAt": "2026-10-10T08:30:45Z",
    "override": true,
    "overrideAuthorizedBy": "admin-001",
    "overrideReason": "Ticket outside window; visitor delayed due to traffic",
    "message": "Admission recorded with override"
  }
}
```

---

## Reports

### Booking Report (Admin)

Export bookings with filters.

```http
GET /api/reports/bookings?monumentId={monumentId}&fromDate=2026-10-01&toDate=2026-10-31
Authorization: Bearer {staff_token}
```

**Query Parameters:**
- `monumentId` (optional)
- `status` (optional)
- `fromDate` (optional)
- `toDate` (optional)
- `format` (optional): `json` | `csv` | `excel` (default: `json`)

**Response (200 OK - JSON):**
```json
{
  "success": true,
  "data": [
    {
      "reference": "BKG-2026-001234",
      "monumentName": "Taj Mahal",
      "visitDate": "2026-10-10",
      "slotTime": "06:00 - 07:00",
      "visitorName": "John Doe",
      "totalVisitors": 2,
      "totalPrice": 220.00,
      "status": "confirmed",
      "paymentStatus": "paid",
      "bookingChannel": "website",
      "createdAt": "2026-10-05T14:15:00Z"
    }
  ],
  "summary": {
    "totalBookings": 150,
    "confirmedBookings": 120,
    "cancelledBookings": 20,
    "totalVisitors": 350,
    "totalRevenue": 35000.00
  }
}
```

### Attendance Report (Admin)

Visitor check-in report.

```http
GET /api/reports/attendance?monumentId={monumentId}&fromDate=2026-10-01
Authorization: Bearer {staff_token}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": [
    {
      "reference": "BKG-2026-001234",
      "visitorName": "John Doe",
      "monumentName": "Taj Mahal",
      "bookingDate": "2026-10-05T14:15:00Z",
      "visitDate": "2026-10-10",
      "bookedVisitors": 2,
      "checkedInAt": "2026-10-10T06:30:45Z",
      "attendance": "present"
    }
  ],
  "summary": {
    "bookedVisitors": 350,
    "checkedInVisitors": 300,
    "noShows": 50,
    "attendanceRate": 85.71
  }
}
```

### Occupancy Report (Admin)

Slot occupancy analysis.

```http
GET /api/reports/occupancy?monumentId={monumentId}&fromDate=2026-10-01
Authorization: Bearer {staff_token}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": [
    {
      "date": "2026-10-10",
      "slotTime": "06:00 - 07:00",
      "capacity": 100,
      "booked": 75,
      "checkedIn": 65,
      "remaining": 25,
      "utilization": 75.0,
      "status": "high_occupancy"
    }
  ],
  "summary": {
    "averageUtilization": 72.5,
    "peakCapacityDays": 5,
    "underutilizedDays": 2
  }
}
```

### Revenue Report (Admin)

Payment and revenue analysis.

```http
GET /api/reports/revenue?fromDate=2026-10-01&toDate=2026-10-31
Authorization: Bearer {admin_token}
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": [
    {
      "date": "2026-10-05",
      "monumentName": "Taj Mahal",
      "bookingsCount": 25,
      "visitorsCount": 60,
      "subtotal": 6000.00,
      "tax": 600.00,
      "total": 6600.00,
      "paid": 6600.00,
      "refunded": 0.00,
      "net": 6600.00
    }
  ],
  "summary": {
    "totalBookings": 300,
    "totalVisitors": 900,
    "grossRevenue": 90000.00,
    "refunds": 5000.00,
    "netRevenue": 85000.00,
    "averageBookingValue": 300.00
  }
}
```

---

## Audit Logs

### Get Audit Logs (Admin/Auditor)

Retrieve audit trail of system changes.

```http
GET /api/audit?entityType=Monument&fromDate=2026-10-01
Authorization: Bearer {admin_token}
```

**Query Parameters:**
- `entityType` (optional): `Monument` | `Booking` | `TicketCategory` | `User`
- `entityId` (optional)
- `action` (optional): `create` | `update` | `delete` | `publish` | `cancel`
- `userId` (optional)
- `fromDate` (optional)
- `toDate` (optional)
- `page` (optional)

**Response (200 OK):**
```json
{
  "success": true,
  "data": [
    {
      "id": "audit-001",
      "timestamp": "2026-10-05T14:20:00Z",
      "userId": "admin-001",
      "userName": "Admin User",
      "action": "create",
      "entityType": "Monument",
      "entityId": "mon-001",
      "entityName": "Taj Mahal",
      "changes": {
        "status": { "oldValue": null, "newValue": "draft" },
        "name": { "oldValue": null, "newValue": "Taj Mahal" }
      },
      "ipAddress": "192.168.1.100"
    }
  ],
  "pagination": {...}
}
```

---

## Error Handling

All API responses follow a consistent error format:

```json
{
  "success": false,
  "error": "Error message",
  "errorCode": "VALIDATION_ERROR",
  "data": {
    "field": "email",
    "detail": "Email format is invalid"
  },
  "timestamp": "2026-10-05T14:30:00Z"
}
```

### HTTP Status Codes

| Code | Meaning | Example |
|---|---|---|
| 200 | Success | Booking confirmed |
| 201 | Created | Monument created |
| 400 | Bad Request | Invalid input, sold-out slot |
| 401 | Unauthorized | Missing or invalid token |
| 403 | Forbidden | Insufficient permissions |
| 404 | Not Found | Monument not found |
| 409 | Conflict | Duplicate check-in, overbooking |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Server Error | Database error |
| 503 | Service Unavailable | Maintenance mode |

### Common Error Codes

| Code | Meaning |
|---|---|
| `VALIDATION_ERROR` | Input validation failed |
| `UNAUTHORIZED` | Authentication required |
| `FORBIDDEN` | Permission denied |
| `NOT_FOUND` | Resource not found |
| `CONFLICT` | State conflict (e.g., duplicate check-in) |
| `SLOT_SOLD_OUT` | No remaining capacity |
| `BOOKING_CANCELLED` | Booking is cancelled |
| `TICKET_INVALID` | QR token invalid or expired |
| `TICKET_ALREADY_USED` | Ticket already checked in |
| `PAYMENT_FAILED` | Payment processing failed |
| `DATABASE_ERROR` | Database operation failed |

---

## Rate Limiting

Rate limits are applied per IP address for public endpoints and per user for authenticated endpoints:

- **Public Endpoints:** 100 requests per minute
- **Staff Endpoints:** 1000 requests per minute
- **Verification:** 500 requests per minute (per operator)

Rate limit headers included in all responses:
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1696512600
```

---

## Pagination

List endpoints support pagination with query parameters:

```http
GET /api/monuments?page=2&pageSize=50
```

Response includes pagination metadata:
```json
{
  "success": true,
  "data": [...],
  "pagination": {
    "page": 2,
    "pageSize": 50,
    "totalRecords": 250,
    "totalPages": 5,
    "hasNextPage": true,
    "hasPreviousPage": true
  }
}
```

---

## API Versioning

Future versions will use URL versioning:

- Current: `/api/monuments`
- Future: `/api/v2/monuments`

---

## Testing Endpoints

Use these test credentials for development:

**Admin User:**
- Email: `admin@monuments.local`
- Password: `admin_password_123`

**Manager User:**
- Email: `manager@monuments.local`
- Password: `manager_password_123`

**Operator User:**
- Email: `operator@monuments.local`
- Password: `operator_password_123`

Test monuments and sample data available in `/scripts/seed-testdata.sql`.

---

**API Version:** 1.0  
**Last Updated:** October 5, 2026