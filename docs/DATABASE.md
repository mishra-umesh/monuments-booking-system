# Database Schema & Design

## Overview

The Monuments Booking System uses a relational database with Entity Framework Core as the ORM. This document outlines the complete database schema, relationships, indexes, and data integrity rules.

**Database Choice:** To be confirmed (SQL Server, PostgreSQL, MySQL, or other)

**ORM:** Entity Framework Core (EF Core) with Code-First migrations

**Key Principles:**
- Database is the authoritative source for all state (capacity, check-ins, bookings)
- All capacity operations use atomic transactions
- Soft deletes for monuments and users (archive status)
- Audit trail for all administrative changes
- Proper foreign key constraints with cascade rules

---

## Table of Contents

1. [Core Entities](#core-entities)
2. [Relationships](#relationships)
3. [Indexes](#indexes)
4. [Constraints & Validation](#constraints--validation)
5. [Migrations & Setup](#migrations--setup)
6. [Sample Data](#sample-data)
7. [Performance Tuning](#performance-tuning)
8. [Backup & Recovery](#backup--recovery)

---

## Core Entities

### 1. Users

Stores staff user accounts with role-based access control.

```sql
CREATE TABLE Users (
    Id NVARCHAR(50) PRIMARY KEY,
    Username NVARCHAR(255) NOT NULL UNIQUE,
    Email NVARCHAR(255) NOT NULL UNIQUE,
    PasswordHash NVARCHAR(255) NOT NULL,
    FullName NVARCHAR(255) NOT NULL,
    Phone NVARCHAR(20),
    Role NVARCHAR(50) NOT NULL, -- SystemAdmin, MonumentManager, Operator, Auditor
    IsActive BIT NOT NULL DEFAULT 1,
    IsArchived BIT NOT NULL DEFAULT 0,
    LastLoginAt DATETIME2,
    CreatedAt DATETIME2 NOT NULL,
    UpdatedAt DATETIME2 NOT NULL,
    CreatedBy NVARCHAR(50),
    UpdatedBy NVARCHAR(50)
);
```

**Columns:**
- `Id`: Unique user identifier
- `Username`: Login identifier (lowercase email)
- `Email`: User email address
- `PasswordHash`: Bcrypt/PBKDF2 hashed password
- `FullName`: Display name
- `Phone`: Contact number
- `Role`: User role determining permissions
- `IsActive`: Soft delete flag
- `IsArchived`: Archive flag
- `LastLoginAt`: Last login timestamp
- `CreatedAt`, `UpdatedAt`: Audit timestamps
- `CreatedBy`, `UpdatedBy`: Audit user references

### 2. Roles

Defines permissions for user roles.

```sql
CREATE TABLE Roles (
    Id NVARCHAR(50) PRIMARY KEY,
    Name NVARCHAR(100) NOT NULL UNIQUE,
    Description NVARCHAR(500),
    Permissions NVARCHAR(MAX), -- JSON array of permission codes
    IsSystem BIT NOT NULL DEFAULT 0, -- System roles cannot be deleted
    CreatedAt DATETIME2 NOT NULL
);
```

**System Roles:**
- `SystemAdmin` – Full system access
- `MonumentManager` – Manage assigned monuments
- `OperatorCounter` – Create walk-in bookings
- `OperatorVerification` – Scan and verify tickets
- `Auditor` – View-only access to reports

### 3. UserMonumentAssignments

Links users to monuments for role-based access.

```sql
CREATE TABLE UserMonumentAssignments (
    Id NVARCHAR(50) PRIMARY KEY,
    UserId NVARCHAR(50) NOT NULL,
    MonumentId NVARCHAR(50) NOT NULL,
    AssignedAt DATETIME2 NOT NULL,
    AssignedBy NVARCHAR(50) NOT NULL,
    FOREIGN KEY (UserId) REFERENCES Users(Id) ON DELETE CASCADE,
    FOREIGN KEY (MonumentId) REFERENCES Monuments(Id) ON DELETE CASCADE,
    UNIQUE(UserId, MonumentId)
);
```

### 4. Monuments

Core entity representing a bookable location.

```sql
CREATE TABLE Monuments (
    Id NVARCHAR(50) PRIMARY KEY,
    Name NVARCHAR(255) NOT NULL,
    Description NVARCHAR(MAX),
    Location_Address NVARCHAR(500),
    Location_City NVARCHAR(100),
    Location_State NVARCHAR(100),
    Location_Country NVARCHAR(100),
    Location_Latitude DECIMAL(9, 6),
    Location_Longitude DECIMAL(9, 6),
    TimeZone NVARCHAR(100) NOT NULL, -- e.g., "Asia/Kolkata"
    Status NVARCHAR(50) NOT NULL, -- draft, published, inactive, archived
    CapacityMode NVARCHAR(50) NOT NULL, -- limited, unlimited
    BookingWindowDays INT NOT NULL DEFAULT 60,
    BookingCutoffMinutes INT NOT NULL DEFAULT 30,
    MaxVisitorsPerBooking INT NOT NULL DEFAULT 10,
    EntryInstructions NVARCHAR(1000),
    MainImageUrl NVARCHAR(500),
    IsActive BIT NOT NULL DEFAULT 1,
    CreatedAt DATETIME2 NOT NULL,
    UpdatedAt DATETIME2 NOT NULL,
    PublishedAt DATETIME2,
    CreatedBy NVARCHAR(50),
    UpdatedBy NVARCHAR(50)
);
```

**Key Constraints:**
- `Status` must be one of: draft, published, inactive, archived
- `CapacityMode` must be: limited, unlimited
- `BookingWindowDays` >= 0
- `BookingCutoffMinutes` >= 0
- `TimeZone` must be valid IANA timezone

**Indexes:**
- `Status, IsActive` (for listing published monuments)
- `PublishedAt` (for sorting recent)
- `Name` (for search)

### 5. MonumentImages

Gallery images for monuments.

```sql
CREATE TABLE MonumentImages (
    Id NVARCHAR(50) PRIMARY KEY,
    MonumentId NVARCHAR(50) NOT NULL,
    ImageUrl NVARCHAR(500) NOT NULL,
    DisplayOrder INT NOT NULL,
    IsMainImage BIT NOT NULL DEFAULT 0,
    UploadedAt DATETIME2 NOT NULL,
    UploadedBy NVARCHAR(50),
    FOREIGN KEY (MonumentId) REFERENCES Monuments(Id) ON DELETE CASCADE,
    UNIQUE(MonumentId, DisplayOrder)
);
```

### 6. TicketCategories

Visitor type pricing for monuments.

```sql
CREATE TABLE TicketCategories (
    Id NVARCHAR(50) PRIMARY KEY,
    MonumentId NVARCHAR(50) NOT NULL,
    Name NVARCHAR(100) NOT NULL, -- Adult, Child, Student
    Price DECIMAL(10, 2) NOT NULL,
    Description NVARCHAR(500),
    IsActive BIT NOT NULL DEFAULT 1,
    CreatedAt DATETIME2 NOT NULL,
    UpdatedAt DATETIME2 NOT NULL,
    FOREIGN KEY (MonumentId) REFERENCES Monuments(Id) ON DELETE CASCADE,
    UNIQUE(MonumentId, Name),
    CHECK (Price >= 0)
);
```

**Constraints:**
- `Price` >= 0 (decimal, never float)
- `Name` unique per monument

### 7. Schedules (Slot Templates)

Recurring slot templates.

```sql
CREATE TABLE Schedules (
    Id NVARCHAR(50) PRIMARY KEY,
    MonumentId NVARCHAR(50) NOT NULL,
    DayOfWeek INT NOT NULL, -- 0=Sunday, 1=Monday, ... 6=Saturday
    StartTime TIME NOT NULL,
    EndTime TIME NOT NULL,
    CapacityMode NVARCHAR(50) NOT NULL, -- limited, unlimited
    Capacity INT, -- NULL if unlimited
    BookingCutoffMinutes INT NOT NULL,
    EntryGraceMinutes INT NOT NULL,
    IsDisabled BIT NOT NULL DEFAULT 0,
    CreatedAt DATETIME2 NOT NULL,
    UpdatedAt DATETIME2 NOT NULL,
    FOREIGN KEY (MonumentId) REFERENCES Monuments(Id) ON DELETE CASCADE,
    CHECK (StartTime < EndTime),
    CHECK (Capacity IS NULL OR Capacity > 0)
);
```

**Constraints:**
- `DayOfWeek` 0-6 (Sunday to Saturday)
- `StartTime` < `EndTime`
- `Capacity` > 0 or NULL (unlimited)

**Indexes:**
- `MonumentId, DayOfWeek` (for retrieving weekly schedule)

### 8. SlotInstances

Specific dated occurrences of a schedule.

```sql
CREATE TABLE SlotInstances (
    Id NVARCHAR(50) PRIMARY KEY,
    ScheduleId NVARCHAR(50) NOT NULL,
    MonumentId NVARCHAR(50) NOT NULL,
    [Date] DATE NOT NULL,
    StartTime TIME NOT NULL,
    EndTime TIME NOT NULL,
    CapacityMode NVARCHAR(50) NOT NULL,
    Capacity INT,
    ConfirmedVisitors INT NOT NULL DEFAULT 0,
    ReservationHoldsVisitors INT NOT NULL DEFAULT 0,
    Status NVARCHAR(50) NOT NULL DEFAULT 'normal', -- normal, override, disabled
    CreatedAt DATETIME2 NOT NULL,
    FOREIGN KEY (ScheduleId) REFERENCES Schedules(Id) ON DELETE CASCADE,
    FOREIGN KEY (MonumentId) REFERENCES Monuments(Id) ON DELETE CASCADE,
    UNIQUE(MonumentId, [Date], StartTime),
    CHECK (ConfirmedVisitors >= 0),
    CHECK (ReservationHoldsVisitors >= 0)
);
```

**Key Constraints:**
- `Capacity` > 0 or NULL
- `ConfirmedVisitors` >= 0
- `ReservationHoldsVisitors` >= 0
- `ConfirmedVisitors + ReservationHoldsVisitors` <= `Capacity` (checked at application level during booking)

**Indexes:**
- `MonumentId, [Date]` (for availability queries)
- `[Date], Status` (for reporting)

### 9. BlockedPeriods

Date/slot closures.

```sql
CREATE TABLE BlockedPeriods (
    Id NVARCHAR(50) PRIMARY KEY,
    MonumentId NVARCHAR(50) NOT NULL,
    StartDate DATE NOT NULL,
    EndDate DATE NOT NULL,
    SlotInstanceId NVARCHAR(50), -- NULL = entire day
    Reason NVARCHAR(100) NOT NULL, -- Maintenance, Holiday, PrivateEvent, Emergency
    PublicMessage NVARCHAR(500),
    AffectedBookingCount INT DEFAULT 0,
    CreatedAt DATETIME2 NOT NULL,
    CreatedBy NVARCHAR(50),
    FOREIGN KEY (MonumentId) REFERENCES Monuments(Id) ON DELETE CASCADE,
    FOREIGN KEY (SlotInstanceId) REFERENCES SlotInstances(Id) ON DELETE SET NULL,
    CHECK (StartDate <= EndDate)
);
```

**Indexes:**
- `MonumentId, StartDate, EndDate` (for closure lookups)

### 10. Bookings

Visitor reservations.

```sql
CREATE TABLE Bookings (
    Id NVARCHAR(50) PRIMARY KEY,
    Reference NVARCHAR(50) NOT NULL UNIQUE, -- BKG-2026-001234
    MonumentId NVARCHAR(50) NOT NULL,
    SlotInstanceId NVARCHAR(50) NOT NULL,
    VisitDate DATE NOT NULL,
    VisitorName NVARCHAR(255) NOT NULL,
    Email NVARCHAR(255),
    Phone NVARCHAR(20),
    TotalVisitors INT NOT NULL,
    Subtotal DECIMAL(10, 2) NOT NULL,
    Tax DECIMAL(10, 2) NOT NULL,
    TotalPrice DECIMAL(10, 2) NOT NULL,
    Currency NVARCHAR(3) NOT NULL DEFAULT 'INR',
    Status NVARCHAR(50) NOT NULL, -- pending, confirmed, cancelled, expired
    PaymentStatus NVARCHAR(50) NOT NULL, -- not_paid, pending, paid, failed, refunded
    BookingChannel NVARCHAR(50) NOT NULL, -- website, counter
    CreatedAt DATETIME2 NOT NULL,
    ConfirmedAt DATETIME2,
    CancelledAt DATETIME2,
    CancellationReason NVARCHAR(500),
    CreatedBy NVARCHAR(50), -- NULL for visitor bookings, operator ID for counter
    FOREIGN KEY (MonumentId) REFERENCES Monuments(Id),
    FOREIGN KEY (SlotInstanceId) REFERENCES SlotInstances(Id),
    CHECK (TotalVisitors > 0),
    CHECK (Subtotal >= 0),
    CHECK (Tax >= 0),
    CHECK (TotalPrice >= 0)
);
```

**Key Constraints:**
- `Reference` is unique and public-facing
- `TotalVisitors` > 0
- All money fields are decimals, never floats
- `Status` and `PaymentStatus` are controlled enums

**Indexes:**
- `Reference` (for lookup by booking reference)
- `Email, Phone` (for visitor self-service)
- `MonumentId, VisitDate, Status` (for reporting)
- `CreatedAt` (for time-based queries)

### 11. BookingItems

Quantity breakdown per ticket category.

```sql
CREATE TABLE BookingItems (
    Id NVARCHAR(50) PRIMARY KEY,
    BookingId NVARCHAR(50) NOT NULL,
    TicketCategoryId NVARCHAR(50) NOT NULL,
    Quantity INT NOT NULL,
    UnitPrice DECIMAL(10, 2) NOT NULL,
    Subtotal DECIMAL(10, 2) NOT NULL,
    FOREIGN KEY (BookingId) REFERENCES Bookings(Id) ON DELETE CASCADE,
    FOREIGN KEY (TicketCategoryId) REFERENCES TicketCategories(Id),
    CHECK (Quantity > 0),
    CHECK (UnitPrice >= 0),
    CHECK (Subtotal >= 0)
);
```

### 12. ReservationHolds

Temporary capacity reservations during booking flow.

```sql
CREATE TABLE ReservationHolds (
    Id NVARCHAR(50) PRIMARY KEY,
    SlotInstanceId NVARCHAR(50) NOT NULL,
    BookingId NVARCHAR(50),
    Visitors INT NOT NULL,
    ExpiresAt DATETIME2 NOT NULL,
    Status NVARCHAR(50) NOT NULL DEFAULT 'active', -- active, released, expired
    CreatedAt DATETIME2 NOT NULL,
    FOREIGN KEY (SlotInstanceId) REFERENCES SlotInstances(Id) ON DELETE CASCADE,
    FOREIGN KEY (BookingId) REFERENCES Bookings(Id) ON DELETE SET NULL,
    CHECK (Visitors > 0)
);
```

**Indexes:**
- `SlotInstanceId, Status, ExpiresAt` (for hold expiry cleanup)

### 13. Payments

Financial transactions.

```sql
CREATE TABLE Payments (
    Id NVARCHAR(50) PRIMARY KEY,
    BookingId NVARCHAR(50) NOT NULL UNIQUE,
    Amount DECIMAL(10, 2) NOT NULL,
    Currency NVARCHAR(3) NOT NULL DEFAULT 'INR',
    Method NVARCHAR(50) NOT NULL, -- card, netbanking, upi, cash, check
    Status NVARCHAR(50) NOT NULL, -- not_initiated, pending, paid, failed, cancelled
    TransactionId NVARCHAR(100),
    ProviderReference NVARCHAR(255), -- e.g., Razorpay order ID
    ErrorMessage NVARCHAR(500),
    ProcessedAt DATETIME2,
    CreatedAt DATETIME2 NOT NULL,
    UpdatedAt DATETIME2 NOT NULL,
    FOREIGN KEY (BookingId) REFERENCES Bookings(Id) ON DELETE CASCADE,
    CHECK (Amount > 0)
);
```

**Indexes:**
- `BookingId` (for lookup by booking)
- `Status, ProcessedAt` (for reconciliation)

### 14. Refunds

Refund records for cancelled bookings.

```sql
CREATE TABLE Refunds (
    Id NVARCHAR(50) PRIMARY KEY,
    PaymentId NVARCHAR(50) NOT NULL,
    BookingId NVARCHAR(50) NOT NULL,
    Amount DECIMAL(10, 2) NOT NULL,
    Reason NVARCHAR(500) NOT NULL,
    Status NVARCHAR(50) NOT NULL, -- pending, processed, failed
    ProcessedAt DATETIME2,
    CreatedAt DATETIME2 NOT NULL,
    CreatedBy NVARCHAR(50),
    FOREIGN KEY (PaymentId) REFERENCES Payments(Id),
    FOREIGN KEY (BookingId) REFERENCES Bookings(Id) ON DELETE CASCADE,
    CHECK (Amount > 0)
);
```

### 15. Tickets

QR credentials for bookings.

```sql
CREATE TABLE Tickets (
    Id NVARCHAR(50) PRIMARY KEY,
    BookingId NVARCHAR(50) NOT NULL UNIQUE,
    Token NVARCHAR(500) NOT NULL UNIQUE, -- Opaque or JWT token
    QrCodeData NVARCHAR(MAX), -- Base64 encoded PNG/SVG QR code
    Status NVARCHAR(50) NOT NULL DEFAULT 'active', -- active, used, revoked, expired
    CreatedAt DATETIME2 NOT NULL,
    RevokedAt DATETIME2,
    RevokedReason NVARCHAR(255),
    FOREIGN KEY (BookingId) REFERENCES Bookings(Id) ON DELETE CASCADE
);
```

**Constraints:**
- `Token` is unique and cryptographically random
- `Token` never stored in audit logs

**Indexes:**
- `Token` (for fast QR verification lookup)
- `BookingId` (for ticket retrieval)

### 16. CheckIns

Admission records.

```sql
CREATE TABLE CheckIns (
    Id NVARCHAR(50) PRIMARY KEY,
    BookingId NVARCHAR(50) NOT NULL UNIQUE, -- One check-in per booking (MVP)
    TicketId NVARCHAR(50) NOT NULL,
    MonumentId NVARCHAR(50) NOT NULL,
    OperatorId NVARCHAR(50) NOT NULL,
    LocationId NVARCHAR(50), -- Gate or counter ID
    CheckInTime DATETIME2 NOT NULL,
    Status NVARCHAR(50) NOT NULL DEFAULT 'success', -- success, failed
    Notes NVARCHAR(500),
    CreatedAt DATETIME2 NOT NULL,
    FOREIGN KEY (BookingId) REFERENCES Bookings(Id) ON DELETE CASCADE,
    FOREIGN KEY (TicketId) REFERENCES Tickets(Id),
    FOREIGN KEY (MonumentId) REFERENCES Monuments(Id),
    FOREIGN KEY (OperatorId) REFERENCES Users(Id)
);
```

**Constraints:**
- `UNIQUE(BookingId)` – Only one check-in per booking (MVP constraint)
- `CheckInTime` must be within entry window

**Indexes:**
- `BookingId` (for lookup)
- `MonumentId, CheckInTime` (for reporting)
- `OperatorId, CheckInTime` (for operator activity)

### 17. VerificationAttempts

Audit log for QR verification activity.

```sql
CREATE TABLE VerificationAttempts (
    Id NVARCHAR(50) PRIMARY KEY,
    TicketId NVARCHAR(50) NOT NULL,
    BookingId NVARCHAR(50) NOT NULL,
    MonumentId NVARCHAR(50) NOT NULL,
    OperatorId NVARCHAR(50) NOT NULL,
    LocationId NVARCHAR(50),
    Outcome NVARCHAR(50) NOT NULL, -- success, invalid_qr, cancelled, wrong_monument, outside_window, already_used, system_error
    AttemptTime DATETIME2 NOT NULL,
    ErrorDetail NVARCHAR(500),
    IpAddress NVARCHAR(50),
    FOREIGN KEY (TicketId) REFERENCES Tickets(Id) ON DELETE CASCADE,
    FOREIGN KEY (BookingId) REFERENCES Bookings(Id),
    FOREIGN KEY (MonumentId) REFERENCES Monuments(Id),
    FOREIGN KEY (OperatorId) REFERENCES Users(Id)
);
```

**Indexes:**
- `MonumentId, AttemptTime` (for audit reports)
- `OperatorId, AttemptTime` (for operator activity tracking)

### 18. AuditLogs

Administrative change audit trail.

```sql
CREATE TABLE AuditLogs (
    Id NVARCHAR(50) PRIMARY KEY,
    Timestamp DATETIME2 NOT NULL,
    UserId NVARCHAR(50),
    Action NVARCHAR(50) NOT NULL, -- create, update, delete, publish, archive, cancel
    EntityType NVARCHAR(50) NOT NULL, -- Monument, Booking, TicketCategory, etc.
    EntityId NVARCHAR(50) NOT NULL,
    EntityName NVARCHAR(255),
    OldValues NVARCHAR(MAX), -- JSON
    NewValues NVARCHAR(MAX), -- JSON
    Changes NVARCHAR(MAX), -- JSON summary of field changes
    IpAddress NVARCHAR(50),
    UserAgent NVARCHAR(500),
    CreatedAt DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    FOREIGN KEY (UserId) REFERENCES Users(Id) ON DELETE SET NULL
);
```

**Indexes:**
- `Timestamp DESC` (for recent activity)
- `EntityType, EntityId` (for change history per entity)
- `UserId, Timestamp` (for user activity)

### 19. Notifications

Email/SMS delivery tracking.

```sql
CREATE TABLE Notifications (
    Id NVARCHAR(50) PRIMARY KEY,
    BookingId NVARCHAR(50) NOT NULL,
    Type NVARCHAR(50) NOT NULL, -- confirmation, cancellation, reminder, refund_issued
    Channel NVARCHAR(50) NOT NULL, -- email, sms
    Recipient NVARCHAR(255) NOT NULL,
    Subject NVARCHAR(500),
    Body NVARCHAR(MAX),
    Status NVARCHAR(50) NOT NULL DEFAULT 'pending', -- pending, sent, failed, bounced
    SentAt DATETIME2,
    FailureReason NVARCHAR(500),
    RetryCount INT DEFAULT 0,
    NextRetryAt DATETIME2,
    CreatedAt DATETIME2 NOT NULL,
    FOREIGN KEY (BookingId) REFERENCES Bookings(Id) ON DELETE CASCADE
);
```

**Indexes:**
- `Status, NextRetryAt` (for background job to retry failed notifications)

### 20. Reports

Generated report exports.

```sql
CREATE TABLE Reports (
    Id NVARCHAR(50) PRIMARY KEY,
    CreatedBy NVARCHAR(50) NOT NULL,
    ReportType NVARCHAR(50) NOT NULL, -- booking, attendance, occupancy, revenue, etc.
    Filters NVARCHAR(MAX), -- JSON of filter criteria
    ExportUrl NVARCHAR(500),
    Format NVARCHAR(20) NOT NULL, -- json, csv, excel, pdf
    RowCount INT,
    FileSize INT,
    CreatedAt DATETIME2 NOT NULL,
    ExpiresAt DATETIME2, -- Auto-delete old reports
    FOREIGN KEY (CreatedBy) REFERENCES Users(Id) ON DELETE SET NULL
);
```

---

## Relationships

### Relationship Diagram

```
Users (1) ──────────── (M) UserMonumentAssignments ──────────── (1) Monuments
  │                                                                    │
  │                                                                    │
  └─────── (1) ─────────────────────────────────────────────────── (M) Bookings
           Role-based                                              Status: pending|
           Authorization                                           confirmed|
                                                                   cancelled|
                                                                   expired
                                                                    │
                                                         ┌─────────┴─────────┐
                                                         │                   │
                                                      (1) Tickets         (1) Payments
                                                      (QR Code)           (Payment Info)
                                                         │                   │
                                                         │            Status: pending|
                                                         │            paid|failed|refunded
                                                         │                   │
                                                      (1) CheckIns       (1) Refunds
                                                      (Admission)
                                                         │
                                                      (M) VerificationAttempts
                                                      (Audit Trail)

Monuments (1) ──────────────────── (M) TicketCategories
    │                               (Pricing)
    │
    ├─── (1) ─────────────────── (M) Schedules (Slot Templates)
    │                            (Recurring)
    │
    ├─── (1) ─────────────────── (M) SlotInstances (Dated Slots)
    │                            (Capacity + Bookings)
    │
    ├─── (1) ─────────────────── (M) BlockedPeriods
    │                            (Closures)
    │
    └─── (1) ─────────────────── (M) MonumentImages
                                 (Gallery)

SlotInstances (1) ──────────────────── (M) Bookings
                                        (Capacity Reserved)

BookingItems     (M) ────────────────── (1) TicketCategories
                                        (Price Snapshot)
```

### Cascade Delete Behavior

| Parent Table | Child Table | Behavior |
|---|---|---|
| Monuments | TicketCategories | CASCADE (soft delete archive instead) |
| Monuments | Schedules | CASCADE |
| Schedules | SlotInstances | CASCADE |
| SlotInstances | Bookings | CASCADE (soft cancel instead) |
| Bookings | BookingItems | CASCADE |
| Bookings | Tickets | CASCADE |
| Bookings | CheckIns | CASCADE |
| Bookings | Payments | CASCADE |
| Bookings | Refunds | CASCADE |
| Users | AuditLogs | SET NULL |

---

## Indexes

### Critical Indexes for Performance

```sql
-- Availability queries (frequent)
CREATE INDEX idx_SlotInstances_MonumentDate ON SlotInstances(MonumentId, [Date], Status);
CREATE INDEX idx_Bookings_MonumentVisitDate ON Bookings(MonumentId, VisitDate, Status);

-- Booking lookup (frequent)
CREATE INDEX idx_Bookings_Reference ON Bookings(Reference);
CREATE INDEX idx_Tickets_Token ON Tickets(Token);

-- Reporting
CREATE INDEX idx_CheckIns_MonumentTime ON CheckIns(MonumentId, CheckInTime);
CREATE INDEX idx_BookingItems_BookingCategory ON BookingItems(BookingId, TicketCategoryId);
CREATE INDEX idx_VerificationAttempts_MonumentTime ON VerificationAttempts(MonumentId, AttemptTime);
CREATE INDEX idx_AuditLogs_TimestampDesc ON AuditLogs(Timestamp DESC);

-- Background jobs
CREATE INDEX idx_ReservationHolds_Expiry ON ReservationHolds(SlotInstanceId, Status, ExpiresAt);
CREATE INDEX idx_Notifications_Retry ON Notifications(Status, NextRetryAt);

-- Listing and search
CREATE INDEX idx_Monuments_StatusActive ON Monuments([Status], IsActive);
CREATE INDEX idx_Monuments_Name ON Monuments(Name);

-- User management
CREATE INDEX idx_Users_UsernameEmail ON Users(Username, Email);
CREATE INDEX idx_UserMonumentAssignments_UserMonument ON UserMonumentAssignments(UserId, MonumentId);
```

---

## Constraints & Validation

### Entity Framework Core Validations

```csharp
// Example: Monument Entity
public class Monument
{
    public string Id { get; set; }
    
    [Required]
    [StringLength(255)]
    public string Name { get; set; }
    
    [StringLength(100)]
    public string TimeZone { get; set; } // Validated against IANA zones
    
    [Range(0, int.MaxValue)]
    public int BookingWindowDays { get; set; }
    
    [EnumDataType(typeof(MonumentStatus))]
    public MonumentStatus Status { get; set; } // draft, published, inactive, archived
}

// Example: Booking Entity
public class Booking
{
    [Range(1, int.MaxValue)]
    public int TotalVisitors { get; set; }
    
    [Column(TypeName = "decimal(10,2)")]
    [Range(0, 999999.99)]
    public decimal Subtotal { get; set; } // Never float!
    
    [EnumDataType(typeof(BookingStatus))]
    public BookingStatus Status { get; set; }
}
```

### Database Constraints

| Constraint | Validation |
|---|---|
| `CHECK (TotalVisitors > 0)` | Bookings must have at least 1 visitor |
| `CHECK (Price >= 0)` | Ticket prices cannot be negative |
| `CHECK (Subtotal >= 0)` | Money fields non-negative |
| `CHECK (StartTime < EndTime)` | Slot start before end |
| `CHECK (StartDate <= EndDate)` | Block period validity |
| `UNIQUE(Reference)` | Booking references are unique |
| `UNIQUE(Token)` | QR tokens are unique |
| `UNIQUE(BookingId)` | One check-in per booking (MVP) |

---

## Migrations & Setup

### EF Core Code-First Workflow

```bash
# Create initial migration
dotnet ef migrations add InitialCreate

# Generate migration script (review before applying)
dotnet ef migrations script -o schema.sql

# Apply migrations to database
dotnet ef database update

# Create database and apply all migrations
dotnet ef database update --verbose
```

### Migration File Structure

```
/src/MonumentsAPI/Data/Migrations/
├── 20261001000000_InitialCreate.cs
├── 20261002000000_AddMonumentImages.cs
├── 20261003000000_AddReservationHolds.cs
└── 20261005000000_AddAuditLogging.cs
```

### Sample DbContext Configuration

```csharp
public class MonumentsDbContext : DbContext
{
    public DbSet<User> Users { get; set; }
    public DbSet<Monument> Monuments { get; set; }
    public DbSet<Booking> Bookings { get; set; }
    public DbSet<Ticket> Tickets { get; set; }
    public DbSet<CheckIn> CheckIns { get; set; }
    // ... other DbSets

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        // Configure unique indexes
        modelBuilder.Entity<Booking>()
            .HasIndex(b => b.Reference)
            .IsUnique();

        modelBuilder.Entity<Ticket>()
            .HasIndex(t => t.Token)
            .IsUnique();

        // Configure cascade behaviors
        modelBuilder.Entity<Booking>()
            .HasOne(b => b.SlotInstance)
            .WithMany()
            .OnDelete(DeleteBehavior.Restrict); // Prevent accidental cascade

        // Configure decimal precision
        modelBuilder.Entity<Booking>()
            .Property(b => b.Subtotal)
            .HasPrecision(10, 2);

        // Configure JSON columns (for AuditLogs)
        modelBuilder.Entity<AuditLog>()
            .Property(a => a.Changes)
            .HasConversion(
                v => JsonSerializer.Serialize(v),
                v => JsonSerializer.Deserialize<Dictionary<string, object>>(v));
    }
}
```

---

## Sample Data

### Seed Test Monuments

```sql
INSERT INTO Monuments (Id, Name, Description, Location_City, TimeZone, Status, CapacityMode, BookingWindowDays, BookingCutoffMinutes)
VALUES 
('mon-001', 'Taj Mahal', 'Ivory-white marble mausoleum', 'Agra', 'Asia/Kolkata', 'published', 'limited', 60, 30),
('mon-002', 'Red Fort', 'Historic fort complex', 'Delhi', 'Asia/Kolkata', 'published', 'limited', 60, 30),
('mon-003', 'Mysore Palace', 'Royal palace and museum', 'Mysore', 'Asia/Kolkata', 'draft', 'unlimited', 30, 15);

INSERT INTO TicketCategories (Id, MonumentId, Name, Price)
VALUES 
('cat-001', 'mon-001', 'Adult', 100.00),
('cat-002', 'mon-001', 'Child', 50.00),
('cat-003', 'mon-002', 'Adult', 75.00);

INSERT INTO Schedules (Id, MonumentId, DayOfWeek, StartTime, EndTime, CapacityMode, Capacity)
VALUES 
('sched-001', 'mon-001', 1, '06:00:00', '07:00:00', 'limited', 100), -- Monday
('sched-002', 'mon-001', 1, '09:00:00', '10:00:00', 'limited', 150),
('sched-003', 'mon-001', 3, '11:00:00', '12:00:00', 'unlimited', NULL); -- Wednesday
```

### Seed Test Users

```sql
INSERT INTO Users (Id, Username, Email, PasswordHash, FullName, Role)
VALUES 
('user-admin', 'admin@monuments.local', 'admin@monuments.local', '[BCRYPT_HASH]', 'System Admin', 'SystemAdmin'),
('user-mgr-001', 'manager@monuments.local', 'manager@monuments.local', '[BCRYPT_HASH]', 'Taj Mahal Manager', 'MonumentManager'),
('user-op-001', 'operator@monuments.local', 'operator@monuments.local', '[BCRYPT_HASH]', 'Counter Operator', 'OperatorCounter');
```

---

## Performance Tuning

### Query Optimization

```csharp
// ❌ Bad: N+1 query problem
var bookings = dbContext.Bookings
    .Where(b => b.Status == BookingStatus.Confirmed)
    .ToList(); // Now will query each BookingItem separately!

foreach (var booking in bookings)
{
    var items = booking.BookingItems; // Query per booking
}

// ✅ Good: Eager loading
var bookings = dbContext.Bookings
    .Where(b => b.Status == BookingStatus.Confirmed)
    .Include(b => b.BookingItems)
    .ThenInclude(bi => bi.TicketCategory)
    .ToList(); // Single query with joins
```

### Availability Check Query (Optimized)

```csharp
// Single query for slot availability and remaining capacity
var slot = await dbContext.SlotInstances
    .Where(s => s.MonumentId == monumentId && s.Date == visitDate && s.StartTime == slotTime)
    .Select(s => new {
        Id = s.Id,
        Capacity = s.Capacity,
        BookedVisitors = s.ConfirmedVisitors,
        AvailableCapacity = s.Capacity - s.ConfirmedVisitors - s.ReservationHoldsVisitors
    })
    .FirstOrDefaultAsync();
```

### Capacity Check Transaction

```csharp
// Atomic capacity reservation with transaction
using (var transaction = await dbContext.Database.BeginTransactionAsync())
{
    try
    {
        // Lock the row for update
        var slot = await dbContext.SlotInstances
            .FromSqlRaw("SELECT * FROM SlotInstances WHERE Id = {0} WITH (UPDLOCK)", slotId)
            .FirstOrDefaultAsync();

        if (slot.ConfirmedVisitors + slot.ReservationHoldsVisitors + visitorCount > slot.Capacity)
            throw new CapacityExceededException();

        // Create reservation hold
        var hold = new ReservationHold { SlotInstanceId = slotId, Visitors = visitorCount };
        dbContext.ReservationHolds.Add(hold);
        await dbContext.SaveChangesAsync();

        await transaction.CommitAsync();
    }
    catch
    {
        await transaction.RollbackAsync();
        throw;
    }
}
```

### Query Plan Analysis

Run EXPLAIN PLAN before production:

```sql
-- SQL Server
SET STATISTICS IO ON;
SELECT * FROM Bookings WHERE MonumentId = 'mon-001' AND VisitDate = '2026-10-10';
SET STATISTICS IO OFF;

-- PostgreSQL
EXPLAIN ANALYZE
SELECT * FROM Bookings WHERE MonumentId = 'mon-001' AND VisitDate = '2026-10-10';
```

---

## Backup & Recovery

### Backup Strategy

| Type | Frequency | Retention |
|---|---|---|
| Full Backup | Daily (2 AM UTC) | 30 days |
| Incremental | Hourly | 7 days |
| Transaction Log | Every 15 minutes | 7 days |

### Backup Commands

```bash
# SQL Server
BACKUP DATABASE MonumentsBooking 
TO DISK = '/backups/monuments_full_2026-10-05.bak'
WITH INIT, COMPRESSION;

# PostgreSQL
pg_dump -U postgres -h localhost monuments_booking > /backups/monuments_full_2026-10-05.sql
```

### Recovery Procedure

```sql
-- Restore from backup (SQL Server)
RESTORE DATABASE MonumentsBooking 
FROM DISK = '/backups/monuments_full_2026-10-05.bak'
WITH REPLACE;

-- Restore transaction log to point-in-time
RESTORE LOG MonumentsBooking 
FROM DISK = '/backups/tlog_2026-10-05_1400.trn'
WITH STOPAT = '2026-10-05 14:00:00';
```

### Point-in-Time Recovery

Maintain detailed audit logs to reconstruct data:

```sql
-- Recover a cancelled booking
SELECT * FROM AuditLogs 
WHERE EntityType = 'Booking' 
AND EntityId = 'bkg-001'
ORDER BY Timestamp DESC;
```

---

## Data Retention Policy

| Data Type | Retention Period | Archive | Notes |
|---|---|---|---|
| Confirmed Bookings | Indefinite | No | Historical records for reporting |
| Cancelled Bookings | 7 years | After 1 year | Tax/legal compliance |
| Audit Logs | 2 years | After 1 year | Compliance with regulations |
| Verification Attempts | 1 year | After 6 months | Security audit trail |
| Notifications | 90 days | No | Cleanup old email/SMS records |
| Generated Reports | 30 days | No | Auto-cleanup old exports |
| Visitor Personal Data | Per booking retention + 90 days | No | GDPR/privacy compliance |

---

**Database Version:** 1.0  
**Last Updated:** October 5, 2026  
**ORM:** Entity Framework Core  
**Database Platform:** To be confirmed (SQL Server / PostgreSQL / MySQL)