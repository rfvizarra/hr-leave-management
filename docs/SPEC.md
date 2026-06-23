# HR Leave Management System — Technical Specification

> **Version**: 1.2  
> **Status**: Draft for implementation  
> **Audience**: Backend (Java/Quarkus) and Frontend (Vue 3) developers  
> **Related**: [UX/UI Design](./UX-DESIGN.md) · [API Design](./API-DESIGN.md)  

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Architecture](#2-architecture)
3. [Technology Stack](#3-technology-stack)
4. [Multi-tenancy](#4-multi-tenancy)
5. [Domain Model](#5-domain-model)
6. [Rule Engine — DMN + BPMN (Kogito)](#6-rule-engine--dmn--bpmn-kogito)
7. [Data Model — PostgreSQL](#7-data-model--postgresql)
8. [Authentication & Authorization](#8-authentication--authorization)
9. [REST API Specification](#9-rest-api-specification)
10. [Event-Driven Architecture — RabbitMQ](#10-event-driven-architecture--rabbitmq)
11. [Notification System](#11-notification-system)
12. [Security](#12-security)
13. [GDPR Compliance](#13-gdpr-compliance)
14. [Internationalization (i18n)](#14-internationalization-i18n)
15. [Frontend Architecture — Vue 3](#15-frontend-architecture--vue-3)
16. [Reporting](#16-reporting)
17. [Testing Strategy](#17-testing-strategy)
18. [Deployment & Infrastructure](#18-deployment--infrastructure)
19. [Future Phases / Backlog](#19-future-phases--backlog)

---

## 1. System Overview

A multi-tenant web application for managing employee leave in Spanish companies, supporting:

- **12+ leave types**: vacation, sick leave, birth/childcare, pregnancy-related, paid leave (15+ subcategories), unpaid leave, parental leave, family care leave, leave of absence (excedencia), reduced working hours, public holidays, and bridge days.
- **Configurable rules**: labor law + collective bargaining agreements (*convenios*) via a DMN/BPMN rule engine, not hardcoded logic.
- **Configurable approval workflows**: per leave type, with delegation, escalation, and SLA tracking.
- **GDPR-compliant** handling of medical/special-category data, with immutable audit trails.
- **SSO authentication** via Google and Zoho, plus local fallback.
- **11 user roles** with fine-grained permissions (RBAC).

---

## 2. Architecture

### 2.1 High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                        Nginx (HTTPS, rate limiting)           │
└──────────┬───────────────────────────────────────┬───────────┘
           │                                       │
           ▼                                       ▼
┌──────────────────────┐              ┌─────────────────────────┐
│   Vue 3 Frontend     │              │   Quarkus REST API      │
│   (SPA, port 443)    │──────────────▶   (port 8080)           │
│                      │   /api/v1/*   │                        │
└──────────────────────┘              │  ┌───────────────────┐  │
                                      │  │ Kogito DMN Engine │  │
                                      │  │ Kogito BPMN Engine│  │
                                      │  └───────────────────┘  │
                                      │  ┌───────────────────┐  │
                                      │  │ Hibernate/Panache │  │
                                      │  └─────────┬─────────┘  │
                                      └────────────┼────────────┘
                                                   │
                                      ┌────────────┼────────────┐
                                      │            ▼            │
                                      │      PostgreSQL 16      │
                                      │   (encrypted volume)    │
                                      └─────────────────────────┘

┌──────────────────────┐              ┌─────────────────────────┐
│   RabbitMQ           │◀─────────────│   Quarkus Reactive      │
│   (events, email)    │              │   Messaging             │
└──────────┬───────────┘              └─────────────────────────┘
           │
           ▼
┌──────────────────────┐
│   SMTP Server        │
│   (email delivery)   │
└──────────────────────┘
```

### 2.2 Key Design Principles

| Principle | Implementation |
|-----------|---------------|
| **Rules as configuration** | Leave rules live in DMN decision tables and DB policies, not in Java code |
| **Stateless API** | JWT-based authentication via httpOnly cookies; no server-side sessions |
| **Event-driven side effects** | Notifications, payroll export, audit logging react to domain events published to RabbitMQ |
| **Immutable audit** | Append-only audit tables, no UPDATE/DELETE on audit rows |
| **Defense in depth** | Rate limiting at Nginx + application level; encryption at rest (DB + filesystem) and in transit (TLS) |
| **API versioning** | All endpoints under `/api/v1/` |

---

## 3. Technology Stack

### 3.1 Backend

| Component | Technology | Version |
|-----------|-----------|---------|
| Runtime | Quarkus | 3.36.3 |
| Language | Java | 21 |
| Build | Maven | 3.9+ |
| ORM | Hibernate ORM with Panache | — |
| DB Driver | JDBC PostgreSQL | — |
| REST | RESTEasy Reactive + Jackson | — |
| Auth (JWT) | SmallRye JWT | — |
| Auth (SSO) | OIDC client (Quarkus OIDC) | — |
| DMN / BPMN | Kogito (Quarkus extension) | — |
| Messaging | SmallRye Reactive Messaging — RabbitMQ connector | — |
| Email | Quarkus Mailer | — |
| Scheduling | Quarkus Scheduler | — |
| Validation | Hibernate Validator (Bean Validation) | — |
| API Docs | SmallRye OpenAPI (Swagger UI) | — |
| Health | SmallRye Health | — |
| Cache | Quarkus Cache (Caffeine) | — |

### 3.2 Frontend

| Component | Technology | Version |
|-----------|-----------|---------|
| Framework | Vue | 3.5.x |
| Language | TypeScript | 6.0.x |
| State | Pinia | 3.0.x |
| Router | Vue Router | 5.1.x |
| Build | Vite | 8.0.x |
| HTTP Client | Native Fetch API (`credentials: 'include'`) | — |
| CSS | Tailwind CSS v4 (OKLCH tokens, `@theme`) | — |
| UI Components | Custom components with Tailwind utilities — no third-party library | — |
| Icons | Lucide Vue (`lucide-vue-next`) | — |
| Forms | vee-validate + zod | — |
| Fonts | Self-hosted via `@fontsource` (Inter, DM Sans) | — |
| Sanitizer | DOMPurify (if `v-html` unavoidable) | — |
| i18n | vue-i18n | — |
| Testing — unit | Vitest + Vue Test Utils + @pinia/testing | 4.x |
| Testing — e2e | Playwright | 1.61.x |
| Linting | ESLint + Oxlint | — |
| Formatting | Oxfmt | — |

### 3.3 Infrastructure

| Component | Technology |
|-----------|-----------|
| Database | PostgreSQL 16 |
| Message Broker | RabbitMQ 3.13+ |
| Reverse Proxy | Nginx (TLS termination, rate limiting) |
| File Storage | Local filesystem on encrypted volume (LUKS or cloud provider encrypted disk) |
| Containerization | Docker (Dockerfiles already present) |

---

## 4. Multi-tenancy

### 4.1 Strategy: Discriminator Column

All tenant-scoped tables carry a `tenant_id` column (UUID, NOT NULL). A Quarkus Hibernate multitenant resolver inspects the JWT claim `tenant_id` on every request and sets the Hibernate filter automatically.

```
JWT: { "sub": "...", "tenant_id": "uuid", "roles": [...] }
      │
      ▼
TenantResolver (implements TenantResolver from Hibernate)
      │
      ▼
Hibernate filter: WHERE tenant_id = currentTenantId()
```

### 4.2 Isolation Rules

- Every query (except super-admin cross-tenant queries) is implicitly filtered by tenant.
- Super-admin may query across tenants (e.g., platform health reports).
- A new tenant is bootstrapped with: one system administrator user, default leave types, default policies, and at least one holiday calendar.

### 4.3 Tenant Onboarding

1. Super admin creates tenant via `/api/v1/admin/tenants`.
2. System auto-creates default roles, default leave types, a "General" convenio, and a default Spain policy set.
3. First user who logs in via SSO (matching tenant domain) becomes system administrator.

---

## 5. Domain Model

### 5.1 Core Entities

#### Tenant
```
Tenant
  id: UUID
  name: String
  domain: String (email domain, e.g., "company.com")
  country: String (ISO 3166-1 alpha-2, default "ES")
  active: Boolean
  created_at: Instant
  updated_at: Instant
```

#### User
```
User
  id: UUID
  tenant_id: UUID
  email: String (unique within tenant)
  first_name: String
  last_name: String
  locale: String (e.g., "es", "en")
  status: Enum [ACTIVE, INACTIVE, TERMINATED]
  auth_provider: Enum [GOOGLE, ZOHO, LOCAL]
  auth_provider_id: String (nullable — subject from SSO)
  password_hash: String (nullable — only for LOCAL provider)
  created_at: Instant
  updated_at: Instant
```

#### Employee
```
Employee
  id: UUID
  user_id: UUID → User
  tenant_id: UUID
  employee_code: String
  employment_start_date: LocalDate
  contract_type: Enum [FULL_TIME, PART_TIME, SHIFT]
  work_center: String
  convenio_id: UUID → Convenio
  professional_category: String
  annual_working_hours: Integer
  annual_vacation_entitlement: Integer (base; DMN may override)
  family_status: JSONB  — e.g., {"children": 2, "dependent_relatives": 1}
  seniority_years: Integer (calculated nightly or on update)
  manager_id: UUID → Employee (self-referencing, nullable)
  department_id: UUID → Department
  created_at: Instant
  updated_at: Instant
```

#### Convenio (Collective Agreement)
```
Convenio
  id: UUID
  tenant_id: UUID
  code: String
  name: String
  country: String
  effective_from: LocalDate
  effective_to: LocalDate (nullable = indefinite)
  active: Boolean
```

#### Department
```
Department
  id: UUID
  tenant_id: UUID
  name: String
  manager_id: UUID → Employee (nullable)
```

### 5.2 Leave Management Entities

#### LeaveType
```
LeaveType
  id: UUID
  tenant_id: UUID
  code: String (e.g., "VACATION", "MARRIAGE", "BIRTH")
  name: String (translatable via i18n key)
  description: String (translatable)
  country: String
  paid: Boolean
  active: Boolean
  requires_documentation: Boolean
  requires_approval: Boolean
  default_max_days: Integer (fallback when no policy matches)
```

Standard codes:  
`VACATION`, `SICK_LEAVE_COMMON`, `SICK_LEAVE_WORK`, `BIRTH_CHILDCARE`, `PREGNANCY_RISK`, `BREASTFEEDING_RISK`, `PRENATAL_EXAM`, `CHILDBIRTH_PREP`, `MARRIAGE`, `DEATH_RELATIVE`, `SERIOUS_ILLNESS_RELATIVE`, `HOSPITALIZATION_RELATIVE`, `SURGERY_REST`, `MOVING_HOUSE`, `PUBLIC_DUTIES`, `JURY_DUTY`, `ELECTORAL_DUTIES`, `TRADE_UNION`, `EXAMS`, `MEDICAL_APPOINTMENT`, `FAMILY_EMERGENCY`, `FORCE_MAJEURE`, `BLOOD_DONATION`, `ORGAN_DONATION`, `UNPAID_LEAVE`, `PARENTAL_LEAVE`, `FAMILY_CARE_LEAVE`, `FAMILY_CARE_REDUCED`, `EXCEDENCIA_VOLUNTARIA`, `EXCEDENCIA_FORZOSA`, `EXCEDENCIA_CHILDCARE`, `EXCEDENCIA_FAMILY`, `EXCEDENCIA_PUBLIC_OFFICE`, `EXCEDENCIA_UNION`, `REDUCED_HOURS_CHILDCARE`, `REDUCED_HOURS_DEPENDENT`

#### Policy
```
Policy
  id: UUID
  tenant_id: UUID
  leave_type_id: UUID → LeaveType
  convenio_id: UUID → Convenio (nullable = general/default)
  name: String
  description: String
  effective_from: LocalDate
  effective_to: LocalDate (nullable)
  priority: Integer (lower = higher priority; used for resolution)
  dmn_eligibility_ref: String (DMN decision ID for eligibility)
  dmn_duration_ref: String (DMN decision ID for duration calculation)
  dmn_entitlement_ref: String (DMN decision ID for balance/entitlement)
  bpmn_process_id: String (BPMN process ID for approval workflow)
  active: Boolean
  version: Integer (monotonically increasing per leave_type + convenio)
  created_at: Instant
  updated_at: Instant
```

#### PolicyRule (for simple, non-DMN rules)
```
PolicyRule
  id: UUID
  policy_id: UUID → Policy
  rule_category: Enum [ELIGIBILITY, DURATION, DOCUMENTATION, ACCOUNTING, WORKFLOW]
  rule_type: String (e.g., "MIN_SENIORITY", "FIXED_DAYS", "REQUIRES_DOCUMENT", "DOCUMENT_TYPE", "NO_BALANCE_IMPACT")
  rule_config: JSONB (the actual rule parameters)
  priority: Integer
```

Example `rule_config` values:
```json
{"type": "MIN_SENIORITY", "months": 12}
{"type": "FIXED_DAYS", "days": 15}
{"type": "DOCUMENT_TYPE", "accepted": ["MARRIAGE_CERTIFICATE"]}
{"type": "APPROVAL_CHAIN", "steps": ["MANAGER", "HR"]}
{"type": "DEDUCT_BALANCE", "balance": "VACATION"}
{"type": "RELATIVE_TYPE", "allowed": ["SPOUSE", "CHILD", "PARENT"]}
```

#### LeaveRequest
```
LeaveRequest
  id: UUID
  tenant_id: UUID
  employee_id: UUID → Employee
  leave_type_id: UUID → LeaveType
  policy_id: UUID → Policy (resolved at submission time)
  policy_version: Integer (frozen at submission)
  start_date: LocalDate
  end_date: LocalDate
  start_time: LocalTime (nullable — for partial-day leaves)
  end_time: LocalTime (nullable)
  total_days: BigDecimal (calculated)
  status: Enum [DRAFT, SUBMITTED, UNDER_REVIEW, APPROVED, REJECTED,
                CANCELLATION_REQUESTED, CANCELLATION_UNDER_REVIEW, CANCELLED]
  cancellation_reason: String (nullable)
  submitted_at: Instant
  resolved_at: Instant
  created_at: Instant
  updated_at: Instant
```

#### LeaveBalance
```
LeaveBalance
  id: UUID
  tenant_id: UUID
  employee_id: UUID → Employee
  leave_type_id: UUID → LeaveType
  year: Integer
  entitlement: BigDecimal
  consumed: BigDecimal
  carried_over: BigDecimal (from previous year)
  carried_over_expiry: LocalDate (nullable)
  remaining: BigDecimal (calculated: entitlement + carried_over − consumed)
  updated_at: Instant
```

#### ApprovalStep
```
ApprovalStep
  id: UUID
  tenant_id: UUID
  leave_request_id: UUID → LeaveRequest
  step_order: Integer (1, 2, 3, ...)
  approver_role: Enum [MANAGER, DEPARTMENT_HEAD, HR, LEGAL, MEDICAL_SPECIALIST]
  approver_id: UUID → User (nullable — assigned when dequeued)
  status: Enum [PENDING, APPROVED, REJECTED, DELEGATED, SKIPPED]
  comment: String (nullable)
  decided_at: Instant
  created_at: Instant
```

#### LeaveDocument
```
LeaveDocument
  id: UUID
  tenant_id: UUID
  leave_request_id: UUID → LeaveRequest
  document_type: String (e.g., "MARRIAGE_CERTIFICATE", "MEDICAL_CERTIFICATE", "HOSPITAL_REPORT")
  file_name: String (original)
  file_path: String (encrypted path on filesystem)
  mime_type: String (whitelist enforced)
  file_size_bytes: Long (max 10 MB)
  encrypted: Boolean (always true in production)
  uploaded_by: UUID → User
  uploaded_at: Instant
```

#### PublicHoliday
```
PublicHoliday
  id: UUID
  tenant_id: UUID
  calendar_id: UUID → HolidayCalendar
  name: String
  date: LocalDate
  autonomous_community: String (nullable — ISO 3166-2 ES code)
  municipality: String (nullable)
  recurring: Boolean
```

#### HolidayCalendar
```
HolidayCalendar
  id: UUID
  tenant_id: UUID
  name: String
  year: Integer (nullable — if null, applies every year)
  country: String
  autonomous_community: String (nullable)
  municipality: String (nullable)
  work_center: String (nullable — null = all work centers)
```

#### BridgeDay
```
BridgeDay
  id: UUID
  tenant_id: UUID
  calendar_id: UUID → HolidayCalendar
  name: String (nullable)
  date: LocalDate
  description: String (nullable)
```

#### AuditLog
```
AuditLog
  id: UUID
  tenant_id: UUID
  entity_type: String (e.g., "LeaveRequest", "LeaveDocument", "Policy")
  entity_id: UUID
  action: Enum [CREATED, UPDATED, DELETED, VIEWED, SUBMITTED, APPROVED,
                REJECTED, CANCELLED, CANCELLATION_REQUESTED, DOCUMENT_UPLOADED,
                DOCUMENT_ACCESSED, DOCUMENT_DELETED, POLICY_ACTIVATED, POLICY_VERSIONED,
                ROLE_ASSIGNED, PERMISSION_GRANTED, LOGIN, LOGOUT]
  changes: JSONB (diff — old value → new value)
  performed_by: UUID → User
  performed_at: Instant
  ip_address: String
  user_agent: String
```

This table is APPEND-ONLY. No UPDATE, no DELETE allowed at DB level.

### 5.3 Role & Permission Entities

```
Role
  id: UUID
  tenant_id: UUID
  name: String (e.g., "Employee", "Line Manager", "HR Administrator")
  description: String
  system: Boolean (true = built-in, cannot be deleted)

Permission
  id: UUID
  role_id: UUID → Role
  code: String (see §8.4 Permission Codes)

UserRole
  user_id: UUID → User
  role_id: UUID → Role
  -- Composite PK (user_id, role_id)
```

---

## 6. Rule Engine — DMN + BPMN (Kogito)

### 6.1 Why Kogito

Kogito is the Quarkus-native implementation of the Decision Model and Notation (DMN) and Business Process Model and Notation (BPMN) standards. DMN decision tables are readable by business analysts and auditable. BPMN processes provide long-running approval workflows with timers, escalations, and compensations.

### 6.2 DMN Decision Tables

#### DMN: Vacation Entitlement (`vacation-entitlement.dmn`)

| Input | Type | Output | Type |
|-------|------|--------|------|
| Seniority Years | number | Base Entitlement | number (days) |
| Contract Type | string | Additional Days (convenio) | number |
| Convenio Code | string | Total Entitlement | number |
| Annual Working Hours | number | Prorated | boolean |
| Employment Start Date | date | Prorated Days | number |
| Request Year | number | | |

#### DMN: Leave Eligibility (`leave-eligibility.dmn`)

| Input | Type | Output | Type |
|-------|------|--------|------|
| Leave Type Code | string | Eligible | boolean |
| Employee ID | string | Ineligibility Reason | string |
| Seniority Months | number | | |
| Contract Type | string | | |
| Family Status | JSON | | |
| Request Dates | date range | | |

#### DMN: Leave Duration (`leave-duration.dmn`)

| Input | Type | Output | Type |
|-------|------|--------|------|
| Leave Type Code | string | Maximum Days | number |
| Policy Duration Rules | JSON | Minimum Days | number |
| Employee Attributes | JSON | Allow Partial Days | boolean |

#### DMN: Proration (`proration.dmn`)

| Input | Type | Output | Type |
|-------|------|--------|------|
| Employment Start Date | date | Prorated Days | number |
| Year | number | | |
| Full Entitlement | number | | |

### 6.3 BPMN Approval Workflows

#### BPMN: Standard Approval (`approval-standard.bpmn`)
```
[Employee Submits] → [Manager Reviews] ──→ [Approve] ──→ [HR Reviews] ──→ [Approved]
                          │                      │
                          └──→ [Reject]          └──→ [Reject]
```
Timer boundary events: reminder at 2 business days, escalation at 5 business days.

#### BPMN: Manager Only (`approval-manager-only.bpmn`)
```
[Employee Submits] → [Manager Reviews] → [Approved/Rejected]
```

#### BPMN: Auto-Approval (`approval-auto.bpmn`)
```
[Employee Submits] → [Service Task: Validate] → [Approved]
```
No human tasks. Used for sick leave, force majeure, etc.

#### BPMN: Multi-level (`approval-multi-level.bpmn`)
```
[Employee] → [Manager] → [Department Head] → [HR] → [Legal] → [Approved]
```
Used for excedencia, unpaid leave > 30 days.

#### BPMN: Cancellation Workflow (`cancellation-workflow.bpmn`)
```
[Employee Requests Cancellation]
    → [Manager Reviews] → [Approve Cancellation]
        → [Service Task: Restore Balances]
        → [Cancelled]
```

### 6.4 Policy Resolution Algorithm

```
1. Resolve employee's convenio.
2. Query Policy table for:
   - leave_type_id = requested type
   - effective_from <= today <= effective_to
   - convenio_id = employee.convenio OR convenio_id IS NULL (fallback)
   - active = true
   ORDER BY priority ASC
3. First matching policy is selected.
4. Policy ID and version are frozen on the LeaveRequest.
5. DMN decisions referenced by the policy are evaluated.
6. BPMN process referenced by the policy is started.
```

---

## 7. Data Model — PostgreSQL

### 7.1 Schema

All tables live in a single database. Multi-tenancy enforced via `tenant_id` column + Hibernate filter.

### 7.2 Table Definitions (Key Tables)

Full DDL includes: `tenants`, `users`, `refresh_tokens`, `roles`, `permissions`, `user_roles`, `convenios`, `departments`, `employees`, `leave_types`, `policies`, `policy_rules`, `leave_requests`, `leave_balances`, `approval_steps`, `leave_documents`, `holiday_calendars`, `public_holidays`, `bridge_days`, `audit_log`, `notifications`.

**Key constraints:**
- `audit_log` has a `BEFORE UPDATE OR DELETE` trigger that raises an exception (append-only).
- `leave_documents.mime_type` CHECK constraint enforces whitelist: `application/pdf`, `image/jpeg`, `image/png`, `image/webp`.
- `leave_documents.file_size_bytes` CHECK ≤ 10,485,760 (10 MB).
- Every tenant-scoped table has composite indexes on `(tenant_id, ...)` for common query patterns.

### 7.3 Database-Level Encryption

PostgreSQL runs on a LUKS-encrypted volume. Sensitive columns use pgcrypto for column-level encryption on `users.password_hash` and `leave_documents.file_path`.

---

## 8. Authentication & Authorization

### 8.1 SSO Flow

#### Google SSO
```
1. Frontend: User clicks "Sign in with Google"
2. Redirect to: /api/v1/auth/google
3. Backend: Redirects user to Google OAuth 2.0 authorization endpoint
   - Scopes: openid email profile
   - Redirect URI: https://<domain>/api/v1/auth/google/callback
4. Google redirects back with authorization code
5. Backend:
   a. Exchanges code for ID token + access token
   b. Validates ID token (signature, audience, expiry)
   c. Extracts: sub, email, name, picture
   d. Resolves or creates User + Employee records
   e. Generates JWT (access token, 8h) + refresh token (opaque, 30 days)
   f. Sets three cookies on the response:
      - access_token:  httpOnly, Secure, SameSite=Strict, Path=/api, Max-Age=28800
      - refresh_token: httpOnly, Secure, SameSite=Strict, Path=/api/v1/auth, Max-Age=2592000
      - csrf_token:    Secure, SameSite=Strict, Path=/, readable by JS (non-httpOnly)
   g. Redirects to: {FRONTEND_URL}/login/callback (no tokens in URL)
6. Frontend: Browser stores cookies automatically; frontend calls GET /api/v1/auth/me
   to fetch user info, permissions, and roles → confirms authenticated session
```

#### Zoho SSO
Identical flow, using Zoho OIDC endpoints.

#### Local Login
```
POST /api/v1/auth/login
Body: { "email": "...", "password": "..." }
Backend validates credentials, sets httpOnly cookies (access_token, refresh_token, csrf_token).
Response: 200 with user info { "id": "...", "email": "...", "roles": [...], "permissions": [...] }
(No tokens in the response body — they are in httpOnly cookies.)
```

### 8.2 JWT Structure

```json
{
  "iss": "hr-leave-management",
  "sub": "<user-uuid>",
  "upn": "user@company.com",
  "preferred_username": "John Doe",
  "tenant_id": "<tenant-uuid>",
  "groups": ["employee", "line_manager"],
  "iat": 1719000000,
  "exp": 1719028800,
  "jti": "<unique-token-id>"
}
```

### 8.3 Token Refresh

```
POST /api/v1/auth/refresh
(No request body — cookies are sent automatically by the browser.)
Cookies sent: refresh_token (httpOnly)
Backend:
  1. Reads refresh_token from cookie
  2. Hashes it (SHA-256) and looks up in refresh_tokens table
     where token_hash = hash AND revoked = false AND expires_at > now()
  3. If valid: revokes old token, generates new access_token + refresh_token pair,
     sets updated httpOnly cookies and a new csrf_token cookie
  4. If invalid or missing: returns 401, clears cookies
Response 200 (cookies updated): { "message": "ok" }
```

Refresh token rotation: every refresh issues a new refresh token and revokes the old one. This limits the window for stolen refresh tokens.

**Frontend refresh flow**: The fetch client (§15.10) intercepts 401 responses, calls `POST /api/v1/auth/refresh`, and retries the original request once. If refresh also fails, the user is redirected to `/login`.

### 8.4 Role Definitions & Permission Codes

#### Role: Employee
```
leave.request.own          — Create/view own leave requests
leave.cancel.own           — Request cancellation of own leaves
leave.balance.view.own     — View own balances
document.upload.own        — Upload documents for own requests
document.view.own          — View own uploaded documents
profile.view.own           — View own profile
calendar.view.team         — View team leave calendar (dates + type only, no medical details)
```

#### Role: Line Manager
```
leave.view.team            — View direct reports' leave requests
leave.approve.team         — Approve/reject as step 1 approver
leave.balance.view.team    — View team balances
calendar.view.team         — View team calendar
approval.delegate          — Delegate pending approvals
notification.receive.team  — Receive team-related notifications
```
Plus all Employee permissions.

#### Role: HR Administrator
```
leave.view.all             — View all employee leave records
leave.approve.hr           — Approve/reject as HR step
leave.override             — Override decisions, correct balances
leave.balance.edit         — Modify leave balances
document.view.all          — View all documents (non-medical)
calendar.manage            — Create/edit holiday calendars
calendar.bulk.assign       — Company-wide closure assignments
report.view.all            — Access all reports
employee.view.all          — View all employee records
employee.edit              — Update employee master data
convenio.manage            — Create/edit convenios
```
Plus all Employee permissions.

#### Role: HR Medical Specialist
```
medical.view              — View medical certificates and health documents
medical.approve           — Approve/reject medical leave
leave.view.medical        — View medical leave requests
```
Plus all Employee permissions.

#### Role: HR Policy Administrator
```
leave_type.manage          — Create/edit leave types
policy.create              — Create policies
policy.edit                — Edit draft policies
policy.publish             — Publish/activate policies
policy.version             — Create new policy versions
policy.view                — View all policies
```
Plus all Employee permissions.

#### Role: Payroll Officer
```
leave.view.payroll         — View payroll-impacting leave data
report.payroll.export      — Export payroll-impacting absences
employee.view.basic        — View basic employee info (not medical)
```

#### Role: Compliance / Legal Officer
```
audit.view                 — Query audit logs
policy.view.history        — View historical policy versions
leave.view.all             — Read-only access to leave records
report.view.all            — Access all reports (read-only)
```
Cannot modify any data.

#### Role: Department Manager
```
leave.view.department      — View department leave calendar
leave.balance.view.department — View department balances
approval.escalated         — Handle escalated approvals
report.view.department     — View department reports
```

#### Role: Leave Administrator
```
leave.process              — Validate and process leave requests
document.request           — Request missing documents from employees
leave.view.processing      — View processing queue
```

#### Role: System Administrator
```
user.manage                — Create/edit/deactivate users
user.role.assign           — Assign/revoke roles
tenant.manage              — Create/configure tenants (super admin only)
integration.configure      — Configure SSO, SMTP, etc.
system.config              — System-level configuration
```
Explicitly NOT granted: `leave.view.all`, `medical.view`, `leave.approve.*`, `policy.*` (separation of duties).

#### Role: Auditor
```
audit.view                 — Query audit logs
policy.view                — Read-only policy access
leave.view.all             — Read-only leave access
report.view.all            — Read-only report access
```
Cannot modify anything.

### 8.5 Permission Matrix (Summary)

| Permission | Employee | Manager | HR Admin | HR Medical | HR Policy | Payroll | Legal | Dept Mgr | Leave Admin | Sys Admin | Auditor |
|-----------|:--------:|:-------:|:--------:|:----------:|:---------:|:-------:|:-----:|:--------:|:-----------:|:---------:|:-------:|
| leave.request.own | ✓ | ✓ | ✓ | ✓ | ✓ | — | — | ✓ | — | — | — |
| leave.view.team | — | ✓ | — | — | — | — | — | — | — | — | — |
| leave.view.all | — | — | ✓ | — | — | — | ✓ | — | — | — | ✓ |
| leave.approve.team | — | ✓ | — | — | — | — | — | — | — | — | — |
| leave.approve.hr | — | — | ✓ | — | — | — | — | — | — | — | — |
| leave.override | — | — | ✓ | — | — | — | — | — | — | — | — |
| medical.view | — | — | — | ✓ | — | — | — | — | — | — | — |
| policy.create/edit/publish | — | — | — | — | ✓ | — | — | — | — | — | — |
| audit.view | — | — | — | — | — | — | ✓ | — | — | — | ✓ |
| user.manage | — | — | — | — | — | — | — | — | — | ✓ | — |
| document.view.all | — | — | ✓ | ✓ | — | — | — | — | — | — | — |

A user may hold multiple roles; permissions are additive.

### 8.6 Backend Authorization Implementation

- JWT extracted from `access_token` httpOnly cookie by SmallRye JWT.
- JWT `groups` claim carries role names.
- Use `@RolesAllowed` annotations on JAX-RS endpoints.
- Fine-grained permission checks (e.g., "own" vs "team" vs "all") are enforced in service-layer methods using `SecurityIdentity` + custom permission evaluator.
- Medical data endpoints: additionally check `medical.view` permission explicitly; log every access to `audit_log` with `DOCUMENT_ACCESSED` action.
- CSRF protection: state-changing requests (POST, PUT, DELETE, PATCH) must include an `X-CSRF-Token` header whose value matches the `csrf_token` cookie. Verified by a Quarkus server filter.

---

## 9. REST API Specification

All endpoints are prefixed with `/api/v1`. Authentication via httpOnly `access_token` cookie (browser sends automatically). CSRF protection via `X-CSRF-Token` header on state-changing requests.  
All request/response bodies are JSON. Dates are ISO 8601 (`YYYY-MM-DD`). Timestamps are ISO 8601 with timezone.  

> **Full API contract**: See [API-DESIGN.md](./API-DESIGN.md) for complete request/response schemas, pagination, filtering, ETags, rate limiting headers, and all ~70 endpoints. The sections below summarize the endpoint inventory.

### 9.1 Authentication

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `POST` | `/auth/login` | None | Local login. Sets httpOnly cookies; returns user info. |
| `POST` | `/auth/refresh` | Cookie (`refresh_token`) | Refresh access token. Updates httpOnly cookies. |
| `POST` | `/auth/logout` | Cookie | Revoke refresh token; clear all auth cookies. |
| `GET` | `/auth/google` | None | Redirect to Google OAuth |
| `GET` | `/auth/google/callback` | None (code param) | Google OAuth callback; sets cookies; redirects to frontend. |
| `GET` | `/auth/zoho` | None | Redirect to Zoho OAuth |
| `GET` | `/auth/zoho/callback` | None (code param) | Zoho OAuth callback; sets cookies; redirects to frontend. |
| `GET` | `/auth/me` | Cookie (`access_token`) | Current user info, roles, permissions. |

#### POST /auth/login
```json
// Request
{ "email": "user@company.com", "password": "s3cret" }

// Response 200 (cookies set: access_token, refresh_token, csrf_token)
{
  "id": "uuid", "email": "user@company.com", "first_name": "John", "last_name": "Doe",
  "locale": "es", "tenant_id": "uuid",
  "roles": ["employee", "line_manager"],
  "permissions": ["leave.request.own", "leave.view.team", "..."],
  "employee_id": "uuid", "employee_code": "EMP001"
}
```

#### POST /auth/refresh
```
// Request: no body (refresh_token sent via httpOnly cookie)
// Response 200 (cookies updated): { "message": "ok" }
// Response 401: cookies cleared
```

#### POST /auth/logout
```
// Request: no body
// Response 200: access_token, refresh_token, csrf_token cookies cleared.
// Backend revokes the refresh_token in the database.
```

#### GET /auth/me
```json
// Response 200 (authenticated via access_token cookie)
{
  "id": "uuid", "email": "user@company.com", "first_name": "John", "last_name": "Doe",
  "locale": "es", "tenant_id": "uuid",
  "roles": ["employee", "line_manager"],
  "permissions": ["leave.request.own", "leave.view.team", "..."],
  "employee_id": "uuid", "employee_code": "EMP001"
}
```

### 9.2 Leave Types

| Method | Path | Roles | Description |
|--------|------|-------|-------------|
| `GET` | `/leave-types` | All authenticated | List active leave types for tenant |
| `GET` | `/leave-types/{id}` | All authenticated | Detail |
| `POST` | `/leave-types` | HR Policy Admin | Create |
| `PUT` | `/leave-types/{id}` | HR Policy Admin | Update |
| `DELETE` | `/leave-types/{id}` | HR Policy Admin | Soft-delete (set active=false) |

### 9.3 Policies

| Method | Path | Roles | Description |
|--------|------|-------|-------------|
| `GET` | `/policies` | HR Policy Admin, Legal | List policies |
| `GET` | `/policies/{id}` | HR Policy Admin, Legal | Detail |
| `POST` | `/policies` | HR Policy Admin | Create draft |
| `PUT` | `/policies/{id}` | HR Policy Admin | Update draft |
| `POST` | `/policies/{id}/publish` | HR Policy Admin | Activate policy |
| `GET` | `/policies/{id}/versions` | HR Policy Admin, Legal | Version history |

### 9.4 Leave Requests

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/leave-requests` | Cookie | List (scoped by role: own / team / all) |
| `POST` | `/leave-requests` | Cookie + CSRF | Create draft |
| `GET` | `/leave-requests/{id}` | Cookie | Detail |
| `PUT` | `/leave-requests/{id}` | Cookie + CSRF | Update draft |
| `DELETE` | `/leave-requests/{id}` | Cookie + CSRF | Delete draft |
| `POST` | `/leave-requests/{id}/submit` | Cookie + CSRF | Submit for approval |
| `POST` | `/leave-requests/{id}/cancel` | Cookie + CSRF | Request cancellation |
| `POST` | `/leave-requests/bulk` | Cookie + CSRF | Company-wide closure (HR Admin) |
| `GET` | `/leave-requests/calendar` | Cookie | Calendar-optimized leave data for a date range |

#### POST /leave-requests (create draft)
```json
// Request
{
  "leave_type_id": "uuid",
  "start_date": "2026-08-01", "end_date": "2026-08-15",
  "start_time": null, "end_time": null,
  "notes": "Summer vacation"
}

// Response 201
{
  "id": "uuid", "status": "DRAFT",
  "leave_type": { "code": "VACATION", "name": "Vacaciones" },
  "start_date": "2026-08-01", "end_date": "2026-08-15",
  "total_days": 15.0, "deductible_days": 11.0,
  "balance_after": 11.0,
  "conflicts": [
    { "employee_name": "Jane Smith", "dates": "2026-08-01 - 2026-08-07" }
  ]
}
```

#### POST /leave-requests/{id}/submit
```json
// Response 200
{
  "id": "uuid", "status": "SUBMITTED",
  "policy_id": "uuid", "policy_version": 3,
  "approval_steps": [
    { "step": 1, "approver_role": "MANAGER", "approver_name": "Alice Manager", "status": "PENDING" },
    { "step": 2, "approver_role": "HR", "approver_name": null, "status": "PENDING" }
  ],
  "submitted_at": "2026-06-23T10:35:00Z"
}
```

### 9.5 Approvals

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/approvals/pending` | Cookie | List pending approvals for current user |
| `POST` | `/approvals/{stepId}/approve` | Cookie + CSRF | Approve step |
| `POST` | `/approvals/{stepId}/reject` | Cookie + CSRF | Reject step |
| `POST` | `/approvals/{stepId}/delegate` | Cookie + CSRF | Delegate to another user |

### 9.6 Balances

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/balances` | Cookie | Leave balances (scoped by role) |
| `GET` | `/balances/me` | Cookie | Current user's balances |

#### GET /balances/me
```json
// Response 200
{
  "year": 2026,
  "balances": [{
    "leave_type": { "code": "VACATION", "name": "Vacaciones" },
    "entitlement": 30.0, "consumed": 5.0,
    "carried_over": 3.0, "remaining": 28.0,
    "carried_over_expiry": "2026-03-31"
  }]
}
```

### 9.7 Documents

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `POST` | `/documents/upload` | Cookie + CSRF | Upload document (multipart/form-data) |
| `GET` | `/documents/{id}` | Cookie | Download (access controlled) |
| `DELETE` | `/documents/{id}` | Cookie + CSRF | Delete |

**File restrictions:** MIME whitelist: `application/pdf`, `image/jpeg`, `image/png`, `image/webp`. Max 10 MB. Encrypted at rest (AES-256-GCM). Medical documents additionally restricted to HR Medical Specialist, Legal, and the employee themself.

### 9.8 Employees

| Method | Path | Roles | Description |
|--------|------|-------|-------------|
| `GET` | `/employees` | HR Admin, Manager (team only) | List |
| `GET` | `/employees/{id}` | HR Admin, Manager (if team) | Detail |
| `POST` | `/employees` | HR Admin, Sys Admin | Create |
| `PUT` | `/employees/{id}` | HR Admin | Update |

### 9.9 Holiday Calendars

| Method | Path | Roles | Description |
|--------|------|-------|-------------|
| `GET` | `/holiday-calendars` | All authenticated | List calendars |
| `POST` | `/holiday-calendars` | HR Admin | Create |
| `PUT` | `/holiday-calendars/{id}` | HR Admin | Update |
| `DELETE` | `/holiday-calendars/{id}` | HR Admin | Delete |
| `GET` | `/holiday-calendars/{id}/holidays` | All authenticated | List holidays |
| `POST` | `/holiday-calendars/{id}/holidays` | HR Admin | Add holiday |
| `POST` | `/holiday-calendars/{id}/holidays/import` | HR Admin | Import from source |
| `POST` | `/holiday-calendars/{id}/bridge-days` | HR Admin | Add bridge day |

### 9.10 Reports

All reports accessed via a consolidated endpoint with the report type as a path parameter.

| Method | Path | Roles | Description |
|--------|------|-------|-------------|
| `GET` | `/reports/{type}` | HR Admin, Manager (scoped), Payroll | Generate report |
| `POST` | `/reports/{type}/generate` | HR Admin, Payroll | Async report generation (for large datasets) |
| `GET` | `/reports/jobs/{jobId}` | Report requester | Poll async job status |

**Supported `{type}` values**: `vacation-balances`, `absences`, `sick-leave`, `absence-rate`, `payroll-impact`, `pending-documents`, `leave-by-type`.

Query parameters: `?from=2026-01-01&to=2026-12-31&department_id=uuid&format=csv`.  
Async: use `Prefer: respond-async` header or automatic redirect for >10,000 rows.  
Job polling: `GET /reports/jobs/{jobId}` returns status + download URL when complete.

### 9.11 Audit

| Method | Path | Roles | Description |
|--------|------|-------|-------------|
| `GET` | `/audit-log` | Legal, Auditor, Sys Admin | Query audit log with filters |
| `GET` | `/audit-log/{entityType}/{entityId}` | Legal, Auditor | Audit trail for entity |

### 9.12 Admin

| Method | Path | Roles | Description |
|--------|------|-------|-------------|
| `GET` | `/admin/users` | Sys Admin | List all users |
| `POST` | `/admin/users` | Sys Admin | Create user |
| `PUT` | `/admin/users/{id}` | Sys Admin | Update user |
| `PUT` | `/admin/users/{id}/roles` | Sys Admin | Assign/revoke roles |
| `GET` | `/admin/roles` | Sys Admin | List roles |
| `POST` | `/admin/roles` | Sys Admin | Create role |
| `PUT` | `/admin/roles/{id}` | Sys Admin | Update role |
| `GET` | `/admin/tenants` | Super Admin | List tenants |
| `POST` | `/admin/tenants` | Super Admin | Create tenant |

### 9.13 GDPR

| Method | Path | Roles | Description |
|--------|------|-------|-------------|
| `GET` | `/gdpr/data-export` | Employee (own) | Export all personal data (JSON) |
| `POST` | `/gdpr/data-export/{employeeId}` | HR Admin, Legal | Export employee data (right of access) |
| `POST` | `/gdpr/deletion-request` | Employee, HR Admin | Request data deletion/anonymization |

### 9.14 Common HTTP Responses

| Code | When |
|------|------|
| 200 | Success (GET, PUT, some POSTs) |
| 201 | Resource created |
| 204 | Success, no body (DELETE) |
| 400 | Validation error (response includes `errors` array) |
| 401 | Missing/invalid session (cookie expired or missing) |
| 403 | Authenticated but insufficient permissions |
| 404 | Resource not found |
| 409 | Conflict (e.g., overlapping leave dates) |
| 429 | Rate limit exceeded |
| 500 | Internal server error |

Error response format:
```json
{
  "error": "validation_error",
  "message": "End date must be after start date",
  "errors": [{ "field": "end_date", "message": "must be after start_date" }],
  "trace_id": "abc-123"
}
```

---

## 10. Event-Driven Architecture — RabbitMQ

### 10.1 Why RabbitMQ

For this use case, RabbitMQ is the better choice over Kafka because:
- **Workflow-oriented**: approval steps, notifications, and reminders map naturally to exchanges + queues.
- **Low throughput**: HR systems generate hundreds-to-thousands of events per day, not millions per second.
- **Operational simplicity**: RabbitMQ is easier to set up, monitor, and maintain for a deployment of this scale.
- **Quarkus support**: SmallRye Reactive Messaging has first-class RabbitMQ connector.

### 10.2 Exchange & Queue Topology

```
Exchange: leave.events (topic exchange)

Routing keys:
  leave.request.submitted    → Queue: notification.queue
  leave.request.approved     → Queue: notification.queue
                               Queue: calendar.sync.queue (future)
                               Queue: payroll.export.queue (future)
  leave.request.rejected     → Queue: notification.queue
  leave.request.cancelled    → Queue: notification.queue
                               Queue: balance.restore.queue
  leave.cancellation.requested → Queue: notification.queue
  leave.started              → Queue: notification.queue
  leave.ended                → Queue: notification.queue
  leave.document.uploaded    → Queue: notification.queue
  leave.approval.reminder    → Queue: notification.queue
  leave.approval.escalated   → Queue: notification.queue
  policy.activated           → Queue: audit.queue
  balance.updated            → Queue: payroll.export.queue (future)
```

### 10.3 Consumers

| Consumer | Queue | Action |
|----------|-------|--------|
| `NotificationService` | `notification.queue` | Send email via SMTP, store in `notifications` table |
| `BalanceRestoreService` | `balance.restore.queue` | Restore leave balance after cancellation |
| `AuditEventLogger` | `audit.queue` | Write domain events to `audit_log` |
| `CalendarSyncService` | `calendar.sync.queue` | (Future) sync with Google/Outlook calendar |

### 10.4 Event Schema

All events are JSON with:
```json
{
  "event_id": "uuid",
  "event_type": "leave.request.submitted",
  "tenant_id": "uuid",
  "timestamp": "2026-06-23T10:35:00Z",
  "payload": {
    "leave_request_id": "uuid", "employee_id": "uuid",
    "employee_name": "John Doe", "leave_type": "VACATION",
    "start_date": "2026-08-01", "end_date": "2026-08-15",
    "total_days": 15.0
  }
}
```

---

## 11. Notification System

### 11.1 Notification Types

| Type | Trigger | Recipient(s) |
|------|---------|--------------|
| `LEAVE_SUBMITTED` | Leave request submitted | Next approver(s) |
| `LEAVE_APPROVED` | Leave approved | Employee |
| `LEAVE_REJECTED` | Leave rejected | Employee |
| `CANCELLATION_REQUESTED` | Cancellation workflow started | Manager |
| `LEAVE_CANCELLED` | Cancellation approved | Employee |
| `APPROVAL_REMINDER` | 2 business days without action | Pending approver |
| `APPROVAL_ESCALATED` | 5 business days without action | Escalation approver + original approver |
| `LEAVE_STARTS_SOON` | 3 days before leave start | Employee + Manager |
| `DOCUMENT_REQUESTED` | Missing documents identified | Employee |
| `LEAVE_CONFLICT` | Overlapping team leave detected | Manager (on submission) |
| `BALANCE_LOW` | Balance below threshold | Employee |
| `RETURN_TO_WORK` | Long leave (>4 weeks) ending | Employee + Manager + HR |

### 11.2 Email Configuration

- Quarkus Mailer with SMTP configuration (configurable per tenant).
- Email templates stored as Qute templates, parameterized with locale.
- In-app notifications persisted in `notifications` table and served via `GET /api/v1/notifications` (paginated) + real-time delivery via `GET /api/v1/notifications/stream` (Server-Sent Events). Frontend opens an SSE connection on login; falls back to polling every 60s if SSE fails.

---

## 12. Security

### 12.1 Encryption

| Layer | Mechanism |
|-------|-----------|
| **In transit** | TLS 1.3 (terminated at Nginx; backend communicates via internal network) |
| **At rest — Database** | LUKS-encrypted volume (or cloud KMS-encrypted disk) |
| **At rest — Files (documents)** | AES-256-GCM encryption before writing to filesystem; keys stored in environment variables or HashiCorp Vault |
| **At rest — Column level** | `password_hash` stored as bcrypt; `file_path` optionally encrypted with pgcrypto |
| **JWT signing** | RS256 (asymmetric); private key never leaves the server |

### 12.2 Rate Limiting

**Layer 1 — Nginx:**
```nginx
limit_req_zone $binary_remote_addr zone=api:10m rate=100r/m;
limit_req_zone $binary_remote_addr zone=login:10m rate=10r/m;
location /api/v1/auth/login { limit_req zone=login burst=5 nodelay; }
location /api/v1/ { limit_req zone=api burst=20 nodelay; }
```

**Layer 2 — Quarkus:** custom `@RateLimited` interceptor for sensitive endpoints.

### 12.3 Input Validation

- All DTOs annotated with Bean Validation (`@NotNull`, `@Size`, `@Email`, `@Future`, custom validators).
- Leave dates: `end_date >= start_date`, cannot be in the past (except sick leave retroactive registration by HR Medical).
- File uploads: MIME type whitelist enforced server-side, max size 10 MB.
- SQL injection: Prevented by Hibernate parameterized queries. No raw SQL except for reporting views.

### 12.4 CORS

```properties
quarkus.http.cors.origins=${FRONTEND_URL}
quarkus.http.cors.methods=GET,POST,PUT,DELETE,OPTIONS
quarkus.http.cors.headers=X-CSRF-Token,Content-Type,Accept-Language
quarkus.http.cors.exposed-headers=Content-Disposition
quarkus.http.cors.access-control-max-age=24H
quarkus.http.cors.access-control-allow-credentials=true
```

All fetch requests from the frontend use `credentials: 'include'` so cookies are sent.

### 12.5 Security Headers

Enforced via Nginx:
```
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Referrer-Policy: strict-origin-when-cross-origin
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

### 12.6 Session & Token Security

- **JWT storage**: `access_token` in httpOnly cookie — never exposed to JavaScript. Eliminates XSS token theft.
- **JWT expiry**: 8 hours, signed RS256.
- **Refresh token**: 30-day expiry, opaque (random 256-bit), stored hashed (SHA-256) in DB, transmitted via httpOnly cookie with `Path=/api/v1/auth` (narrowest scope).
- **Refresh token rotation**: old token revoked on each refresh. Reuse detection: if a revoked token is presented, all tokens for that user are revoked.
- **CSRF protection**: Double-submit cookie pattern — `csrf_token` cookie (readable by JS) must match `X-CSRF-Token` request header on all state-changing operations. Verified server-side.
- **Terminated users**: JWT verification rejects tokens where `user.status == 'TERMINATED'`.
- **Cookie attributes**: `Secure` (HTTPS only), `SameSite=Strict` (no cross-site requests), `httpOnly` (no JS access for auth tokens).
- **Logout**: clears all cookies + revokes refresh token in DB.

---

## 13. GDPR Compliance

### 13.1 Data Classification

| Category | Examples | Protection |
|----------|----------|------------|
| **Personal data** | Name, email, employee code | Access controlled by RBAC |
| **Special category data** | Medical certificates, diagnoses, pregnancy status, disability info | Encrypted at rest, restricted to HR Medical Specialist + Legal, logged on every access |
| **Employment data** | Contract type, salary-impacting leave, seniority | Restricted to HR Admin, Payroll |

### 13.2 Access Control

- Managers see leave type + dates + status — **not** medical details, diagnoses, or certificates.
- Medical documents viewable only by: the employee themself, HR Medical Specialist, and Legal/Compliance (audit-only).
- Every access to a medical document is logged in `audit_log` with `DOCUMENT_ACCESSED` action.

### 13.3 Data Retention Policy

| Data | Retention Period | After Retention |
|------|-----------------|-----------------|
| Leave requests (non-medical) | 5 years after end of leave | Anonymized |
| Medical certificates & documents | 5 years after end of leave | **Deleted** |
| Employee master data | 5 years after termination | Anonymized |
| Audit logs | 10 years | Archived to cold storage |
| Refresh tokens | 30 days (expiry) or on logout | Deleted immediately |
| Notifications | 1 year | Deleted |

A scheduled task (`@Scheduled`) runs weekly to identify records past retention and anonymize/delete accordingly.

### 13.4 Right of Access / Data Portability

- `GET /api/v1/gdpr/data-export` returns a machine-readable JSON file containing all personal data for the requesting user.
- HR can trigger the same export for an employee via `POST /api/v1/gdpr/data-export/{employeeId}` (logged in audit).

### 13.5 Right to Erasure

- `POST /api/v1/gdpr/deletion-request` initiates a workflow: employee requests erasure → HR/Legal reviews → if approved, data is anonymized/deleted within 30 days.
- Certain data (audit logs, payroll-impacting records) may be retained per legal obligation; the response explains what was retained and why.

---

## 14. Internationalization (i18n)

### 14.1 Supported Languages

- **Spanish (es)** — default
- **English (en)** — fallback
- Extensible: add more languages via resource files

### 14.2 Language Resolution

A `useLocale()` composable implements the resolution chain:

```
1. User profile locale (synced from backend on login) → use that
2. localStorage key 'app-locale' (persisted manual override) → use that
3. navigator.languages[0] / Accept-Language → match against supported locales
4. Fallback → 'en'
```

- On first login, sync backend-stored locale to localStorage.
- Allow manual override via a locale switcher in the UI.
- The switcher updates the Pinia store, localStorage, and the backend via `PUT /api/v1/profile`.

### 14.3 Backend i18n

- API error messages and validation errors are localized based on `Accept-Language` header.
- Quarkus i18n via resource bundles in `src/main/resources/i18n/messages_{locale}.properties`.
- DMN/BPMN: decision names and descriptions are in English in the models; UI labels are translated in the frontend.

### 14.4 Frontend i18n

- `vue-i18n` with JSON locale files in `src/locales/es.json`, `src/locales/en.json`.
- Translation keys organized by feature namespace: `dashboard.title`, `leave.form.startDate`.
- ICU pluralization and interpolation.
- All user-visible strings are translatable — no hardcoded text in components.
- Number, date, and currency formatting use `Intl.DateTimeFormat`, `Intl.NumberFormat`, `Intl.RelativeTimeFormat`.
- **Design for text expansion**: layouts must tolerate 30–40% longer strings.
- Icon-only buttons always include a translated `aria-label`.

---

## 15. Frontend Architecture — Vue 3

### 15.1 Design System

**CSS Framework**: Tailwind CSS v4 with OKLCH color space. No third-party component library — all components are custom-built with Tailwind utilities.

**Theme tokens** defined in `src/assets/main.css` via `@theme` (see Appendix D). Tokens consumed as Tailwind utilities: `bg-primary-500`, `text-neutral-950`, `font-heading`.

**Dark mode**: Uses Tailwind `class` strategy. A `useTheme` composable reads `prefers-color-scheme` on first visit, toggles the `.dark` class on `<html>`, persists the choice to `localStorage`, and exposes a reactive `theme` ref.

**Typography**:
- Body: Inter Variable, 16px minimum, line-height ≥ 1.5.
- Headings: DM Sans Variable.
- Self-hosted via `@fontsource` (no Google Fonts CDN) — imported in `main.ts`.
- System font stack fallback: `system-ui, -apple-system, 'Segoe UI', Roboto, sans-serif`.

**Spacing**: Ample padding and margins between sections, cards, and form elements. Prefer whitespace over dividers.

**Animation**: CSS transitions and `@keyframes` exclusively. All animations respect `prefers-reduced-motion` via Tailwind's `motion-reduce:` variant.

### 15.2 Project Structure

```
src/
├── api/                  # Fetch wrapper + API service modules
│   ├── client.ts         # Cookie-based fetch (credentials, CSRF, refresh-on-401)
│   ├── auth.ts
│   ├── leaveRequests.ts
│   ├── balances.ts
│   ├── employees.ts
│   ├── policies.ts
│   ├── calendars.ts
│   ├── documents.ts
│   ├── reports.ts
│   └── notifications.ts
├── assets/
│   └── main.css          # Tailwind import + @theme tokens
├── components/
│   └── common/           # Reusable UI primitives
│       ├── AppHeader.vue
│       ├── AppSidebar.vue
│       ├── DataTable.vue
│       ├── DateRangePicker.vue
│       ├── StatusBadge.vue
│       ├── ConfirmDialog.vue
│       ├── FileUpload.vue
│       ├── SkeletonLoader.vue
│       ├── EmptyState.vue
│       ├── ErrorState.vue
│       └── ThemeToggle.vue
├── composables/          # use*() functions — no templates rendered
│   ├── useAuth.ts        # Session check, login/logout, permission Set
│   ├── useCan.ts         # useCan('leave.approve.team') → boolean
│   ├── useTheme.ts       # Dark/light toggle + persistence
│   ├── useLocale.ts      # Locale resolution + switching
│   ├── useLeaveRequest.ts
│   ├── useNotifications.ts
│   └── useCalendar.ts
├── directives/
│   └── permission.ts     # v-permission directive
├── layouts/
│   ├── DefaultLayout.vue  # Sidebar + header + <slot />
│   ├── AuthLayout.vue     # Minimal layout for login
│   └── AdminLayout.vue
├── locales/
│   ├── es.json
│   └── en.json
├── router/
│   ├── index.ts
│   └── guards.ts
├── stores/               # Pinia stores
│   ├── auth.ts
│   ├── leaveRequests.ts
│   ├── balances.ts
│   ├── calendar.ts
│   ├── approvals.ts
│   ├── employees.ts
│   ├── policies.ts
│   ├── notifications.ts
│   └── tenants.ts
├── types/
│   ├── api.ts
│   ├── leave.ts
│   ├── user.ts
│   └── policy.ts
├── utils/
│   ├── cn.ts             # clsx + tailwind-merge class-name utility
│   ├── date.ts
│   ├── format.ts
│   └── validators.ts
├── views/                # Route-level page components
│   ├── DashboardView.vue
│   ├── LoginView.vue
│   ├── LoginCallbackView.vue
│   ├── LeavesListView.vue
│   ├── LeaveNewView.vue
│   ├── LeaveDetailView.vue
│   ├── CalendarView.vue
│   ├── BalancesView.vue
│   ├── ProfileView.vue
│   ├── team/
│   │   ├── TeamCalendarView.vue
│   │   └── ApprovalsView.vue
│   ├── hr/
│   │   ├── EmployeeListView.vue
│   │   ├── EmployeeDetailView.vue
│   │   ├── EmployeeEditView.vue
│   │   ├── PoliciesView.vue
│   │   ├── PolicyNewView.vue
│   │   ├── PolicyDetailView.vue
│   │   ├── LeaveTypesView.vue
│   │   ├── CalendarsView.vue
│   │   ├── ConveniosView.vue
│   │   ├── ReportsView.vue
│   │   ├── DocumentsView.vue
│   │   ├── AuditLogView.vue
│   │   └── DataExportView.vue
│   ├── medical/
│   │   ├── MedicalRequestsView.vue
│   │   └── MedicalDocumentsView.vue
│   └── admin/
│       ├── UsersView.vue
│       ├── UserEditView.vue
│       ├── RolesView.vue
│       ├── TenantsView.vue
│       └── SettingsView.vue
├── __tests__/            # Vitest tests mirroring src structure
│   ├── components/
│   ├── composables/
│   ├── stores/
│   └── views/
├── App.vue
└── main.ts
```

### 15.3 Routes

```
/                          → Redirect to /dashboard
/login                     → Login page (SSO buttons + local form)
/login/callback            → Calls /auth/me to verify session, redirects to /dashboard
/dashboard                 → Employee dashboard (balances, upcoming leaves, pending actions)

/leaves                    → My leave requests list
/leaves/new                → New leave request form
/leaves/:id                → Leave request detail + timeline
/leaves/:id/cancel         → Cancellation request form

/calendar                  → Team calendar view
/calendar/company          → Company-wide calendar (HR/Manager)

/balances                  → My leave balances
/documents                 → My uploaded documents
/notifications             → Notification list
/profile                   → My profile + locale + theme settings

/team                      → Team leave calendar + balances (Manager)
/team/approvals            → Pending approvals (Manager)

/hr/employees              → Employee list (HR Admin)
/hr/employees/:id          → Employee detail
/hr/employees/:id/edit     → Edit employee
/hr/leave-types            → Leave types management
/hr/policies               → Policy list
/hr/policies/new           → New policy
/hr/policies/:id           → Policy detail + version history
/hr/calendars              → Holiday calendar management
/hr/convenios              → Convenio management
/hr/reports                → Reports dashboard
/hr/documents              → Document management
/hr/audit-log              → Audit log viewer
/hr/data-export            → GDPR data export

/medical/requests          → Medical leave queue (HR Medical)
/medical/documents         → Medical document management

/admin/users               → User management (Sys Admin)
/admin/users/:id           → User detail + role assignment
/admin/roles               → Role management
/admin/tenants             → Tenant management (Super Admin)
/admin/settings            → System settings
```

### 15.4 Route Guards

```typescript
// router/guards.ts
router.beforeEach(async (to) => {
  const auth = useAuthStore()
  if (to.path.startsWith('/login')) return true

  if (!auth.user) await auth.fetchSession()

  if (!auth.isAuthenticated) {
    return { path: '/login', query: { redirect: to.fullPath } }
  }

  const perm = to.meta.permission as string | undefined
  if (perm && !auth.hasPermission(perm)) return { path: '/dashboard' }
  return true
})
```

### 15.5 Key Composables

#### useAuth
```typescript
// No tokens in memory — httpOnly cookies managed by browser.
// Store caches: user, permissions: Set<string>, roles: string[]
// Actions: fetchSession(), login(), logout(), refreshSession()
// SSO: loginWithGoogle(), loginWithZoho() — redirects to /api/v1/auth/google or zoho
```

#### useCan
```typescript
export function useCan(permission: string): boolean {
  return useAuthStore().hasPermission(permission)  // O(1) Set lookup
}
```

#### useTheme
```typescript
const theme = ref<'light' | 'dark'>(
  localStorage.getItem('app-theme') ??
  (window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light')
)
watchEffect(() => {
  document.documentElement.classList.toggle('dark', theme.value === 'dark')
  localStorage.setItem('app-theme', theme.value)
})
```

### 15.6 Four-State Coverage (Mandatory)

Every component that fetches or displays data must handle four states:

| State | Behavior |
|-------|----------|
| **Loading** | Render `<SkeletonLoader>` or spinner while data resolves |
| **Empty** | Show `<EmptyState>` with helpful message and call to action |
| **Error** | Show `<ErrorState>` with user-friendly message and Retry button |
| **Success** | Render the actual data |

### 15.7 Form Validation (vee-validate + zod)

All forms use **vee-validate** + **zod** — validate on blur and on submit:

```typescript
import { useForm } from 'vee-validate'
import { toTypedSchema } from '@vee-validate/zod'
import { z } from 'zod'

const schema = toTypedSchema(z.object({
  leave_type_id: z.string().uuid(),
  start_date: z.string().min(1),
  end_date: z.string().min(1),
}).refine(d => d.end_date >= d.start_date, {
  message: 'End date must be after start date',
  path: ['end_date'],
}))

const { handleSubmit, errors, isSubmitting } = useForm({ validationSchema: schema })
```

- Inline error messages next to the relevant field.
- Submit buttons disabled during submission (`isSubmitting`).
- Server-side validation errors (400) mapped to form fields or shown as summary.

### 15.8 Error Handling by Severity

| Severity | Display | Example |
|----------|---------|---------|
| Field-level | Inline below the field | "Email is invalid" |
| Action-level | Toast (auto-dismiss) | "Leave submitted successfully" |
| Section-level | Inline state replacement | "No results" where the list would be |
| App-wide | Persistent banner | "You're offline" |
| Catastrophic | Full-page | 404 / 500 error page |

Critical component trees wrapped with `onErrorCaptured` (Vue's error boundary) to prevent a single rendering failure from crashing the entire app.

### 15.9 Permission Directive

```vue
<button v-permission="'leave.approve.team'" @click="approve">Approve</button>
```
```typescript
// directives/permission.ts
export const vPermission: Directive<HTMLElement, string> = {
  mounted(el, binding) {
    if (!useAuthStore().hasPermission(binding.value)) {
      el.parentNode?.removeChild(el)
    }
  }
}
```

### 15.10 HTTP Client (Cookie-Based Fetch)

No tokens in JavaScript. The browser sends httpOnly cookies automatically. CSRF token is read from the readable `csrf_token` cookie:

```typescript
// api/client.ts
const BASE_URL = import.meta.env.VITE_API_BASE_URL + '/api/v1'

function getCsrfToken(): string {
  const match = document.cookie.match(/(?:^|;\s*)csrf_token=([^;]*)/)
  return match ? match[1] : ''
}

async function request<T>(path: string, options: RequestInit = {}): Promise<T> {
  const headers: Record<string, string> = {
    'Accept': 'application/json',
    ...(options.headers as Record<string, string>),
  }

  if (['POST','PUT','DELETE','PATCH'].includes(options.method?.toUpperCase() ?? '')) {
    headers['X-CSRF-Token'] = getCsrfToken()
  }

  if (options.body && typeof options.body !== 'string' && !(options.body instanceof FormData)) {
    headers['Content-Type'] = 'application/json'
    options.body = JSON.stringify(options.body)
  }

  let response = await fetch(`${BASE_URL}${path}`, { ...options, headers, credentials: 'include' })

  // 401 → refresh → retry once
  if (response.status === 401 && !(options as any).__retry) {
    const refreshRes = await fetch(`${BASE_URL}/auth/refresh`, {
      method: 'POST', credentials: 'include',
      headers: { 'X-CSRF-Token': getCsrfToken() },
    })
    if (refreshRes.ok) {
      (options as any).__retry = true
      response = await fetch(`${BASE_URL}${path}`, { ...options, headers, credentials: 'include' })
    } else {
      throw new ApiError(401, { error: 'Session expired' })
    }
  }

  if (!response.ok) {
    const data = await response.json().catch(() => null)
    throw new ApiError(response.status, data)
  }

  if (response.status === 204) return undefined as T
  return response.json() as Promise<T>
}

export const api = {
  get: <T>(p: string) => request<T>(p, { method: 'GET' }),
  post: <T>(p: string, body?: unknown) => request<T>(p, { method: 'POST', body }),
  put: <T>(p: string, body?: unknown) => request<T>(p, { method: 'PUT', body }),
  delete: <T>(p: string) => request<T>(p, { method: 'DELETE' }),
  upload: <T>(p: string, fd: FormData) => request<T>(p, { method: 'POST', body: fd }),
}
```

### 15.11 Performance Guidelines

- **`shallowRef`** for large objects treated as immutable (full replacement on change).
- **`computed`** for all derived state.
- **Lazy route loading** via dynamic imports for all routes.
- **`<KeepAlive>`** for expensive views that toggle frequently (Dashboard, Calendar).
- **`v-if`** for permission-gated/rarely-shown content; **`v-show`** for tabs/panels.
- **Debounce** search inputs (~300ms).
- **Virtualized rendering** for lists exceeding ~100 items.
- Dynamic import for heavy third-party libraries only when needed.

### 15.12 Accessibility (WCAG Compliance)

- Semantic HTML: `<nav>`, `<main>`, `<header>`, `<button>`, `<form>`.
- All interactive elements are keyboard-navigable (Tab, Enter, Escape).
- Focus trapping inside modals; focus returns to trigger on close.
- `aria-label` on icon-only buttons (using translated strings).
- Sufficient contrast ratios (OKLCH color space).
- `<Teleport to="body">` for modals, tooltips, dropdowns, and toasts.
- Respect `prefers-reduced-motion`.

### 15.13 Security (Frontend)

- **No tokens in `localStorage` or `sessionStorage`** — httpOnly cookies only.
- **CSRF**: `X-CSRF-Token` header on all POST/PUT/DELETE/PATCH requests.
- **XSS prevention**: Never use `v-html` with user-supplied content. If rich text is unavoidable, sanitize with DOMPurify.
- **CSP**: No inline scripts/styles — everything in `.vue` SFCs.
- **Secrets**: Only `VITE_`-prefixed env vars for public config; typed via `ImportMetaEnv` augmentation.
- **429 handling**: Show retry delay on rate-limited responses; implement exponential backoff for automatic retries.
- **Dependency auditing**: run `npm audit` regularly.

---

## 16. Reporting

### 16.1 Report Endpoints

All reports support date range filtering (`from`, `to`), department filtering (`department_id`), and format selection (`format=json|csv`).

### 16.2 Report Types

| Report | Description | Access |
|--------|-------------|--------|
| Vacation Balances | Per employee: entitlement, consumed, remaining, carry-over | HR Admin, Manager (team only) |
| Upcoming Absences | All approved leaves in a date range | HR Admin, Manager (team only) |
| Sick Leave Statistics | Frequency, duration, trends | HR Admin, Medical |
| Absence Rate | Percentage by department/month | HR Admin, Dept Mgr |
| Leave by Type | Utilization breakdown by leave type | HR Admin |
| Payroll Impact | Unpaid leave, reduced hours, contract suspensions | HR Admin, Payroll |
| Excedencia Tracking | Active excedencias, return deadlines | HR Admin, Legal |
| Pending Documentation | Leaves missing required documents | HR Admin, Leave Admin |

### 16.3 Implementation

- Reports use read-only database views (or named queries) for performance.
- Heavy reports are generated asynchronously: `POST /reports/{type}/generate` → `202 Accepted` with `job_id` → `GET /reports/jobs/{jobId}`.
- Light reports (<1000 rows) return synchronously.

---

## 17. Testing Strategy

### 17.1 Backend (Quarkus)

| Layer | Tool | Coverage Target | Description |
|-------|------|----------------|-------------|
| **Unit tests** | JUnit 5 + Mockito | ≥ 80% line coverage | Service layer, DMN decisions, validators, utilities. No DB or external services. |
| **Integration tests** | Quarkus `@QuarkusIntegrationTest` + RestAssured | ≥ 70% of endpoints | REST endpoints with real DB (Testcontainers PostgreSQL), real RabbitMQ (Testcontainers). |
| **DMN tests** | Kogito test framework | 100% of decision tables | Each DMN decision table tested with representative input combinations. |
| **BPMN tests** | Kogito test framework | All process paths | Each BPMN process tested: happy path, rejection, escalation, timer events. |
| **Security tests** | Custom + RestAssured | All roles × critical endpoints | Verify 401/403 responses for unauthenticated/unauthorized requests. Also verify CSRF rejection when `X-CSRF-Token` is missing or mismatched. |

### 17.2 Frontend (Vue 3)

| Layer | Tool | Coverage Target | Description |
|-------|------|----------------|-------------|
| **Unit tests** | Vitest + Vue Test Utils | ≥ 80% line coverage | Composables (`useAuth`, `useCan`, `useTheme`, `useLocale`), stores (all Pinia stores), utility functions (`cn`, `formatDate`, validators). |
| **Component tests** | Vitest + Vue Test Utils + `@pinia/testing` | ≥ 70% of components | Render with `createTestingPinia`, mock API layer, verify 4-state handling (loading/empty/error/success), verify `v-permission` behavior. |
| **E2E tests** | Playwright | All critical user flows | Full journeys: SSO login (cookie-based), leave request → approval, cancellation, permission enforcement, i18n switching, dark mode toggle. |

**Testing priorities**: focus on critical paths, edge cases, and error states. The ≥80% line coverage gate remains for CI.

### 17.3 Critical Test Scenarios (E2E)

1. **Employee vacation request flow**: Login → Dashboard → New request → See conflict warning → Submit.
2. **Manager approval flow**: Login → Approval queue → Review team calendar → Approve.
3. **Cancellation flow**: Approved leave → Cancel → Manager approves → Balances restored.
4. **Medical document access control**: Upload certificate → Manager cannot view → HR Medical can view → Audit logged.
5. **Policy versioning**: Policy v1 → submit leave → publish v2 → new leave uses v2 → old leave shows v1.
6. **SSO login (cookie-based)**: Google SSO → httpOnly cookies set → `/auth/me` succeeds → Dashboard loads. No token in URL.
7. **Rate limiting**: Rapid-fire logins → 429 → UI shows retry delay.
8. **i18n**: Switch locale → all UI text, dates, numbers update immediately.
9. **Dark mode**: Toggle theme → `.dark` class applies → persists across reloads.
10. **Four-state coverage**: Skeleton → empty → error → success states render correctly.
11. **Form validation**: Empty submit → inline errors → fix → button disabled during submission → success.
12. **Error boundary**: Rendering error in child → fallback UI → Retry recovers.

### 17.4 CI/CD Gates

- All unit tests must pass.
- Line coverage ≥ 80% for backend, ≥ 80% for frontend (enforced by build).
- E2E tests (smoke suite) must pass.
- Linting (ESLint + Oxlint) must pass with zero errors.
- OpenAPI spec must be valid and up-to-date.
- `vue-tsc --noEmit` must pass (no type errors).

---

## 18. Deployment & Infrastructure

### 18.1 Docker Compose (Development)

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: hr_leave
      POSTGRES_USER: hr_app
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  rabbitmq:
    image: rabbitmq:3.13-management
    environment:
      RABBITMQ_DEFAULT_USER: hr_app
      RABBITMQ_DEFAULT_PASS: ${RABBITMQ_PASSWORD}
    ports:
      - "5672:5672"
      - "15672:15672"

  api:
    build: ./hr-leave-management-api
    environment:
      QUARKUS_DATASOURCE_JDBC_URL: jdbc:postgresql://postgres:5432/hr_leave
      QUARKUS_DATASOURCE_USERNAME: hr_app
      QUARKUS_DATASOURCE_PASSWORD: ${DB_PASSWORD}
      QUARKUS_RABBITMQ_URL: amqp://hr_app:${RABBITMQ_PASSWORD}@rabbitmq:5672
      SMTP_USERNAME: ${SMTP_USERNAME}
      SMTP_PASSWORD: ${SMTP_PASSWORD}
      JWT_PRIVATE_KEY_PATH: /etc/keys/privateKey.pem
      JWT_PUBLIC_KEY_PATH: /etc/keys/publicKey.pem
      ENCRYPTION_KEY: ${ENCRYPTION_KEY}
    volumes:
      - docs:/var/lib/hr-leave/documents
      - ./keys:/etc/keys:ro
    ports:
      - "8080:8080"

  frontend:
    build: ./hr-leave-management-front
    ports:
      - "3000:3000"

  nginx:
    image: nginx:alpine
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./certs:/etc/nginx/certs:ro
    ports:
      - "443:443"
    depends_on:
      - api
      - frontend
```

### 18.2 Production Considerations

- PostgreSQL on managed service (AWS RDS, Cloud SQL) with encrypted storage.
- RabbitMQ as a cluster or managed service.
- JWT keys and encryption keys stored in a secrets manager (Vault, AWS Secrets Manager).
- Kubernetes for orchestration if multi-instance scaling is required.
- Health endpoints (`/q/health/live`, `/q/health/ready`) for liveness and readiness probes.

### 18.3 Environment Variables

| Variable | Description |
|----------|-------------|
| `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USERNAME`, `DB_PASSWORD` | Database connection |
| `RABBITMQ_URL` | RabbitMQ connection string |
| `SMTP_HOST`, `SMTP_PORT`, `SMTP_USERNAME`, `SMTP_PASSWORD` | Email delivery |
| `JWT_PRIVATE_KEY_PATH`, `JWT_PUBLIC_KEY_PATH` | JWT signing keys (RS256) |
| `ENCRYPTION_KEY` | AES-256 key for document encryption (base64, 32 bytes) |
| `FRONTEND_URL` | Frontend URL for CORS and OAuth redirects |
| `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` | Google OAuth credentials |
| `ZOHO_CLIENT_ID`, `ZOHO_CLIENT_SECRET` | Zoho OAuth credentials |
| `APP_DOMAIN` | Public domain of the application |

---

## 19. Future Phases / Backlog

### Phase 2 (Next Priority)

| # | Feature | Notes |
|---|---------|-------|
| 1 | Substitution/Backup person when leave approved | Visible on team calendar |
| 2 | Malware scanning for uploads | ClamAV or cloud API |
| 3 | Legacy data import | CSV/Excel for historical data |
| 4 | Calendar integration | Google Calendar / Outlook two-way sync |
| 5 | Payroll integration | Outbound API/webhook |
| 6 | Public holiday auto-import | Government APIs or ics feeds |
| 7 | DMN editor in UI | HR Policy Admin can edit decision tables |
| 8 | Advanced analytics | Absence trends, predictive staffing |

### Phase 3 (Long-term)

| # | Feature | Notes |
|---|---------|-------|
| 9 | Mobile application | React Native or PWA |
| 10 | Multi-country support | Non-Spanish jurisdictions |
| 11 | AI-assisted conflict resolution | Optimal leave dates |
| 12 | Advanced delegation rules | Auto-delegate based on org chart |
| 13 | Additional SSO providers | Azure AD, Okta |

---

## Appendix A: DMN Decision Table Example

### `vacation-entitlement.dmn` (logic)

| Seniority (years) | Contract Type | Annual Hours | Convenio | Base Days | Additional |
|-------------------|---------------|-------------|----------|-----------|------------|
| < 1 | FULL_TIME | — | — | 30 (prorated) | 0 |
| 1–5 | FULL_TIME | — | — | 30 | 0 |
| > 5 | FULL_TIME | — | — | 30 | +2 |
| Any | PART_TIME | < 1000 | — | 30 × (hours/1780) | 0 |
| Any | PART_TIME | ≥ 1000 | — | 30 × (hours/1780) | 0 |
| > 10 | FULL_TIME | — | METAL | 30 | +3 |
| > 15 | FULL_TIME | — | CONSULTING | 30 | +5 |

## Appendix B: Permission Code Reference

```
# Leave management
leave.request.own           leave.request.submit         leave.request.cancel
leave.view.own              leave.view.team              leave.view.department
leave.view.all              leave.view.medical           leave.view.payroll
leave.approve.team          leave.approve.hr             leave.approve.medical
leave.override              leave.process
leave.balance.view.own      leave.balance.view.team      leave.balance.view.department
leave.balance.edit

# Documents
document.upload.own         document.view.own            document.view.all
document.delete.own         document.delete.all          document.request

# Medical
medical.view                medical.approve              medical.document.access

# Employees
employee.view.own           employee.view.basic          employee.view.all
employee.edit               employee.create

# Policies
leave_type.manage           policy.create                policy.edit
policy.publish              policy.version               policy.view
policy.view.history

# Calendar
calendar.view.team          calendar.view.company        calendar.manage
calendar.bulk.assign

# Reports
report.view.all             report.view.department       report.payroll.export

# Audit
audit.view

# System
user.manage                 user.role.assign             tenant.manage
integration.configure       system.config                role.manage

# Notifications
notification.receive.team   notification.manage

# Approval delegation
approval.delegate           approval.escalated
```

## Appendix C: Quarkus Extensions (Maven Dependencies)

```xml
<!-- Core -->
<dependency><groupId>io.quarkus</groupId><artifactId>quarkus-arc</artifactId></dependency>
<dependency><groupId>io.quarkus</groupId><artifactId>quarkus-rest-jackson</artifactId></dependency>
<dependency><groupId>io.quarkus</groupId><artifactId>quarkus-hibernate-orm-panache</artifactId></dependency>
<dependency><groupId>io.quarkus</groupId><artifactId>quarkus-jdbc-postgresql</artifactId></dependency>
<dependency><groupId>io.quarkus</groupId><artifactId>quarkus-hibernate-validator</artifactId></dependency>

<!-- Auth -->
<dependency><groupId>io.quarkus</groupId><artifactId>quarkus-smallrye-jwt</artifactId></dependency>
<dependency><groupId>io.quarkus</groupId><artifactId>quarkus-smallrye-jwt-build</artifactId></dependency>
<dependency><groupId>io.quarkus</groupId><artifactId>quarkus-oidc</artifactId></dependency>

<!-- Kogito (DMN + BPMN) -->
<dependency><groupId>org.kie.kogito</groupId><artifactId>kogito-quarkus</artifactId></dependency>
<dependency><groupId>org.kie.kogito</groupId><artifactId>kogito-quarkus-decisions</artifactId></dependency>
<dependency><groupId>org.kie.kogito</groupId><artifactId>kogito-quarkus-processes</artifactId></dependency>

<!-- Messaging -->
<dependency><groupId>io.quarkus</groupId><artifactId>quarkus-smallrye-reactive-messaging-rabbitmq</artifactId></dependency>

<!-- Email -->
<dependency><groupId>io.quarkus</groupId><artifactId>quarkus-mailer</artifactId></dependency>

<!-- Scheduling -->
<dependency><groupId>io.quarkus</groupId><artifactId>quarkus-scheduler</artifactId></dependency>

<!-- Observability -->
<dependency><groupId>io.quarkus</groupId><artifactId>quarkus-smallrye-openapi</artifactId></dependency>
<dependency><groupId>io.quarkus</groupId><artifactId>quarkus-smallrye-health</artifactId></dependency>
<dependency><groupId>io.quarkus</groupId><artifactId>quarkus-cache</artifactId></dependency>

<!-- Testing -->
<dependency><groupId>io.quarkus</groupId><artifactId>quarkus-junit5</artifactId><scope>test</scope></dependency>
<dependency><groupId>io.rest-assured</groupId><artifactId>rest-assured</artifactId><scope>test</scope></dependency>
<dependency><groupId>io.quarkus</groupId><artifactId>quarkus-junit5-mockito</artifactId><scope>test</scope></dependency>
```

## Appendix D: Frontend NPM Dependencies

```bash
# Core (already in project)
npm install                         # vue, vue-router, pinia

# Tailwind CSS v4
npm install -D tailwindcss @tailwindcss/vite

# Icons
npm install lucide-vue-next

# Forms
npm install vee-validate zod @vee-validate/zod

# i18n
npm install vue-i18n

# Security
npm install dompurify
npm install -D @types/dompurify

# Fonts (self-hosted)
npm install @fontsource/inter @fontsource/dm-sans

# Utilities
npm install clsx tailwind-merge

# Testing
npm install -D @pinia/testing
```

### Tailwind Setup

`vite.config.ts`:
```typescript
import tailwindcss from '@tailwindcss/vite'
export default defineConfig({
  plugins: [vue(), tailwindcss()],
})
```

`src/assets/main.css`:
```css
@import "tailwindcss";

@theme {
  --color-primary-50: oklch(0.97 0.01 250);
  --color-primary-500: oklch(0.55 0.18 250);
  --color-primary-900: oklch(0.25 0.08 250);
  --color-accent-500: oklch(0.70 0.19 145);
  --color-neutral-50: oklch(0.97 0.005 250);
  --color-neutral-950: oklch(0.15 0.02 250);
  --font-sans: 'Inter Variable', system-ui, -apple-system, 'Segoe UI', Roboto, sans-serif;
  --font-heading: 'DM Sans Variable', var(--font-sans);
}

.dark {
  --color-primary-500: oklch(0.65 0.15 250);
  --color-accent-500: oklch(0.75 0.17 145);
}
```

### Font Loading

`src/main.ts`:
```typescript
import '@fontsource/inter/variable.css'
import '@fontsource/dm-sans/variable.css'
```

---

> **End of Specification**  
> For questions, clarifications, or change requests, contact the architecture team.
