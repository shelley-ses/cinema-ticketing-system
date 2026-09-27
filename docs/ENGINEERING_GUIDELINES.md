# AGENTS.md — AI Developer & Autonomous Agent Guidelines
## BJRS Cinema Ticketing System

**Document Version:** 1.0.0  
**Target Audience:** AI Coding Assistants, LLM Agents, and Software Engineers  
**Applies to:** Entire `cinema-ticketing-system` Repository  
**Tech Stack:** **Laravel 11.x (REST API Backend)** + **React 18+ (Vite SPA Frontend)**  

---

## 1. Context Poisoning Prevention & Anti-Patterns (CRITICAL)

To prevent architectural drift and hallucinated legacy patterns, all AI agents and developers **MUST** observe the following strict boundaries:

### ❌ FORBIDDEN LEGACY ANTI-PATTERNS (DO NOT USE)
1. **NO Raw Session Management:** Do NOT use `$_SESSION`, `session_start()`, or manual cookie headers. Use **Laravel Sanctum** token-based authentication.
2. **NO Procedural SQL Calls:** Do NOT use raw `sqlsrv_connect()`, `sqlsrv_query()`, or inline unparameterized SQL strings. Use **Laravel Eloquent Models**, **Query Builder**, or parameterized **DB Transactions / Stored Procedures**.
3. **NO Raw Header / Echo Responses:** Do NOT write `header('Content-Type: application/json')` followed by `echo json_encode()`. Always return typed Laravel `JsonResponse` (`response()->json(...)`) or `JsonResource` collections.


---

## 2. Target Architecture & Project Structure

The project follows a clean, decoupled monorepo or standard multi-tier layout:

```
cinema-ticketing-system/
├── backend/                            # Laravel 11.x REST API Backend
│   ├── app/
│   │   ├── Http/
│   │   │   ├── Controllers/Api/v1/     # API Controllers (Auth, Branch, Movie, Showtime, Booking)
│   │   │   ├── Middleware/             # RoleMiddleware, BranchScopeMiddleware
│   │   │   ├── Requests/               # Form Requests (CreateBranchRequest, CheckoutRequest, etc.)
│   │   │   └── Resources/              # JsonResources (MovieResource, BookingResource, SeatMapResource)
│   │   ├── Models/                     # Eloquent Models (Branch, Admin, Room, Seat, Movie, Showtime, Booking, Ticket, Payment, User)
│   │   ├── Services/                   # Business Services (BookingService, SeatLockService, ShowtimeConflictService)
│   │   └── Mail/                       # Mailables (StaffWelcomeMail, TicketConfirmationMail)
│   ├── database/
│   │   ├── migrations/                 # Laravel database schema migrations
│   │   └── seeders/                    # Database seeders for roles, movies, and test branches
│   ├── routes/
│   │   ├── api.php                     # Versioned RESTful API routes (/api/v1/...)
│   │   └── console.php                 # Artisan console commands
│   ├── config/                         # Laravel configuration (database, sanctum, mail, queue)
│   └── tests/                          # Feature and Unit tests (PHPUnit / Pest)
│
├── frontend/                           # React 18+ Single Page Application (Vite)
│   ├── src/
│   │   ├── assets/                     # Static media, icons, cinema themes
│   │   ├── components/                 # Reusable UI components
│   │   │   ├── common/                 # Button, Modal, Input, Badge, Toast, Spinner
│   │   │   ├── layout/                 # Navbar, Footer, AdminSidebar, ProtectedRoute
│   │   │   ├── movie/                  # MovieCard, MovieGrid, TrailerModal
│   │   │   ├── seatmap/                # InteractiveSeatGrid, SeatLegend, SeatItem
│   │   │   └── ticket/                 # ETicketCard, QRCodeView, ReceiptModal
│   │   ├── context/                    # AuthContext, BookingCartContext
│   │   ├── hooks/                      # Custom hooks (useAuth, useSeatMap, useMovies)
│   │   ├── pages/                      # Page components
│   │   │   ├── public/                 # HomePage, MovieDetailPage, SeatPickerPage, CheckoutPage
│   │   │   ├── customer/               # LoginPage, RegisterPage, HistoryPage, ETicketPage
│   │   │   ├── branch-admin/           # BADashboard, RoomSetupPage, SchedulePage, SalesPage
│   │   │   └── super-admin/            # SADashboard, BranchListPage, StaffPage, CatalogPage
│   │   ├── services/                   # Axios API client and endpoints (api.js, authService.js)
│   │   ├── routes/                     # React Router DOM configuration
│   │   ├── App.jsx                     # Root application component
│   │   └── main.jsx                    # Vite entry point
│   ├── package.json                    # React dependencies (Vite, React Router, Tailwind, Axios, Lucide)
│   └── tailwind.config.js              # TailwindCSS configuration
│
├── docs/                               # Official System Documentation
│   ├── PRD.md                          # Product Requirements Document
│   ├── SDD.md                          # System Design Document
│   ├── USER_FLOW.md                    # User Flow & State Diagrams
│   ├── USER_JOURNEY.md                 # User Journey Maps & Personas
│   └── AGENTS.md                       # This Developer & Agent Guideline
└── README.md                           # Main Project Overview & Setup Instructions
```

---

## 3. Core Development Rules for Agents

### 3.1 Backend Rules (Laravel 11)
1. **API Response Consistency:** Every API controller method must return a standard response structure:
   ```php
   return response()->json([
       'success' => true,
       'message' => 'Operation successful',
       'data'    => $data,
       'errors'  => null,
   ], 200);
   ```
2. **Form Request Validation:** Never validate incoming requests directly inside controllers with `$request->validate()`. Always create dedicated Form Requests in `app/Http/Requests/` with clear validation rules and custom error messages.
3. **Database Transactions:** Always wrap multi-entity state mutations (e.g., booking seats + inserting tickets + processing payments) inside `DB::transaction(function () { ... })` to ensure absolute ACID atomicity.
4. **Pessimistic / Atomic Locking:** When processing checkout, lock the affected seat records (`Seat::whereIn('id', $seatIds)->lockForUpdate()->get()`) or verify the active temporary hold to prevent double-booking.
5. **Asynchronous Mail & Notifications:** Email dispatchers (e.g. `Mail::to($email)->queue(new StaffWelcomeMail($credentials))`) must implement `ShouldQueue` to keep API response times $< 100\text{ ms}$.
6. **Soft Deletes:** Use `SoftDeletes` trait on Models representing physical infrastructure and transactions (`Branch`, `Room`, `Showtime`, `Booking`, `User`) to safeguard financial and historical audit data.

### 3.2 Frontend Rules (React 18+ / Vite)
1. **Component Modularity:** Keep components small, focused, and pure. Extract complex logic into custom React hooks (`useSeatMap`, `useShowtimeFilter`).
2. **Axios Centralization:** All HTTP calls must route through a centralized Axios client instance (`services/api.js`) equipped with:
   - Base URL configuration (`/api/v1`).
   - Request Interceptor: Injects `Authorization: Bearer <token>` from AuthContext/localStorage.
   - Response Interceptor: Catches `401 Unauthorized` (clears token, triggers logout) and `422 Unprocessable Entity` (formats validation errors).
3. **State Management & Caching:** Maintain customer seat selection and cart state in `BookingCartContext` or `Zustand`. Clear temporary selections on unmount or session expiration.
4. **Responsive & Modern Styling:** Use TailwindCSS with a sleek cinema aesthetic (dark background `#0F172A`, gold `#F59E0B` or crimson `#E11D48` accents, glassmorphic cards, clear seat state colors: available, selected, occupied, VIP).
5. **No Placeholders in Production UI:** Always provide realistic fallback data, clean loaders/skeletons, and meaningful empty states.

---

## 4. Key Recipes for Agents

### 4.1 Creating a New API Endpoint in Laravel
1. **Define Migration & Model:** Generate migration and model if adding a new table (`php artisan make:model CinemaHall -m`).
2. **Create Form Request:** Generate request class (`php artisan make:request StoreShowtimeRequest`).
3. **Create Service / Business Logic:** Place complex validation or calculation in `app/Services/`.
4. **Create Controller:** Implement controller in `app/Http/Controllers/Api/v1/` and inject services.
5. **Register Route:** Add versioned route in `routes/api.php` under appropriate Sanctum / Role middleware.
6. **Write Feature Test:** Add test in `tests/Feature/` asserting JSON structure and status code.

### 4.2 Creating a New UI Feature in React
1. **Define Service Function:** Add API call method in `src/services/` (e.g., `bookingService.holdSeats(seatIds)`).
2. **Create/Update Component:** Build functional component with React hooks, accessible markup, and Tailwind styling.
3. **Handle Loading & Errors:** Include loading skeletons, error toasts, and disabled states during network operations.
4. **Connect Routes:** Register page in `src/routes/AppRoutes.jsx` with appropriate access guard (`ProtectedRoute`).

---

## 5. Pre-Commit Verification Checklist for Agents

Before completing any task, ensure:
- [ ] No raw PHP scripts, legacy `$_SESSION` calls, or monolithic view scripts were created.
- [ ] Backend routes are registered in `routes/api.php` and adhere to `/api/v1/` REST standard.
- [ ] Database mutations use transactions and soft deletes where applicable.
- [ ] Frontend API calls use the centralized Axios service with Bearer token injection.
- [ ] React components handle loading, error, and empty states gracefully.
- [ ] All updated documentation in `docs/` is coherent and aligned with the Laravel + React architecture.
