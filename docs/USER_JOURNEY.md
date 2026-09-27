# User Journey Document
## BJRS Cinema Ticketing System

**Document Version:** 1.0.0  
**Status:** Approved  
**Target Architecture:** Laravel 11 API Backend + React 18 SPA Frontend  
**Focus:** User Experience, Emotional Curve, Touchpoints, and Opportunities across Modern SPA Workflows  

---

## 1. Persona 1: Sarah — The Weekend Moviegoer

- **Demographics:** 28-year-old marketing professional, enjoys weekend cinema outings with friends and family.
- **Goal:** Find an evening screening of a new blockbuster at the nearby cinema branch, reserve two VIP recliner seats together without losing them mid-selection, and get instant digital tickets on her smartphone.
- **Primary Channels:** React SPA on Mobile Browser / Desktop Web.

### 1.1 Customer Journey Map

```
Stage           Discovery       Selection         Seat Choice       Payment          Fulfillment       Attendance
─────────────────────────────────────────────────────────────────────────────────────────────────────────────
Actions         Browses films   Filters by        Picks VIP Hall    Enters card/     Receives email &  Presents QR code
                on React home;  Branch & Date;    & picks 2 couple  wallet info;     instantly views   at cinema hall
                watches trailer checks times      recliners live    clicks Pay       e-ticket on SPA   entrance scanner
─────────────────────────────────────────────────────────────────────────────────────────────────────────────
Touchpoints     React Home      Movie Details     Interactive       Simulated        E-Ticket Modal &  Mobile Screen /
                Hero Banner     & Schedule Tab    Canvas Seat Map   Checkout View    Ticket History    Gate Scanner
─────────────────────────────────────────────────────────────────────────────────────────────────────────────
Emotion         😊 Excited       🤔 Evaluating     🤩 Delighted      😌 Reassured      🎉 Relieved       🍿 Enjoying
                "Looks great!"  "7:30 PM fits"   "Smooth map!"     "5-min lock on"  "Ticket saved!"   "Zero queues"
─────────────────────────────────────────────────────────────────────────────────────────────────────────────
Pain Points     Laggy lists &   Showtime not      Seats get taken   Hidden booking   Slow email        Lost ticket /
                page reloads    available         mid-selection     fees             delivery          dim screen
─────────────────────────────────────────────────────────────────────────────────────────────────────────────
Opportunity /   Fast Vite SPA   Instant reactive  Temporary 5-min   Transparent      Instant on-screen High-contrast
System Solution instant filter  tab filtering     Redis seat hold   price breakdown  QR + async mail   QR code screen
```

### Detailed Stage Breakdown:
1. **Discovery (Awareness):** Sarah lands on the React SPA homepage. Movies are dynamically grouped into "Now Showing" and "Coming Soon". Clicking a movie card triggers a smooth modal with HD trailer playback and synopsis.
2. **Showtime Selection (Consideration):** Switching branch locations or dates updates showtime chips instantly via Axios without refreshing the entire page. Regular and VIP auditoriums are clearly labeled with badge indicators.
3. **Interactive Seat Map (Decision):** Sarah enters the interactive seat selector component. The UI renders an SVG/CSS grid showing screen orientation, aisle spacing, seat tiers (Standard vs. VIP Recliner), and real-time availability. Clicking two seats places a 5-minute reservation hold via `/api/v1/bookings/hold-seats`.
4. **Checkout & Payment (Conversion):** The checkout component displays a 5-minute countdown timer, clear price summary, and validated card/e-wallet form.
5. **E-Ticket & Entry (Delight):** Upon transaction confirmation, Sarah is immediately routed to her digital e-ticket with a high-contrast QR code and booking reference number. She also receives a copy via email and can access it anytime from her authenticated customer portal.

---

## 2. Persona 2: Alex — The Cinema Branch Manager

- **Demographics:** 35-year-old Cinema Operations Manager at the Downtown branch.
- **Goal:** Configure a newly renovated VIP screening hall, auto-generate its 60-seat recliner layout, and schedule 4 daily showtimes for the weekend premiere without scheduling collisions.
- **Primary Channels:** React Admin Portal / Desktop Browser.

### 2.1 Branch Manager Journey Map

```
Stage           Morning Prep       Hall Setup        Scheduling        Monitoring        End of Day
─────────────────────────────────────────────────────────────────────────────────────────────────────────────
Actions         Logs in to React   Creates "Hall 3"  Assigns new film  Monitors real-    Exports daily
                Admin portal;      as VIP type;      to Hall 3 across  time occupancy    revenue and ticket
                reviews screenings sets rows/cols    4 time slots      for prime shows   sales report
─────────────────────────────────────────────────────────────────────────────────────────────────────────────
Touchpoints     Sanctum Auth &     Room Setup View   Schedule Calendar Real-Time Sales   Export CSV &
                Branch Dashboard   & Matrix Modal    & Overlap Check   Occupancy Widget  Analytics View
─────────────────────────────────────────────────────────────────────────────────────────────────────────────
Emotion         😐 Focused         😊 Productive     😌 Confident      🤩 Pleased        🏆 Satisfied
                "Let's get ready"  "Grid generated"  "No collisions"   "85% booked"      "Target hit"
─────────────────────────────────────────────────────────────────────────────────────────────────────────────
Pain Points     Session timeout    Manual seat-by-   Accidental time   Laggy dashboard   Complex manual
                while working      seat creation     overlaps          refresh           spreadsheet math
─────────────────────────────────────────────────────────────────────────────────────────────────────────────
Opportunity /   Persistent token   Automated matrix  Backend overlap   Live reactive     One-click revenue
System Solution refresh            generation API    conflict detector stat cards        breakdown API
```

### Key Milestones:
- **Instant Grid Generation:** Alex inputs 6 rows and 10 columns; Laravel's `RoomService` automatically batch-inserts 60 seat records with proper row letters (`A-F`) and numbers (`1-10`) in a single database transaction.
- **Conflict Prevention Engine:** When Alex schedules a 7:00 PM showtime, the backend cross-references movie runtime (135 minutes) plus a 15-minute turnaround buffer. Attempting to schedule a conflicting show at 8:30 PM immediately triggers a clear validation toast without corrupting existing schedules.

---

## 3. Persona 3: Marcus — The Super Administrator (Operations Director)

- **Demographics:** 44-year-old Corporate Director overseeing 12 cinema branches across the region.
- **Goal:** Onboard a new branch location, provision a new branch manager account, and ensure automated, encrypted credential delivery via email.
- **Primary Channels:** Corporate Laptop / Super Admin React Console.

### 3.1 Super Admin Journey Map

```
Stage           Strategic Review   Branch Creation   Staff Provisioning  Activation        System Health
─────────────────────────────────────────────────────────────────────────────────────────────────────────────
Actions         Reviews regional   Fills branch      Enters manager      Verifies email    Inspects global
                revenue & ticket   name, address,    details; triggers   dispatch status;  movie catalog
                volumes            phone & hours     auto-password gen   branch goes live  & preview assets
─────────────────────────────────────────────────────────────────────────────────────────────────────────────
Touchpoints     Executive KPI      Branch Modal &    BranchController    Laravel Mailer    Movie Catalog
                Dashboard Cards    Form Validation   & User Service      Queue Worker      Management Grid
─────────────────────────────────────────────────────────────────────────────────────────────────────────────
Emotion         🤔 Analytical      ⚡ Efficient       🛡️ Secure          🎉 Reassured      😎 In Control
                "North Mall ready" "Form is clean"   "Auto-encrypted"   "Credentials sent" "Full visibility"
─────────────────────────────────────────────────────────────────────────────────────────────────────────────
Pain Points     Scattered data     Duplicate branch  Manual passwords   Email bounce      Inconsistent film
                across sheets      names             leaked in chat     unnoticed         metadata
─────────────────────────────────────────────────────────────────────────────────────────────────────────────
Opportunity /   Unified executive  Unique constraint Automatic secure   Async queue       Centralized film
System Solution analytics API      validation rules  mailer delivery    status telemetry  asset repository
```

### Key Milestones:
- **One-Step Unified Provisioning:** Marcus enters branch details and manager email in a single React form. The Laravel backend handles database insertion, Bcrypt password generation, and triggers an asynchronous `StaffWelcomeMail` via the background queue.
- **Audit & Soft-Deletion Safety:** Deleting a branch or retiring staff uses soft-deletes (`deleted_at`), ensuring historical ticket sales, revenue metrics, and booking audit trails remain intact.
