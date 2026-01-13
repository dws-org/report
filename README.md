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

# Database Model

## Overview

Schema-based multi-tenancy on a single PostgreSQL instance:

- `public` – Event Management
- `tickets` – Ticket Management

---

## public.Event

| Column | Type | Notes |
| --- | --- | --- |
| id | TEXT | PK |
| name | TEXT |  |
| description | TEXT |  |
| date | TIMESTAMP |  |
| location | TEXT |  |
| capacity | INTEGER |  |
| organizerId | TEXT | FK → Organizer.id |
| status | TEXT | planned | ongoing | completed | cancelled |
| createdAt | TIMESTAMP |  |
| updatedAt | TIMESTAMP |  |

---

## public.Organizer

| Column | Type | Notes |
| --- | --- | --- |
| id | TEXT | PK |
| name | TEXT |  |
| email | TEXT |  |
| phone | TEXT |  |
| address | TEXT |  |
| createdAt | TIMESTAMP |  |
| updatedAt | TIMESTAMP |  |

---

## tickets.Tickets

| Column | Type | Notes |
| --- | --- | --- |
| id | TEXT | PK |
| userId | TEXT |  |
| eventId | TEXT | FK → public.Event.id |
| quantity | INTEGER |  |
| totalPrice | NUMERIC(65,30) |  |
| status | TEXT | pending | confirmed | cancelled | used |
| createdAt | TIMESTAMP |  |
| updatedAt | TIMESTAMP |  |

---

## Relationships

- Organizer 1 — N Event
- Event 1 — N Tickets

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

### SQL Injection Protection

**Implementation:**
Our project is protected against SQL injection attacks through the use of **Prisma ORM**. Prisma automatically uses parameterized queries (prepared statements) which separate SQL code from user input data. This is the industry-standard defense against SQL injection.

**How it works:**

- All database queries are constructed through Prisma's type-safe API
- User input is never directly concatenated into SQL queries
- Parameters are automatically escaped and validated
- Example of safe query:

```go
// User input is safely parameterized
event, err := prisma.Event.FindUnique(
  db.Event.ID.Equals(userProvidedID),
).Exec(ctx)

```

**Why we're protected:**

- Prisma translates our Go code to SQL with proper parameterization
- Even if `userProvidedID` contains malicious SQL, it's treated as a data value, not executable code
- No string concatenation or user input directly in queries

---

### XSS (Cross-Site Scripting) Protection

**Implementation:**
Our project is protected against XSS attacks through **React's built-in security features** in the Next.js frontend. React automatically escapes all user-generated content by default.

**How it works:**

- React treats all content as text by default, not HTML
- Special characters (`<`, `>`, `&`, etc.) are automatically HTML-escaped
- User input rendered in JSX is safe from script injection
- Example of safe rendering:

```jsx
// Even if userComment contains "<script>alert('xss')</script>",
// React will display it as plain text, not execute it
<div>{userComment}</div>
// Renders as: &lt;script&gt;alert('xss')&lt;/script&gt;

```

**Additional protections:**

- We avoid using `dangerouslySetInnerHTML` with unsanitized content
- Input validation on both frontend and backend
- Content Security Policy (CSP) headers could be added for additional defense

**Why we're protected:**

- React's default behavior is to escape content
- Users cannot inject executable scripts through normal data fields
- Even if malicious input reaches the frontend, it's rendered as harmless text

## Certificate Management and Automatic Renewal

Our project uses **cert-manager** with **Let's Encrypt** to automatically manage SSL/TLS certificates for all external communications. This ensures that HTTPS certificates are provisioned and renewed without manual intervention.

### Implementation

The cluster runs cert-manager to manage certificates through Let's Encrypt's ACME protocol. We have configured a production ClusterIssuer (`letsencrypt-prod`) that validates domain ownership using HTTP-01 challenges with our Traefik Ingress Controller. The ClusterIssuer is registered  and maintains an account with Let's Encrypt.

Currently, the following services are protected with valid TLS certificates: ArgoCD, Frontend, Keycloak, Grafana, and Prometheus. All certificates show Ready status and are actively serving HTTPS traffic.

### Automatic Renewal Process

Let's Encrypt certificates are valid for 90 days. cert-manager continuously monitors certificate expiration and automatically requests new certificates 30 days before expiration. This happens entirely in the background without any service downtime. When a certificate is renewed, it's automatically stored in the corresponding Kubernetes Secret and the Ingress controller picks up the new certificate transparently.

# GDPR Compliance and Data Privacy Measures

### Data Collection & Storage

Our application follows the principle of data minimization by collecting only essential information needed for event management functionality. User personal data (email, name, authentication details) is managed exclusively by **Keycloak**, our dedicated identity provider. Our application database only stores event-related information (event details, organizer IDs) and does not replicate personal user data.

This separation of concerns ensures that sensitive personal information is handled by a specialized system (Keycloak) rather than distributed across multiple services, reducing the overall data handling footprint.

### Authentication & Access Control

All API endpoints require authentication through Keycloak, ensuring that only registered users can access the system. The application implements role-based access control where only users with the "Organiser" role can create events. Users cannot access or view other users' personal profiles—the system only exposes event data, not user directories or contact information.

### User Rights

**Right to be Forgotten:** Keycloak has "User-managed access" enabled, which allows users to manage and delete their own accounts directly through the Keycloak user portal. Users can export their account data and request account deletion without administrator intervention.

### Data Protection in Transit

All communications between client and server are encrypted using HTTPS/TLS. Certificate management is automated through Let's Encrypt and cert-manager, ensuring encryption certificates are continuously valid and renewed automatically.

### Current Limitations & Future Improvements

The following could be added in future versions:

- Explicit consent banner for data processing
- Formal privacy policy documentation
- Application-level data export functionality (currently only available in Keycloak)
