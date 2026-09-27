# System Design Document (SDD)
## BJRS Cinema Ticketing System

**Document Version:** 1.0.0  
**Status:** Approved  
**Author:** Architecture & Engineering Team  
**Tech Stack:** Laravel 11.x (REST API) | React 18+ (Vite SPA) | Supabase (PostgreSQL) | Laravel Sanctum | Redis / Queues | TailwindCSS  

---

## 1. System Overview & Architecture

The BJRS Cinema Ticketing System employs a modern decoupled **Client-Server Architecture** consisting of an independent **React Single-Page Application (SPA)** communicating over HTTP/HTTPS with a high-performance **Laravel 11 RESTful API Backend**.

```
┌────────────────────────────────────────────────────────────────────────┐
│                   React 18+ Client Layer (Vite SPA)                    │
│   React Router DOM | State (Zustand / Context) | TailwindCSS | Axios   │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ HTTPS / REST JSON API
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                   Laravel 11 Routing & Middleware                      │
│      routes/api.php ──► Sanctum Auth ──► RoleMiddleware (RBAC)         │
│                        ──► BranchScopeMiddleware                       │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Form Requests (Validation)
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                      Laravel API Controllers                           │
│   AuthController | BranchController | ShowtimeController               │
│   SeatController | BookingController | MovieController | Dashboard     │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                        Service & Business Layer                        │
│   BookingService | SeatLockService | ShowtimeConflictService           │
│   CredentialDispatchService | AnalyticsService                         │
└──────────────────┬─────────────────────────────────┬───────────────────┘
                   │                                 │
                   ▼                                 ▼
┌─────────────────────────────────────┐  ┌───────────────────────────────┐
│     Async Queue & Notification      │  │        Eloquent ORM           │
│    Mailables (StaffWelcomeMail,     │  │   Models, Repositories, DB    │
│       TicketConfirmationMail)       │  │   Transactions & Optimistic/  │
│        Redis / DB Queue             │  │   Pessimistic Seat Locking    │
└─────────────────────────────────────┘  └───────────────┬───────────────┘
                                                         │
                                                         ▼
                                         ┌───────────────────────────────┐
                                         │     Supabase (PostgreSQL)     │
                                         │  Tables, Foreign Keys, Indexes│
                                         └───────────────────────────────┘
```

---

## 2. Technology Stack & Framework Choices

| Layer | Technology | Rationale |
| :--- | :--- | :--- |
| **Frontend Runtime** | React 18+ with Vite | Blazing fast build tooling, declarative component architecture, zero-page-reload user flows, dynamic interactive seat layout rendering. |
| **Frontend Styling** | TailwindCSS + Modern CSS | Utility-first responsive design, dark cinema aesthetic, glassmorphic UI components, consistent design tokens. |
| **Frontend State & HTTP**| Axios + Zustand / React Context | Clean HTTP interceptors for automatic Bearer token injection, predictable state management for seat selection carts. |
| **Backend Framework**| Laravel 11.x (PHP 8.2+) | Robust ecosystem: native routing, dependency injection, Eloquent ORM, Form Request validation, background queues, and Mailables. |
| **API Authentication** | Laravel Sanctum | Secure token-based authentication for SPAs and mobile/external clients with granular token abilities. |
| **Database Engine** | Supabase (PostgreSQL) | Managed PostgreSQL with connection pooling, transactional integrity during seat reservations, foreign key cascades, and soft deletes. |
| **Async Queuing** | Laravel Queues (Database / Redis) | Offloads email generation and external notifications from the HTTP request cycle for $< 100\text{ ms}$ response times. |

---

## 3. Database Schema Overview

The relational database layer models the cinema domain with foreign key relationships, soft-delete tracking (`deleted_at`), and timestamp audit trails:

| Table Name | Primary Purpose | Key Relationships |
| :--- | :--- | :--- |
| `branches` | Cinema branch locations and contact metadata. | Has many `admins`, `rooms` |
| `admins` | Super Admins and Branch Admins with hashed passwords and roles. | Belongs to `branches` (optional for Super Admin) |
| `rooms` | Screening auditoriums (Regular, VIP) with row/column dimensions. | Belongs to `branches`, has many `seats`, `showtimes` |
| `seats` | Physical seat entities (row, number, label, type like Standard/VIP Recliner). | Belongs to `rooms`, referenced by `tickets` |
| `movies` | Global movie catalog (title, synopsis, rating, duration, poster/trailer URLs). | Has many `showtimes` |
| `showtimes` | Scheduled movie screenings in specific rooms with regular/VIP pricing. | Belongs to `movies` and `rooms`, has many `bookings` |
| `bookings` | Customer transaction header with unique reference, total, and status. | Belongs to `users` and `showtimes`, has many `tickets`, has one `payments` |
| `tickets` | Individual reserved seat line-item within a booking. | Belongs to `bookings` and `seats` |
| `payments` | Settlement record tracking payment method, transaction ID, and amount paid. | Belongs to `bookings` |
| `users` | Customer accounts for authentication, booking history, and e-ticket storage. | Has many `bookings` |

---

## 4. RESTful API Architecture & Endpoint Catalog

All API endpoints follow RESTful conventions, adhere to standard HTTP status codes, and return consistent JSON response envelopes:

```json
{
  "success": true,
  "message": "Resource retrieved successfully",
  "data": { ... },
  "errors": null
}
```

### 4.1 Authentication Endpoints (`/api/v1/auth`)
| Method | Route | Description | Auth Required |
| :--- | :--- | :--- | :--- |
| `POST` | `/api/v1/auth/customer/register` | Register customer account | No |
| `POST` | `/api/v1/auth/customer/login` | Authenticate customer & issue token | No |
| `POST` | `/api/v1/auth/admin/login` | Authenticate Admin (Super / Branch) | No |
| `POST` | `/api/v1/auth/logout` | Revoke active Sanctum token | Yes |
| `GET` | `/api/v1/auth/me` | Get authenticated user/admin profile | Yes |

### 4.2 Super Admin Endpoints (`/api/v1/admin/super`)
*Protected by `auth:sanctum` and `role:super_admin`*

| Method | Route | Description |
| :--- | :--- | :--- |
| `GET` | `/api/v1/admin/super/dashboard` | High-level metrics across all branches |
| `GET` | `/api/v1/admin/super/branches` | List all branches with staff records |
| `POST` | `/api/v1/admin/super/branches` | Create branch with manager & auto-send credentials |
| `GET` | `/api/v1/admin/super/branches/{id}` | Get specific branch details and staff |
| `PUT` | `/api/v1/admin/super/branches/{id}` | Update branch metadata and active status |
| `DELETE`| `/api/v1/admin/super/branches/{id}` | Soft-delete a cinema branch |
| `POST` | `/api/v1/admin/super/movies` | Add new movie to global catalog |
| `PUT` | `/api/v1/admin/super/movies/{id}` | Update movie metadata and screening status |
| `DELETE`| `/api/v1/admin/super/movies/{id}` | Soft-delete movie |

### 4.3 Branch Admin Endpoints (`/api/v1/admin/branch`)
*Protected by `auth:sanctum` and `role:branch_admin` (scoped to assigned `branch_id`)*

| Method | Route | Description |
| :--- | :--- | :--- |
| `GET` | `/api/v1/admin/branch/dashboard` | Real-time branch sales and room occupancy |
| `GET` | `/api/v1/admin/branch/rooms` | List screening halls in branch |
| `POST` | `/api/v1/admin/branch/rooms` | Create hall & auto-generate seat matrix |
| `GET` | `/api/v1/admin/branch/rooms/{id}/seats` | Fetch seat grid configuration for a room |
| `GET` | `/api/v1/admin/branch/showtimes` | List scheduled showtimes for branch |
| `POST` | `/api/v1/admin/branch/showtimes` | Schedule showtime with overlap conflict validation |
| `DELETE`| `/api/v1/admin/branch/showtimes/{id}` | Cancel/remove scheduled showtime |

### 4.4 Public & Customer Endpoints (`/api/v1`)
| Method | Route | Description | Auth Required |
| :--- | :--- | :--- | :--- |
| `GET` | `/api/v1/movies` | Get "Now Showing" and "Coming Soon" movies | No |
| `GET` | `/api/v1/movies/{slug}` | Get movie details, cast, and trailer | No |
| `GET` | `/api/v1/branches` | List all active cinema branches | No |
| `GET` | `/api/v1/showtimes` | Get available showtimes by movie, branch, and date | No |
| `GET` | `/api/v1/showtimes/{id}/seat-map` | Get real-time seat map with occupancy mask | No / Optional |
| `POST` | `/api/v1/bookings/hold-seats` | Temporarily hold seats for checkout (5 min lock) | Yes |
| `POST` | `/api/v1/bookings/checkout` | Process booking transaction & confirm payment | Yes |
| `GET` | `/api/v1/customer/bookings` | Get customer booking history | Yes |
| `GET` | `/api/v1/customer/bookings/{ref}` | Get digital e-ticket details & QR data | Yes |

---

## 5. Concurrency & Seat Locking Engine

To ensure **0.00% double-booking probability**, the booking engine utilizes a two-tier seat locking strategy:

```
[Customer Selects Seats] ──► POST /hold-seats ──► Redis / DB Lock Table (TTL: 5 Minutes)
                                                     │
                                                     ▼
                                            Seats marked "Held"
                                            in realtime seat-map
                                                     │
[Customer Confirms Payment] ──► POST /checkout
                                     │
                                     ▼
                      DB Transaction: BEGIN TRANSACTION
                      SELECT seats FOR UPDATE (Pessimistic Lock)
                      Verify: Are seats free or held by CURRENT user?
                               ├── YES: Insert BOOKING, TICKETS, PAYMENT
                               │        Commit Transaction, Release Hold
                               │        Dispatch Async Confirmation Mail
                               └── NO:  Rollback Transaction
                                        Return 409 Conflict Error
```

---

## 6. Security Architecture & Controls

1. **Token Authentication (Laravel Sanctum):** Secure API token management. Tokens are revoked on logout and expired after inactivity.
2. **Role & Branch Authorization Middleware:**
   - `CheckRole:super_admin`: Restricts access to global provisioning APIs.
   - `CheckRole:branch_admin` + `BranchScope`: Ensures branch managers cannot read or manipulate data belonging to other branches.
3. **Form Request Validation:** All incoming data is rigorously typed and sanitized before reaching controller logic.
4. **Password Security:** Salted Bcrypt hashing (`Hash::make($password)`). Auto-generated manager passwords meet enterprise complexity standards.
5. **CORS & Rate Limiting:** Configured `throttle:api` middleware prevents brute-force login and API flooding.

---

## 7. Frontend (React SPA) Architecture

```
frontend/src/
├── assets/             # Logos, icons, background assets
├── components/         # Reusable UI components
│   ├── common/         # Button, Modal, Card, Input, Badge, Toast
│   ├── layout/         # Navbar, Footer, AdminSidebar, PageContainer
│   ├── movie/          # MovieCard, MovieGrid, TrailerModal
│   ├── seatmap/        # InteractiveSeatGrid, SeatLegend, SeatItem
│   └── ticket/         # ETicketCard, QRCodeView, PrintableReceipt
├── context/            # React Contexts (AuthContext, BookingCartContext)
├── hooks/              # Custom hooks (useAuth, useSeatMap, useMovies)
├── pages/
│   ├── public/         # HomePage, MovieDetailPage, ShowtimeSelectPage, SeatPickerPage, CheckoutPage
│   ├── customer/       # CustomerLoginPage, RegisterPage, BookingHistoryPage, TicketViewPage
│   ├── branch-admin/   # BADashboardPage, RoomSetupPage, SchedulePage, SalesReportPage
│   └── super-admin/    # SADashboardPage, BranchListPage, StaffProvisionPage, MovieCatalogPage
├── services/           # Axios API instances & service functions (api.js, authService.js, bookingService.js)
├── routes/             # AppRoutes.jsx (Public, ProtectedCustomer, ProtectedBranchAdmin, ProtectedSuperAdmin)
└── App.jsx             # App entry, router provider, global toast notifications
```

---

## 8. Error Handling, Logging, and Observability

- **Unified Exception Handler:** Laravel's `bootstrap/app.php` or `app/Exceptions/Handler.php` captures all HTTP, database, and validation exceptions, converting them into structured JSON error envelopes with proper HTTP status codes (`400`, `401`, `403`, `404`, `422`, `409`, `500`).
- **Structured Logging:** Errors, booking failures, and payment anomalies are logged to `storage/logs/laravel.log` with contextual request IDs and user references.
- **Frontend Interceptor Trapping:** Axios response interceptors intercept `401 Unauthorized` (clearing auth state and redirecting to login) and `422 Validation Error` (populating field error messages automatically).
