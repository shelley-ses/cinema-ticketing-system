# Product Requirements Document (PRD)
## BJRS Cinema Ticketing System

**Document Version:** 1.0.0  
**Target Tech Stack:** Laravel 11.x (PHP 8.2+) / React 18+ (Vite, Modern SPA) / Supabase (PostgreSQL) / Laravel Sanctum / SMTP Mailer  

---

## 1. Executive Summary
The **BJRS Cinema Ticketing System** is an enterprise-grade multi-branch cinema management and e-ticketing platform built with a modern decoupled architecture: a **Laravel RESTful API backend** and a **React Single-Page Application (SPA) frontend**.

The platform centralizes operations across diverse geographic cinema branches, automates movie scheduling and conflict detection, manages interactive regular and VIP seat grids, handles atomic seat reservation and simulated checkout, and enforces granular Role-Based Access Control (Super Admin, Branch Admin, and Registered/Guest Customer).

---

## 2. Problem Statement & Objectives

### 2.1 Problem Statement
- **Fragmented Operations:** Cinema chains struggle with decentralized branch scheduling, leading to time overlaps and manual administrative overhead.
- **Seat Race Conditions:** High-demand movie releases result in concurrent seat collisions and double bookings without atomic locking.
- **Legacy Technical Debt:** Monolithic architectures with tight coupling between UI and server render updates slow, brittle, and difficult to maintain.
- **Insecure Staff Provisioning:** Manual generation and transmission of branch manager credentials creates security risks.

### 2.2 Objectives & Key Results (OKRs)
- **Decoupled Modern Architecture:** 100% separation between Laravel API backend and React SPA frontend.
- **Zero Double-Booking Guarantee:** 0.00% seat collision rate through transactional DB locks and temporary reservation holds during checkout.
- **Sub-Second Response Times:** API response latency $< 150\text{ ms}$ for standard queries; interactive, zero-reload seat selection on the React frontend.
- **Automated Communication:** 100% automated async email delivery for staff onboarding credentials and customer e-tickets with QR codes.
- **Rapid Branch Onboarding:** Super Admins can provision a new cinema branch and onboard its manager in $< 30\text{ seconds}$.

---

## 3. Target User Personas

| Role | Persona | Key Needs & Goals | Pain Points Addressed |
| :--- | :--- | :--- | :--- |
| **Super Admin** | *Marcus (Operations Director)* | Multi-branch provisioning, global film catalog oversight, staff role assignment, system-wide analytics. | Fragmented branch metrics, manual credential distribution. |
| **Branch Admin** | *Alex (Branch Manager)* | Screening room/hall configuration (Regular vs. VIP), conflict-free showtime scheduling, real-time branch occupancy & sales. | Scheduling overlaps, manual seat-by-seat grid setup, unauthorized cross-branch edits. |
| **End User / Customer** | *Sarah (Moviegoer)* | Rich movie browsing, trailer previews, branch & date filtering, interactive visual seat picking, instant digital QR e-tickets. | Cluttered legacy pages, slow checkout, seat hijacking during selection, lost physical tickets. |

---

## 4. System Roles & Access Control Matrix

| Feature / Domain | Super Admin | Branch Admin | Customer (Auth) | Guest (Unauth) |
| :--- | :---: | :---: | :---: | :---: |
| **Branch Management (CRUD)** | Full CRUD | View Only (Assigned) | View Active List | View Active List |
| **Staff Provisioning & Credentials** | Full CRUD | No Access | No Access | No Access |
| **Global Movie Catalog** | Full CRUD | View Only | View Only | View Only |
| **Room & Seat Map Config** | Full CRUD | CRUD (Assigned Branch) | No Access | No Access |
| **Showtime & Scheduling Engine** | Full CRUD | CRUD (Assigned Branch) | View Available | View Available |
| **Revenue & Sales Analytics** | All Branches | Assigned Branch Only | No Access | No Access |
| **Seat Booking & Checkout** | No Access | POS / Over-the-counter | Full Access | Redirect to Auth |
| **Booking History & Digital Ticket** | No Access | Validate by Ref/QR | Own Bookings Only | No Access |
| **Profile & Security Settings** | Self | Self | Self | No Access |

---

## 5. Functional Requirements

### 5.1 Super Administrator Module (Laravel API + React Portal)
- **FR-SA-01: Multi-Branch Management**
  - Create, view, update, and soft-delete cinema branches (`name`, `location`, `phone`, `email`, `operating_hours`, `is_active`).
- **FR-SA-02: Branch Staff Provisioning & Automated Dispatch**
  - Provision branch managers linked to specific branch IDs.
  - Automatically generate secure random passwords, hash via Bcrypt, and dispatch credentials via Laravel Mail queue.
- **FR-SA-03: Global Analytics & Dashboard**
  - High-level KPIs: Total active branches, operational staff count, platform-wide revenue, and top-grossing films.
- **FR-SA-04: Film Catalog Management**
  - Manage movies, release dates, durations, age ratings, poster URLs, trailer embed URLs, synopsis, genre tags, and screening status (`Now Showing`, `Coming Soon`, `Archived`).

### 5.2 Branch Administrator Module (Laravel API + React Portal)
- **FR-BA-01: Cinema Hall / Room Configuration**
  - Create screening rooms (e.g., "Hall 1 - Regular", "Hall 2 - VIP Recliner").
  - Dynamically generate row-column seat matrices (e.g., Rows A–F, Numbers 1–10) with custom seat types (`standard`, `vip_recliner`, `couple`).
- **FR-BA-02: Conflict-Free Showtime Scheduler**
  - Assign movies to rooms with date, start time, end time, and tiered seat pricing (`price_regular`, `price_vip`).
  - Automated validation preventing overlapping showtimes in the same room (including a customizable cleanup/turnaround buffer).
- **FR-BA-03: Movie Screening Status Toggle**
  - Quick-toggle status for movies screening at their branch.
- **FR-BA-04: Branch Performance & Real-Time Monitoring**
  - Live occupancy meters, daily ticket revenue, and showtime capacity metrics for the assigned branch.

### 5.3 End User / Customer Module (Laravel API + React Public App)
- **FR-EU-01: Movie Discovery & Rich Exploration**
  - Browse "Now Showing" and "Coming Soon" movies with instant search, genre filtering, and modal video trailer previews.
- **FR-EU-02: Branch & Showtime Selection**
  - Filter screenings by branch location and calendar date; view segmented Regular and VIP showtimes with live pricing.
- **FR-EU-03: Reactive Interactive Seat Selector**
  - Dynamic visual seat map rendering hall orientation, screen curve, aisle spaces, available seats, user selections, and locked/occupied seats.
  - Live price calculation reflecting selected seat classes.
- **FR-EU-04: Concurrency-Safe Checkout & Seat Reservation**
  - Temporary atomic hold on selected seats during checkout to prevent duplicate claims.
  - Simulated payment gateway processing with card/e-wallet validation.
- **FR-EU-05: Digital E-Ticket & Email Delivery**
  - Instant post-purchase e-ticket displaying unique Booking Reference, QR code, seat labels, hall name, showtime, and branch address.
  - Automated HTML confirmation email containing the digital receipt and e-ticket payload.
- **FR-EU-06: Customer Account & Booking History**
  - Authentication (Register/Login via Laravel Sanctum), profile updates, and historical ticket archive with reprint/re-view capability.

---

## 6. Non-Functional Requirements (NFRs)

### 6.1 Performance & Scalability
- **API Latency:** $P_{95}$ response time $< 150\text{ ms}$ for standard endpoints.
- **Frontend Responsiveness:** Fast initial load via Vite bundle splitting; instant state transitions without full-page reloads.
- **Concurrency & Locking:** Pessimistic/transactional locking on seat records during booking execution to ensure 0 double bookings under peak traffic.

### 6.2 Security & Compliance
- **Authentication:** Token/cookie-based session authentication via Laravel Sanctum with token expiration and refresh capabilities.
- **Password Protection:** Industry-standard Bcrypt/Argon2id password hashing.
- **Authorization Middleware:** Role-based middleware (`auth:sanctum`, `role:super_admin`, `role:branch_admin`, `branch.scope`) ensuring strict data isolation between branches.
- **Input Validation & Sanitization:** 100% structured validation via Laravel Form Requests, preventing SQL injection, mass assignment, and XSS.
- **CORS Protection:** Configured cross-origin resource sharing for trusted frontend domains.

### 6.3 Reliability & Maintainability
- **Data Integrity:** Soft deletes (`deleted_at`) on branches, rooms, showtimes, and users to preserve historical financial records.
- **Asynchronous Processing:** Email dispatches and heavy notifications processed via Laravel background Queues to keep HTTP requests fast.
- **Clean Architecture:** Strict adherence to Laravel MVC/Service Layer patterns and React component modularity.

---

## 7. Technology Stack Summary

| Domain | Technology | Specification |
| :--- | :--- | :--- |
| **Backend Framework** | Laravel 11.x | PHP 8.2+, Composer, Eloquent ORM, Form Requests, Resources |
| **Frontend Framework** | React 18+ | Vite, Modern Hooks, React Router DOM v6+, Axios |
| **Styling & Icons** | Modern CSS / TailwindCSS | Glassmorphism cinema theme, Lucide Icons |
| **Database** | Supabase (PostgreSQL) | Managed PostgreSQL with connection pooling, migrations, and ACID transactions |
| **Authentication** | Laravel Sanctum | Secure SPA Cookies / Bearer API Tokens |
| **Queue & Cache** | Redis / Database Queue | Async email sending, seat lock management |
| **Mailer** | Laravel Mail | SMTP/TLS with Blade-rendered responsive HTML email templates |

---

## 8. Success Metrics & Key Performance Indicators (KPIs)
- **Seat Conflict Rate:** 0.00% double-booked seats.
- **Booking Funnel Drop-off:** $< 15\%$ drop-off between seat selection and checkout confirmation.
- **API Availability:** $\ge 99.9\%$ uptime.
- **Staff Onboarding Speed:** $< 30\text{ seconds}$ from form submission to credentials received in mailbox.
