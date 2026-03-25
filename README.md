# RetailFixIT — Field Service Management Platform

A production-grade, multi-tenant field service operations platform that coordinates service jobs between customers and ~1,000 vendors. Built on a fully event-driven Azure architecture with AI-assisted dispatch, real-time push notifications, strict RBAC isolation, and a seamless dev→prod provider swap strategy — no application code changes required between environments.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Tech Stack](#tech-stack)
3. [Project Structure](#project-structure)
4. [Setup Instructions](#setup-instructions)
5. [Example Event Flow](#example-event-flow)
6. [AI Integration](#ai-integration)
7. [Performance Considerations](#performance-considerations)
8. [Engineering Reasoning](#engineering-reasoning)
9. [API Reference](#api-reference)
10. [User Roles & Permissions](#user-roles--permissions)

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                         BROWSER  (React 19 SPA)                       │
│   TanStack Query · Zustand · @azure/msal-browser · @microsoft/signalr │
└──────────────────────────────┬───────────────────────────────────────┘
                               │  HTTPS REST  +  WSS SignalR
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    ASP.NET Core 10  (Clean Architecture)              │
│                                                                        │
│  ┌──────────────┐   ┌────────────────────┐   ┌─────────────────────┐ │
│  │  Controllers │   │  MediatR CQRS       │   │   SignalR Hub       │ │
│  │  (REST API)  │──►│  Commands + Queries │   │   /hubs/jobs        │ │
│  └──────────────┘   │  + FluentValidation │   └──────────┬──────────┘ │
│                     └─────────┬───────────┘              │            │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │                  Infrastructure Layer                             │ │
│  │  ┌────────────┐  ┌─────────────┐  ┌──────────────┐  ┌────────┐ │ │
│  │  │ EF Core    │  │ MassTransit │  │ IAIProvider  │  │ ICache │ │ │
│  │  │ Cosmos DB  │  │ Consumers   │  │ Gemini / OAI │  │ Service│ │ │
│  │  └────────────┘  └─────────────┘  └──────────────┘  └────────┘ │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │  Middleware: TenantResolution · ExceptionHandling · Auth (MSAL)  │ │
│  └──────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────┬───────────────────────────────────────┘
                               │
          ┌────────────────────┼─────────────────────┐
          ▼                    ▼                      ▼
  ┌───────────────┐   ┌──────────────────┐   ┌──────────────────┐
  │  Azure        │   │  Azure           │   │  Azure SignalR   │
  │  Cosmos DB    │   │  Service Bus     │   │  Service         │
  └───────────────┘   └──────────────────┘   └──────────────────┘
          │                                          │
  ┌───────────────┐                        ┌──────────────────┐
  │  Azure Cache  │                        │  Azure OpenAI    │
  │  for Redis    │                        │  / Gemini API    │
  └───────────────┘                        └──────────────────┘
```

### Layer Responsibilities

| Layer | Responsibility |
|---|---|
| **Domain** | Entities, enums, domain events, repository interfaces — zero external dependencies |
| **Application** | MediatR commands/queries, FluentValidation pipelines, AutoMapper profiles, service interfaces |
| **Infrastructure** | EF Core + Cosmos DB, MassTransit consumers, AI providers, cache, SignalR notifications, auth |
| **API** | Controllers, SignalR hub, middleware (tenant resolution, exception handling), Program.cs DI root |
| **Frontend** | React SPA — TanStack Query for server state, Zustand for auth, MSAL for token lifecycle |

### Dev vs Production Provider Swap

Every external service is controlled by a single `"Provider"` config key in `appsettings.json`. **No consumer or application code changes between dev and prod.**

| Service | Dev | Production |
|---|---|---|
| Auth | JWT Bearer (local) | Microsoft Entra ID (`Microsoft.Identity.Web` + MSAL) |
| Database | Azure Cosmos DB | Azure Cosmos DB |
| AI | Google Gemini 1.5 Flash (free) | Azure OpenAI (GPT-4o) |
| Messaging | MassTransit in-memory | MassTransit + Azure Service Bus |
| Real-time | ASP.NET Core SignalR in-process | Azure SignalR Service |
| Cache | `IMemoryCache` | Azure Cache for Redis |

---

## Tech Stack

### Backend

| Technology | Purpose |
|---|---|
| ASP.NET Core 10 | Web API host |
| Clean Architecture | Strict layer boundaries: Domain → Application → Infrastructure → API |
| Entity Framework Core + Cosmos DB | Multi-tenant data access with global `TenantId` query filters |
| MediatR | CQRS — commands and queries separated from controllers |
| FluentValidation | Pipeline-level request validation before handlers run |
| MassTransit | Abstracted event bus (in-memory ↔ Azure Service Bus via config) |
| Microsoft.Identity.Web | Azure Entra ID Bearer token validation |
| AutoMapper | Entity ↔ DTO mapping profiles |
| Serilog | Structured JSON logging (console in dev, App Insights in prod) |

### Frontend

| Technology | Purpose |
|---|---|
| React 19 + TypeScript + Vite | SPA with fast HMR |
| TanStack Query v5 | Server state: caching, pagination, optimistic updates |
| Zustand v5 | Client state: resolved user info after auth |
| React Router v7 | Protected routes with role-based guards |
| `@azure/msal-browser` v5 | Entra ID popup auth, token caching, silent refresh |
| `@microsoft/signalr` | Real-time hub client with auto-reconnect |
| React Hook Form + Zod | Forms with schema validation |
| Tailwind CSS v4 | Utility-first styling |
| TanStack Table | Headless job dashboard table |
| lucide-react | Icon set |

---

<img width="1840" height="784" alt="image" src="https://github.com/user-attachments/assets/af6e2d30-6db1-44bf-8f60-3e64e110f1fd" />


## Project Structure

```
RetailFixIT/
├── docker-compose.yml                    # Cosmos DB emulator (local dev alternative)
├── frontend/
│   └── src/
│       ├── api/                          # Typed axios API modules (jobs, vendors, assignments, AI)
│       ├── features/
│       │   ├── auth/                     # LoginPage — MSAL popup flow
│       │   ├── dashboard/                # DashboardPage, JobTable, JobFilters
│       │   ├── jobs/                     # JobDetailPage, CreateJobModal, JobTimeline
│       │   ├── assignments/              # AssignmentPanel, VendorSelector
│       │   ├── recommendations/          # AIRecommendationCard, RecommendationPanel
│       │   └── vendors/                  # VendorListPage, VendorForm
│       ├── hooks/                        # useSignalR, usePermissions, useJobs, useVendors
│       ├── lib/                          # msalConfig, queryClient, signalr factory
│       ├── router/                       # index.tsx, ProtectedRoute (role-based)
│       └── store/                        # authStore (Zustand — user info only)
└── backend/
    ├── tests.http                         # Runnable HTTP test file (REST Client / JetBrains)
    └── src/
        ├── RetailFixIT.Domain/
        │   ├── Entities/                  # Job, Vendor, Assignment, AIRecommendation, AuditLog
        │   ├── Enums/                     # JobStatus, Priority, UserRoleType, AssignmentStatus
        │   ├── Events/                    # JobCreated, JobAssigned, AIRecommendation{Requested,Generated}
        │   └── Interfaces/                # IJobRepository, IVendorRepository, IAssignmentRepository
        ├── RetailFixIT.Contracts/          # MassTransit event message contracts
        ├── RetailFixIT.Application/
        │   ├── Jobs/Commands/             # CreateJob, UpdateJob, CancelJob, PatchJobStatus
        │   ├── Jobs/Queries/              # GetJobs (paged+filtered), GetJobById, GetJobTimeline
        │   ├── Vendors/                   # GetVendors, GetVendorById, CreateVendor, UpdateVendor
        │   ├── Assignments/               # AssignVendor, UnassignVendor, GetAssignments
        │   └── AI/Commands/               # RequestAIRecommendation, GetRecommendations
        └── RetailFixIT.Infrastructure/
            ├── Persistence/               # AppDbContext (global tenant filters), Repositories
            ├── Messaging/Consumers/       # 4 event consumers (full pipeline)
            ├── AI/                        # IAIProvider, GeminiAIProvider, AzureOpenAIProvider
            ├── Caching/                   # MemoryCacheService, RedisCacheService
            ├── Auth/                      # JwtTokenService, CurrentUserService
            └── Realtime/                  # SignalRNotificationService
```

---

## Setup Instructions

### Prerequisites

| Tool | Version |
|---|---|
| .NET SDK | 10.0+ |
| Node.js | 18+ |
| Docker Desktop | For Cosmos DB emulator (optional — use Azure Cosmos DB directly) |

### 1. Get a free Gemini API key

Visit https://aistudio.google.com/app/apikey and create a key. It's free with generous rate limits.

### 2. Configure the backend

`backend/src/RetailFixIT.API/appsettings.Development.json` — fill in real values:

```jsonc
{
  "Auth": {
    "Provider": "AzureAd",              // "Jwt" for local JWT-only dev
    "AzureAd": {
      "TenantId":         "<entra-tenant-id>",
      "ClientId":         "<app-registration-client-id>",
      "Audience":         "api://<client-id>",
      "InternalTenantId": "11111111-1111-1111-1111-111111111111"
    }
  },
  "CosmosDb": {
    "Endpoint":      "<cosmos-db-endpoint>",
    "AccountKey":    "<cosmos-account-key>",
    "DatabaseName":  "RetailFixIT"
  },
  "AI": {
    "Provider": "Gemini",
    "Gemini": {
      "ApiKey": "<your-gemini-api-key>",
      "Model":  "gemini-1.5-flash"
    }
  },
  "Messaging": { "Provider": "InMemory" },     // "AzureServiceBus" + connection string for prod
  "SignalR":   { "Provider": "InMemory" },     // "AzureSignalR" + connection string for prod
  "Cache":     { "Provider": "Memory" }        // "Redis" + connection string for prod
}
```

### 3. Run the backend

```bash
dotnet run --project backend/src/RetailFixIT.API
# API → http://localhost:5000
# Swagger → http://localhost:5000/swagger
# Cosmos DB containers + seed data created automatically on first run
```

### 4. Run the frontend

```bash
cd frontend
npm install
npm run dev
# SPA → http://localhost:5173  (proxies /api and /hubs → http://localhost:5000)
```

### 5. Azure App Registration (for Entra ID login)

1. **App Registration → Authentication → Add platform → Single-page application**
   - Redirect URI: `http://localhost:5173`
2. **Expose an API → Add a scope**: `access_as_user`
3. **App roles → Create role** for each: `Dispatcher`, `VendorManager`, `Admin`, `SupportAgent`
4. **Enterprise Applications → [your app] → Users and groups** → assign roles to users

### 6. Local dev test accounts (JWT mode — `"Provider": "Jwt"`)

When running with `Auth.Provider = "Jwt"`, the `/api/v1/auth/login` endpoint accepts hardcoded test credentials:

| Email | Password | Role |
|---|---|---|
| dispatcher@acme.com | Test1234! | Dispatcher |
| vendormgr@acme.com | Test1234! | VendorManager |
| admin@acme.com | Test1234! | Admin |
| support@acme.com | Test1234! | SupportAgent |

---

## Example Event Flow

The full lifecycle from job creation to dashboard update across five asynchronous stages:

```
  Browser              ASP.NET Core API           MassTransit Bus        AI Provider
     │                       │                          │                     │
     │── POST /api/v1/jobs ──►│                          │                     │
     │                       │─ Validate (FluentVal.)   │                     │
     │                       │─ CreateJobCommandHandler │                     │
     │                       │─ SaveJob → Cosmos DB     │                     │
     │                       │─ Publish JobCreatedEvent ►│                     │
     │◄── 201 Created ────────│                          │                     │
     │   (returns immediately)│                          │                     │
     │                       │                          │                     │
     │                       │      ┌─────────────────────────────────────┐   │
     │                       │      │  JobCreatedConsumer                  │   │
     │                       │◄─────│  · Set Job.Status = InReview        │   │
     │                       │      │  · Create AIRecommendation(Pending) │   │
     │                       │      │  · Publish AIRecommendationRequested►│   │
     │                       │      └─────────────────────────────────────┘   │
     │                       │                          │                     │
     │                       │      ┌──────────────────────────────────────────────┐
     │                       │      │  AIRecommendationRequestedConsumer           │
     │                       │◄─────│  · Build AIJobContext (job + vendor list)    │
     │                       │      │  · Call IAIProvider.GenerateAsync()  ───────►│
     │                       │      │                                   Gemini API │
     │                       │      │◄──────────────── { jobSummary, vendorIds,    │
     │                       │      │                    reasoning }               │
     │                       │      │  · Validate vendorIds against known vendors  │
     │                       │      │  · Save AIRecommendation(Completed)          │
     │                       │      │  · Publish AIRecommendationGenerated ───────►│
     │                       │      └──────────────────────────────────────────────┘
     │                       │                          │                     │
     │                       │      ┌────────────────────────────────────────┐│
     │                       │      │  AIRecommendationGeneratedConsumer     ││
     │                       │◄─────│  · SignalR → tenant group broadcast    ││
     │◄── [SignalR push] ─────│      │    "AIRecommendationReady"            ││
     │  toast: "AI ready"    │      └────────────────────────────────────────┘│
     │                       │                                                  │
     │── POST /assignments ──►│                                                  │
     │                       │─ AssignVendorCommandHandler                       │
     │                       │─ Create Assignment record                         │
     │                       │─ Publish JobAssignedEvent ─────────────────────►│ │
     │◄── 201 Created ────────│                                                  │
     │                       │      ┌──────────────────────────────────────┐     │
     │                       │      │  JobAssignedConsumer                  │     │
     │                       │◄─────│  · Write immutable AuditLog entry    │     │
     │                       │      │  · SignalR → tenant group broadcast  │     │
     │                       │      │    "JobAssigned"                     │     │
     │◄── [SignalR push] ─────│      └──────────────────────────────────────┘     │
     │  all dashboards update│
```

### Key Properties

| Property | Detail |
|---|---|
| **Non-blocking** | `POST /jobs` returns `201` in ~80ms before Gemini is called |
| **Idempotent consumers** | Each consumer checks current state before acting (e.g., only transitions `New → InReview`) |
| **Tenant isolation** | Every SignalR broadcast targets `TenantGroup/{tenantId}` — other tenants never receive events |
| **Graceful AI failure** | After 3 retries, `AIRecommendation.Status = Failed`; job remains fully assignable without AI |
| **Audit trail** | Every state change writes an immutable `AuditLog` entry before the event is published |

---

## AI Integration

<img width="1842" height="846" alt="image" src="https://github.com/user-attachments/assets/2cecb1a9-53e7-4b33-8584-6b5cfc34bd20" />

### Interface abstraction

The application layer knows only one contract — `IAIProvider`. Swapping Gemini for Azure OpenAI (or any future model) requires zero changes outside the infrastructure layer:

```csharp
public interface IAIProvider
{
    Task<AIRecommendationResult> GenerateRecommendationAsync(
        AIJobContext context,
        CancellationToken ct = default);
}
```

`AIJobContext` carries: job title, description, service type, location, and the full list of available vendors (id, name, service area, specializations, rating, available capacity slots). The command handler fetches this context before publishing the event, ensuring the AI has complete information for ranking.

### Prompt design

The system prompt establishes the AI as a **field service dispatch expert**. The structured user prompt supplies:

- Job details: title, description, service type, location
- Vendor list with: skills, service area, rating, available capacity

The model is instructed to respond **only with valid JSON** in a fixed schema:

```json
{
  "jobSummary": "2-3 sentence summary of the job and what needs to be done",
  "recommendedVendorIds": ["uuid1", "uuid2"],
  "reasoning": "Why these vendors were chosen based on skills, location, and capacity"
}
```

### Response validation

The parser:
1. Extracts the JSON block from the response (robust to markdown code fences)
2. Cross-validates every returned UUID against the known vendor list — hallucinated IDs are silently dropped
3. Returns `Success: false` with the error message if the JSON is malformed

### Model settings

- `temperature: 0.3` — deterministic, consistent vendor ranking (low creative variance)
- `maxOutputTokens: 1024` — sufficient for 1–3 vendors with reasoning; avoids runaway costs

### Resilience

| Layer | Strategy |
|---|---|
| Retry | 3 attempts with exponential backoff (2s, 4s) |
| Timeout | 30-second per-attempt `CancellationTokenSource` |
| Mock fallback | Missing/unconfigured API key returns a deterministic mock so the pipeline never stalls |
| Failure record | Unrecoverable errors persist `AIRecommendation.Status = Failed` with the error message |
| Human override | Dispatchers can assign any vendor manually, independent of AI availability |

---

Service Bus
<img width="1825" height="756" alt="image" src="https://github.com/user-attachments/assets/2db88a30-2001-40b6-bae8-d11a48449396" />
<img width="1830" height="809" alt="image" src="https://github.com/user-attachments/assets/cc5702eb-b10b-4db5-8b07-2e8a5bc17a17" />

Redis
<img width="1514" height="329" alt="image" src="https://github.com/user-attachments/assets/c3014fe8-eb17-4b39-905c-c2091a40e87c" />


## Performance Considerations

### Backend

| Concern | Strategy |
|---|---|
| **AI latency off critical path** | Job creation returns `201` before Gemini is called. P99 of `POST /jobs` is unaffected by LLM response time — AI runs async through the message bus. |
| **Database reads** | `ICacheService` caches vendor lists and common read queries per tenant. Redis in production eliminates repeat Cosmos DB round-trips. |
| **Cosmos DB efficiency** | Global `TenantId` query filters align with partition keys — zero cross-partition fan-out for all standard queries. |
| **Optimistic concurrency** | Jobs carry a `RowVersion` token preventing double-assignment race conditions when two dispatchers act simultaneously. |
| **Message throughput** | MassTransit consumer concurrency is configurable. Azure Service Bus provides at-least-once delivery with dead-letter queues for failed events. |
| **SignalR scale-out** | Azure SignalR Service removes server affinity — any API instance can broadcast to any connected client without sticky sessions. |

### Frontend

| Concern | Strategy |
|---|---|
| **No polling** | TanStack Query `staleTime: 30s` prevents redundant refetches. SignalR events call `queryClient.invalidateQueries()` for surgical cache busting — the efficiency of push with the freshness of pull. |
| **Pagination** | Page-based fetching on the job dashboard; TanStack Table virtualises rendered rows to keep DOM size constant. |
| **Optimistic updates** | Assignment and status changes apply to the TanStack Query cache immediately, giving sub-10ms UI feedback before server confirmation. |
| **Bundle size** | Vite tree-shakes unused lucide icons. Route-level lazy loading defers non-critical chunks. MSAL + SignalR are the largest chunks (~213 kB gzip). |
| **Token management** | MSAL handles token caching and silent refresh in `localStorage` — zero custom token logic in application code. |

---


## API Reference

```
# Auth
POST   /api/v1/auth/login              # Dev-only: returns JWT for hardcoded test accounts
GET    /api/v1/auth/me                 # Returns resolved user info from validated token claims

# Jobs
GET    /api/v1/jobs                    # ?page&pageSize&status&priority&search&sortBy
GET    /api/v1/jobs/{id}
GET    /api/v1/jobs/{id}/timeline
POST   /api/v1/jobs                    # [Dispatcher, Admin]
PUT    /api/v1/jobs/{id}               # [Dispatcher, Admin]
PATCH  /api/v1/jobs/{id}/status        # [Dispatcher, Admin]
DELETE /api/v1/jobs/{id}/cancel        # [Admin]

# Vendors
GET    /api/v1/vendors                 # ?page&isActive&hasCapacity
GET    /api/v1/vendors/{id}
POST   /api/v1/vendors                 # [VendorManager, Admin]
PUT    /api/v1/vendors/{id}            # [VendorManager, Admin]

# Assignments
GET    /api/v1/jobs/{jobId}/assignments
POST   /api/v1/jobs/{jobId}/assignments          # [Dispatcher, Admin]
DELETE /api/v1/jobs/{jobId}/assignments/{id}     # [Dispatcher, Admin]

# AI Recommendations
POST   /api/v1/jobs/{jobId}/recommendations      # [Dispatcher, Admin] — triggers async AI
GET    /api/v1/jobs/{jobId}/recommendations
GET    /api/v1/jobs/{jobId}/recommendations/{id}

# Audit Logs
GET    /api/v1/audit-logs              # [Admin only]

# SignalR Hub
WS     /hubs/jobs
# Server → Client events:
#   JobCreated            { jobId, jobNumber, title, status }
#   JobUpdated            { jobId, status, updatedAt }
#   JobAssigned           { jobId, vendorName, assignedAt }
#   AIRecommendationReady { jobId, recommendationId }
```

For runnable HTTP examples, see `backend/tests.http`.

---

## User Roles & Permissions

<img width="1830" height="607" alt="image" src="https://github.com/user-attachments/assets/03a63a40-6d48-4513-b796-7f3acf8d33e9" />


| Action | Dispatcher | VendorManager | Admin | SupportAgent |
|---|:---:|:---:|:---:|:---:|
| View Dashboard | ✓ | ✓ | ✓ | ✓ |
| Create / Update Job | ✓ | | ✓ | |
| Assign Vendor | ✓ | | ✓ | |
| Request AI Recommendation | ✓ | | ✓ | |
| Manage Vendors | | ✓ | ✓ | |
| Cancel Job | | | ✓ | |
| View Audit Logs | | | ✓ | |

---

## End-to-End Verification

1. Sign in as `dispatcher@...` (Entra ID) or `dispatcher@acme.com / Test1234!` (JWT mode)
2. **Create a job** → dashboard row appears immediately via SignalR
3. **Open the job → AI Analysis tab** → "AI ready" toast fires within seconds; recommendation shows ranked vendors with reasoning
4. **Assignment tab** → AI-recommended vendors are highlighted; select one → Confirm Assignment
5. All open dashboard tabs update instantly via SignalR `JobAssigned` broadcast
6. Sign in as `admin@...` → **Audit Logs** → full immutable trail of every action

---

*Built as a full-stack technical assessment demonstrating event-driven Azure architecture, AI-assisted dispatch, and production-grade engineering practices across a Clean Architecture .NET 10 + React 19 stack.*
