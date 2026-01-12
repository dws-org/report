# Project Report

# Motivation

We have built an application that allows organizations, such as student associations, to sell tickets to students. Users can log in, view available events, and purchase tickets. Organizers can also log in, where they have the option to manage existing events and create new ones. 

## Why is this a dynamic Websystem?

It is a dynamic web system because the application uses server-side content generation with Next.js and the App Router, where content is generated at runtime on the server rather than being prebuilt as static HTML files; each request receives dynamically generated content based on 

- database queries (events, tickets, and user data),
- query parameters such as search, filters, and pagination
- the user session for authentication and authorization.

# Technology Stack

```julia
┌─────────────────────────────────────────┐
│        Frontend Layer                    │
│  (Next.js, React, TypeScript)           │
└────────────────┬────────────────────────┘
                 │
    ┌────────────┴────────────┐
    ↓                         ↓
┌──────────────────┐  ┌──────────────────┐
│ Event Service    │  │ Ticket Service   │
│     (Go)         │  │     (Go)         │
└────────┬─────────┘  └────────┬─────────┘
         │                     │
    ┌────┴─────────────────────┘
    ↓
┌──────────────────────────────┐
│  PostgreSQL / Supabase       │
│  (Centralized Data Store)    │
└──────────────────────────────┘

External Services:
├── RabbitMQ (Message Queue)
├── Keycloak (Authentication)
└── Kubernetes (Orchestration)
```

We chose Next.js as the frontend framework because it is well suited for building modern, dynamic, and scalable web applications.

**Main reasons:**

- Next.js allows content to be generated at runtime on the server based on user, event, and session data.
- Performance and scalability out of the box ****with ****Features like automatic code splitting and optimization improve performance while keeping the application maintainable.
- We had some prior knowledge

For our backend services we choosed Go 

**Main reasons:**

- Go offers high performance and good scalability, making it well suited for containerized deployments such as Kubernetes.
- Go provides a rich set of built-in standard libraries, which reduces the need for external dependencies.
- We already had prior experience with Go, allowing us to develop the backend efficiently and reliably.

# **Our Two Microservices**

## **1. dws-event-service (Go)**

The **Event Service** is responsible for managing the core business logic of events. It handles event creation, updates, deletions, and event-related queries. This service exposes REST APIs for the frontend to retrieve event listings, event details, and event metadata. It communicates with PostgreSQL for persistent storage and publishes events to RabbitMQ when important actions occur (e.g., event created, event updated), allowing other services to react asynchronously.

**API Endpoints**:

- `GET /api/v1/events` - List Events
- `POST /api/v1/events` - Create Event
- `GET /api/v1/events/:id` - Get Event Detail

## **2. dws-ticket-service (Go)**

The **Ticket Service** manages the ticketing system for event registrations and purchases. It handles ticket creation, ticket cancellation, refunds, and ticket lifecycle management. This service integrates with RabbitMQ to consume events from the Event Service (e.g., when an event is cancelled, tickets must be refunded) and publishes its own events (e.g., ticket purchased, ticket refunded). It also manages payment integration and tracks ticket statuses in PostgreSQL.

```julia
Frontend (Next.js)
    ↓
Event Service (Get/Create Events) ←→ RabbitMQ ←→ Ticket Service (Process Tickets)
    ↓                                                    ↓
    └────────────────────────────────────────→ PostgreSQL (Shared Database)
```

## Data & Communication Flow

### Request Flow: Frontend → Backend

```
1. User Action in Frontend
   ↓
2. Next.js Client sends HTTP Request
   (mit Auth Token / JWT)
   ↓
3. Kubernetes Ingress routes Request
   ↓
4. Service Endpoint receives Request
   (e.g., dws-event-service: 8080)
   ↓
5. Go Handler processes Request
   - Validates Input (Zod/Custom)
   - Queries Database (Prisma)
   - May publish to RabbitMQ
   ↓
6. Response sent back to Frontend
   (JSON)
   ↓
7. React updates UI State

```

# **GitOps CI/CD**

```julia
Developer
  ↓
App Repo (GitHub)
  ↓
GitHub Actions (CI)
  ├─ build + push image → GHCR
  └─ update image tag → GitOps Repo
                           ↓
                     Argo CD watches
                           ↓
                     Kubernetes sync
                           ↓
                        Users

```

## Testing and Monitoring

# Database

### Prisma Schema Event Service

```
model Event {
  id        String    @id @default(cuid())
  title     String
  description String?
  startDate DateTime
  endDate   DateTime
  location  String?
  capacity  Int?
  status    EventStatus @default(ACTIVE)

  tickets   Ticket[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

enum EventStatus {
  DRAFT
  ACTIVE
  CANCELLED
  COMPLETED
}

model Ticket {
  id        String    @id @default(cuid())
  eventId   String
  event     Event     @relation(fields:  [eventId], references: [id])
  userId    String

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

```

### Prisma Schema Ticket Service

```
model Ticket {
  id        String    @id @default(cuid())
  eventId   String
  userId    String

  status    TicketStatus @default(PURCHASED)
  price     Decimal

  paymentId String?

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

enum TicketStatus {
  PURCHASED
  USED
  CANCELLED
  REFUNDED
}

```

# Deployment

```
┌─────────────────────────────────────┐
│    Kubernetes Cluster               │
├─────────────────────────────────────┤
│                                     │
│  Namespace: default (or app)        │
│  ├── Pod: dws-frontend-xxxxx        │
│  ├── Pod: dws-event-service-xxxxx   │
│  ├── Pod: dws-ticket-service-xxxxx  │
│  └── Service: Internal DNS          │
│                                     │
│  Namespace: infrastructure          │
│  ├── RabbitMQ                       │
│  ├── PostgreSQL (or external)       │
│  └── Monitoring Stack               │
│                                     │
└─────────────────────────────────────┘

```

**Repository**: `gitops`

**GitOps-Workflow**:

```
Developer Push
    ↓
GitHub Repo Update
    ↓
ArgoCD Detects Change (polls gitops repo)
    ↓
ArgoCD Syncs to Kubernetes Cluster
    ↓
Kubernetes Applies Manifests
    ↓
Services Deployed/Updated

```

# Scalabily

# Security

## Authentifizierung & Autorisierung

### Frontend Authentication

### API Authorization

**Frontend Requests** (mit Token):

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

```

**Backend Validation**:

```go
// Middleware in Go Service
func AuthMiddleware(next http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    token := r.Header.Get("Authorization")

    // Validate JWT
    if ! validateToken(token) {
      http.Error(w, "Unauthorized", http.StatusUnauthorized)
      return
    }

    next.ServeHTTP(w, r)
  })
}

```

```tsx
// Keycloak Login Flow
const keycloak = new Keycloak({
  url: process.env.KEYCLOAK_URL,
  realm: 'dws-realm',
  clientId: 'dws-frontend'
});

keycloak.init({ onLoad: 'login-required' }).then(() => {
  // User authenticated, JWT available
  const token = keycloak.token;
});

```
