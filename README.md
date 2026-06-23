# HR Leave Management  

A multi-tenant web application for managing employee leave in Spanish companies. Supports 30+ leave types across legal mandates and collective bargaining agreements (*convenios*), configurable approval workflows, DMN/BPMN rule engine, SSO authentication, and GDPR-compliant handling of medical data.  

---

## Tech Stack  

| Layer | Technology |
|-------|-----------|
| **Backend** | Java 21 · Quarkus 3.36 · Hibernate/Panache · Kogito (DMN + BPMN) · SmallRye JWT |
| **Frontend** | Vue 3.5 · TypeScript 6 · Tailwind CSS v4 · Pinia · Vue Router |
| **Database** | PostgreSQL 16 |
| **Cache** | Valkey 8 |
| **Message Broker** | RabbitMQ 3.13 |
| **Reverse Proxy** | Nginx (TLS termination, rate limiting) |
| **Auth** | Google SSO · Zoho SSO · Local (OIDC + httpOnly cookies + CSRF) |

---

## Quick Start (Development)  

### Prerequisites  

- **Java 21+** with Maven 3.9+  
- **Node.js 22+** with npm  
- **Docker** with Compose v2 (for PostgreSQL, Valkey, RabbitMQ)  

### 1. Start Infrastructure  

```bash
cd hr-leave-management-api
docker compose -f src/main/docker/compose-dev.yml up -d
```

This starts PostgreSQL, Valkey, and RabbitMQ with default dev credentials.  

### 2. Start the API  

```bash
cd hr-leave-management-api
./mvnw quarkus:dev
```

API runs at `http://localhost:8080`. Swagger UI at `http://localhost:8080/q/swagger-ui`.  

### 3. Start the Frontend  

```bash
cd hr-leave-management-front
npm install
npm run dev
```

Frontend runs at `http://localhost:3000`.  

### 4. First Login  

The first user to authenticate via SSO (Google or Zoho) is auto-assigned the `System Administrator` role. For local development, create a test user via the database.  

---

## Project Structure  

```
hr-leave-management/
│
├── hr-leave-management-api/          # Quarkus backend (Java 21)
│   ├── src/main/java/es/mylair/
│   │   ├── boundary/                 # JAX-RS resources (HTTP endpoints)
│   │   ├── dto/                      # Request/response DTOs
│   │   ├── service/                  # Business logic
│   │   ├── entity/                   # Panache entities
│   │   ├── repository/               # Panache + native SQL repositories
│   │   ├── security/                 # Auth filters, CSRF, permission evaluator
│   │   ├── event/                    # RabbitMQ publishers & consumers
│   │   └── cache/                    # Valkey cache configuration
│   ├── src/main/resources/
│   │   ├── dmn/                      # Kogito DMN decision tables
│   │   ├── bpmn/                     # Kogito BPMN approval workflows
│   │   └── db/migration/             # Flyway SQL migrations
│   └── pom.xml
│
├── hr-leave-management-front/        # Vue 3 frontend (TypeScript)
│   ├── src/
│   │   ├── api/                      # Fetch client + API modules
│   │   ├── components/common/        # Reusable UI primitives
│   │   ├── composables/              # useAuth, useCan, useTheme, useLocale
│   │   ├── layouts/                  # AuthLayout, DefaultLayout, AdminLayout
│   │   ├── locales/                  # es.json, en.json (vue-i18n)
│   │   ├── router/                   # Routes + navigation guards
│   │   ├── stores/                   # Pinia stores (auth, leaves, approvals, ...)
│   │   ├── types/                    # TypeScript interfaces
│   │   ├── utils/                    # cn(), formatters, validators
│   │   └── views/                    # Route-level page components
│   └── package.json
│
└── docs/                             # Documentation
    ├── SPEC.md                       # Functional & technical specification
    ├── UX-DESIGN.md                  # UX/UI design system + screen designs
    ├── API-DESIGN.md                 # REST API contract (~70 endpoints)
    ├── API-IMPL-SPEC.md              # API implementation guide
    ├── OPERATIONS.md                 # Operations, backup/restore, runbooks
    └── README.md                     # This file
```

---

## Documentation Index  

| Document | Audience | Content |
|----------|----------|---------|
| [SPEC.md](./docs/SPEC.md) | Developers, Architects | Complete functional and technical specification: domain model, data model, auth flows, all endpoints, RBAC, GDPR, reporting, deployment |
| [UX-DESIGN.md](./docs/UX-DESIGN.md) | Frontend Developers | Design system (OKLCH colors, typography, spacing), 10 common components, 11 screen designs with ASCII wireframes, dark mode, responsive strategy, accessibility |
| [API-DESIGN.md](./docs/API-DESIGN.md) | Backend Developers | REST API contract: conventions, pagination, filtering, ETags, rate limiting, ~70 endpoints with request/response JSON examples, SSE notifications |
| [API-IMPL-SPEC.md](./docs/API-IMPL-SPEC.md) | Backend Developers | Implementation details: layered architecture, caching with Valkey, multi-tenancy, DMN/BPMN integration, outbox pattern, testing strategy, CI/CD |
| [OPERATIONS.md](./docs/OPERATIONS.md) | System Administrators | Production deployment (Docker Compose), backup/restore with GPG encryption, monitoring, key rotation, disaster recovery, runbooks, security hardening |

---

## Key Features  

- **30+ leave types** — vacation, sick leave, birth/childcare, pregnancy-related, paid leave (15+ subcategories), unpaid leave, parental leave, family care, excedencia, reduced hours, public holidays, bridge days  
- **Configurable rules engine** — DMN decision tables for eligibility/duration/entitlement. BPMN for approval workflows. Policies versioned per collective agreement. No hardcoded leave logic.  
- **11 user roles** — Employee, Line Manager, HR Administrator, HR Medical Specialist, HR Policy Administrator, Payroll Officer, Legal Officer, Department Manager, Leave Administrator, System Administrator, Auditor — with 60+ fine-grained permissions  
- **Cookie-based auth** — JWT in httpOnly cookies, CSRF double-submit pattern, SSO via Google and Zoho, refresh token rotation  
- **GDPR compliant** — medical data restricted to specialized roles, every access logged, encrypted at rest, configurable retention with automated anonymization/deletion  
- **Multi-tenant** — discriminator column per tenant, Hibernate filter enforcement, Valkey cache key scoping  
- **Event-driven** — RabbitMQ for notifications, audit logging, balance updates. Outbox pattern for reliable delivery.  
- **SSE real-time notifications** — Server-Sent Events for instant approval alerts, fallback to polling  

---

## Security  

- **In transit**: TLS 1.3 (Nginx termination)  
- **At rest — DB**: LUKS-encrypted volume  
- **At rest — Files**: AES-256-GCM per document  
- **At rest — Backups**: GPG-encrypted before leaving the server  
- **Auth**: httpOnly cookies (no tokens in JavaScript), CSRF double-submit, RS256 JWT signing  
- **RBAC**: `@RolesAllowed` + custom permission evaluator, medical data access logged  
- **Rate limiting**: Nginx (per-IP) + Quarkus (per-user, per-tenant)  

See [SPEC.md §12](./docs/SPEC.md#12-security) and [OPERATIONS.md §11](./docs/OPERATIONS.md#11-security-hardening-checklist).  

---

## Development Workflow  

```bash
# Backend
cd hr-leave-management-api
./mvnw quarkus:dev              # Hot reload (Java changes)
./mvnw test                     # Unit tests
./mvnw verify -DskipITs=false   # Integration tests (requires Docker)

# Frontend
cd hr-leave-management-front
npm run dev                     # Vite dev server (HMR)
npm run test:unit               # Vitest unit tests
npm run test:e2e                # Playwright E2E tests
npm run type-check              # vue-tsc type checking
npm run lint                    # ESLint + Oxlint

# Full build
cd hr-leave-management-api && ./mvnw package
cd hr-leave-management-front && npm run build
```

### Pre-Commit Checklist  

- [ ] `./mvnw test` passes (backend)  
- [ ] `npm run test:unit` passes (frontend)  
- [ ] `npm run type-check` passes (frontend)  
- [ ] `npm run lint` passes (frontend)  
- [ ] New endpoints documented in [API-DESIGN.md](./docs/API-DESIGN.md)  
- [ ] New components follow [UX-DESIGN.md](./docs/UX-DESIGN.md) patterns  
- [ ] Database changes have a Flyway migration in `src/main/resources/db/migration/`  

---

## Testing  

| Layer | Tool | Coverage |
|-------|------|----------|
| Backend unit | JUnit 5 + Mockito | ≥ 80% |
| Backend integration | Quarkus `@QuarkusIntegrationTest` + Testcontainers | ≥ 70% endpoints |
| DMN decisions | Kogito test framework | 100% paths |
| Frontend unit | Vitest + Vue Test Utils + @pinia/testing | ≥ 80% |
| Frontend E2E | Playwright | All critical flows |
| Security | Custom + RestAssured | All roles × critical endpoints |

---

## License  

Proprietary. All rights reserved.
