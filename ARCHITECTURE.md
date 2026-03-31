# BimaKavach Platform Architecture

## Table of Contents
- [1. System Overview](#1-system-overview)
- [2. Technical Architecture](#2-technical-architecture)
  - [2.1 High-Level System Diagram](#21-high-level-system-diagram)
  - [2.2 Service Details](#22-service-details)
  - [2.3 Inter-Service Communication](#23-inter-service-communication)
  - [2.4 Database Architecture](#24-database-architecture)
  - [2.5 External Integrations](#25-external-integrations)
  - [2.6 LeadSquared Integration](#26-leadsquared-integration)
  - [2.7 Sibro Integration](#27-sibro-integration)
  - [2.8 Infrastructure & Deployment](#28-infrastructure--deployment)
- [3. Usability Architecture](#3-usability-architecture)
  - [3.1 User Roles & Access](#31-user-roles--access)
  - [3.2 Customer Journey (BK Web)](#32-customer-journey-bk-web)
  - [3.3 Admin Journey (bi-admin)](#33-admin-journey-bi-admin)
  - [3.4 POS Agent Journey](#34-pos-agent-journey)
  - [3.5 Feature-to-Service Mapping](#35-feature-to-service-mapping)

---

## 1. System Overview

BimaKavach is a **B2B insurance aggregation platform** that enables businesses to compare, purchase, and manage commercial insurance policies from 25+ insurers across 20+ product categories. The platform consists of three user-facing applications backed by a microservices architecture.

```mermaid
graph TB
    subgraph "User-Facing Applications"
        WEB["BK Web V2.0<br/><i>Customer Portal</i><br/>Next.js 14 | Port 3000"]
        ADMIN["bi-admin V2<br/><i>Internal Admin Panel</i><br/>Next.js 13 | Port 3000"]
        subgraph "POS Platform"
            POS_APP["POS Mobile App<br/><i>Agent App</i><br/>React Native + Expo"]
            POS_ADMIN["POS Admin<br/><i>POS Admin Panel</i><br/>Next.js 16 | Port 3012"]
            POS_REPORT["POS Reporting<br/><i>Analytics Dashboard</i><br/>React | CloudFront"]
        end
    end

    subgraph "Backend Services"
        GW["API Gateway<br/>NestJS | Port 3005"]
        POS_API["POS Backend<br/>NestJS | Port 3011"]
        COMMON["Common Service<br/>NestJS | Port 3006"]
        QUESTION["Question Service<br/>NestJS | Port 3007"]
        EMAIL["Email Service<br/>NestJS | Port 3008"]
        PROPOSAL["Proposal Form Service<br/>NestJS | Port 3009"]
    end

    WEB -->|HTTPS + HMAC| GW
    ADMIN -->|HTTPS + JWT| GW
    POS_APP -->|HTTPS + JWT| POS_API
    POS_ADMIN -->|NextAuth + HMAC| POS_API
    POS_REPORT -->|Google Sheets API| SHEETS[(Google Sheets)]

    GW -->|TCP| COMMON
    GW -->|TCP| QUESTION
    GW -->|TCP| EMAIL
    GW -->|TCP| PROPOSAL

    POS_API -->|TCP| COMMON
    POS_API -->|TCP| QUESTION

    QUESTION -->|TCP| COMMON
    QUESTION -->|TCP| EMAIL
    QUESTION -->|TCP| PROPOSAL
    PROPOSAL -->|TCP| COMMON
    PROPOSAL -->|TCP| EMAIL
    PROPOSAL -->|TCP| QUESTION
    EMAIL -->|TCP| COMMON
```

---

## 2. Technical Architecture

### 2.1 High-Level System Diagram

```mermaid
graph LR
    subgraph "Clients"
        C1["Browser<br/>(BK Web)"]
        C2["Browser<br/>(Admin)"]
        C3["Mobile<br/>(POS App)"]
        C4["Browser<br/>(POS Admin)"]
    end

    subgraph "API Layer"
        GW["API Gateway :3005"]
        POS["POS API :3011"]
    end

    subgraph "Microservices (TCP)"
        MS_COMMON["Common :3006"]
        MS_QUESTION["Question :3007"]
        MS_EMAIL["Email :3008"]
        MS_PROPOSAL["Proposal Form :3009"]
    end

    subgraph "Data Stores"
        PG[(PostgreSQL)]
        S3[(AWS S3)]
        GCS[(Google Cloud Storage)]
    end

    subgraph "External Services"
        SG[SendGrid]
        ANVIL[Anvil PDF]
        LSQ[LeadSquared CRM]
        SIBRO[Sibro Broker]
        PAY[Payment Gateways]
        MSG91[MSG91 OTP]
        PROBE[Probe CIN API]
        CHOLA[Chola Insurance API]
        FIREBASE[Firebase FCM]
        WHATSAPP[AiSensy WhatsApp]
        GDOC[Google Document AI]
    end

    C1 --> GW
    C2 --> GW
    C3 --> POS
    C4 --> POS

    GW --> MS_COMMON & MS_QUESTION & MS_EMAIL & MS_PROPOSAL
    POS --> MS_COMMON & MS_QUESTION

    MS_COMMON & MS_QUESTION & MS_EMAIL & MS_PROPOSAL --> PG
    GW --> PG
    POS --> PG

    GW & MS_EMAIL & MS_QUESTION & MS_PROPOSAL --> S3
    MS_QUESTION --> GCS
    MS_EMAIL --> SG
    MS_PROPOSAL --> ANVIL
    GW --> LSQ & PROBE & MSG91
    MS_QUESTION --> SIBRO & CHOLA & GDOC
    GW --> PAY
    POS --> FIREBASE & WHATSAPP & MSG91
```

### 2.2 Service Details

| Service | Repo | Tech Stack | Port | Purpose |
|---------|------|-----------|------|---------|
| **BK Web V2.0** | `BK_Web_V2.0` | Next.js 14, React 18, MUI, Framer Motion | 3000 | Customer-facing insurance portal |
| **bi-admin V2** | `bi-admin-V2` | Next.js 13, TypeScript, Redux Toolkit, MUI, Tailwind, Ant Design | 3000 | Internal admin dashboard |
| **API Gateway** | `microservice-api-gateway` | NestJS 10, TypeScript, TypeORM | 3005 | Central API entry point, auth, routing |
| **Common Service** | `microservice-common` | NestJS 10, TypeScript, TypeORM | 3006 | Master data, products, pricing, CIN |
| **Question Service** | `microservice-question` | NestJS 10, TypeScript, TypeORM, Puppeteer | 3007 | Questions, quotes, policies, claims, OCR |
| **Email Service** | `microservice-email` | NestJS 10, TypeScript, TypeORM, Puppeteer | 3008 | Email templates, sending, scheduling |
| **Proposal Form Service** | `microservice-proposal-form` | NestJS 10, TypeScript, TypeORM | 3009 | Proposal form lifecycle, PDF generation |
| **POS Backend** | `pos/microservice-pos` | NestJS 11, TypeScript, TypeORM | 3011 | POS agent platform backend |
| **POS Admin** | `pos/pos-admin` | Next.js 16, React 19, NextAuth | 3012 | POS admin dashboard |
| **POS Mobile App** | `pos/pos-app` | React Native 0.81, Expo 54 | -- | Agent mobile application |
| **POS Reporting** | `pos/pos-reporting` | React 19, Google Sheets API | -- | Analytics & reporting (CloudFront) |

### 2.3 Inter-Service Communication

All backend microservices communicate via **NestJS TCP Transport** using RPC-style message patterns.

```mermaid
graph TD
    subgraph "API Gateway :3005"
        GW_AUTH["Auth & Guards<br/>JWT, HMAC, Roles"]
        GW_ROUTES["60+ Controllers<br/>REST → TCP routing"]
        GW_CRON["Cron Jobs<br/>Daily 4:30 AM IST"]
        GW_WS["WebSocket<br/>Real-time"]
    end

    subgraph "Common :3006"
        C_PRODUCTS["Products & Insurers"]
        C_MASTER["Master Data<br/>States, Industries, Occupancy"]
        C_PRICING["Pricing Engine"]
        C_CIN["CIN/Probe Data"]
        C_MARINE["Marine Metadata"]
        C_TP["Third-Party API Config"]
    end

    subgraph "Question :3007"
        Q_L1L2["L1/L2 Questions"]
        Q_POLICY["Policy Management"]
        Q_QUOTES["Quotes Engine"]
        Q_CLAIMS["Claims Processing"]
        Q_OCR["OCR / Document AI"]
        Q_SIBRO["Sibro Integration"]
        Q_RECO["Recommendations"]
    end

    subgraph "Email :3008"
        E_TEMPLATES["Templates & Bodies"]
        E_SEND["SendGrid Sending"]
        E_SCHEDULE["Scheduler & Retry"]
        E_CASE["Case Studies"]
    end

    subgraph "Proposal Form :3009"
        P_FORMS["Form Templates"]
        P_ANSWERS["Client Answers"]
        P_PDF["Anvil PDF Generation"]
        P_STATUS["Status Tracking"]
    end

    GW_ROUTES -->|"products, insurers,<br/>pricing, master data"| C_PRODUCTS
    GW_ROUTES -->|"questions, quotes,<br/>policies, claims"| Q_L1L2
    GW_ROUTES -->|"send email,<br/>templates, logs"| E_TEMPLATES
    GW_ROUTES -->|"proposal forms,<br/>answers, PDF"| P_FORMS

    Q_POLICY -->|"get products,<br/>get config"| C_PRODUCTS
    Q_POLICY -->|"send email"| E_SEND
    Q_QUOTES -->|"proposal form link"| P_STATUS

    P_ANSWERS -->|"get products,<br/>get config"| C_PRODUCTS
    P_ANSWERS -->|"send email,<br/>fill PDF"| E_SEND
    P_STATUS -->|"update link"| Q_POLICY

    E_SCHEDULE -->|"get config"| C_PRODUCTS
```

**Key Message Patterns:**

| From | To | Patterns |
|------|----|----------|
| Gateway → Common | | `get-product-list`, `get-insurer-list`, `get-pricing`, `get-configuration-details-by-key` |
| Gateway → Question | | `get-question-list`, `get-quotes`, `create-policy`, `get-policy-list` |
| Gateway → Email | | `send-email`, `get-email-template-list`, `create-email-trigger` |
| Gateway → Proposal | | `get-proposal-form`, `update-form`, `initiate-proposal-form` |
| Question → Common | | `get-product-by-id`, `get-short-insurer-list` |
| Question → Email | | `send-email`, `schedule-email` |
| Proposal → Email | | `send-email`, `proposal-form-fill`, `create-email-logs-and-scheduler` |
| Proposal → Common | | `get-product-by-id`, `get-configuration-details-by-key` |
| POS API → Common | | `get-product-list`, `get-insurer-list` |
| POS API → Question | | `get-question-list`, L1/L2 answers |

### 2.4 Database Architecture

All services connect to **PostgreSQL** (AWS RDS in production). Each service manages its own set of tables but shares the same database instance.

```mermaid
erDiagram
    API_GATEWAY {
        table User
        table Company
        table RefreshToken
        table UserCompany
        table Partner
        table LeadSquare
        table UserOtp
        table ProbeDetails
    }

    COMMON_SERVICE {
        table Product
        table ProductGroup
        table Insurer
        table Industry
        table Pricing
        table FireOccupancy
        table StateMaster
        table Roles
        table Resources
        table CinDetails
        table MarineMetadata
    }

    QUESTION_SERVICE {
        table L1Question
        table L2Question
        table Policy
        table Quote
        table QuoteItem
        table Claim
        table ClientAnswerForm
        table PolicyRequest
        table OcrAuditLog
        table SibroInsurer
    }

    EMAIL_SERVICE {
        table EmailLog
        table Template
        table EmailBody
        table Trigger
        table EmailVariables
        table ClientEmailScheduler
    }

    PROPOSAL_SERVICE {
        table ProposalForm
        table ProposalFormQuestion
        table ProposalFormAnswer
        table PolicyProposalFormDetails
        table ProposalFormTemplate
    }

    POS_SERVICE {
        table PosUser
        table PosClient
        table PosPolicy
        table PosBrokerage
        table PosQuote
        table PosClaim
        table PosCampaign
        table PosRegion
        table PosEndorsement
        table PosNotification
    }
```

### 2.5 External Integrations

```mermaid
graph TB
    subgraph "Payment Gateways"
        FG["Future Generali"]
        CHOLA_PAY["Chola Insurance"]
        MAGMA["Magma HDI"]
        CASHFREE["Cashfree"]
    end

    subgraph "CRM & Lead Management"
        LSQ["LeadSquared<br/>Lead tracking, activities"]
        SIBRO_EXT["Sibro<br/>Broker platform sync"]
    end

    subgraph "Communication"
        SG["SendGrid<br/>Transactional email"]
        MSG91_EXT["MSG91<br/>SMS OTP"]
        FIREBASE_EXT["Firebase FCM<br/>Push notifications"]
        AISENSY["AiSensy<br/>WhatsApp messaging"]
    end

    subgraph "Document & Data"
        ANVIL_EXT["Anvil.io<br/>PDF form filling"]
        PROBE_EXT["Probe API<br/>CIN/Company verification"]
        GDOCAI["Google Document AI<br/>OCR extraction"]
        TESSERACT["Tesseract.js<br/>Text extraction"]
    end

    subgraph "Cloud Infrastructure"
        S3_EXT["AWS S3<br/>File storage"]
        CW["AWS CloudWatch<br/>Centralized logging"]
        CF["AWS CloudFront<br/>CDN"]
        RDS["AWS RDS<br/>PostgreSQL"]
        GCS_EXT["Google Cloud Storage"]
    end

    subgraph "Analytics"
        GTM["Google Tag Manager"]
        GSHEETS["Google Sheets<br/>POS Reporting"]
    end

    GW_NODE["API Gateway"] --> FG & CHOLA_PAY & MAGMA & LSQ & PROBE_EXT & MSG91_EXT
    Q_NODE["Question Service"] --> SIBRO_EXT & GDOCAI & TESSERACT & CASHFREE
    E_NODE["Email Service"] --> SG
    P_NODE["Proposal Service"] --> ANVIL_EXT
    POS_NODE["POS Backend"] --> FIREBASE_EXT & AISENSY & MSG91_EXT
    WEB_NODE["BK Web"] --> GTM
```

| Integration | Service | Purpose |
|-------------|---------|---------|
| **SendGrid** | Email Service | Transactional emails, templates, bulk sending |
| **Anvil.io** | Proposal Form Service | PDF form filling and signing |
| **LeadSquared** | API Gateway | CRM - lead creation, activity tracking, webhooks |
| **Sibro** | Question Service | Insurance broker platform sync (policies, transactions) |
| **Probe API** | API Gateway | Company CIN/registration verification |
| **Google Document AI** | Question Service | Policy document OCR extraction |
| **Chola Insurance API** | Question Service | Premium calculation, occupancy data |
| **Future Generali** | API Gateway | Payment processing |
| **Magma HDI** | API Gateway | Payment processing |
| **Cashfree** | Question Service | Payment gateway |
| **MSG91** | API Gateway, POS | SMS OTP delivery |
| **Firebase FCM** | POS Backend | Mobile push notifications |
| **AiSensy** | POS Backend | WhatsApp message automation |
| **AWS S3** | Multiple | Document & file storage (ap-south-1) |
| **AWS CloudWatch** | Multiple | Centralized logging |
| **Google Sheets** | POS Reporting | Analytics data source |

### 2.6 LeadSquared Integration

LeadSquared is the CRM system used for lead lifecycle management -- from website visitor tracking through policy purchase and renewal.

```mermaid
graph TB
    subgraph "Lead Sources"
        WEB_TRACK["BK Web Tracker.js<br/><i>Behavioral tracking</i><br/>Account: 66484"]
        QUOTE_FLOW["Quote Generation<br/><i>L1 form → OTP → Quote</i>"]
        BULK_CSV["Bulk CSV Upload<br/><i>Renewal leads</i>"]
        PARTNER_LEAD["Partner Leads<br/><i>via Partner API</i>"]
        OFFLINE["Offline / Manual<br/><i>Admin-created leads</i>"]
    end

    subgraph "API Gateway :3005"
        LSQ_MODULE["LeadSquared Module"]
        LSQ_BULK["Bulk Operations Service<br/><i>5 req/sec rate limit</i>"]
        LSQ_LAPP["LAPP Service<br/><i>Renewal management</i>"]
        LSQ_CRON["Cron Jobs<br/><i>Daily 4:30 AM IST</i><br/>- Leads not converted (45 days)<br/>- Policy reminders<br/>- Proposal form reminders"]
    end

    subgraph "Question Service :3007"
        LSQ_QS["LeadSquared Service<br/><i>Product-specific lead creation</i><br/>CPM, EAR, CAR, Crime,<br/>Cyber, CGL, D&O, E&O"]
    end

    subgraph "LeadSquared API"
        direction TB
        LSQ_CAPTURE["Lead.Capture<br/><i>Create/update leads</i>"]
        LSQ_ACTIVITY["ProspectActivity.Create<br/><i>Log activities</i>"]
        LSQ_OPP["Opportunity<br/><i>Cross-sell, renewals</i>"]
        LSQ_GET["Lead.GetById<br/><i>Fetch lead details</i>"]
        LSQ_WEBHOOK["Webhooks<br/><i>Callbacks to Gateway</i>"]
    end

    WEB_TRACK -->|"Page views, clicks,<br/>form interactions"| LSQ_CAPTURE
    QUOTE_FLOW --> LSQ_MODULE
    BULK_CSV --> LSQ_BULK
    PARTNER_LEAD --> LSQ_MODULE
    OFFLINE --> LSQ_MODULE

    LSQ_MODULE -->|"Lead data + custom fields"| LSQ_CAPTURE
    LSQ_MODULE -->|"Policy events"| LSQ_ACTIVITY
    LSQ_MODULE -->|"Renewal/cross-sell"| LSQ_OPP
    LSQ_LAPP -->|"Renewal bulk"| LSQ_CAPTURE
    LSQ_QS -->|"Product-specific leads"| LSQ_CAPTURE

    LSQ_WEBHOOK -->|"Payment complete,<br/>callback requests"| LSQ_MODULE
    LSQ_GET -->|"Lead details"| LSQ_MODULE

    style WEB_TRACK fill:#e3f2fd
    style LSQ_CAPTURE fill:#fff3e0
    style LSQ_WEBHOOK fill:#fce4ec
```

#### Data Flow Summary

**Outbound (BimaKavach → LeadSquared):**

| Trigger | Data Sent | Custom Fields (mx_*) |
|---------|-----------|---------------------|
| Quote generated | Lead: name, email, phone, company | Product_Name, Lead_Intent, OTP_Verification_Status |
| Payment completed | Activity update with policy details | Policy number, premium, insurer |
| Product-specific L1 form | Lead with product attributes | Sum_Insured, City, Policy_Type + product-specific fields |
| Bulk CSV upload | Multiple leads/activities | Renewal requirements, dates |
| Cron jobs (daily) | Reminder activities | Stale lead notifications, policy reminders |

**Inbound (LeadSquared → BimaKavach):**

| Trigger | Data Received | Action Taken |
|---------|--------------|-------------|
| Payment webhook (`/lead-square/byId/:id`) | Lead prospect ID | Fetch lead → create policy → send confirmation email |
| Cross-sell webhook (`/lead-square/byOpportunityId/:id`) | Opportunity ID | Process cross-sell payment |
| Callback request | Phone + prospect ID | Queue callback for sales team |
| Renewal webhook (`/lead-square/renewal-management/*`) | Lead ID | Send payment link or process renewal |

#### Database Tables

```
LeadSquare                      → Internal lead record (productId, isConverted, leadCreated source)
LeadSquareIdMapping             → Maps internal lead ID ↔ LSQ prospect ID
L1FormLsqActivityIdMapping      → Maps L1 form submission ↔ LSQ activity ID + opportunity ID
LsqCallbackRequests             → Stores phone + prospect_id for callback queue
LsqBulkOperation                → Tracks bulk upload jobs (QUEUED → COMPLETE/FAILED)
LsqBulkOperationResult          → Individual row results with payload + response + errors
LsqEntitySubtype                → Schema definitions for bulk operation field types
LsqFieldMaster                  → Field definitions per entity (key, name, dataType)
```

#### Lead Sources (leadCreated enum)
`Online` | `Offline` | `Bulk-Upload` | `Lead-Square` | `Recommendations` | `Partner-Lead`

---

### 2.7 Sibro Integration

Sibro is the insurance broker management platform used for policy record-keeping, regulatory compliance, and commission tracking. BimaKavach syncs master data FROM Sibro and pushes policy/transaction data TO Sibro.

```mermaid
graph TB
    subgraph "Sibro Platform"
        direction TB
        S_INSURERS["Insurers & Branches"]
        S_PRODUCTS["Products / Policies"]
        S_CRM["CRM Users"]
        S_BOWNERS["Business Owners"]
        S_CFIELDS["Policy Custom Fields"]
        S_CLIENT["Client Management"]
        S_POLICY["Policy Records"]
        S_PREMIUM["Premium Transactions"]
    end

    subgraph "Question Service :3007 (sibro module)"
        SYNC_SVC["Sync Service<br/><i>Master data pull</i>"]
        CLIENT_SVC["Client Service<br/><i>Search & create clients</i>"]
        POLICY_SVC["Policy Service<br/><i>Push policies on creation</i>"]
        MAPPING_SVC["Mapping Services<br/><i>Insurer, Product, CRM User</i>"]
    end

    subgraph "Internal Tables"
        direction TB
        T_INSURERS["sibro_insurers<br/>sibro_insurer_branches"]
        T_PRODUCTS["sibro_products"]
        T_CRM["sibro_crm_users"]
        T_BOWNERS["sibro_business_owners"]
        T_CFIELDS_T["sibro_policy_custom_fields<br/>sibro_policy_custom_field_options"]
        T_MAPPING["sibro_insurer_mapping<br/>sibro_product_mapping<br/>sibro_crm_user_mapping"]
        T_CLIENT["sibro_client_mapping"]
        T_TX["sibro_client_transactions"]
    end

    %% Inbound sync (Sibro → BK)
    S_INSURERS -->|"GET /insurers"| SYNC_SVC
    S_PRODUCTS -->|"GET /policies?page=N"| SYNC_SVC
    S_CRM -->|"GET /crm-users?page=N"| SYNC_SVC
    S_BOWNERS -->|"GET /business-owners"| SYNC_SVC
    S_CFIELDS -->|"GET /policy-custom-fields/{id}"| SYNC_SVC

    SYNC_SVC --> T_INSURERS & T_PRODUCTS & T_CRM & T_BOWNERS & T_CFIELDS_T
    MAPPING_SVC --> T_MAPPING

    %% Outbound sync (BK → Sibro)
    POLICY_SVC -->|"POST /add-policy"| S_POLICY
    POLICY_SVC -->|"POST /add-premium-transaction/{id}"| S_PREMIUM
    CLIENT_SVC -->|"POST /client (search)"| S_CLIENT
    CLIENT_SVC -->|"POST /client (create)"| S_CLIENT
    CLIENT_SVC --> T_CLIENT
    POLICY_SVC --> T_TX

    style S_INSURERS fill:#e8f5e9
    style S_PRODUCTS fill:#e8f5e9
    style S_CRM fill:#e8f5e9
    style S_POLICY fill:#fff3e0
    style S_PREMIUM fill:#fff3e0
    style S_CLIENT fill:#fff3e0
```

#### Sync Direction & Triggers

**Inbound (Sibro → BimaKavach) -- Master Data Pull:**

| Data | Endpoint | Message Pattern | Trigger | Strategy |
|------|----------|----------------|---------|----------|
| Insurers + Branches | `GET /insurers` | `sibro.sync.insurers` | Manual / scheduled | Upsert + deactivate stale |
| Products | `GET /policies?page=N` | `sibro.sync.products` | Manual / scheduled | Paginated upsert |
| CRM Users | `GET /crm-users?page=N` | `sibro.sync.crmUsers` | Manual / scheduled | Paginated upsert |
| Business Owners | `GET /business-owners` | `sibro.sync.businessOwners` | Manual / scheduled | Upsert + deactivate |
| Custom Fields | `GET /policy-custom-fields/{id}` | `sibro.sync.policyCustomFields` | Manual / scheduled | Per-product sync |

All master syncs follow an **upsert + deactivate** pattern: mark all existing records inactive → upsert from API response → stale records remain inactive.

**Outbound (BimaKavach → Sibro) -- On Policy Creation:**

```mermaid
flowchart TD
    POLICY_CREATED["Policy Created in BimaKavach"]
    CHECK_FLAG{"Feature flag<br/>SEND_POLICY_TO_SIBRO<br/>enabled?"}
    FIND_CLIENT["Search Sibro Client<br/><i>PAN → GST-PAN → Email → Phone</i>"]
    MATCH{"Match<br/>result?"}
    EXACT["Exact Match (1 client)<br/>→ Auto-save mapping"]
    MULTIPLE["Multiple Matches<br/>→ Require manual selection"]
    NO_MATCH["No Match<br/>→ Create new client in Sibro"]
    CREATE_POLICY["POST /add-policy<br/><i>Client ID, business owner,<br/>CRM user, custom fields</i>"]
    CREATE_TX["POST /add-premium-transaction<br/><i>Insurer, branch, amount,<br/>payment mode, date</i>"]
    SAVE_TX["Save to sibro_client_transactions"]
    ERROR_EMAIL["Send error email to admin<br/><i>with stage checkpoint</i>"]

    POLICY_CREATED --> CHECK_FLAG
    CHECK_FLAG -->|Yes| FIND_CLIENT
    CHECK_FLAG -->|No| SKIP["Skip Sibro sync"]
    FIND_CLIENT --> MATCH
    MATCH -->|"1 result"| EXACT
    MATCH -->|"2+ results"| MULTIPLE
    MATCH -->|"0 results"| NO_MATCH
    EXACT --> CREATE_POLICY
    NO_MATCH --> CREATE_POLICY
    CREATE_POLICY --> CREATE_TX
    CREATE_TX --> SAVE_TX
    CREATE_POLICY -.->|"On failure"| ERROR_EMAIL
    CREATE_TX -.->|"On failure"| ERROR_EMAIL

    style POLICY_CREATED fill:#e3f2fd
    style SAVE_TX fill:#c8e6c9
    style ERROR_EMAIL fill:#ffcdd2
```

**Client Search Priority:** PAN → PAN extracted from GST → Email → Phone

#### Mapping Tables

| Table | Links | Purpose |
|-------|-------|---------|
| `sibro_insurer_mapping` | Sibro insurer ↔ BK insurer | Route policies to correct Sibro insurer |
| `sibro_product_mapping` | Sibro product ↔ BK product | Map product catalog between systems |
| `sibro_crm_user_mapping` | Sibro CRM user ↔ BK internal user | Assign policies to correct CRM owner |
| `sibro_client_mapping` | BK user+company ↔ Sibro client ID | Avoid duplicate client creation |
| `sibro_client_transactions` | Sibro policy ID ↔ BK policy ID | Track which policies have been synced |

#### Authentication
- **Method:** Bearer token (`SIBRO_AUTH_TOKEN` env var)
- **Base URL:** `https://bimakavachbroking.sibro.xyz/api/v1/`
- **Override:** Sync endpoints accept optional `auth_token` in payload

#### Error Handling
- Staged execution tracking (resolve_client → create_policy → create_premium_transaction → save_transaction)
- Email alerts to admin on any failure with exact failure stage
- CloudWatch logging for all operations

---

### 2.8 Infrastructure & Deployment

```mermaid
graph TB
    subgraph "Docker Network: local"
        subgraph "Frontend Containers"
            D_WEB["BK Web<br/>:3000<br/>Node 18.17"]
            D_ADMIN["bi-admin<br/>:3000<br/>Node 18.16"]
            D_POS_ADMIN["POS Admin<br/>:3012<br/>Node 22"]
        end
        subgraph "Backend Containers"
            D_GW["API Gateway<br/>:3005<br/>Node 18-slim"]
            D_COMMON["Common<br/>:3006<br/>Node 18-slim"]
            D_QUESTION["Question<br/>:3007<br/>Node 18-slim<br/>+ Chromium"]
            D_EMAIL["Email<br/>:3008<br/>Node 18-slim<br/>+ Puppeteer"]
            D_PROPOSAL["Proposal Form<br/>:3009<br/>Node 18-slim"]
            D_POS_API["POS API<br/>:3011<br/>Node 22-slim"]
        end
    end

    subgraph "AWS (ap-south-1)"
        RDS_PG[(RDS PostgreSQL)]
        S3_BUCK["S3 Buckets<br/>bimakavach-v2<br/>bimakavach-policies<br/>bk-pos-policies"]
        CW_LOGS["CloudWatch Logs"]
        CF_CDN["CloudFront CDN"]
    end

    D_GW & D_COMMON & D_QUESTION & D_EMAIL & D_PROPOSAL & D_POS_API --> RDS_PG
    D_GW & D_QUESTION & D_EMAIL & D_PROPOSAL & D_POS_API --> S3_BUCK
    D_GW --> CW_LOGS
```

**Tech Stack Summary:**
- **Runtime:** Node.js 18 (main platform), Node.js 22 (POS)
- **Backend Framework:** NestJS 10-11
- **Frontend Frameworks:** Next.js 13-16, React Native/Expo 54
- **Database:** PostgreSQL via TypeORM (synchronize mode)
- **Transport:** TCP (NestJS Microservices) between backend services
- **Auth:** JWT + HMAC-SHA256 signing + OTP (MSG91)
- **Containerization:** Docker + Docker Compose, shared `local` network
- **Cloud:** AWS (RDS, S3, CloudWatch, CloudFront), Google Cloud (Document AI, Storage)
- **Region:** ap-south-1 (Mumbai)

---

## 3. Usability Architecture

### 3.1 User Roles & Access

```mermaid
graph TB
    subgraph "BK Web Portal"
        CLIENT["Client / Business Owner<br/>- Get quotes & compare<br/>- Purchase policies<br/>- View dashboard<br/>- File claims<br/>- Fill proposal forms"]
    end

    subgraph "bi-admin Panel"
        ADMIN_SUPER["Super Admin<br/>- Full system access<br/>- User management<br/>- Master data config"]
        SALES["Sales Head / Team<br/>- Lead management<br/>- Client tracking<br/>- Quote oversight"]
        CS["Customer Success<br/>- Policy support<br/>- Claims processing<br/>- Client communication"]
        MARKETING["Marketing Team<br/>- Email campaigns<br/>- Newsletter management<br/>- Case studies"]
    end

    subgraph "POS Platform"
        POS_AGENT["POS Agent<br/>- Quote generation<br/>- Policy creation<br/>- Claims filing<br/>- Client management"]
        POS_HEAD["POS Head / Manager<br/>- Agent oversight<br/>- Campaign management<br/>- KPI tracking"]
        POS_ADMIN_ROLE["POS Admin<br/>- Agent onboarding<br/>- KYC approval<br/>- Commission management"]
    end

    CLIENT -->|"HTTPS"| WEB_APP["BK Web V2.0"]
    ADMIN_SUPER & SALES & CS & MARKETING -->|"HTTPS"| ADMIN_APP["bi-admin V2"]
    POS_AGENT -->|"Mobile"| MOBILE_APP["POS Mobile App"]
    POS_HEAD -->|"Browser"| REPORT_APP["POS Reporting"]
    POS_ADMIN_ROLE -->|"Browser"| POS_ADMIN_APP["POS Admin Panel"]
```

### 3.2 Customer Journey (BK Web)

```mermaid
flowchart TD
    START([Customer visits BK Web]) --> BROWSE["Browse insurance products<br/><i>20+ product categories</i>"]
    BROWSE --> SELECT_PRODUCT["Select product<br/><i>e.g. Fire, Marine, Cyber, WC</i>"]
    SELECT_PRODUCT --> L1_QUESTIONS["Fill L1 Questions<br/><i>Basic risk info: turnover, industry, etc.</i>"]
    L1_QUESTIONS --> OTP["OTP Verification<br/><i>Mobile/Email via MSG91</i>"]
    OTP --> GET_QUOTES["Get Quotes from Insurers<br/><i>Compare premiums, coverage, features</i>"]

    GET_QUOTES --> COMPARE["Compare Quotes<br/><i>Side-by-side comparison</i>"]
    COMPARE --> SELECT_INSURER["Select Insurer & Plan"]

    SELECT_INSURER --> L2_QUESTIONS["Fill L2 Questions<br/><i>Detailed underwriting info</i>"]
    L2_QUESTIONS --> COMPANY_DETAILS["Provide Company Details<br/><i>CIN, GST, PAN, documents</i>"]
    COMPANY_DETAILS --> PROPOSAL_FORM["Fill Proposal Form<br/><i>Insurer-specific PDF form via Anvil</i>"]
    PROPOSAL_FORM --> PAYMENT["Make Payment<br/><i>Future Generali / Chola / Magma</i>"]
    PAYMENT --> POLICY_ISSUED["Policy Issued"]

    POLICY_ISSUED --> DASHBOARD["Customer Dashboard"]
    DASHBOARD --> VIEW_POLICIES["View Policies"]
    DASHBOARD --> FILE_CLAIM["File a Claim"]
    DASHBOARD --> RENEW["Renew Policy"]
    DASHBOARD --> DOCS["Download Documents"]

    style START fill:#e1f5fe
    style POLICY_ISSUED fill:#c8e6c9
    style DASHBOARD fill:#fff3e0
```

**Key Touchpoints:**

| Step | Service Involved | Key Action |
|------|-----------------|------------|
| Browse Products | Common Service | Fetch product groups, product list |
| L1 Questions | Question Service | Dynamic questionnaire per product |
| OTP Verification | API Gateway | MSG91 SMS/Email OTP |
| Get Quotes | Question Service | Calculate premiums per insurer |
| Compare Quotes | BK Web (client-side) | Side-by-side comparison UI |
| L2 Questions | Question Service | Insurer-specific detailed questions |
| Company Details | API Gateway | CIN verification via Probe API |
| Proposal Form | Proposal Form Service | Anvil PDF generation & signing |
| Payment | API Gateway | Payment gateway integration |
| Policy Issued | Question Service | Policy record creation, email trigger |
| Dashboard | Multiple | Policy listing, claims, documents |

### 3.3 Admin Journey (bi-admin)

```mermaid
flowchart TD
    LOGIN([Admin Login<br/><i>JWT + Role-based access</i>])

    LOGIN --> DASH["Dashboard<br/><i>Policy stats, KPIs</i>"]

    DASH --> CLIENT_MGMT["Client Management"]
    CLIENT_MGMT --> VIEW_CLIENTS["View Clients & Policies"]
    CLIENT_MGMT --> COMPANY_MGMT["Manage Company Details"]
    CLIENT_MGMT --> FILE_CLAIM_ADMIN["File Claim on behalf"]

    DASH --> LEAD_MGMT["Lead Management"]
    LEAD_MGMT --> LEADS["Track Leads<br/><i>LeadSquared sync</i>"]
    LEAD_MGMT --> NEWSLETTER["Newsletter Subscribers"]
    LEAD_MGMT --> CASE_STUDY["Case Study Distribution"]

    DASH --> PRODUCT_CONFIG["Product Configuration"]
    PRODUCT_CONFIG --> PRODUCTS["Products & Groups"]
    PRODUCT_CONFIG --> INSURERS["Insurance Companies"]
    PRODUCT_CONFIG --> PRICING_CFG["Pricing Setup"]
    PRODUCT_CONFIG --> QUESTIONS_CFG["L1/L2 Questions"]
    PRODUCT_CONFIG --> PROPOSAL_CFG["Proposal Forms"]

    DASH --> COMMS["Communications"]
    COMMS --> EMAIL_TEMPLATES["Email Templates & Bodies"]
    COMMS --> EMAIL_TRIGGERS["Email Triggers & Scheduling"]
    COMMS --> EMAIL_LOGS["Email History"]

    DASH --> QCR_MGMT["QCR Management<br/><i>Quick Clarification Requests</i>"]

    DASH --> MASTER_DATA["Master Data"]
    MASTER_DATA --> INDUSTRIES["Industries"]
    MASTER_DATA --> OCCUPANCIES["Fire/WC Occupancies"]
    MASTER_DATA --> GEO["States, Cities, Pincodes"]
    MASTER_DATA --> ROLES_RES["Roles & Permissions"]

    style LOGIN fill:#e1f5fe
    style DASH fill:#fff3e0
```

### 3.4 POS Agent Journey

```mermaid
flowchart TD
    AGENT_LOGIN([Agent Login via OTP<br/><i>Mobile App</i>])

    AGENT_LOGIN --> HOME["Home Dashboard<br/><i>KPIs, recent activity</i>"]

    HOME --> NEW_QUOTE["Generate Quote"]
    NEW_QUOTE --> SELECT_CLIENT["Select/Create Client"]
    SELECT_CLIENT --> SELECT_PROD["Select Product"]
    SELECT_PROD --> FILL_L1["Fill L1 Questions"]
    FILL_L1 --> RECEIVE_QUOTE["Receive Quote(s)"]
    RECEIVE_QUOTE --> SHARE_QUOTE["Share with Client<br/><i>WhatsApp / Email</i>"]

    HOME --> MANAGE_POLICIES["Manage Policies"]
    MANAGE_POLICIES --> VIEW_POL["View Active Policies"]
    MANAGE_POLICIES --> ENDORSEMENTS["Request Endorsements"]
    MANAGE_POLICIES --> RENEWALS["Track Renewals"]

    HOME --> CLAIMS_FLOW["Claims"]
    CLAIMS_FLOW --> FILE_CLAIM_POS["File Claim"]
    CLAIMS_FLOW --> TRACK_CLAIM["Track Claim Status"]

    HOME --> CLIENTS_FLOW["Client Management"]
    CLIENTS_FLOW --> ADD_CLIENT["Add New Client<br/><i>Retail or Corporate</i>"]
    CLIENTS_FLOW --> CLIENT_KYC["Submit KYC Documents"]

    HOME --> CAMPAIGNS["Campaigns<br/><i>Sales targets & milestones</i>"]

    HOME --> COMMISSIONS["View Commissions<br/><i>Brokerage tracking</i>"]

    HOME --> NOTIF["Notifications<br/><i>Push (Firebase) + WhatsApp</i>"]

    style AGENT_LOGIN fill:#e1f5fe
    style HOME fill:#fff3e0
    style SHARE_QUOTE fill:#c8e6c9
```

**POS Admin manages:** Agent onboarding, KYC approval, campaign creation, commission grids, WhatsApp templates, claims review.

**POS Reporting provides:** Premium trends, regional performance, agent leaderboards, KPI summaries.

### 3.5 Feature-to-Service Mapping

```mermaid
graph TB
    subgraph "Features"
        F_QUOTE["Quote<br/>Comparison"]
        F_POLICY["Policy<br/>Management"]
        F_CLAIMS["Claims<br/>Processing"]
        F_PROPOSAL["Proposal<br/>Forms"]
        F_EMAIL["Email<br/>Campaigns"]
        F_PAYMENT["Payment<br/>Processing"]
        F_CRM["CRM / Lead<br/>Management"]
        F_MASTER["Master Data<br/>Config"]
        F_OCR["Document<br/>OCR"]
        F_POS_AGENT["POS Agent<br/>Operations"]
        F_NOTIF["Push & WhatsApp<br/>Notifications"]
        F_ANALYTICS["Reporting &<br/>Analytics"]
    end

    subgraph "Services"
        S_GW["API Gateway"]
        S_COMMON["Common"]
        S_QUESTION["Question"]
        S_EMAIL["Email"]
        S_PROPOSAL["Proposal Form"]
        S_POS["POS Backend"]
    end

    F_QUOTE --> S_QUESTION & S_COMMON
    F_POLICY --> S_QUESTION & S_GW
    F_CLAIMS --> S_QUESTION
    F_PROPOSAL --> S_PROPOSAL & S_EMAIL
    F_EMAIL --> S_EMAIL & S_GW
    F_PAYMENT --> S_GW & S_QUESTION
    F_CRM --> S_GW
    F_MASTER --> S_COMMON
    F_OCR --> S_QUESTION
    F_POS_AGENT --> S_POS & S_COMMON & S_QUESTION
    F_NOTIF --> S_POS
    F_ANALYTICS --> S_POS
```

| Feature | Primary Service | Supporting Services | External Dependencies |
|---------|----------------|--------------------|-----------------------|
| **Quote Generation** | Question :3007 | Common :3006 (products, pricing) | Chola API |
| **Policy Management** | Question :3007 | Gateway :3005, Email :3008 | LeadSquared, Sibro |
| **Claims Processing** | Question :3007 | Email :3008 | -- |
| **Proposal Forms** | Proposal :3009 | Email :3008, Common :3006 | Anvil.io (PDF) |
| **Email Campaigns** | Email :3008 | Common :3006 | SendGrid |
| **Payment** | Gateway :3005 | Question :3007 | Future Generali, Chola, Magma, Cashfree |
| **CRM/Leads** | Gateway :3005 | -- | LeadSquared |
| **Master Data** | Common :3006 | -- | -- |
| **Document OCR** | Question :3007 | -- | Google Document AI, Tesseract |
| **Company Verification** | Gateway :3005 | -- | Probe API |
| **POS Operations** | POS :3011 | Common :3006, Question :3007 | Firebase, AiSensy, MSG91 |
| **POS Analytics** | POS Reporting | -- | Google Sheets |

---

*Generated on 2026-03-31 from codebase analysis.*
