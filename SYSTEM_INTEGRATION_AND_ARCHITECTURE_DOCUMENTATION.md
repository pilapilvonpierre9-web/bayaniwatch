# BayaniWatch: System Integration & Architecture Final Diagrams

**System Title:** BayaniWatch: Barangay Danhug Digital Incident Blotter, Emergency SOS Telemetry & GIS Command System  
**Course:** System Integration and Architecture (SIA) / Information Assurance and Security 1 (IAS1)  

---

## 1. Use Case Diagram (System Users & Functionalities)

```mermaid
flowchart LR
    subgraph Actors ["System Actors"]
        RES["Resident / Community Member"]
        TAN["Barangay Tanod / Responder"]
        ADM["Barangay Administrator"]
        EXT_AI["Google Gemini AI API"]
        EXT_CF["Cloudflare Edge CDN"]
    end

    subgraph Boundaries ["BayaniWatch System Boundary"]
        UC1(["Register & Verify Account via OTP"])
        UC2(["Authenticate & Session Login"])
        UC3(["File Incident Blotter with Photo/Video"])
        UC4(["Trigger Realtime SOS Emergency Telemetry"])
        UC5(["View GIS Spatial Map & Incident Blotter"])
        UC6(["Receive Siren & Dispatch Notifications"])
        UC7(["Update Incident Status (In Progress)"])
        UC8(["Resolve Incident with Proof Media"])
        UC9(["Post Community Advisories"])
        UC10(["Manage User Accounts & Roles (RBAC)"])
        UC11(["Review Security Audit Trails"])
        UC12(["Execute AI Risk Analysis Scan"])
        UC13(["Execute Database Backup & Restore"])
        UC14(["Offline Staging & Auto-Synchronization"])
    end

    RES --> UC1
    RES --> UC2
    RES --> UC3
    RES --> UC4
    RES --> UC5
    RES --> UC14

    TAN --> UC2
    TAN --> UC5
    TAN --> UC6
    TAN --> UC7

    ADM --> UC2
    ADM --> UC5
    ADM --> UC6
    ADM --> UC7
    ADM --> UC8
    ADM --> UC9
    ADM --> UC10
    ADM --> UC11
    ADM --> UC12
    ADM --> UC13

    UC3 -.-> EXT_CF
    UC8 -.-> EXT_CF
    UC12 -.-> EXT_AI
```

---

## 2. System Flow Diagram (Swimlanes & Pools)

```mermaid
sequenceDiagram
    autonumber
    actor Resident as Resident (Mobile PWA)
    participant ClientApp as PWA Engine & LocalStorage
    participant CF_Worker as Cloudflare Edge CDN Worker
    participant Supabase as Supabase (PostgreSQL & Realtime)
    actor AdminDesk as Barangay Admin / Tanod Desk

    Note over Resident,AdminDesk: Phase 1: Incident / SOS Ingestion
    Resident->>ClientApp: Trigger SOS or Submit Incident with Photo/Video
    alt Device is Offline (No Signal)
        ClientApp->>ClientApp: Save to 'bayaniwatch_offline_queue' (Base64)
        ClientApp-->>Resident: Show 'SAVED OFFLINE' Staged Badge
    else Device is Online
        ClientApp->>Supabase: Upload Media to bucket 'incident-photos'
        Supabase-->>ClientApp: Return Supabase Storage URL
        ClientApp->>ClientApp: Rewrite to Cloudflare Worker URL
        ClientApp->>Supabase: Insert record into 'incidents' / 'sos_alerts'
        ClientApp->>Supabase: Broadcast 'sos_trigger' via WebSocket Room
    end

    Note over Resident,AdminDesk: Phase 2: Realtime Dispatch & Response
    Supabase->>AdminDesk: Push Realtime Broadcast Event
    AdminDesk->>AdminDesk: Play Siren Alarm ('alarm.mp3') & Show Red Banner
    AdminDesk->>AdminDesk: Plot Dynamic Marker on GIS Leaflet Map

    Note over Resident,AdminDesk: Phase 3: Edge Caching for Evidence Retrieval
    AdminDesk->>CF_Worker: Request Incident Evidence Media
    alt Edge Cache Hit
        CF_Worker-->>AdminDesk: Serve Media from Cloudflare Cache (0 KB Egress)
    else Edge Cache Miss
        CF_Worker->>Supabase: Fetch Origin Media File Once
        CF_Worker->>CF_Worker: Store in Edge RAM/SSD (TTL: 30 Days)
        CF_Worker-->>AdminDesk: Return Media Stream to Officer
    end

    Note over Resident,AdminDesk: Phase 4: Resolution & Audit Logging
    AdminDesk->>ClientApp: Set Status to 'Resolved' + Resolution Notes
    ClientApp->>Supabase: Update status & write to 'log_trails' table
    Supabase-->>Resident: Status Updates on Resident Blotter Feed
```

---

## 3. Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    USERS ||--o{ INCIDENTS : "submits"
    USERS ||--o{ SOS_ALERTS : "triggers"
    USERS ||--o{ LOG_TRAILS : "generates"
    USERS ||--o{ PASSWORD_RESETS : "requests"
    USERS ||--o{ ANNOUNCEMENTS : "publishes"

    USERS {
        uuid user_id PK "Primary Key"
        varchar username UK "Unique username"
        varchar email UK "Unique verified email"
        varchar password_hash "SHA-256 hashed password"
        varchar full_name "Citizen / Official name"
        varchar contact_number "Mobile phone number"
        varchar role "Resident | Tanod | Admin"
        boolean is_active "Account status flag"
        boolean is_locked "Lockout protection flag"
        integer failed_attempts "Consecutive failed login counter"
        timestamp lockout_until "Lockout expiration timestamp"
        timestamp created_at "Registration timestamp"
    }

    INCIDENTS {
        bigserial incident_id PK "Tracking ID"
        uuid user_id FK "References USERS(user_id)"
        varchar title "Incident headline"
        varchar category "Noise, Theft, Medical, Fire, etc."
        text description "Detailed description"
        varchar location_address "Barangay Danhug address"
        numeric latitude "GPS Coordinate Latitude"
        numeric longitude "GPS Coordinate Longitude"
        varchar status "Pending | In Progress | Resolved"
        varchar image_url "Cloudflare CDN Proxied Media URL"
        boolean is_anonymous "Anonymous whistleblower flag"
        text resolution_notes "Tanod / Admin response summary"
        varchar resolution_image_url "Cloudflare CDN Proof URL"
        timestamp reported_at "ISO 8601 Creation Timestamp"
    }

    SOS_ALERTS {
        bigserial sos_id PK "SOS Log Identifier"
        uuid user_id FK "References USERS(user_id)"
        varchar category "Medical, Fire, Crime, Disaster"
        text description "Emergency situation notes"
        numeric latitude "Real-time Telemetry Lat"
        numeric longitude "Real-time Telemetry Lng"
        varchar media_url "Cloudflare CDN proxied clip"
        varchar media_type "photo | video"
        varchar status "ACTIVE | ACKNOWLEDGED | RESOLVED"
        timestamp triggered_at "Timestamp of SOS activation"
    }

    LOG_TRAILS {
        bigserial log_id PK "Audit trail primary key"
        uuid user_id FK "Actor user reference"
        varchar username "Logged username at event"
        varchar module "AUTH | INCIDENT | SOS | SYSTEM | USER_MGMT"
        text action "Audit event description"
        varchar ip_address "Client IP address"
        varchar result "SUCCESS | FAILED | WARNING | CRITICAL"
        timestamp logged_at "Tamper-proof audit timestamp"
    }

    ANNOUNCEMENTS {
        bigserial announcement_id PK "Advisory identifier"
        uuid created_by FK "References USERS(user_id)"
        varchar title "Advisory headline"
        text content "Barangay memorandum body"
        varchar category "SECURITY | HEALTH | WEATHER | GENERAL"
        boolean is_urgent "Priority flashing badge flag"
        timestamp created_at "Publication timestamp"
    }

    PASSWORD_RESETS {
        bigserial reset_id PK "Reset token identifier"
        uuid user_id FK "References USERS(user_id)"
        varchar otp_code "6-Digit One-Time Password"
        boolean is_used "One-time usage flag"
        timestamp expires_at "Expiration limit (15 mins)"
        timestamp created_at "Issuance timestamp"
    }
```

---

## 4. System Flowchart

```mermaid
flowchart TD
    START([User Launches BayaniWatch PWA]) --> AUTH_CHECK{Authenticated Session Exists?}
    
    %% Authentication Branch
    AUTH_CHECK -- No --> LOGIN_SCREEN[/Enter Username & Password/]
    LOGIN_SCREEN --> VAL_CREDS{Validate Credentials against SHA-256 Hash?}
    
    VAL_CREDS -- Invalid --> INC_FAIL[Increment Failed Attempts]
    INC_FAIL --> ATTEMPTS_CHECK{Failed Attempts >= 3?}
    ATTEMPTS_CHECK -- Yes --> LOCK_ACC[Lock Account with Progressive Timer & Trigger CAPTCHA]
    LOCK_ACC --> LOG_FAIL[Write FAIL event to Security Log Trail]
    LOG_FAIL --> LOGIN_SCREEN
    ATTEMPTS_CHECK -- No --> DISP_REMAIN[/Show Remaining Attempts/] --> LOGIN_SCREEN
    
    VAL_CREDS -- Valid --> RESET_ATTEMPTS[Reset Failed Counter to 0]
    RESET_ATTEMPTS --> ROLE_BRANCH{Evaluate User Role}
    
    ROLE_BRANCH -- Admin / Tanod --> ADMIN_DASH[Load Admin Command Center & GIS Telemetry]
    ROLE_BRANCH -- Resident --> RES_DASH[Load Resident Blotter & SOS Panel]
    AUTH_CHECK -- Yes --> ROLE_BRANCH

    %% Inactivity Timer
    RES_DASH --> INACTIVITY{15 Mins Inactivity?}
    ADMIN_DASH --> INACTIVITY
    INACTIVITY -- Yes --> AUTO_LOGOUT[Show Session Warning -> Terminate Session -> Log Out]
    AUTO_LOGOUT --> START

    %% Incident Submission Branch
    RES_DASH --> USER_ACTION{Select Action}
    USER_ACTION -- File Incident / Trigger SOS --> CAPTURE_MEDIA[/Capture Photo or 5s WebM Video/]
    CAPTURE_MEDIA --> NET_CHECK{Network Online?}
    
    NET_CHECK -- Offline --> LOCAL_STAGE[Convert Media to Base64 & Enqueue to 'bayaniwatch_offline_queue']
    LOCAL_STAGE --> LOCAL_FEED[/Render in Blotter with 'Offline Staged' Badge/]
    
    NET_CHECK -- Online --> UPLOAD_MEDIA[Upload Media Blob to Supabase Storage]
    UPLOAD_MEDIA --> CDN_PROXY[Transform URL to Cloudflare Edge Worker URL]
    CDN_PROXY --> DB_INSERT[(Insert Record into PostgreSQL Database)]
    DB_INSERT --> REALTIME_BC[Broadcast SOS Event across WebSocket Channel]
    
    %% Realtime Tanod Alert
    REALTIME_BC --> SIREN[Trigger Audible Siren on Officer Terminals]
    SIREN --> PLOT_GIS[Plot Pin on GIS Leaflet Map]

    %% Officer Resolution
    PLOT_GIS --> DISPATCH[Tanod Responds to Scene]
    DISPATCH --> RESOLVE_INC[/Officer Enters Resolution Summary & Proof Photo/]
    RESOLVE_INC --> UPDATE_DB[(Update Status = 'Resolved')]
    UPDATE_DB --> AUDIT_LOG[(Append Entry to 'log_trails' Audit Table)]
    AUDIT_LOG --> END_NODE([Cycle Complete])
```

---

## 5. Network Diagram (Network Flow)

```mermaid
graph TB
    subgraph ClientZone ["Client Endpoints (Edge Access)"]
        RES_MOBILE["Resident Smartphone (PWA / Android / iOS)"]
        ADMIN_PC["Barangay Hall Admin Workstation (Chrome / Edge)"]
        TANOD_DEV["Barangay Tanod Mobile Patrol Unit"]
    end

    subgraph PerimeterNetwork ["Perimeter & Edge Acceleration Layer"]
        CF_DNS["Cloudflare Authoritative DNS & DDoS Shield"]
        CF_WORKER["Cloudflare Edge Workers (Proxy & Caching Engine)<br/>Domain: supabase-storage-cache.pilapilvonpierre9.workers.dev"]
        CF_CACHE[("Cloudflare Global Cache Store (RAM / NVMe SSD)")]
    end

    subgraph CloudInfrastructure ["Cloud Backend Infrastructure (Supabase AWS Cluster)"]
        API_GATEWAY["Kong API Gateway (Port 443 HTTPS / WSS)"]
        AUTH_SVC["Supabase GoTrue Authentication Engine"]
        REST_ENGINE["PostgREST RESTful API Engine"]
        REALTIME_SVC["Supabase Realtime Engine (WebSockets / Elixir)"]
        PG_DB[("PostgreSQL 15 Relational Database Cluster<br/>(RLS & AES-256 Storage Encryption)")]
        STORAGE_BUCKET[("Supabase S3 Object Storage<br/>Bucket: 'incident-photos'")]
    end

    subgraph ExternalAPIs ["Cognitive & Location Services Tier"]
        GEMINI_AI["Google Gemini 1.5 Flash AI Cloud API"]
        OSM_TILES["OpenStreetMap / Leaflet Tile CDN Servers"]
    end

    %% Client Traffic
    RES_MOBILE -- "HTTPS / WSS (TLS 1.3)" --> CF_DNS
    ADMIN_PC -- "HTTPS / WSS (TLS 1.3)" --> CF_DNS
    TANOD_DEV -- "HTTPS / WSS (TLS 1.3)" --> CF_DNS

    %% Edge Routing
    CF_DNS --> CF_WORKER
    CF_WORKER <--> CF_CACHE
    CF_WORKER -- "Cache Miss: Forward to Origin" --> STORAGE_BUCKET

    %% Database & Realtime Traffic
    CF_DNS -- "REST Queries & Auth Requests" --> API_GATEWAY
    API_GATEWAY --> AUTH_SVC
    API_GATEWAY --> REST_ENGINE
    API_GATEWAY --> REALTIME_SVC
    REST_ENGINE <--> PG_DB
    AUTH_SVC <--> PG_DB

    %% Realtime Push
    REALTIME_SVC -. "WSS Bi-directional Push" .-> ADMIN_PC
    REALTIME_SVC -. "WSS Bi-directional Push" .-> TANOD_DEV

    %% External Connections
    ADMIN_PC -- "AI Risk Analysis Prompts" --> GEMINI_AI
    ADMIN_PC & RES_MOBILE -- "GIS Map Tiles Fetch" --> OSM_TILES

    classDef client fill:#e0f2fe,stroke:#0284c7,stroke-width:2px;
    classDef edge fill:#fef3c7,stroke:#d97706,stroke-width:2px;
    classDef cloud fill:#dcfce7,stroke:#16a34a,stroke-width:2px;
    classDef ext fill:#f3e8ff,stroke:#9333ea,stroke-width:2px;

    class RES_MOBILE,ADMIN_PC,TANOD_DEV client;
    class CF_DNS,CF_WORKER,CF_CACHE edge;
    class API_GATEWAY,AUTH_SVC,REST_ENGINE,REALTIME_SVC,PG_DB,STORAGE_BUCKET cloud;
    class GEMINI_AI,OSM_TILES ext;
```
