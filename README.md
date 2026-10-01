# Passia Backend

> **Production-oriented Java backend architecture case study focused on secure APIs, domain boundaries, cloud deployment, identity, authorization and evolutionary architecture.**

**Passia** is a portfolio project for a pet walking and safe pet transportation platform.

The backend was designed not only as an API implementation, but as an architecture case study demonstrating how product requirements, security, domain modeling, data consistency, cloud deployment and technical trade-offs can be translated into an evolvable software solution.

---

## Executive Summary

The Passia backend is currently implemented as a **modular monolith** using Java and Spring Boot.

The architecture intentionally prioritizes:

- explicit domain boundaries;
- secure user-delegated authentication;
- server-side resource ownership;
- stable REST contracts;
- relational consistency;
- testability;
- stateless application design;
- cloud-ready containerization;
- architecture documentation;
- incremental evolution based on real operational evidence.

Current core stack:

```text
Java 21
Spring Boot 4
Spring Security
Spring Modulith
OAuth 2.0 / OpenID Connect
Auth0
JWT
REST APIs
OpenAPI 3.1
PostgreSQL 18
JPA / Hibernate
Flyway
Docker
Maven
JUnit
Testcontainers
Spring Boot Actuator
Render Cloud
```

The project deliberately avoids introducing distributed infrastructure before the MVP requires it.

Redis, Kafka, Kubernetes, PostGIS and microservice extraction are therefore treated as **architectural options with explicit adoption criteria**, rather than mandatory technologies.

---

# Architecture

## Architecture Goals

The main architectural goals are:

1. keep business rules independent from infrastructure details;
2. maintain explicit boundaries between product domains;
3. protect Tutor/Pet ownership on the server side;
4. delegate authentication to a specialized Identity Provider;
5. maintain the application stateless between requests;
6. use PostgreSQL as the durable source of truth;
7. keep external providers behind architectural boundaries;
8. support future tracking and realtime capabilities without prematurely distributing the system;
9. make architectural decisions traceable through ADRs;
10. evolve infrastructure based on metrics instead of assumptions.

---

## System Context

The application is composed of a Flutter mobile client, an external Identity Provider, the Passia backend and a relational database.

### Architecture Diagram

> **Diagram placeholder — System Context / C4 Level 1**
>
> Export your diagram from Notion/Figma/Draw.io and replace this block with:
>
> `![Passia System Context](docs/images/01-system-context.png)`

---

## High-Level Architecture

```text
                       ┌─────────────────────┐
                       │        Tutor        │
                       └──────────┬──────────┘
                                  │
                                  ▼
                       ┌─────────────────────┐
                       │    Flutter Mobile   │
                       │      Passia App     │
                       └───────┬──────┬──────┘
                               │      │
                  OIDC + PKCE  │      │ HTTPS / JWT
                               │      │
                               ▼      ▼
                        ┌──────────┐  ┌───────────────────────────┐
                        │  Auth0   │  │      Passia Backend       │
                        │   IdP    │  │                           │
                        └────┬─────┘  │ Java 21 / Spring Boot     │
                             │        │ Spring Security           │
                             │ JWKS   │ Spring Modulith           │
                             └───────▶│ REST / OpenAPI            │
                                      │                           │
                                      │ Identity                  │
                                      │ Tutor / Pet               │
                                      │ Walk                      │
                                      │ Route                     │
                                      │ Tracking                  │
                                      │ Notification              │
                                      └─────────────┬─────────────┘
                                                    │
                                                    ▼
                                      ┌───────────────────────────┐
                                      │       PostgreSQL 18       │
                                      │      JPA / Hibernate      │
                                      │          Flyway           │
                                      └───────────────────────────┘
```

---

# Backend Architecture

## Architectural Style

Passia uses a **modular monolith**.

This is a deliberate architectural decision.

The application currently does not have the traffic volume, independent team ownership, availability requirements or different scaling profiles that would justify the operational cost of microservices.

Instead, business capabilities are separated into explicit modules while remaining part of a single deployable application.

```text
br.com.passia

├── identity
│   ├── application
│   ├── domain
│   ├── infrastructure
│   └── web
│
├── tutorpet
│   ├── application
│   ├── domain
│   ├── infrastructure
│   └── web
│
├── walk
│   ├── application
│   ├── domain
│   ├── infrastructure
│   └── web
│
├── route
│   ├── application
│   ├── domain
│   └── infrastructure
│
├── tracking
│   ├── application
│   ├── domain
│   ├── infrastructure
│   └── web
│
├── notification
│   ├── application
│   └── infrastructure
│
└── shared
    ├── web
    ├── observability
    └── security
```

The objective is to preserve **domain boundaries without paying the distributed-systems cost prematurely**.

---

## Module Diagram

> **Diagram placeholder — Backend Modules / C4 Component Diagram**
>
> Suggested file:
>
> `docs/images/02-backend-modules.png`

---

# Domain Boundaries

## Identity

Responsible for:

- authentication boundary;
- JWT validation;
- authenticated actor resolution;
- integration between Auth0 identity and Passia domain identity;
- cross-cutting authorization concerns.

Identity infrastructure does not own the Tutor business entity.

---

## Tutor / Pet

Responsible for:

- Tutor profile;
- Pet lifecycle;
- Tutor → Pet ownership;
- authorization policies related to pets.

Ownership is always validated server-side.

The backend never trusts a `tutorId` received from the client as proof of identity.

---

## Walk

The main business domain.

Responsible for the lifecycle of a walk, its invariants and business transitions.

The architecture is designed so controllers, persistence entities and provider DTOs do not become the domain model.

---

## Route

Responsible for concepts such as:

```text
GeoPoint
Address
Waypoint
RoutePlan
RouteEstimate
```

External map/provider representations should be translated by adapters before entering the application/domain layers.

---

## Tracking

Designed as an independent module because tracking has a different load and storage profile from transactional Walk operations.

The architectural direction supports:

- offline collection;
- batched synchronization;
- sequence-based ordering;
- deduplication;
- delayed delivery;
- future independent scalability.

Tracking is currently kept inside the modular monolith.

Extraction into a dedicated service is considered only if real traffic demonstrates the need.

---

## Notification

Responsible for reacting to application/domain events and integrating with notification providers.

Notification does not directly control the Walk lifecycle.

---

# Authentication & Identity Architecture

Passia uses **OAuth 2.0 / OpenID Connect with Auth0**.

The Flutter application is treated as a **public native client**.

Authentication uses:

```text
Authorization Code Flow
+
PKCE (S256)
```

No Client Secret is embedded in the mobile application.

---

## Authentication Flow

> **Diagram placeholder — OIDC / PKCE Authentication Sequence**
>
> Suggested file:
>
> `docs/images/03-authentication-sequence.png`

```text
Tutor
  │
  ▼
Flutter App
  │
  │ Authorization Code + PKCE
  ▼
Auth0
  │
  │ Access Token
  ▼
Flutter App
  │
  │ Authorization: Bearer <JWT>
  ▼
Passia Backend
  │
  ├── validate signature
  ├── validate issuer
  ├── validate audience
  ├── validate expiration
  └── resolve Passia Tutor identity
```

---

# Identity Mapping

External authentication identity and business identity are intentionally separated.

The Auth0 user contains a controlled domain association in:

```text
app_metadata.tutor_id
```

A Post Login Action publishes a namespaced claim:

```text
https://passia.com.br/tutor_id
```

The backend accepts **only this namespaced claim** as the authenticated Tutor domain identifier.

The value must be a valid UUID.

There is intentionally no fallback to:

```text
sub
tutor_id
request body tutorId
query parameter tutorId
```

This keeps the identity boundary explicit and prevents the mobile client from impersonating another domain user.

---

# Authorization & Ownership

Authentication and authorization are treated as separate concerns.

A valid JWT proves the identity of an authenticated actor.

It does **not** automatically grant access to every resource.

Passia performs ownership validation server-side.

Validated behavior in the development environment:

| Scenario | Result |
|---|---|
| Request without authentication | `401 Unauthorized` |
| Authenticated Tutor accesses own Pet | `200 OK` |
| Authenticated Tutor accesses another Tutor's Pet | `403 Forbidden` |
| Ownership violation | `ACCESS_DENIED` |

This provides a clear security boundary between:

```text
Authentication
      ↓
Identity resolution
      ↓
Authorization
      ↓
Resource ownership
```

---

# Data Architecture

PostgreSQL is the primary transactional datastore.

Current stack:

```text
PostgreSQL 18
JPA
Hibernate
Flyway
UUID domain identifiers
```

PostgreSQL remains the **source of truth for durable application state**.

Flyway provides versioned database migrations and prevents schema evolution from depending on manual environment changes.

---

## Database Architecture Diagram

> **Diagram placeholder — Data Model / ER Diagram**
>
> Suggested file:
>
> `docs/images/04-data-architecture.png`

---

# Why PostgreSQL?

The MVP requires:

- transactional consistency;
- relational ownership;
- predictable constraints;
- mature indexing;
- ACID transactions;
- strong tooling;
- straightforward operational behavior.

A relational model currently provides more value than introducing multiple specialized databases.

NoSQL storage may be evaluated later if a domain develops access patterns that justify it.

---

# Why Redis Is Not Used

Redis is **intentionally absent from the current MVP architecture**.

This is an architectural decision rather than an unfinished infrastructure task.

At the current stage there is no evidence of:

```text
high request volume
database saturation
repeated expensive reads requiring cache
distributed session state
distributed locks
distributed rate limiting
shared ephemeral state
low-latency cross-instance coordination
```

Adding Redis now would introduce:

- another runtime dependency;
- cache invalidation strategies;
- additional failure modes;
- monitoring requirements;
- security configuration;
- operational cost;
- duplicated state.

without solving a demonstrated problem.

PostgreSQL therefore remains the source of truth.

Redis will be reconsidered if production metrics demonstrate a real need.

### Redis adoption triggers

Possible future triggers include:

```text
measurable database read bottlenecks
high cache hit potential
distributed rate limiting
shared ephemeral state
cross-instance coordination
distributed locking requirements
realtime presence
high-volume read workloads
```

This decision is documented as an Architecture Decision Record.

---

# Why a Modular Monolith Instead of Microservices?

Microservices are not treated as an architecture maturity badge.

They introduce real costs:

```text
network failure modes
distributed tracing
service discovery
deployment coordination
eventual consistency
distributed transactions
schema ownership
operational overhead
observability complexity
```

The current Passia product does not yet present a measurable reason to pay those costs.

Instead:

```text
Modular Monolith
      │
      ├── strong internal boundaries
      ├── independent domain concepts
      ├── ports/adapters
      ├── explicit dependencies
      └── single deployment
```

A module should be extracted only when there is a concrete driver such as:

- independent scaling;
- different availability requirements;
- independent team ownership;
- regulatory isolation;
- incompatible deployment lifecycle;
- specialized storage;
- materially different workload profile.

Tracking is currently the strongest candidate for future independent scaling because its workload may eventually differ significantly from transactional operations.

---

# Event-Driven Architecture

Passia is designed to allow domain/application events without requiring an external broker from day one.

For MVP workloads, events may remain inside the application boundary.

Examples of future events include:

```text
WalkStarted
WalkCompleted
WalkCancelled
WalkAborted
TrackingSessionStarted
TrackingSessionCompleted
```

An external event broker such as **Apache Kafka** should only be introduced when there is a requirement for:

- durable asynchronous processing;
- independent consumers;
- replay;
- high-volume event streaming;
- cross-service communication;
- asynchronous fan-out;
- stronger decoupling between independently deployed workloads.

Until then, a Kafka cluster would add operational complexity without solving an existing requirement.

---

# Cloud-Native Evolution

The backend is packaged as a container and designed to remain stateless between HTTP requests.

This means the architecture can evolve toward platforms such as:

```text
Kubernetes
OpenShift
AWS ECS / EKS
Azure Kubernetes Service
Google Kubernetes Engine
```

without requiring the domain model to change.

The current MVP does **not** require Kubernetes.

The development environment uses a managed container runtime because it provides enough capability for the current traffic and operational requirements.

This keeps infrastructure proportional to product maturity.

---

# Container & Deployment Architecture

> **Diagram placeholder — Deployment / Container Architecture**
>
> Suggested file:
>
> `docs/images/05-deployment-architecture.png`

Current development topology:

```text
Git Repository
      │
      ▼
Container Build
      │
      ▼
Managed Runtime
      │
      ├──────────────▶ Auth0 / JWKS
      │
      ▼
Spring Boot Backend
      │
      ▼
Managed PostgreSQL
```

The backend and database run in the same cloud region in the development environment.

Secrets are provided through runtime configuration and are never committed to source control.

---

# API Design

Passia exposes REST APIs with versioned routes.

Example:

```text
/api/v1/...
```

API concerns include:

- HTTP semantics;
- stable resource contracts;
- validation;
- standardized error responses;
- authentication;
- ownership authorization;
- versioning;
- OpenAPI documentation.

The project uses **OpenAPI 3.1** as the machine-readable API contract.

---

# Error Model

External errors should use stable application codes rather than expose implementation details.

Conceptually:

```json
{
  "code": "ACCESS_DENIED",
  "message": "You are not allowed to access this resource.",
  "traceId": "...",
  "timestamp": "..."
}
```

Clients should not receive:

```text
SQL errors
database structure
stack traces
internal exception classes
credentials
JWT contents
```

---

# Non-Functional Requirements

Architecture is evaluated not only by feature delivery but by non-functional requirements.

| Concern | Current approach |
|---|---|
| Security | OAuth2/OIDC, JWT validation, ownership |
| Maintainability | modular boundaries and explicit dependencies |
| Reliability | transactional persistence and controlled errors |
| Scalability | stateless backend and measurable evolution |
| Data consistency | PostgreSQL transactions and constraints |
| Deployability | containerized application |
| Observability | Actuator, readiness and structured logging |
| Testability | unit + integration + Testcontainers |
| Privacy | minimization of sensitive location logging |
| Cost | infrastructure proportional to MVP traffic |
| Evolvability | ADRs and measurable extraction criteria |

---

# Observability

The backend exposes Spring Boot Actuator health capabilities.

Development readiness endpoint:

```text
/actuator/health/readiness
```

The architectural observability model includes:

```text
structured logs
request correlation
request duration
HTTP status distribution
database connection health
readiness
domain error metrics
future tracing
```

Sensitive values should never be logged.

That includes:

```text
access tokens
refresh tokens
credentials
passwords
full tracking payloads
unnecessary precise coordinates
```

Distributed tracing infrastructure should be introduced when distributed execution actually exists.

---

# Reliability & Resilience

The architecture favors simple and deterministic consistency mechanisms before distributed resilience tooling.

Relevant patterns include:

```text
database transactions
ownership validation
optimistic concurrency where required
idempotent commands where required
timeouts for external integrations
controlled retries
stable error taxonomy
```

Circuit breakers, distributed locks and advanced retry policies should be introduced only around integrations whose failure characteristics justify them.

---

# Testing Strategy

The project uses multiple testing levels.

```text
Unit Tests
    ↓
Application Tests
    ↓
Persistence Integration Tests
    ↓
API / Security Integration Tests
```

Technologies include:

```text
JUnit
Spring Test
Testcontainers
PostgreSQL
Maven Verify
```

At the security architecture checkpoint:

```text
24 backend tests
0 failures
0 errors
```

The database integration tests use real PostgreSQL through Testcontainers instead of relying exclusively on an in-memory substitute.

---

## Security Regression Scenarios

Three scenarios form an important minimum security regression set:

```text
No token
    → 401

Valid token + own resource
    → 200

Valid token + foreign resource
    → 403
```

This ensures future refactoring cannot accidentally collapse authentication and authorization into the same concern.

---

# Running the Test Suite

Requirements:

```text
Java 21
Docker
```

Run:

```bash
./mvnw clean verify
```

Integration tests may start PostgreSQL containers through Testcontainers.

---

# Local Runtime Configuration

No production secrets are stored in the repository.

Expected configuration follows this model:

```text
DATABASE_URL=jdbc:postgresql://<host>:5432/<database>
DATABASE_USERNAME=<username>
DATABASE_PASSWORD=<secret>

JWT_ISSUER_URI=https://<your-auth0-tenant>/
JWT_AUDIENCE=<your-api-audience>
```

Do not commit:

```text
database passwords
Auth0 credentials
access tokens
refresh tokens
private keys
real .env files
production connection strings
```

A public portfolio repository should contain only safe examples such as:

```text
.env.example
application-local.example.yml
```

---

# Architecture Decision Records

Architecture decisions are documented under:

```text
docs/architecture/adr/
```

ADRs explain:

```text
Context
Decision Drivers
Alternatives
Decision
Consequences
Risks
Revisit Criteria
```

Relevant decisions include:

### Authentication Boundary

Auth0 handles authentication.

Passia remains responsible for translating authenticated identity into domain identity and enforcing business authorization.

### Modular Monolith

A single deployment with strong module boundaries is currently preferable to distributed services.

### PostgreSQL as Source of Truth

Durable application state remains transactional and relational.

### Redis Deferred

Redis will only be introduced after measurable evidence demonstrates a requirement for caching or shared ephemeral state.

### External Broker Deferred

Kafka or another event broker is considered when durable asynchronous communication or event streaming becomes a real requirement.

### PostGIS Deferred

Basic latitude/longitude storage is sufficient until advanced spatial queries justify specialized geospatial capabilities.

---

# Architecture Decision Diagram

> **Diagram placeholder — Architecture Evolution / Decision Map**
>
> Suggested file:
>
> `docs/images/06-architecture-evolution.png`

Suggested visual:

```text
MVP
 │
 ▼
Modular Monolith
REST
PostgreSQL
OIDC/JWT
Docker
 │
 ├────────────── traffic growth ─────────────▶ horizontal scale
 │
 ├────────────── read bottleneck ────────────▶ evaluate Redis
 │
 ├────────────── durable async events ───────▶ evaluate Kafka
 │
 ├────────────── spatial queries ────────────▶ evaluate PostGIS
 │
 ├────────────── complex orchestration ──────▶ evaluate Kubernetes
 │
 └────────────── independent workloads ──────▶ evaluate service extraction
```

---

# Architecture vs Technology Checklist

This project intentionally distinguishes **architectural capability** from simply adding technologies to a stack.

| Architecture capability | Passia evidence | Status |
|---|---|---|
| Java backend engineering | Java 21 + Spring Boot | Implemented |
| REST API architecture | Versioned API + OpenAPI | Implemented |
| Authentication | OAuth2/OIDC + Auth0 | Implemented |
| API security | Spring Security + JWT | Implemented |
| Authorization | server-side ownership | Implemented |
| Domain boundaries | modular monolith | Implemented |
| Relational persistence | PostgreSQL + Hibernate | Implemented |
| Schema evolution | Flyway | Implemented |
| Containerization | Docker | Implemented |
| Cloud deployment | managed runtime + managed DB | Implemented |
| Integration testing | Testcontainers | Implemented |
| Health monitoring | Spring Boot Actuator | Implemented |
| Architecture documentation | ADRs + diagrams | Implemented |
| Distributed cache | Redis | Intentionally deferred |
| Event streaming | Kafka | Intentionally deferred |
| Container orchestration | Kubernetes | Not required for MVP |
| Advanced geospatial DB | PostGIS | Intentionally deferred |
| Microservice extraction | evolutionary option | Not currently justified |

---

# Mapping Business Constraints to Architecture

A Solution Architect should be able to explain **why** a technology exists, not only how to configure it.

| Product / Engineering Constraint | Architecture Decision |
|---|---|
| MVP with limited traffic | modular monolith |
| Need secure mobile login | Auth0 + OIDC + PKCE |
| Client cannot be trusted with identity | server-side domain identity resolution |
| Tutor resources must remain isolated | ownership authorization |
| Transactional domain data | PostgreSQL |
| Reproducible schema evolution | Flyway |
| Need integration confidence | Testcontainers |
| Future offline tracking | dedicated Tracking boundary |
| Low current read volume | no Redis yet |
| No durable event fan-out requirement | no Kafka yet |
| No advanced spatial query requirement | no PostGIS yet |
| Low infrastructure complexity requirement | managed container runtime |
| Future scalability | stateless application design |

---

# Solution Architecture Skills Demonstrated

The project was designed to demonstrate competencies commonly expected from senior Java architecture roles:

**Software Architecture**

```text
Architecture trade-off analysis
Modular architecture
DDD-oriented boundaries
Ports & Adapters concepts
Clean Architecture principles
System design
Architecture evolution
Architecture Decision Records
```

**Java Platform**

```text
Java 21
Spring Boot
Spring Security
Spring Data / JPA
Hibernate
Spring Modulith
Maven
```

**Integration & APIs**

```text
REST
OpenAPI
OAuth 2.0
OpenID Connect
JWT
External provider boundaries
Error contracts
API versioning
```

**Data**

```text
PostgreSQL
Relational modeling
Transactions
Schema migration
Flyway
Database integration testing
```

**Cloud & DevOps**

```text
Docker
Managed cloud runtime
Externalized configuration
Secrets management principles
Health/readiness probes
CI/CD-ready build
Stateless deployment model
```

**Quality & Reliability**

```text
Unit testing
Integration testing
Testcontainers
Security regression testing
Observability design
Resilience patterns
Error handling
```

**Distributed Systems**

```text
Service extraction criteria
Event-driven architecture boundaries
Idempotency concepts
Asynchronous processing considerations
Distributed cache trade-offs
Event broker trade-offs
Consistency considerations
```

The project intentionally does not claim that every distributed technology must be deployed to demonstrate understanding of its architectural role.

---

# Architecture for Future Scale

The current architecture provides an evolutionary path rather than assuming future scale.

```text
                    ┌──────────────────┐
                    │ Current MVP      │
                    │ Modular Monolith │
                    │ PostgreSQL       │
                    └────────┬─────────┘
                             │
                     operational metrics
                             │
          ┌──────────────────┼───────────────────┐
          │                  │                   │
          ▼                  ▼                   ▼
   Horizontal Scale     Tracking Scale      Read Bottleneck
          │                  │                   │
          ▼                  ▼                   ▼
 Multiple Instances    Partition/Service      Evaluate Cache
                              │                   │
                              ▼                   ▼
                         Kafka / Broker         Redis
                        only if required     only if required
```

The architecture should evolve because of **measured system behavior**, not because a technology is fashionable.

---

# Repository Structure

A public portfolio version can follow this structure:

```text
.
├── docs
│   ├── architecture
│   │   ├── adr
│   │   ├── diagrams
│   │   └── security
│   │
│   └── api
│       └── openapi.yaml
│
├── src
│   ├── main
│   │   └── java
│   │       └── br
│   │           └── com
│   │               └── passia
│   │
│   └── test
│
├── Dockerfile
├── compose.yaml
├── pom.xml
├── mvnw
├── mvnw.cmd
└── README.md
```

---

# Diagram Gallery

Use this section after exporting the diagrams.

## System Context

<!--
![System Context](docs/images/01-system-context.png)
-->

> Image placeholder.

---

## Backend Modular Architecture

<!--
![Backend Modular Architecture](docs/images/02-backend-modules.png)
-->

> Image placeholder.

---

## Authentication Sequence

<!--
![Authentication Sequence](docs/images/03-authentication-sequence.png)
-->

> Image placeholder.

---

## Data Architecture

<!--
![Data Architecture](docs/images/04-data-architecture.png)
-->

> Image placeholder.

---

## Deployment Architecture

<!--
![Deployment Architecture](docs/images/05-deployment-architecture.png)
-->

> Image placeholder.

---

## Architecture Evolution

<!--
![Architecture Evolution](docs/images/06-architecture-evolution.png)
-->

> Image placeholder.

---

# Engineering Principles

```text
Business requirements before infrastructure
Security by design
Explicit domain ownership
Measure before scaling
Prefer simple systems until complexity is justified
Use architecture boundaries before network boundaries
Keep secrets outside source control
Design for evolution instead of predicting the future
Document important technical decisions
```

---

# Current Scope

Implemented baseline:

```text
Java/Spring backend
Modular architecture
Tutor/Pet vertical slice
REST API
PostgreSQL persistence
Flyway migrations
Auth0 integration
OAuth2/OIDC
JWT validation
Domain identity mapping
Server-side ownership
Docker
Cloud development environment
Health/readiness
Automated tests
Architecture documentation
```

Still evolving:

```text
Walk lifecycle implementation
Route integration
Tracking
Offline synchronization
Realtime capabilities
Analytics
Production-grade CI/CD
Infrastructure as Code
Performance baselines
Production observability stack
```

---

# Security Notes

This repository is intended to be safe for public portfolio use.

It must not contain:

```text
production credentials
database passwords
access tokens
refresh tokens
Auth0 secrets
private keys
real user data
private database URLs containing credentials
cloud secrets
```

Configuration examples should always use placeholders.

The public repository should also be created from a **sanitized Git history** rather than exposing the private development repository history.

---

# Portfolio Purpose

Passia is a portfolio project designed to demonstrate not only software implementation, but the reasoning behind software architecture decisions.

The main objective is to show the ability to move between:

```text
Product requirement
        ↓
Business rule
        ↓
Architecture decision
        ↓
Technical design
        ↓
Implementation
        ↓
Testing
        ↓
Deployment
        ↓
Operational feedback
        ↓
Architecture evolution
```

The architecture is intentionally evolutionary.

Technologies are introduced when they solve a measurable problem, not simply to increase the size of the technology stack.

---

# Key Takeaway

> **Good architecture is not the number of technologies in a diagram.**
>
> It is the ability to make explicit trade-offs, protect important system qualities and evolve the solution when evidence shows that the current architecture is no longer enough.

---

## Author

**Raquel Silva**

Software Architecture • Solution Architecture • Technical Leadership • Java • Cloud • Mobile

---

## Project Status

**Development paused — Portfolio / MVP**

Passia is currently paused due to a temporary health-related leave by its creator.

The repository is preserved as an architecture and engineering portfolio case study, documenting the technical decisions, implemented components, tests, security model and evolution strategy developed up to the current project checkpoint.

### Current availability

This repository should **not be considered a currently operational production application**.

The architecture and implemented components remain available for technical review, but the complete Passia ecosystem may not be running continuously and external development services may be unavailable.

Development may resume when circumstances allow.

**Last active development checkpoint:** Jan 2026.
