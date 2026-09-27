# User Flow Document
## BJRS Cinema Ticketing System

**Document Version:** 1.0.0  
**Status:** Approved  
**Target Architecture:** Laravel 11 REST API + React 18 SPA  
**Coverage:** End User / Customer Booking, Super Admin Provisioning, Branch Admin Operations, and Sanctum Authentication  

---

## 1. End User / Customer Booking Flow

This flow illustrates the complete reactive journey of a moviegoer on the React frontend interacting with the Laravel backend.

```mermaid
flowchart TD
    Start([User visits React SPA Homepage]) --> Browse[Browse Movies: Now Showing / Coming Soon]
    Browse --> SelectMovie[Click Movie Card]
    SelectMovie --> MovieDetails[View Details, Modal Trailer, Synopsis & Cast]
    MovieDetails --> PickBranchDate[Select Cinema Branch & Date Filter]
    PickBranchDate --> ViewShowtimes[Fetch & Display Showtimes by Room Type: Regular vs VIP]
    ViewShowtimes --> ChooseShowtime[Select Showtime Slot]
    
    ChooseShowtime --> CheckAuth{Is Customer Authenticated?}
    CheckAuth -- No --> AuthModal[Render React Auth Modal / Login / Register]
    AuthModal --> CheckAuth
    CheckAuth -- Yes --> FetchSeatMap[Axios: GET /api/v1/showtimes/:id/seat-map]
    
    FetchSeatMap --> RenderSeatGrid[React Component: Render Dynamic Interactive Seat Grid]
    RenderSeatGrid --> UserSelectSeats[User clicks available seats]
    UserSelectSeats --> UpdateCart[Update Reactive Cart Summary & Price Breakdown]
    
    UpdateCart --> ClickCheckout[Click 'Proceed to Checkout']
    ClickCheckout --> HoldSeatsAPI[Axios: POST /api/v1/bookings/hold-seats]
    
    HoldSeatsAPI --> HoldCheck{Lock Successful?}
    HoldCheck -- No: Seat Taken --> ShowConflictToast[Show Conflict Toast & Refresh Seat Map]
    ShowConflictToast --> RenderSeatGrid
    
    HoldCheck -- Yes: 5 Min Lock --> RenderCheckoutPage[Navigate to Checkout View with 5-Min Timer]
    RenderCheckoutPage --> SubmitPayment[Enter Payment Details & Confirm Booking]
    
    SubmitPayment --> CheckoutAPI[Axios: POST /api/v1/bookings/checkout]
    CheckoutAPI --> TxCheck{Payment & Transaction Valid?}
    
    TxCheck -- Failed --> ShowPaymentError[Show Payment Error Notification]
    ShowPaymentError --> RenderCheckoutPage
    
    TxCheck -- Success --> ShowETicket[Render E-Ticket View with QR Code & Booking Ref]
    ShowETicket --> AsyncEmail[Laravel Queue Dispatches Confirmation & Ticket Email]
    AsyncEmail --> End([User Saves E-Ticket / Attends Screening])
```

---

## 2. Super Administrator Provisioning Flow

This flow maps how the Super Administrator manages branches, provisions branch managers, and oversees global cinema operations.

```mermaid
flowchart TD
    SALogin([Super Admin Login on React Portal]) --> SAAuthAPI[POST /api/v1/auth/admin/login]
    SAAuthAPI --> SAAuthCheck{Valid Super Admin?}
    SAAuthCheck -- No --> SAError[Display Invalid Credentials Alert]
    SAError --> SALogin
    SAAuthCheck -- Yes --> SetSanctumToken[Store Sanctum Token & Redirect]
    SetSanctumToken --> SADashboard[Super Admin Executive Dashboard]
    
    SADashboard --> SAMenu{Select Admin Action}
    
    SAMenu --> BranchMgmt[Branch Management]
    SAMenu --> StaffMgmt[Staff Management]
    SAMenu --> CatalogMgmt[Movie Catalog Management]
    SAMenu --> AnalyticsMgmt[Global Revenue & Performance Reports]
    
    %% Branch Sub-flow
    BranchMgmt --> FetchBranches[GET /api/v1/admin/super/branches]
    FetchBranches --> ViewBranchesTable[Render Branches Table with Staff Badges]
    ViewBranchesTable --> BranchAction{Action?}
    
    BranchAction -- Add Branch with Manager --> CreateBranchModal[Open Create Branch Modal]
    CreateBranchModal --> SubmitBranch[POST /api/v1/admin/super/branches]
    SubmitBranch --> LaravelBackendBranch[Laravel Service: Create Branch + Generate Manager Credentials]
    LaravelBackendBranch --> QueueEmail[Queue StaffWelcomeMail with Login Credentials]
    QueueEmail --> RefreshBranches[Re-fetch Branches List & Show Success Toast]
    RefreshBranches --> ViewBranchesTable
    
    BranchAction -- Edit Branch --> EditBranchModal[Open Edit Modal]
    EditBranchModal --> PutBranch[PUT /api/v1/admin/super/branches/:id]
    PutBranch --> ViewBranchesTable
    
    BranchAction -- Delete Branch --> DeleteBranch[DELETE /api/v1/admin/super/branches/:id]
    DeleteBranch --> ViewBranchesTable
```

---

## 3. Branch Administrator Operational Flow

This flow illustrates daily management tasks performed by a Branch Manager (screening hall setup, conflict-free scheduling, and monitoring).

```mermaid
flowchart TD
    BALogin([Branch Admin Login on React Portal]) --> BAAuthAPI[POST /api/v1/auth/admin/login]
    BAAuthAPI --> BAAuthCheck{Valid Branch Admin?}
    BAAuthCheck -- No --> BAError[Show Error Alert]
    BAAuthCheck -- Yes --> BADashboard[Branch Dashboard: Active Screenings & Today's Sales]
    
    BADashboard --> BAMenu{Select Operation}
    
    %% Room Management Sub-flow
    BAMenu --> RoomMgmt[Hall / Room Management]
    RoomMgmt --> FetchRooms[GET /api/v1/admin/branch/rooms]
    FetchRooms --> ViewRoomsGrid[Display Hall Cards with Capacity]
    ViewRoomsGrid --> AddRoomAction[Click 'Add Screening Hall']
    AddRoomAction --> RoomModal[Specify Hall Name, Room Type Regular/VIP, Rows & Columns]
    RoomModal --> SubmitRoom[POST /api/v1/admin/branch/rooms]
    SubmitRoom --> AutoGenSeats[Laravel Service: Creates Room & Batch-Generates Seat Matrix]
    AutoGenSeats --> ViewRoomsGrid
    
    %% Showtime Management Sub-flow
    BAMenu --> ScheduleMgmt[Showtime Scheduling]
    ScheduleMgmt --> FetchSchedule[GET /api/v1/admin/branch/showtimes]
    FetchSchedule --> ViewCalendar[Render Showtime Calendar & Timeline]
    ViewCalendar --> AddShowtimeAction[Click 'Add Showtime']
    AddShowtimeAction --> ShowtimeForm[Select Movie, Room, Start Time, Regular & VIP Prices]
    ShowtimeForm --> SubmitShowtime[POST /api/v1/admin/branch/showtimes]
    SubmitShowtime --> ConflictCheck{Laravel Validator: Room or Time Overlap?}
    ConflictCheck -- Overlap Detected --> ConflictToast[Show Overlap Conflict Warning with Overlapping Film]
    ConflictToast --> ShowtimeForm
    ConflictCheck -- No Conflict --> SaveShowtime[Insert Showtime & Refresh Schedule]
    SaveShowtime --> ViewCalendar
    
    %% Status Toggle Sub-flow
    BAMenu --> StatusToggle[Movie Screening Status Toggle]
    StatusToggle --> ToggleAction[Switch Now Showing / Coming Soon]
    ToggleAction --> SaveStatus[PUT /api/v1/admin/branch/movies/:id/status]
    SaveStatus --> ViewCalendar
```

---

## 4. Authentication, Token Management & Role Guard Flow

```mermaid
sequenceDiagram
    autonumber
    actor User as User / Admin
    participant Client as React SPA (Axios Interceptors)
    participant API as Laravel 11 API (Sanctum)
    participant Queue as Laravel Background Queue (Mail)
    participant DB as Relational Database

    User->>Client: Enters Login Credentials
    Client->>API: POST /api/v1/auth/login {email, password}
    API->>DB: Query User/Admin record & verify Bcrypt Hash
    DB-->>API: User Record with Role & BranchID
    API->>API: Generate Sanctum PersonalAccessToken with Abilities
    API-->>Client: 200 OK {token, user: {id, name, email, role, branch_id}}
    Client->>Client: Store Token in secure Storage / Memory Context
    
    Note over Client,API: Subsequent Authenticated Requests
    Client->>API: GET /api/v1/admin/branch/rooms (Header: Authorization Bearer <token>)
    API->>API: Sanctum Middleware validates token
    API->>API: RoleMiddleware verifies role === 'branch_admin'
    API->>API: BranchScope restricts query to auth()->user()->branch_id
    API->>DB: SELECT * FROM rooms WHERE branch_id = :id AND deleted_at IS NULL
    DB-->>API: Rooms dataset
    API-->>Client: 200 OK {success: true, data: [...]}
    Client->>User: Renders dynamic UI view
```
