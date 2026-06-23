# HR Leave Management — API Design Specification  

> **Version**: 1.0  
> **Base URL**: `https://<domain>/api/v1`  
> **Audience**: Backend developers implementing Quarkus REST endpoints  
> **References**: [SPEC.md](./SPEC.md), [UX-DESIGN.md](./UX-DESIGN.md)  

---

## Table of Contents  

1. [Design Principles](#1-design-principles)  
2. [Conventions](#2-conventions)  
3. [Authentication & CSRF](#3-authentication--csrf)  
4. [Error Handling — RFC 7807](#4-error-handling--rfc-7807)  
5. [Pagination](#5-pagination)  
6. [Filtering, Sorting & Sparse Fieldsets](#6-filtering-sorting--sparse-fieldsets)  
7. [ETags & Optimistic Concurrency](#7-etags--optimistic-concurrency)  
8. [Rate Limiting](#8-rate-limiting)  
9. [Versioning & Deprecation](#9-versioning--deprecation)  
10. [Cross-Cutting Headers](#10-cross-cutting-headers)  
11. [Endpoint Catalog](#11-endpoint-catalog)  
    - [11.1 Authentication](#111-authentication)  
    - [11.2 Profile](#112-profile)  
    - [11.3 Leave Types](#113-leave-types)  
    - [11.4 Eligible Leave Types](#114-eligible-leave-types)  
    - [11.5 Leave Requests](#115-leave-requests)  
    - [11.6 Leave Requests — Calendar](#116-leave-requests--calendar)  
    - [11.7 Approvals](#117-approvals)  
    - [11.8 Balances](#118-balances)  
    - [11.9 Documents](#119-documents)  
    - [11.10 Employees](#1110-employees)  
    - [11.11 Holiday Calendars](#1111-holiday-calendars)  
    - [11.12 Policies](#1112-policies)  
    - [11.13 Reports](#1113-reports)  
    - [11.14 Audit Log](#1114-audit-log)  
    - [11.15 Notifications](#1115-notifications)  
    - [11.16 GDPR](#1116-gdpr)  
    - [11.17 Convenios](#1117-convenios)  
    - [11.18 Search](#1118-search)  
    - [11.19 Admin — Users & Roles](#1119-admin--users--roles)  
    - [11.20 Admin — Tenants](#1120-admin--tenants)  
    - [11.21 Admin — Integrations](#1121-admin--integrations)  
12. [SSE Notification Stream](#12-sse-notification-stream)  
13. [Consistency Notes](#13-consistency-notes)  

---

## 1. Design Principles  

| Principle | Implementation |
|-----------|---------------|
| **RESTful** | Resources named with nouns, HTTP methods express actions. Sub-resources for nested relationships. |
| **Predictable** | Consistent URL patterns, request/response shapes, error formats. No surprises. |
| **Self-describing** | HATEOAS links, `Content-Type` negotiation, OpenAPI 3.1 spec auto-generated from annotations. |
| **Secure by default** | httpOnly cookies + CSRF tokens. RBAC on every endpoint. Medical data access logged. |
| **Scalable** | Pagination on all list endpoints. Keyset for append-only streams, offset for navigable tables. ETags for caching. Async processing for slow reports. |
| **Evolvable** | Versioned under `/api/v1`. Backward-compatible additions only within a version. Breaking changes → new major version. |
| **Minimal over-the-wire** | Sparse fieldsets (`?fields=`), related resource inclusion (`?include=`), conditional requests (`If-None-Match`). |

---

## 2. Conventions  

### 2.1 URL Structure  

```
/api/v1/{resource}
/api/v1/{resource}/{id}
/api/v1/{resource}/{id}/{sub-resource}
/api/v1/{resource}/{id}/{sub-resource}/{subId}
```

- Resource names are **plural kebab-case** nouns: `leave-requests`, `holiday-calendars`, `leave-types`.  
- Sub-resources express ownership: `/leave-requests/{id}/documents`.  
- Actions that aren't CRUD use a verb suffix: `/leave-requests/{id}/submit`, `/approvals/{id}/approve`.  
- Bulk operations use a `/bulk-{action}` sub-resource: `/approvals/bulk-approve`, `/leave-requests/bulk`.  

### 2.2 HTTP Methods  

| Method | Usage |
|--------|-------|
| `GET` | Read. Never mutates state. Safe + idempotent. |
| `POST` | Create new resource or trigger an action. Not idempotent (except bulk/batch endpoints). |
| `PUT` | Full replacement of a resource. Idempotent. Requires `If-Match` for mutable resources. |
| `DELETE` | Remove a resource. Idempotent. |
| `PATCH` | Not used. Use `PUT` with full resource representation to keep semantics simple. |

### 2.3 Response Envelope  

Every successful list response follows this shape:

```json
{
  "data": [ ... ],
  "meta": {
    "page": 1,
    "limit": 25,
    "total": 287,
    "totalPages": 12
  },
  "links": {
    "self": "/api/v1/leave-requests?page=1&limit=25",
    "next": "/api/v1/leave-requests?page=2&limit=25",
    "prev": null,
    "first": "/api/v1/leave-requests?page=1&limit=25",
    "last": "/api/v1/leave-requests?page=12&limit=25"
  }
}
```

Single-resource responses omit the envelope; the resource is the root object:

```json
{
  "id": "uuid",
  "status": "APPROVED",
  ...
}
```

### 2.4 Date & Time Formats  

| Type | Format | Example |
|------|--------|---------|
| Date | `YYYY-MM-DD` (ISO 8601) | `"2026-08-01"` |
| Time | `HH:mm:ss` (ISO 8601) | `"14:30:00"` |
| Timestamp | ISO 8601 with UTC offset | `"2026-06-23T10:35:00Z"` |
| Duration | ISO 8601 duration | Not used; durations are expressed as decimal days |

### 2.5 IDs  

All resource IDs are **UUID version 7** (time-ordered). This provides:  
- Global uniqueness across tenants (even in cross-tenant queries).  
- Lexicographic sortability (time component first).  
- No sequential ID enumeration risk.  

---

## 3. Authentication & CSRF  

### 3.1 Cookie-Based Auth  

The backend sets three cookies after successful authentication:  

| Cookie | Attributes | Purpose |
|--------|-----------|---------|
| `access_token` | `httpOnly, Secure, SameSite=Strict, Path=/api, Max-Age=28800` | JWT for authorization (8h) |
| `refresh_token` | `httpOnly, Secure, SameSite=Strict, Path=/api/v1/auth, Max-Age=2592000` | Opaque token for renewal (30d) |
| `csrf_token` | `Secure, SameSite=Strict, Path=/` (NOT httpOnly — readable by JS) | CSRF double-submit token |

### 3.2 Frontend Requirements  

- All `fetch()` calls include `credentials: 'include'` (sends cookies).  
- All state-changing requests (`POST`, `PUT`, `DELETE`) include header `X-CSRF-Token: <value from csrf_token cookie>`.  
- The frontend never reads `access_token` or `refresh_token` cookies.  
- On 401, the fetch client calls `POST /auth/refresh` (cookies auto-sent), then retries the original request.  

### 3.3 Backend Enforcement  

- SmallRye JWT extracts the JWT from the `access_token` cookie.  
- A Quarkus `ContainerRequestFilter` on all non-GET/HEAD/OPTIONS requests verifies that `X-CSRF-Token` header matches the `csrf_token` cookie value.  
- Mismatch → `403 Forbidden` with error code `csrf_mismatch`.  

### 3.4 Endpoint Auth Column  

Throughout this document, endpoints are marked:  

| Label | Meaning |
|-------|---------|
| `None` | No authentication required |
| `Cookie` | Valid `access_token` cookie required |
| `Cookie + CSRF` | Valid cookie + `X-CSRF-Token` header required |

---

## 4. Error Handling — RFC 7807  

All errors use the **RFC 7807 Problem Details** format (`Content-Type: application/problem+json`).  

### 4.1 Structure  

```json
{
  "type": "https://api.example.com/problems/validation-error",
  "title": "Validation Error",
  "status": 400,
  "detail": "The request body contains invalid fields.",
  "instance": "/api/v1/leave-requests",
  "traceId": "01J1234ABCDEF789",
  "errors": [
    {
      "field": "end_date",
      "message": "End date must be after start date",
      "code": "date_range_invalid"
    }
  ]
}
```

### 4.2 Standard Problem Types  

| HTTP Status | `type` URI | Usage |
|-------------|-----------|-------|
| 400 | `.../validation-error` | Bean Validation failures, malformed JSON |
| 401 | `.../unauthorized` | Missing/expired `access_token` cookie |
| 403 | `.../forbidden` | Authenticated but insufficient permissions |
| 403 | `.../csrf-mismatch` | `X-CSRF-Token` header missing or wrong |
| 404 | `.../not-found` | Resource does not exist or is not in tenant scope |
| 409 | `.../conflict` | Overlapping leave dates, policy version conflict, balance exhausted |
| 412 | `.../precondition-failed` | `If-Match` ETag mismatch (concurrent edit) |
| 422 | `.../unprocessable-entity` | Semantically invalid (e.g., submitting an already-submitted leave) |
| 429 | `.../rate-limited` | Too many requests |
| 500 | `.../internal-error` | Unexpected server error (traceId logged) |
| 503 | `.../service-unavailable` | Downstream service unavailable |

### 4.3 Field-Level Errors  

The `errors` array is present for 400 and 422 responses. Each entry:  

| Field | Description |
|-------|-------------|
| `field` | JSON path to the invalid field (dot notation: `employee.contract_type`) |
| `message` | Human-readable error (localized via `Accept-Language` header) |
| `code` | Machine-readable error code for frontend logic (`balance_exhausted`, `policy_inactive`) |

---

## 5. Pagination  

### 5.1 Strategy Per Resource  

| Resource | Strategy | Rationale |
|----------|----------|-----------|
| `leave-requests` | **Offset** | Users expect page numbers; dataset bounded by leave history per employee |
| `employees` | **Offset** | HR browses pages; dataset bounded by company size |
| `approvals/pending` | **Offset** | Small dataset (pending items), expect page numbers |
| `audit-log` | **Keyset** | Append-only, potentially millions of rows, no page-jumping needed |
| `notifications` | **Keyset** | Infinite scroll, append-only consumption pattern |
| `policies` | **Offset** | Small dataset, rarely changes |
| `leave-types` | **No pagination** | < 50 items, always returned in full |
| `holiday-calendars` | **No pagination** | < 10 items per tenant |
| `documents` | **Offset** | Per-request bounded |
| `reports` | **No pagination** | Exports or pre-aggregated |
| `convenios` | **No pagination** | < 20 items per tenant |
| `roles` | **No pagination** | < 20 items |
| `search` | **Keyset** | Potentially large result sets |

### 5.2 Offset Pagination Parameters  

```
?page=1&limit=25
```

- `page`: 1-indexed. Default: `1`.  
- `limit`: Items per page. Default: `25`. Maximum: `100`.  

Response `meta`:

```json
{
  "page": 1,
  "limit": 25,
  "total": 287,
  "totalPages": 12
}
```

### 5.3 Keyset Pagination Parameters  

```
?after=<cursor>&limit=25
```

- `after`: Opaque cursor from previous response (base64-encoded). Omit for first page.  
- `limit`: Items per page. Default: `25`. Maximum: `100`.  

Response `meta`:

```json
{
  "limit": 25,
  "nextCursor": "eyJpZCI6IjAxSj...",
  "hasMore": true
}
```

The `nextCursor` is derived from the last item's sort key. When `hasMore` is `false`, there are no more pages.

### 5.4 Linked Resources Pagination  

When `?include=employee` is used, the `included` array contains all referenced resources and is NOT paginated separately. The parent resource's pagination applies.

---

## 6. Filtering, Sorting & Sparse Fieldsets  

### 6.1 Filtering  

All list endpoints support filtering via query parameters using a **rich operator syntax**:  

```
?{field}={operator}:{value}
```

**Operators**:  

| Operator | Syntax | Example | Notes |
|----------|--------|---------|-------|
| Equals | `field=value` | `?status=APPROVED` | Default; operator omitted |
| Not equals | `field=ne:value` | `?status=ne:DRAFT` | |
| In list | `field=in:a,b,c` | `?status=in:APPROVED,REJECTED` | |
| Greater than | `field=gt:value` | `?total_days=gt:5` | |
| Greater/equal | `field=gte:value` | `?start_date=gte:2026-01-01` | |
| Less than | `field=lt:value` | `?end_date=lt:2026-12-31` | |
| Less/equal | `field=lte:value` | `?total_days=lte:30` | |
| Contains | `field=like:text` | `?employee_name=like:John` | Case-insensitive substring |
| Between | `field=between:a,b` | `?start_date=between:2026-01-01,2026-12-31` | Inclusive range |

Multiple filters are AND-ed: `?status=APPROVED&leave_type=VACATION`.  

To OR filters on the same field, use `in`: `?status=in:APPROVED,PENDING`.  

**Standard filterable fields per resource** are listed in each endpoint's specification below.

### 6.2 Sorting  

```
?sort=field1,-field2
```

- `field` = ascending, `-field` = descending.  
- Multiple fields separated by comma.  
- Default sort per resource is specified in each endpoint.  

### 6.3 Sparse Fieldsets (JSON:API)  

```
?fields=id,status,start_date,end_date,total_days
```

Returns only the requested fields in each resource object. The `id` field is always included. This reduces payload size for mobile or dashboard contexts where only a subset of columns is needed.

### 6.4 Related Resource Inclusion  

```
?include=employee,approvals
```

Embeds related resources in the response:

```json
{
  "data": [
    {
      "id": "uuid",
      "status": "APPROVED",
      "employee": { "id": "uuid", "name": "John Smith" },
      "approvals": [
        { "id": "uuid", "status": "APPROVED", "approver_name": "Alice Manager" }
      ]
    }
  ]
}
```

Included resources are full objects (respecting `?fields=`). The `?include=` parameter supports:  
- `employee` — embeds basic employee info (name, department, employee_code) — never medical data  
- `approvals` — embeds approval step history  
- `leave_type` — embeds leave type info  
- `policy` — embeds the policy applied  
- `documents` — embeds document metadata (not file contents)  

---

## 7. ETags & Optimistic Concurrency  

### 7.1 ETag Generation  

Every `GET` response for a single resource includes an `ETag` header:

```
ETag: "abc123def456"
```

The ETag is derived from the resource's `updated_at` timestamp and version (if applicable, e.g., policies). For list responses, a weak ETag represents the collection state.

### 7.2 Conditional Requests  

| Header | Method | Behavior |
|--------|--------|----------|
| `If-None-Match: "etag"` | `GET` | If the ETag matches, return `304 Not Modified` with no body. Saves bandwidth for cached resources (leave types, policies, holiday calendars). |
| `If-Match: "etag"` | `PUT` | If the ETag doesn't match, return `412 Precondition Failed`. Prevents lost updates from concurrent edits. |

### 7.3 Resources Requiring `If-Match`  

| Resource | Endpoint | Rationale |
|----------|----------|-----------|
| `policies` | `PUT /policies/{id}` | HR Policy Admins may edit concurrently |
| `employees` | `PUT /employees/{id}` | HR Admins may edit concurrently |
| `leave-requests` (draft) | `PUT /leave-requests/{id}` | Employee may have two tabs open |
| `holiday-calendars` | `PUT /holiday-calendars/{id}` | HR Admins may edit concurrently |
| `convenios` | `PUT /convenios/{id}` | HR Admins may edit concurrently |

`POST` (create) endpoints do not require `If-Match`. `PUT` on resources without an ETag requirement (e.g., `PUT /notifications/{id}/read`) also do not.

---

## 8. Rate Limiting  

### 8.1 Two-Layer Defense  

| Layer | Scope | Implementation |
|-------|-------|---------------|
| **Nginx** | Per IP | Coarse rate limiting (100 req/min for API, 10 req/min for login). Blocks DDoS and brute-force. |
| **Quarkus** | Per user / per tenant | Fine-grained quotas based on JWT `sub` and `tenant_id` claims. Different limits per endpoint group. |

### 8.2 Quarkus Rate Limit Tiers  

| Tier | Limit | Applies To |
|------|-------|-----------|
| **Auth** | 10 req/min per IP | `/auth/login`, `/auth/*/callback` |
| **Read** | 300 req/min per user | All `GET` requests |
| **Write** | 60 req/min per user | `POST`, `PUT`, `DELETE` |
| **Upload** | 20 req/min per user | `POST /documents/upload` |
| **Export** | 5 req/min per tenant | `GET /reports/*`, `GET /gdpr/data-export` |

### 8.3 Rate Limit Headers  

Every response includes:

```
RateLimit-Limit: 300
RateLimit-Remaining: 287
RateLimit-Reset: 1719000060
```

- `RateLimit-Limit`: Maximum requests in the window.  
- `RateLimit-Remaining`: Remaining requests in the current window.  
- `RateLimit-Reset`: Unix timestamp when the window resets.  

When the limit is exceeded, the response is `429 Too Many Requests` with a `Retry-After` header (seconds until reset).

---

## 9. Versioning & Deprecation  

### 9.1 URL Versioning  

```
/api/v1/leave-requests
```

The version is in the URL path. A new major version (`/api/v2/`) is created only for breaking changes (removing fields, changing semantics, removing endpoints). Additive changes (new endpoints, new optional fields, new query parameters) are backward-compatible within a version.

### 9.2 Deprecation Headers  

When an endpoint or feature is scheduled for removal, responses include:

```
Deprecation: true
Sunset: Sat, 31 Dec 2027 23:59:59 GMT
```

The `Sunset` header gives the date after which the deprecated feature may be removed. Frontend developers should monitor these headers in non-production environments.

---

## 10. Cross-Cutting Headers  

### 10.1 Request Headers (Sent by Frontend)  

| Header | Required | Description |
|--------|----------|-------------|
| `Accept` | Yes | `application/json` |
| `Accept-Language` | Yes | `es` or `en` (defaults to `en`) |
| `X-CSRF-Token` | On POST/PUT/DELETE | CSRF token from cookie |
| `If-Match` | On PUT for mutable resources | ETag for optimistic concurrency |
| `If-None-Match` | Optional on GET | Conditional request for caching |
| `Prefer` | Optional | `respond-async` for long-running operations |

### 10.2 Response Headers (Returned by API)  

| Header | Always? | Description |
|--------|---------|-------------|
| `Content-Type` | Yes | `application/json` or `application/problem+json` |
| `X-Request-Id` | Yes | Unique request ID for tracing (UUID v7) |
| `X-Unread-Notifications` | Yes | Count of unread in-app notifications |
| `ETag` | On GET single resource | Entity tag for caching + concurrency |
| `Cache-Control` | On GET | `private, max-age=300` for cachable resources |
| `RateLimit-Limit` | Yes | Rate limit quota |
| `RateLimit-Remaining` | Yes | Remaining quota |
| `RateLimit-Reset` | Yes | Unix timestamp of quota reset |
| `Deprecation` | If applicable | `true` |
| `Sunset` | If applicable | Deprecation date |
| `Location` | On 201 Created | URL of the created resource |

---

## 11. Endpoint Catalog  

### 11.1 Authentication  

| Method | Path | Auth | CSRF | Description |
|--------|------|------|------|-------------|
| `POST` | `/auth/login` | None | No | Local username/password login. Sets cookies. |
| `POST` | `/auth/refresh` | Cookie (`refresh_token`) | Yes | Refreshes `access_token`. Sets new cookies. |
| `POST` | `/auth/logout` | Cookie | Yes | Clears auth cookies. Revokes refresh token. |
| `GET` | `/auth/google` | None | No | Redirects to Google OAuth consent screen. |
| `GET` | `/auth/google/callback` | None | No | Google OAuth callback. Sets cookies. Redirects to frontend. |
| `GET` | `/auth/zoho` | None | No | Redirects to Zoho OAuth consent screen. |
| `GET` | `/auth/zoho/callback` | None | No | Zoho OAuth callback. Sets cookies. Redirects to frontend. |
| `GET` | `/auth/me` | Cookie | No | Returns current user info, roles, permissions, employee profile. |

#### POST /auth/login  

```
Request:
{
  "email": "user@company.com",
  "password": "s3cret"
}

Response 200 (cookies set: access_token, refresh_token, csrf_token):
{
  "id": "01J1234ABC...",
  "email": "user@company.com",
  "firstName": "John",
  "lastName": "Doe",
  "locale": "es",
  "tenantId": "01J1234DEF...",
  "tenantName": "Acme Corporation",
  "roles": ["employee", "line_manager"],
  "permissions": ["leave.request.own", "leave.view.team", "..."],
  "employeeId": "01J1234GHI...",
  "employeeCode": "EMP001"
}
```

#### GET /auth/me  

```
Response 200:
{
  "id": "01J1234ABC...",
  "email": "user@company.com",
  "firstName": "John",
  "lastName": "Doe",
  "locale": "es",
  "tenantId": "01J1234DEF...",
  "tenantName": "Acme Corporation",
  "roles": ["employee", "line_manager"],
  "permissions": ["leave.request.own", "leave.view.team", "..."],
  "employeeId": "01J1234GHI...",
  "employeeCode": "EMP001",
  "departmentId": "01J1234JKL...",
  "departmentName": "Engineering",
  "managerId": "01J1234MNO...",
  "managerName": "Alice Manager",
  "avatarUrl": null
}

Response 401: (access_token cookie missing/expired)
{
  "type": ".../unauthorized",
  "title": "Unauthorized",
  "status": 401,
  "detail": "Access token missing or expired."
}
```

---

### 11.2 Profile  

| Method | Path | Auth | CSRF | Roles | Description |
|--------|------|------|------|-------|-------------|
| `GET` | `/profile` | Cookie | No | All authenticated | Get current user's profile |
| `PUT` | `/profile` | Cookie | Yes | All authenticated | Update own profile (locale, preferences) |

#### PUT /profile  

```
Request:
{
  "locale": "es"
}

Response 200:
{
  "id": "01J1234ABC...",
  "email": "user@company.com",
  "firstName": "John",
  "lastName": "Doe",
  "locale": "es"
}
```

Only `locale` is mutable by the user. Other fields (name, email) are managed by SSO or HR Admin.

---

### 11.3 Leave Types  

| Method | Path | Auth | CSRF | Roles | Description |
|--------|------|------|------|-------|-------------|
| `GET` | `/leave-types` | Cookie | No | All authenticated | List active leave types |
| `GET` | `/leave-types/{id}` | Cookie | No | All authenticated | Detail |
| `POST` | `/leave-types` | Cookie | Yes | HR Policy Admin | Create |
| `PUT` | `/leave-types/{id}` | Cookie | Yes | HR Policy Admin | Update |
| `DELETE` | `/leave-types/{id}` | Cookie | Yes | HR Policy Admin | Soft-delete (sets active=false) |

#### GET /leave-types  

```
Response 200:
{
  "data": [
    {
      "id": "01J1234...",
      "code": "VACATION",
      "name": "Vacaciones",
      "description": "Annual paid vacation leave",
      "paid": true,
      "active": true,
      "requiresDocumentation": false,
      "requiresApproval": true,
      "defaultMaxDays": 30,
      "icon": "sun"
    },
    {
      "id": "01J1234...",
      "code": "MARRIAGE",
      "name": "Matrimonio",
      "description": "Paid leave for marriage or registered partnership",
      "paid": true,
      "active": true,
      "requiresDocumentation": true,
      "requiresApproval": true,
      "defaultMaxDays": 15,
      "icon": "heart"
    }
  ],
  "links": {
    "self": "/api/v1/leave-types"
  }
}
```

**Notes**:  
- No pagination (always < 50 items).  
- Cacheable: `Cache-Control: private, max-age=300`. Supports `If-None-Match`.  
- The `icon` field maps to a Lucide icon name for the frontend.  

---

### 11.4 Eligible Leave Types  

| Method | Path | Auth | CSRF | Roles | Description |
|--------|------|------|------|-------|-------------|
| `GET` | `/employees/{id}/eligible-leave-types` | Cookie | No | Employee (own), Manager (team), HR Admin | List leave types the employee can request + eligibility + balance |

#### GET /employees/{id}/eligible-leave-types  

```
Response 200:
{
  "data": [
    {
      "leaveTypeId": "01J1234...",
      "code": "VACATION",
      "name": "Vacaciones",
      "icon": "sun",
      "eligible": true,
      "ineligibilityReason": null,
      "requiresDocumentation": false,
      "balance": {
        "entitlement": 30.0,
        "consumed": 5.0,
        "carriedOver": 3.0,
        "remaining": 28.0,
        "unit": "days"
      }
    },
    {
      "leaveTypeId": "01J1234...",
      "code": "PARENTAL_LEAVE",
      "name": "Permiso Parental",
      "icon": "baby",
      "eligible": false,
      "ineligibilityReason": "No children under 8 years old registered",
      "requiresDocumentation": true,
      "balance": null
    }
  ],
  "links": {
    "self": "/api/v1/employees/01J1234.../eligible-leave-types"
  }
}
```

**Behavior**:  
- Resolves the employee's convenio.  
- Evaluates DMN `leave-eligibility` decision for each active leave type.  
- Returns current balance for types with an entitlement (vacation, parental leave).  
- For leave types where eligibility depends on an event (marriage, death of relative), `eligible` is `true` (the event is validated at submission time via documents).  
- Caches per employee + leave type combination (invalidated on balance change or policy activation).  

---

### 11.5 Leave Requests  

| Method | Path | Auth | CSRF | Roles | Description |
|--------|------|------|------|-------|-------------|
| `GET` | `/leave-requests` | Cookie | No | Employee (own), Manager (team), HR (all) | List leave requests |
| `POST` | `/leave-requests` | Cookie | Yes | Employee | Create draft |
| `GET` | `/leave-requests/{id}` | Cookie | No | Employee (own), Manager (team), HR (all) | Detail |
| `PUT` | `/leave-requests/{id}` | Cookie | Yes | Employee (own, draft only) | Update draft |
| `DELETE` | `/leave-requests/{id}` | Cookie | Yes | Employee (own, draft only) | Delete draft |
| `POST` | `/leave-requests/{id}/submit` | Cookie | Yes | Employee (own) | Submit for approval |
| `POST` | `/leave-requests/{id}/cancel` | Cookie | Yes | Employee (own, approved) | Request cancellation |
| `POST` | `/leave-requests/bulk` | Cookie | Yes | HR Admin (calendar.bulk.assign) | Company-wide closure |

#### GET /leave-requests  

**Filterable fields**: `status`, `leave_type`, `start_date`, `end_date`, `employee_id` (HR only), `department_id` (HR/Manager).  

**Default sort**: `-submitted_at` (newest first).  

```
GET /api/v1/leave-requests?status=in:APPROVED,PENDING&leave_type=VACATION&page=1&limit=25&include=employee,leave_type

Response 200:
{
  "data": [
    {
      "id": "01J1234...",
      "status": "APPROVED",
      "startDate": "2026-08-01",
      "endDate": "2026-08-15",
      "startTime": null,
      "endTime": null,
      "totalDays": 15.0,
      "deductibleDays": 11.0,
      "notes": "Summer vacation",
      "submittedAt": "2026-06-23T10:35:00Z",
      "resolvedAt": "2026-06-24T09:15:00Z",
      "employee": {
        "id": "01J1234...",
        "firstName": "John",
        "lastName": "Smith",
        "employeeCode": "EMP001",
        "departmentName": "Engineering"
      },
      "leaveType": {
        "id": "01J1234...",
        "code": "VACATION",
        "name": "Vacaciones",
        "icon": "sun"
      }
    }
  ],
  "meta": {
    "page": 1,
    "limit": 25,
    "total": 4,
    "totalPages": 1
  },
  "links": {
    "self": "/api/v1/leave-requests?page=1&limit=25&include=employee,leave_type",
    "next": null,
    "prev": null,
    "first": "/api/v1/leave-requests?page=1&limit=25&include=employee,leave_type",
    "last": "/api/v1/leave-requests?page=1&limit=25&include=employee,leave_type"
  }
}
```

**Scoping rules**:  
- Employee without elevated roles: only own requests.  
- Line Manager: own requests + direct reports' requests.  
- Department Manager: own + department members' requests.  
- HR Admin / Legal / Auditor: all requests in tenant.  
- Medical data (diagnoses, certificate contents) is never included. HR Medical Specialist sees a `hasMedicalDocument: true` flag.  

#### POST /leave-requests (create draft)  

```
Request:
{
  "leaveTypeId": "01J1234...",
  "startDate": "2026-08-01",
  "endDate": "2026-08-15",
  "startTime": null,
  "endTime": null,
  "notes": "Summer vacation"
}

Response 201:
Location: /api/v1/leave-requests/01J1234...

{
  "id": "01J1234...",
  "status": "DRAFT",
  "leaveType": {
    "id": "01J1234...",
    "code": "VACATION",
    "name": "Vacaciones",
    "icon": "sun"
  },
  "startDate": "2026-08-01",
  "endDate": "2026-08-15",
  "startTime": null,
  "endTime": null,
  "totalDays": 15.0,
  "deductibleDays": 11.0,
  "balanceBefore": 28.0,
  "balanceAfter": 17.0,
  "conflicts": [
    {
      "employeeId": "01J1234...",
      "employeeName": "Jane Smith",
      "startDate": "2026-08-01",
      "endDate": "2026-08-07"
    }
  ],
  "notes": "Summer vacation",
  "createdAt": "2026-06-23T12:00:00Z"
}
```

**Validation**:  
- `start_date` ≤ `end_date` (400 if violated).  
- `start_date` must be ≥ today (400 if in past — except HR Medical can retroactively register sick leave via a separate internal endpoint).  
- Employee must be eligible for the leave type (422 with `ineligible_leave_type` code).  
- If the leave type is a one-time entitlement (marriage) and employee has already consumed it, returns 422 with `entitlement_exhausted`.  

**Conflict detection**: Queries approved leaves in the team within the same date range. Does NOT block creation — just informs.  

#### POST /leave-requests/{id}/submit  

```
Request: (empty body, or optionally)
{
  "notes": "Updated notes before submission"
}

Response 200:
{
  "id": "01J1234...",
  "status": "SUBMITTED",
  "policyId": "01J1234...",
  "policyVersion": 3,
  "approvalSteps": [
    {
      "id": "01J1234...",
      "stepOrder": 1,
      "approverRole": "MANAGER",
      "approverId": "01J1234...",
      "approverName": "Alice Manager",
      "status": "PENDING"
    },
    {
      "id": "01J1234...",
      "stepOrder": 2,
      "approverRole": "HR",
      "approverId": null,
      "approverName": null,
      "status": "PENDING"
    }
  ],
  "submittedAt": "2026-06-23T12:05:00Z"
}
```

**Backend actions on submit**:  
1. Resolve employee's convenio.  
2. Select matching policy (highest priority, effective date range).  
3. Freeze policy ID + version on the leave request.  
4. Evaluate DMN eligibility → if ineligible, return 422.  
5. Evaluate DMN duration → validate requested days ≤ max.  
6. Check documentation requirements → flag if documents missing.  
7. Start BPMN approval process (creates `approval_steps` rows).  
8. Publish `leave.request.submitted` event to RabbitMQ.  

**Validations**:  
- Status must be `DRAFT` (422 if already submitted).  
- If the leave type requires documentation and none uploaded, return 422 with `documents_missing` code and list of required document types. The employee can still submit and upload later — this is a soft validation (configurable per policy).  

#### GET /leave-requests/{id}  

```
GET /api/v1/leave-requests/01J1234...?include=approvals,documents

Response 200:
{
  "id": "01J1234...",
  "status": "APPROVED",
  "leaveType": {
    "id": "01J1234...",
    "code": "VACATION",
    "name": "Vacaciones",
    "icon": "sun"
  },
  "employee": {
    "id": "01J1234...",
    "firstName": "John",
    "lastName": "Smith",
    "employeeCode": "EMP001",
    "departmentName": "Engineering"
  },
  "startDate": "2026-08-01",
  "endDate": "2026-08-15",
  "startTime": null,
  "endTime": null,
  "totalDays": 15.0,
  "deductibleDays": 11.0,
  "notes": "Summer vacation",
  "policyId": "01J1234...",
  "policyVersion": 3,
  "policyName": "Spain General — Vacation 2026",
  "balanceImpact": {
    "before": 28.0,
    "after": 17.0
  },
  "submittedAt": "2026-06-23T10:35:00Z",
  "resolvedAt": "2026-06-24T09:15:00Z",
  "approvals": [
    {
      "id": "01J1234...",
      "stepOrder": 1,
      "approverRole": "MANAGER",
      "approverName": "Alice Manager",
      "status": "APPROVED",
      "comment": "Team coverage confirmed.",
      "decidedAt": "2026-06-24T09:00:00Z"
    },
    {
      "id": "01J1234...",
      "stepOrder": 2,
      "approverRole": "HR",
      "approverName": "Bob HR",
      "status": "APPROVED",
      "comment": "Approved per policy.",
      "decidedAt": "2026-06-24T09:15:00Z"
    }
  ],
  "documents": [
    {
      "id": "01J1234...",
      "documentType": "MEDICAL_CERTIFICATE",
      "fileName": "certificate.pdf",
      "uploadedAt": "2026-06-23T12:10:00Z"
    }
  ],
  "createdAt": "2026-06-23T10:30:00Z",
  "updatedAt": "2026-06-24T09:15:00Z"
}
```

**Access control**: Same scoping as the list endpoint. Medical documents in the `documents` array only include metadata (never file contents) and are only visible to HR Medical Specialist, Legal, and the employee themself.

#### POST /leave-requests/{id}/cancel  

```
Request:
{
  "reason": "Changed plans, no longer need leave"
}

Response 200:
{
  "id": "01J1234...",
  "status": "CANCELLATION_REQUESTED",
  "cancellationReason": "Changed plans, no longer need leave",
  "approvalSteps": [
    {
      "id": "01J1234...",
      "stepOrder": 1,
      "approverRole": "MANAGER",
      "status": "PENDING"
    }
  ]
}
```

**Constraints**:  
- Only cancellable if status is `APPROVED` and `start_date` is in the future.  
- Starts a BPMN cancellation workflow. On final approval, balances are restored and `leave.request.cancelled` event is published.  

#### POST /leave-requests/bulk  

```
Request:
{
  "leaveTypeId": "01J1234...",
  "startDate": "2026-12-24",
  "endDate": "2026-12-31",
  "employeeIds": ["01J1234...", "01J1234...", "..."],
  "notes": "Company-wide Christmas closure"
}

Response 202:
{
  "jobId": "01J1234...",
  "status": "PROCESSING",
  "totalEmployees": 48,
  "message": "Bulk leave creation in progress. Poll /leave-requests/bulk/{jobId} for status."
}
```

Creates individual leave requests for each employee. Processing is asynchronous for large lists (>50 employees). For ≤50, returns synchronously with `201` and the created request IDs.  

**GET /leave-requests/bulk/{jobId}**:

```
Response 200:
{
  "jobId": "01J1234...",
  "status": "COMPLETED",
  "totalEmployees": 48,
  "succeeded": 45,
  "failed": 3,
  "failures": [
    { "employeeId": "01J...", "reason": "Employee terminated" }
  ]
}
```

---

### 11.6 Leave Requests — Calendar  

| Method | Path | Auth | CSRF | Roles | Description |
|--------|------|------|------|-------|-------------|
| `GET` | `/leave-requests/calendar` | Cookie | No | Employee+, scoped by role | Calendar-optimized leave data for a date range |

#### GET /leave-requests/calendar  

```
GET /api/v1/leave-requests/calendar?from=2026-06-01&to=2026-06-30

Response 200:
{
  "data": [
    {
      "id": "01J1234...",
      "employeeId": "01J1234...",
      "employeeName": "Ana López",
      "leaveTypeCode": "VACATION",
      "leaveTypeName": "Vacaciones",
      "leaveTypeColor": "oklch(0.65 0.180 235)",
      "startDate": "2026-06-10",
      "endDate": "2026-06-20",
      "status": "APPROVED"
    },
    {
      "id": "01J1234...",
      "employeeId": "01J1234...",
      "employeeName": "Carlos Vega",
      "leaveTypeCode": "SICK_LEAVE_COMMON",
      "leaveTypeName": "Baja por enfermedad",
      "leaveTypeColor": "oklch(0.58 0.170 15)",
      "startDate": "2026-06-15",
      "endDate": "2026-06-18",
      "status": "APPROVED"
    }
  ],
  "holidays": [
    {
      "id": "01J1234...",
      "name": "San Juan",
      "date": "2026-06-24",
      "type": "public_holiday"
    }
  ],
  "links": {
    "self": "/api/v1/leave-requests/calendar?from=2026-06-01&to=2026-06-30"
  }
}
```

**Scoping**: Same as `GET /leave-requests` (own, team, department, or all depending on role). No pagination — bounded by the date range (max 366 days).  

**Response shape**: Flat list of leave periods (not grouped by day). The frontend groups by date for the calendar grid. Public holidays and bridge days for the applicable calendar are merged into the `holidays` array.  

**Filtering**: Supports `?employee_id=uuid` for filtering to specific team members (frontend checkbox filtering).  

---

### 11.7 Approvals  

| Method | Path | Auth | CSRF | Roles | Description |
|--------|------|------|------|-------|-------------|
| `GET` | `/approvals/pending` | Cookie | No | Manager, HR, etc. | Pending approvals for current user |
| `POST` | `/approvals/{stepId}/approve` | Cookie | Yes | Assigned approver | Approve a single step |
| `POST` | `/approvals/{stepId}/reject` | Cookie | Yes | Assigned approver | Reject a single step |
| `POST` | `/approvals/{stepId}/delegate` | Cookie | Yes | Assigned approver | Delegate to another user |
| `POST` | `/approvals/bulk-approve` | Cookie | Yes | Assigned approver | Approve multiple steps |
| `POST` | `/approvals/bulk-reject` | Cookie | Yes | Assigned approver | Reject multiple steps |

#### GET /approvals/pending  

**Filterable fields**: `leave_type`.  
**Default sort**: `-created_at` (oldest pending first).  

```
GET /api/v1/approvals/pending?leave_type=VACATION&page=1&limit=25&include=employee,leave_request

Response 200:
{
  "data": [
    {
      "id": "01J1234...",
      "leaveRequestId": "01J1234...",
      "stepOrder": 1,
      "approverRole": "MANAGER",
      "status": "PENDING",
      "employee": {
        "id": "01J1234...",
        "firstName": "Ana",
        "lastName": "López",
        "departmentName": "Sales"
      },
      "leaveRequest": {
        "id": "01J1234...",
        "leaveTypeCode": "MARRIAGE",
        "leaveTypeName": "Matrimonio",
        "startDate": "2026-09-10",
        "endDate": "2026-09-25",
        "totalDays": 15.0
      },
      "conflicts": [
        {
          "employeeId": "01J1234...",
          "employeeName": "Pedro Sánchez",
          "startDate": "2026-09-15",
          "endDate": "2026-09-20"
        }
      ],
      "pendingSince": "2026-06-23T10:35:00Z",
      "slaStatus": "OK"
    }
  ],
  "meta": {
    "page": 1,
    "limit": 25,
    "total": 2,
    "totalPages": 1
  },
  "links": { ... }
}
```

**SLA status**:  
- `OK` — less than 2 business days pending.  
- `REMINDER_SENT` — 2 business days, reminder email sent.  
- `ESCALATED` — 5 business days, escalated to next level.  

#### POST /approvals/{stepId}/approve  

```
Request:
{
  "comment": "Team coverage confirmed. Approved."
}

Response 200:
{
  "id": "01J1234...",
  "status": "APPROVED",
  "decidedAt": "2026-06-24T09:15:00Z",
  "nextStep": {
    "id": "01J1234...",
    "stepOrder": 2,
    "approverRole": "HR",
    "status": "PENDING"
  },
  "leaveRequestStatus": "UNDER_REVIEW"
}
```

If the approved step is the last step in the workflow, the leave request transitions to `APPROVED` and `leave.request.approved` event is published.  

#### POST /approvals/{stepId}/reject  

```
Request:
{
  "comment": "Insufficient team coverage during this period. Please choose different dates."
}

Response 200:
{
  "id": "01J1234...",
  "status": "REJECTED",
  "decidedAt": "2026-06-24T09:15:00Z",
  "leaveRequestStatus": "REJECTED"
}
```

The BPMN process terminates on rejection. Remaining steps are set to `SKIPPED`.  

#### POST /approvals/bulk-approve  

```
Request:
{
  "stepIds": ["01J1234...", "01J1234...", "01J1234..."],
  "comment": "Bulk approved — all team coverage confirmed."
}

Response 200:
{
  "approved": 3,
  "failed": 0,
  "failures": []
}
```

Processing is **synchronous** and **atomic per step**: each step is approved independently. If one fails (e.g., already decided by another approver), the others still succeed. Failures are reported in the `failures` array.

#### POST /approvals/bulk-reject  

Same shape as `bulk-approve` but sets status to `REJECTED`.

---

### 11.8 Balances  

| Method | Path | Auth | CSRF | Roles | Description |
|--------|------|------|------|-------|-------------|
| `GET` | `/balances/me` | Cookie | No | Employee | Current user's balances for current year |
| `GET` | `/balances/me?year=2025` | Cookie | No | Employee | Current user's balances for a specific year |
| `GET` | `/balances?employee_id=uuid&year=2026` | Cookie | No | Manager (team), HR (all) | Balances for specific employee |

#### GET /balances/me  

```
GET /api/v1/balances/me

Response 200:
{
  "data": [
    {
      "leaveTypeId": "01J1234...",
      "leaveTypeCode": "VACATION",
      "leaveTypeName": "Vacaciones",
      "year": 2026,
      "entitlement": 30.0,
      "consumed": 5.0,
      "carriedOver": 3.0,
      "carriedOverExpiry": "2026-03-31",
      "remaining": 28.0,
      "unit": "days"
    }
  ],
  "links": { ... }
}
```

**No pagination** (one entry per leave type with an entitlement). Types without entitlements (sick leave, force majeure) are excluded. Use `GET /employees/{id}/eligible-leave-types` for those.  

---

### 11.9 Documents  

| Method | Path | Auth | CSRF | Roles | Description |
|--------|------|------|------|-------|-------------|
| `POST` | `/documents/upload` | Cookie | Yes | Employee, HR Admin | Upload a document |
| `GET` | `/documents/{id}` | Cookie | No | Owner, HR Admin, Medical (if medical), Legal | Download document |
| `GET` | `/documents/{id}/meta` | Cookie | No | Owner, HR Admin, Medical, Legal | Document metadata (no file bytes) |
| `DELETE` | `/documents/{id}` | Cookie | Yes | Owner, HR Admin | Delete document |

#### POST /documents/upload  

```
Content-Type: multipart/form-data
Fields:
  - file: (binary, max 10 MB)
  - leaveRequestId: "01J1234..."
  - documentType: "MARRIAGE_CERTIFICATE"

Response 201:
Location: /api/v1/documents/01J1234...

{
  "id": "01J1234...",
  "leaveRequestId": "01J1234...",
  "documentType": "MARRIAGE_CERTIFICATE",
  "fileName": "marriage_cert.pdf",
  "mimeType": "application/pdf",
  "fileSizeBytes": 245760,
  "encrypted": true,
  "uploadedAt": "2026-06-23T12:10:00Z"
}
```

**Validation**:  
- MIME type must be `application/pdf`, `image/jpeg`, `image/png`, or `image/webp` (415 otherwise).  
- File size ≤ 10,485,760 bytes (413 otherwise).  
- `leaveRequestId` must reference an existing request owned by the uploader (403 otherwise).  
- Medical document types (`MEDICAL_CERTIFICATE`, `HOSPITAL_REPORT`) are flagged as sensitive.  

#### GET /documents/{id}  

Returns the file bytes with appropriate `Content-Type` and `Content-Disposition: attachment; filename="..."` headers.  

**Access control**:  
- Owner of the associated leave request → allowed.  
- HR Medical Specialist → allowed for medical documents.  
- HR Admin → allowed for non-medical documents.  
- Legal / Auditor → allowed (read-only, logged).  
- All other roles → 403.  
- Every access to a medical document is logged to `audit_log` with `DOCUMENT_ACCESSED`.  

#### GET /documents/{id}/meta  

Returns the document metadata without the file bytes (same shape as the upload response). Useful for listing documents attached to a leave request.

---

### 11.10 Employees  

| Method | Path | Auth | CSRF | Roles | Description |
|--------|------|------|------|-------|-------------|
| `GET` | `/employees` | Cookie | No | HR Admin, Manager (team), Payroll (basic) | List employees |
| `GET` | `/employees/{id}` | Cookie | No | HR Admin, Manager (if team), Payroll (basic) | Detail |
| `POST` | `/employees` | Cookie | Yes | HR Admin, Sys Admin | Create |
| `PUT` | `/employees/{id}` | Cookie | Yes | HR Admin | Update |

#### GET /employees  

**Filterable fields**: `department_id`, `contract_type`, `convenio_id`, `status`, `name` (like).  
**Sortable fields**: `last_name`, `employee_code`, `employment_start_date`, `seniority_years`.  
**Default sort**: `last_name`.  

```
GET /api/v1/employees?department_id=01J1234...&contract_type=FULL_TIME&page=1&limit=25

Response 200:
{
  "data": [
    {
      "id": "01J1234...",
      "userId": "01J1234...",
      "employeeCode": "EMP001",
      "firstName": "John",
      "lastName": "Smith",
      "email": "john@company.com",
      "status": "ACTIVE",
      "departmentId": "01J1234...",
      "departmentName": "Engineering",
      "contractType": "FULL_TIME",
      "workCenter": "Madrid",
      "convenioId": "01J1234...",
      "convenioName": "Metal Industry Agreement",
      "professionalCategory": "Senior Engineer",
      "employmentStartDate": "2020-03-15",
      "seniorityYears": 6.3,
      "managerId": "01J1234...",
      "managerName": "Alice Manager"
    }
  ],
  "meta": {
    "page": 1,
    "limit": 25,
    "total": 287,
    "totalPages": 12
  },
  "links": { ... }
}
```

**Scope rules**:  
- HR Admin, Legal, Auditor: all employees.  
- Manager: direct reports only.  
- Payroll Officer: all employees but with `?fields=` limited to payroll-relevant fields.  
- Employee: 403 (use `/profile` for own data).  

#### PUT /employees/{id}  

Requires `If-Match` header with current ETag. Returns `412` if the employee was modified since the last `GET`.  

```
Request:
{
  "contractType": "FULL_TIME",
  "workCenter": "Barcelona",
  "convenioId": "01J1234...",
  "professionalCategory": "Staff Engineer",
  "annualWorkingHours": 1780,
  "annualVacationEntitlement": 30,
  "managerId": "01J1234...",
  "departmentId": "01J1234..."
}

Response 200:
{
  "id": "01J1234...",
  ...
  "updatedAt": "2026-06-23T12:15:00Z"
}
```

---

### 11.11 Holiday Calendars  

| Method | Path | Auth | CSRF | Roles | Description |
|--------|------|------|------|-------|-------------|
| `GET` | `/holiday-calendars` | Cookie | No | All authenticated | List calendars |
| `POST` | `/holiday-calendars` | Cookie | Yes | HR Admin | Create calendar |
| `GET` | `/holiday-calendars/{id}` | Cookie | No | All authenticated | Calendar detail |
| `PUT` | `/holiday-calendars/{id}` | Cookie | Yes | HR Admin | Update calendar |
| `DELETE` | `/holiday-calendars/{id}` | Cookie | Yes | HR Admin | Delete calendar |
| `GET` | `/holiday-calendars/{id}/holidays` | Cookie | No | All authenticated | List holidays in calendar |
| `POST` | `/holiday-calendars/{id}/holidays` | Cookie | Yes | HR Admin | Add holiday |
| `POST` | `/holiday-calendars/{id}/holidays/import` | Cookie | Yes | HR Admin | Import from external source |
| `DELETE` | `/holiday-calendars/{id}/holidays/{holidayId}` | Cookie | Yes | HR Admin | Remove holiday |
| `POST` | `/holiday-calendars/{id}/bridge-days` | Cookie | Yes | HR Admin | Add bridge day |
| `DELETE` | `/holiday-calendars/{id}/bridge-days/{bridgeId}` | Cookie | Yes | HR Admin | Remove bridge day |

#### POST /holiday-calendars/{id}/holidays/import  

```
Request (JSON):
{
  "source": "BOE_2026_MADRID",
  "year": 2026
}

Response 202:
{
  "jobId": "01J1234...",
  "status": "PROCESSING",
  "message": "Importing holidays from BOE 2026 Madrid calendar. Poll /holiday-calendars/{id}/import-jobs/{jobId} for status."
}
```

Async import from external sources (government APIs, ICS feeds). The `source` parameter determines the integration.

---

### 11.12 Policies  

| Method | Path | Auth | CSRF | Roles | Description |
|--------|------|------|------|-------|-------------|
| `GET` | `/policies` | Cookie | No | HR Policy Admin, Legal | List policies |
| `GET` | `/policies/{id}` | Cookie | No | HR Policy Admin, Legal | Detail |
| `POST` | `/policies` | Cookie | Yes | HR Policy Admin | Create draft |
| `PUT` | `/policies/{id}` | Cookie | Yes | HR Policy Admin | Update draft |
| `POST` | `/policies/{id}/publish` | Cookie | Yes | HR Policy Admin | Activate policy |
| `GET` | `/policies/{id}/versions` | Cookie | No | HR Policy Admin, Legal | Version history |
| `POST` | `/policies/{id}/preview` | Cookie | Yes | HR Policy Admin | Evaluate policy against sample employee |

#### POST /policies  

```
Request:
{
  "leaveTypeId": "01J1234...",
  "convenioId": null,
  "name": "Spain General — Marriage Leave 2027",
  "description": "20 calendar days of paid leave for marriage (updated per 2027 labor law)",
  "effectiveFrom": "2027-01-01",
  "effectiveTo": null,
  "priority": 10,
  "dmnEligibilityRef": "leave-eligibility",
  "dmnDurationRef": "leave-duration",
  "bpmnProcessId": "approval-manager-only",
  "rules": [
    {
      "ruleCategory": "DURATION",
      "ruleType": "FIXED_DAYS",
      "ruleConfig": { "days": 20 },
      "priority": 0
    },
    {
      "ruleCategory": "DOCUMENTATION",
      "ruleType": "DOCUMENT_TYPE",
      "ruleConfig": { "accepted": ["MARRIAGE_CERTIFICATE"] },
      "priority": 0
    }
  ]
}

Response 201:
Location: /api/v1/policies/01J1234...

{
  "id": "01J1234...",
  "version": 1,
  "active": false,
  "status": "DRAFT",
  ...
}
```

#### POST /policies/{id}/publish  

```
Request: (empty body)

Response 200:
{
  "id": "01J1234...",
  "version": 1,
  "active": true,
  "status": "PUBLISHED",
  "publishedAt": "2026-06-23T12:20:00Z"
}
```

**Behavior**:  
- If a policy with the same `leave_type_id` + `convenio_id` combination is currently active with overlapping dates, it is deactivated (its `effective_to` is set to the new policy's `effective_from` minus 1 day).  
- Publishes `policy.activated` event.  

#### POST /policies/{id}/preview  

```
Request:
{
  "employeeId": "01J1234..."
}

Response 200:
{
  "policyId": "01J1234...",
  "policyVersion": 1,
  "policyName": "Spain General — Marriage Leave 2027",
  "employeeId": "01J1234...",
  "employeeName": "John Smith",
  "evaluation": {
    "eligible": true,
    "eligibilityChecks": [
      { "rule": "MIN_SENIORITY", "required": 12, "actual": 75, "passed": true },
      { "rule": "EMPLOYMENT_STATUS", "required": ["FULL_TIME", "PART_TIME"], "actual": "FULL_TIME", "passed": true }
    ],
    "maxDurationDays": 20,
    "documentsRequired": ["MARRIAGE_CERTIFICATE"],
    "balanceImpact": "NONE"
  }
}
```

Evaluates the draft policy against a real employee without activating it. Useful for HR Policy Admins to test policy changes before publishing.

---

### 11.13 Reports  

| Method | Path | Auth | CSRF | Roles | Description |
|--------|------|------|------|-------|-------------|
| `GET` | `/reports/{type}` | Cookie | No | HR Admin, Manager (scoped), Payroll | Generate report |
| `POST` | `/reports/{type}/generate` | Cookie | Yes | HR Admin, Payroll | Async report generation |
| `GET` | `/reports/jobs/{jobId}` | Cookie | No | Report requester | Poll async job status |

**Report types**: `vacation-balances`, `absences`, `sick-leave`, `absence-rate`, `payroll-impact`, `pending-documents`, `leave-by-type`.  

#### GET /reports/{type}  

**Supported `{type}` values**: `vacation-balances` | `absences` | `sick-leave` | `absence-rate` | `payroll-impact` | `pending-documents` | `leave-by-type`.  

```
GET /api/v1/reports/vacation-balances?from=2026-01-01&to=2026-12-31&department_id=01J1234...&format=csv

Response 200 (Content-Type: text/csv with Content-Disposition: attachment):
employee_code,first_name,last_name,department,entitlement,consumed,carried_over,remaining
EMP001,John,Smith,Engineering,30.0,5.0,3.0,28.0
...
```

**Format selection**:  
- `?format=json` → standard JSON response with `data` array.  
- `?format=csv` → CSV download with `Content-Disposition: attachment`.  

For reports expected to return >10,000 rows, the API returns `303 See Other` redirecting to `POST /reports/{type}/generate` for async processing. The client can optionally send `Prefer: respond-async` to always trigger async processing.  

#### POST /reports/{type}/generate  

```
Request:
{
  "from": "2026-01-01",
  "to": "2026-12-31",
  "departmentId": "01J1234...",
  "format": "csv"
}

Response 202:
Location: /api/v1/reports/jobs/01J1234...

{
  "jobId": "01J1234...",
  "status": "PROCESSING"
}
```

#### GET /reports/jobs/{jobId}  

```
Response 200 (complete):
{
  "jobId": "01J1234...",
  "status": "COMPLETED",
  "downloadUrl": "/api/v1/reports/jobs/01J1234.../download",
  "expiresAt": "2026-06-24T12:25:00Z"
}

Response 200 (still processing):
{
  "jobId": "01J1234...",
  "status": "PROCESSING",
  "progress": { "processed": 150, "total": 287 }
}

Response 200 (failed):
{
  "jobId": "01J1234...",
  "status": "FAILED",
  "error": "Report generation timed out"
}
```

Download URLs expire after 24 hours.

---

### 11.14 Audit Log  

| Method | Path | Auth | CSRF | Roles | Description |
|--------|------|------|------|-------|-------------|
| `GET` | `/audit-log` | Cookie | No | Legal, Auditor, Sys Admin | Query audit log |
| `GET` | `/audit-log/{entityType}/{entityId}` | Cookie | No | Legal, Auditor | Audit trail for a specific entity |

#### GET /audit-log  

**Filterable fields**: `entity_type`, `entity_id`, `action`, `performed_by`, `performed_at` (range).  
**Pagination**: Keyset (append-only table).  
**Default sort**: `-performed_at` (newest first).  

```
GET /api/v1/audit-log?entity_type=LeaveRequest&action=APPROVED&performed_at=gte:2026-01-01&limit=50

Response 200:
{
  "data": [
    {
      "id": "01J1234...",
      "entityType": "LeaveRequest",
      "entityId": "01J1234...",
      "action": "APPROVED",
      "changes": {
        "status": { "from": "SUBMITTED", "to": "APPROVED" },
        "resolvedAt": { "from": null, "to": "2026-06-24T09:15:00Z" }
      },
      "performedBy": "01J1234...",
      "performedByName": "Alice Manager",
      "performedAt": "2026-06-24T09:15:00Z",
      "ipAddress": "192.168.1.100"
    }
  ],
  "meta": {
    "limit": 50,
    "nextCursor": "eyJwZXJmb3JtZWRfYXQiOiIyMDI2LTA2LTI0VDA5OjE1OjAwWiIsImlkIjoiMDFKMTIzNC4uLiJ9",
    "hasMore": true
  },
  "links": {
    "self": "/api/v1/audit-log?entity_type=LeaveRequest&action=APPROVED&limit=50",
    "next": "/api/v1/audit-log?entity_type=LeaveRequest&action=APPROVED&limit=50&after=eyJw..."
  }
}
```

---

### 11.15 Notifications  

| Method | Path | Auth | CSRF | Roles | Description |
|--------|------|------|------|-------|-------------|
| `GET` | `/notifications` | Cookie | No | All authenticated | List user's notifications |
| `PUT` | `/notifications/{id}/read` | Cookie | Yes | All authenticated | Mark as read |
| `PUT` | `/notifications/read-all` | Cookie | Yes | All authenticated | Mark all as read |
| `GET` | `/notifications/stream` | Cookie | No | All authenticated | SSE stream of real-time notifications |

#### GET /notifications  

**Pagination**: Keyset (infinite scroll).  
**Default sort**: `-created_at` (newest first).  
**Notification types**: See [SPEC.md §11.1](./SPEC.md#111-notification-types) for the full catalog (LEAVE_SUBMITTED, LEAVE_APPROVED, LEAVE_REJECTED, CANCELLATION_REQUESTED, LEAVE_CANCELLED, APPROVAL_REMINDER, APPROVAL_ESCALATED, LEAVE_STARTS_SOON, DOCUMENT_REQUESTED, LEAVE_CONFLICT, BALANCE_LOW, RETURN_TO_WORK). The `type` field in each notification object matches these values.

```
Response 200:
{
  "data": [
    {
      "id": "01J1234...",
      "type": "LEAVE_APPROVED",
      "subject": "Your vacation leave has been approved",
      "body": "Your vacation leave from Aug 1 to Aug 15, 2026 has been approved by Alice Manager and HR.",
      "read": false,
      "createdAt": "2026-06-24T09:15:00Z",
      "link": "/leaves/01J1234..."
    }
  ],
  "meta": {
    "limit": 25,
    "nextCursor": "eyJ...",
    "hasMore": false
  },
  "links": { ... }
}
```

#### X-Unread-Notifications Header  

Every API response includes this header with the current unread count:

```
X-Unread-Notifications: 3
```

The frontend reads this from any response and updates the bell badge. No separate endpoint needed.

#### PUT /notifications/{id}/read  

```
Request: (empty body)
Response: 204 No Content
```

#### PUT /notifications/read-all  

```
Request: (empty body)
Response: 204 No Content
```

Marks all of the user's unread notifications as read.

---

### 11.16 GDPR  

| Method | Path | Auth | CSRF | Roles | Description |
|--------|------|------|------|-------|-------------|
| `GET` | `/gdpr/data-export` | Cookie | No | Employee (own) | Export own data |
| `POST` | `/gdpr/data-export/{employeeId}` | Cookie | Yes | HR Admin, Legal | Export employee data |
| `POST` | `/gdpr/deletion-request` | Cookie | Yes | Employee, HR Admin | Request erasure |

#### GET /gdpr/data-export  

```
Response 200 (async for large datasets):
{
  "jobId": "01J1234...",
  "status": "PROCESSING"
}
```

For small datasets (< 1 MB), returns synchronously with `Content-Type: application/json` and `Content-Disposition: attachment; filename="gdpr-export-2026-06-23.json"`. For larger datasets, returns `202 Accepted` with a job ID (poll via `/reports/jobs/{jobId}`).  

The exported file contains all personal data: user profile, employee record, leave request history, balances, uploaded documents (metadata only, not file contents), and audit log entries where the user is the subject.

---

### 11.17 Convenios  

| Method | Path | Auth | CSRF | Roles | Description |
|--------|------|------|------|-------|-------------|
| `GET` | `/convenios` | Cookie | No | HR Admin, HR Policy Admin | List convenios |
| `GET` | `/convenios/{id}` | Cookie | No | HR Admin, HR Policy Admin | Detail |
| `POST` | `/convenios` | Cookie | Yes | HR Admin | Create |
| `PUT` | `/convenios/{id}` | Cookie | Yes | HR Admin | Update |

#### GET /convenios  

No pagination (always < 20 items). Cacheable.

```
Response 200:
{
  "data": [
    {
      "id": "01J1234...",
      "code": "METAL",
      "name": "Metal Industry Agreement",
      "country": "ES",
      "effectiveFrom": "2024-01-01",
      "effectiveTo": null,
      "active": true
    }
  ],
  "links": { ... }
}
```

---

### 11.18 Search  

| Method | Path | Auth | CSRF | Roles | Description |
|--------|------|------|------|-------|-------------|
| `GET` | `/search` | Cookie | No | All authenticated (scoped by role) | Cross-entity search |

#### GET /search  

```
GET /api/v1/search?q=john&types=employee,leave_request&limit=10

Response 200:
{
  "data": [
    {
      "type": "employee",
      "id": "01J1234...",
      "title": "John Smith",
      "subtitle": "Engineering · EMP001",
      "link": "/hr/employees/01J1234..."
    },
    {
      "type": "leave_request",
      "id": "01J1234...",
      "title": "Vacation — Aug 1–15, 2026",
      "subtitle": "John Smith · Approved",
      "link": "/leaves/01J1234..."
    }
  ],
  "meta": {
    "limit": 10,
    "nextCursor": "eyJ...",
    "hasMore": false
  }
}
```

**Searchable entity types**: `employee`, `leave_request`, `policy`, `convenio`, `document`.  

**Scoping**:  
- Search results are always scoped to the user's permissions (employees see only themselves in results; managers see their team; HR sees all).  
- Medical data never appears in search results.  
- Results ranked by relevance (text match score).  

**Pagination**: Keyset (results can grow large).  

This endpoint powers the `⌘K` command palette in the header.

---

### 11.19 Admin — Users & Roles  

| Method | Path | Auth | CSRF | Roles | Description |
|--------|------|------|------|-------|-------------|
| `GET` | `/admin/users` | Cookie | No | Sys Admin | List all users in tenant |
| `POST` | `/admin/users` | Cookie | Yes | Sys Admin | Invite/create user |
| `GET` | `/admin/users/{id}` | Cookie | No | Sys Admin | User detail |
| `PUT` | `/admin/users/{id}` | Cookie | Yes | Sys Admin | Update user (status, name) |
| `PUT` | `/admin/users/{id}/roles` | Cookie | Yes | Sys Admin | Assign/revoke roles |
| `GET` | `/admin/roles` | Cookie | No | Sys Admin | List roles |
| `POST` | `/admin/roles` | Cookie | Yes | Sys Admin | Create custom role |
| `PUT` | `/admin/roles/{id}` | Cookie | Yes | Sys Admin | Update role permissions |

#### PUT /admin/users/{id}/roles  

```
Request:
{
  "roleIds": ["01J1234...", "01J1234..."]
}

Response 200:
{
  "userId": "01J1234...",
  "roles": [
    { "id": "01J1234...", "name": "Employee" },
    { "id": "01J1234...", "name": "Line Manager" }
  ]
}
```

**Constraints**:  
- A user must always have at least one role (422 if attempting to remove all).  
- There must always be at least one System Administrator in the tenant (422 if removing the last one).  

#### POST /admin/users  

```
Request:
{
  "email": "new.employee@company.com",
  "firstName": "New",
  "lastName": "Employee",
  "roleIds": ["01J1234..."]  // At minimum "Employee" role
}

Response 201:
{
  "id": "01J1234...",
  "email": "new.employee@company.com",
  "firstName": "New",
  "lastName": "Employee",
  "status": "ACTIVE",
  "authProvider": null,  // Will be set on first SSO login
  "roles": [{ "id": "01J1234...", "name": "Employee" }]
}
```

Creates the user record. The user completes onboarding via SSO on first login (matched by email).

---

### 11.20 Admin — Tenants  

| Method | Path | Auth | CSRF | Roles | Description |
|--------|------|------|------|-------|-------------|
| `GET` | `/admin/tenants` | Cookie | No | Super Admin (cross-tenant) | List all tenants |
| `POST` | `/admin/tenants` | Cookie | Yes | Super Admin | Create tenant |

#### POST /admin/tenants  

```
Request:
{
  "name": "Acme Corporation",
  "domain": "acme.com",
  "country": "ES"
}

Response 201:
{
  "id": "01J1234...",
  "name": "Acme Corporation",
  "domain": "acme.com",
  "country": "ES",
  "active": true,
  "createdAt": "2026-06-23T12:30:00Z"
}
```

**Backend bootstrap**: Creates default roles, default leave types, a "General" convenio, and a default Spain policy set. First SSO login from matching domain becomes the tenant's System Administrator.

---

### 11.21 Admin — Integrations  

| Method | Path | Auth | CSRF | Roles | Description |
|--------|------|------|------|-------|-------------|
| `GET` | `/admin/integrations` | Cookie | No | Sys Admin | Get integration configs |
| `PUT` | `/admin/integrations/sso` | Cookie | Yes | Sys Admin | Configure SSO providers |
| `PUT` | `/admin/integrations/smtp` | Cookie | Yes | Sys Admin | Configure SMTP |
| `PUT` | `/admin/integrations/general` | Cookie | Yes | Sys Admin | General tenant settings |

#### GET /admin/integrations  

```
Response 200:
{
  "sso": {
    "google": { "enabled": true, "clientId": "***-masked-***" },
    "zoho": { "enabled": false },
    "local": { "enabled": true }
  },
  "smtp": {
    "host": "smtp.company.com",
    "port": 587,
    "username": "noreply@company.com",
    "tls": true
  },
  "general": {
    "defaultLocale": "es",
    "workingDays": ["MONDAY", "TUESDAY", "WEDNESDAY", "THURSDAY", "FRIDAY"],
    "workingHoursPerDay": 8,
    "timezone": "Europe/Madrid"
  }
}
```

Sensitive values (client secrets, passwords) are masked. Update endpoints accept full or partial configurations.

---

## 12. SSE Notification Stream  

### 12.1 Endpoint  

```
GET /api/v1/notifications/stream
Accept: text/event-stream
```

### 12.2 Connection  

The frontend opens an `EventSource` connection after authentication:

```typescript
const eventSource = new EventSource('/api/v1/notifications/stream', {
  withCredentials: true  // sends httpOnly cookies
})
```

Since `EventSource` does not support custom headers, CSRF protection for SSE is handled by requiring the `access_token` cookie and validating it on connection. The stream is read-only (no state-changing operations), so CSRF risk is minimal.

### 12.3 Events  

```
event: notification
data: {"id":"01J1234...","type":"LEAVE_APPROVED","subject":"Your vacation leave has been approved","body":"...","createdAt":"2026-06-24T09:15:00Z","link":"/leaves/01J1234..."}

event: unread_count
data: {"count":5}

event: heartbeat
data: {"timestamp":"2026-06-23T12:30:00Z"}
```

- `notification` — a new notification was created. The frontend prepends it to the notification list and shows a toast.  
- `unread_count` — sent on connection and whenever the count changes. Updates the header bell badge.  
- `heartbeat` — sent every 30 seconds to keep the connection alive.  

### 12.4 Reconnection  

If the SSE connection drops, the frontend falls back to polling `GET /notifications` every 60 seconds until the SSE reconnects. `EventSource` handles automatic reconnection natively.

---

## 13. Consistency Notes  

### 13.1 Reconciliation with UX-DESIGN.md  

| UX Requirement | API Support |
|---------------|-------------|
| Dashboard — balances, upcoming, approvals count, recent | Separate calls to `/balances/me`, `/leave-requests`, `/approvals/pending`, `/notifications` (count via header) |
| New Request — leave type cards with eligibility + balance | `GET /employees/{id}/eligible-leave-types` |
| New Request — conflict warning on draft | Included in `POST /leave-requests` and `PUT /leave-requests/{id}` response |
| New Request — balance after preview | Included in draft response (`balanceBefore`, `balanceAfter`) |
| Leave Detail — approval timeline | `GET /leave-requests/{id}?include=approvals` |
| Team Calendar — color-coded leave bars | `GET /leave-requests/calendar` returns `leaveTypeColor` |
| Approval Queue — tabs with counts | Frontend computes counts from `GET /approvals/pending` (or use `?leave_type=` filter + `meta.total`) |
| Approval Queue — bulk approve | `POST /approvals/bulk-approve` |
| Policy Editor — preview | `POST /policies/{id}/preview` |
| Policy Editor — version history | `GET /policies/{id}/versions` |
| HR Employee List — filters | `GET /employees?contract_type=...&convenio_id=...` |
| Reports — async for large datasets | `POST /reports/{type}/generate` + `GET /reports/jobs/{jobId}` |
| ⌘K search | `GET /search?q=...` |
| Header bell — unread count | `X-Unread-Notifications` header on every response + SSE events |
| Profile — locale update | `PUT /profile` |

### 13.2 Reconciliation with SPEC.md Decisions  

| SPEC Requirement | Implementation |
|-----------------|---------------|
| API versioning under `/api/v1` | ✅ Applied |
| httpOnly cookies + CSRF double-submit | ✅ Applied |
| Health endpoints `/q/health/*` | ✅ Applied (Quarkus built-in) |
| OpenAPI 3.1 auto-generated from annotations | ✅ Applied (SmallRye OpenAPI) |
| Rate limiting at Nginx + Quarkus | ✅ Applied |
| Keyset pagination for append-only, offset for navigable tables | ✅ Applied |
| Rich filter operators | ✅ Applied |
| Sparse fieldsets | ✅ Applied (`?fields=`) |
| Related resource inclusion | ✅ Applied (`?include=`) |
| ETags + `If-Match` for concurrency | ✅ Applied |
| `X-Request-Id` tracing | ✅ Applied |
| `X-Unread-Notifications` count header | ✅ Applied |
| SSE for real-time notifications | ✅ Applied — replaces polling from SPEC.md §11.3 |
| `Prefer: respond-async` for long operations | ✅ Applied |
| RFC 7807 Problem Details for errors | ✅ Applied |
| `Location` header on 201 Created | ✅ Applied |
| Deprecation + Sunset headers | ✅ Applied |
| `Cache-Control` + `ETag` on cachable GETs | ✅ Applied |
| Rate limit headers on all responses | ✅ Applied |
| Per-user / per-tenant rate limits via JWT | ✅ Applied |
| Company-wide closures via `POST /leave-requests/bulk` | ✅ Applied |
| Bulk approval via separate `/bulk-approve` and `/bulk-reject` | ✅ Applied |
| `PUT /profile` for locale | ✅ Applied |
| `GET /leave-requests/calendar` for calendar data | ✅ Applied |
| `GET /employees/{id}/eligible-leave-types` | ✅ Applied |
| `POST /policies/{id}/preview` for policy testing | ✅ Applied |
| `GET /search` for cross-entity ⌘K search | ✅ Applied |

---

> **End of API Design Specification**  
> This document is implementable as Quarkus JAX-RS endpoints with the extensions listed in SPEC.md Appendix C.  
> For questions, contact the architecture team.
