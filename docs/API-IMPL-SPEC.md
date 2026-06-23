# HR Leave Management — API Implementation Technical Specification  

> **Version**: 1.0  
> **Audience**: Backend developers implementing the Quarkus API  
> **References**: [SPEC.md](./SPEC.md) · [API-DESIGN.md](./API-DESIGN.md) · [UX-DESIGN.md](./UX-DESIGN.md)  

---

## Table of Contents  

1. [Technology Stack](#1-technology-stack)  
2. [Layered Architecture](#2-layered-architecture)  
3. [Caching Strategy — Valkey](#3-caching-strategy--valkey)  
4. [Multi-Tenancy Implementation](#4-multi-tenancy-implementation)  
5. [Security Implementation](#5-security-implementation)  
6. [DMN & BPMN Integration](#6-dmn--bpmn-integration)  
7. [Pagination & Filtering Implementation](#7-pagination--filtering-implementation)  
8. [ETags & Optimistic Concurrency](#8-etags--optimistic-concurrency)  
9. [Event-Driven Architecture](#9-event-driven-architecture)  
10. [Error Handling — RFC 7807](#10-error-handling--rfc-7807)  
11. [Observability](#11-observability)  
12. [Configuration & Secrets](#12-configuration--secrets)  
13. [Testing Strategy](#13-testing-strategy)  
14. [Build & CI/CD](#14-build--cicd)  

---

## 1. Technology Stack  

### 1.1 Core Runtime  

| Component | Technology | Version | Purpose |
|-----------|-----------|---------|---------|
| Runtime | Quarkus | 3.36.x | JAX-RS, CDI, configuration, build |
| Language | Java | 21 | Virtual threads, pattern matching, records |
| Build | Maven | 3.9+ | Dependency management, packaging |

### 1.2 Persistence — Hybrid Approach  

**80% Panache / 20% native SQL**. Panache (Hibernate) handles CRUD, relationships, and multi-tenant filtering. Native SQL via `EntityManager#createNativeQuery` or raw `DataSource` handles reports, bulk operations, and performance-critical calendar queries. Flyway manages schema migrations for both.

| Component | Technology | Purpose |
|-----------|-----------|---------|
| ORM (default) | Hibernate ORM + Panache | CRUD, relationships, automatic tenant filtering, audit hooks |
| Native SQL | `@Inject EntityManager` → `createNativeQuery()` | Reports, bulk inserts, calendar data, complex aggregations |
| Raw JDBC | `@Inject DataSource` → `getConnection()` | Bulk batch operations (>50 rows), performance-critical batch inserts |
| Database | PostgreSQL 16 | Primary data store |
| Migrations | Flyway | Version-controlled schema migrations (works with both ORM and JDBC) |
| Connection Pool | Agroal (Quarkus default) | Connection pooling with health checks |

### 1.3 Caching — Valkey  

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Cache Server | Valkey 8.x | In-memory data store, Redis-compatible |
| Client | Quarkus Redis (`quarkus-redis-client`) | Non-blocking Redis client |
| Cache Annotations | `@CacheResult`, `@CacheInvalidate` | Declarative caching on service methods |
| Spring Cache compat | Quarkus Cache (`quarkus-cache`) | Caffeine for local, Redis for distributed |

**Why Valkey instead of Redis**: Valkey is the open-source fork after Redis's license change. It's drop-in compatible with the Quarkus Redis client, has active community maintenance, and avoids licensing concerns for commercial deployments. Same O(1) performance for cache operations.

### 1.4 Messaging  

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Broker | RabbitMQ 3.13+ | Domain events, async notifications |
| Client | SmallRye Reactive Messaging (`quarkus-smallrye-reactive-messaging-rabbitmq`) | Declarative `@Outgoing` / `@Incoming` channels |
| Outbox | Custom PostgreSQL outbox table + Debezium (future) | Reliable event publication |

### 1.5 Security  

| Component | Technology | Purpose |
|-----------|-----------|---------|
| JWT Validation | SmallRye JWT (`quarkus-smallrye-jwt`) | Extract JWT from httpOnly cookie |
| JWT Generation | SmallRye JWT Build (`quarkus-smallrye-jwt-build`) | Issue tokens on login/refresh |
| OIDC Client | Quarkus OIDC (`quarkus-oidc`) | Google & Zoho SSO flows |
| RBAC | `@RolesAllowed` + custom `PermissionEvaluator` | Fine-grained permission checks |
| CSRF | Custom `ContainerRequestFilter` | Double-submit cookie validation |
| Encryption | AES-256-GCM (JCA) | Document encryption at rest |

### 1.6 Rule Engine  

| Component | Technology | Purpose |
|-----------|-----------|---------|
| DMN Decisions | Kogito (`kogito-quarkus-decisions`) | Eligibility, duration, entitlement calculations |
| BPMN Processes | Kogito (`kogito-quarkus-processes`) | Approval workflows, escalations, timers |

### 1.7 Observability  

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Logging | Quarkus Logging (JBoss Log Manager) + Logstash JSON encoder | Structured JSON logs |
| Metrics | Micrometer (`quarkus-micrometer`) + Prometheus | Request latency, cache hit rates, error rates |
| Tracing | OpenTelemetry (`quarkus-opentelemetry`) | Distributed tracing across services |
| Health | SmallRye Health (`quarkus-smallrye-health`) | Liveness, readiness, Valkey/DB/RabbitMQ checks |
| API Docs | SmallRye OpenAPI (`quarkus-smallrye-openapi`) | Auto-generated OpenAPI 3.1 from annotations |

### 1.8 Mapping & Validation  

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Entity ↔ DTO | MapStruct 1.6+ | Compile-time type-safe mapping |
| Validation | Hibernate Validator (Bean Validation 3.0) | DTO validation, custom validators |
| JSON | Jackson (Quarkus default) | Serialization with `@JsonView` for field-level control |

---

## 2. Layered Architecture  

### 2.1 Package Structure  

```
es.mylair.leave
├── boundary/            # JAX-RS Resource classes (HTTP layer)
│   ├── LeaveRequestResource.java
│   ├── ApprovalResource.java
│   ├── PolicyResource.java
│   └── ...
├── dto/                 # Data Transfer Objects (request/response)
│   ├── request/
│   │   ├── CreateLeaveRequestDto.java
│   │   └── ...
│   ├── response/
│   │   ├── LeaveRequestResponse.java
│   │   └── ...
│   └── PageResponse.java  # Generic pagination envelope
├── service/             # Business logic (CDI beans)
│   ├── LeaveRequestService.java
│   ├── PolicyResolutionService.java
│   ├── EligibilityService.java
│   ├── BalanceService.java
│   └── ...
├── entity/              # Panache entities (Hibernate) — 80% of persistence
│   ├── LeaveRequest.java
│   ├── Employee.java
│   └── ...
├── repository/           # PanacheRepository + native query repositories
│   ├── LeaveRequestRepository.java     # Panache queries
│   ├── CalendarRepository.java        # Native SQL (calendar data)
│   ├── ReportRepository.java          # Native SQL (aggregations, CTEs)
│   ├── BulkOperationRepository.java   # Raw JDBC (batch inserts/updates)
│   └── AuditLogRepository.java        # Native SQL (keyset pagination)
├── mapper/              # MapStruct interfaces
│   ├── LeaveRequestMapper.java
│   └── ...
├── security/            # Auth infrastructure
│   ├── CsrfFilter.java
│   ├── PermissionEvaluator.java
│   ├── TenantResolver.java
│   └── SecurityIdentityProducer.java
├── event/               # RabbitMQ event publishers & consumers
│   ├── LeaveEventPublisher.java
│   ├── NotificationConsumer.java
│   └── ...
├── cache/               # Cache configuration & custom key generators
│   ├── CacheKeyGenerator.java
│   └── CacheConfig.java
├── config/              # Quarkus configuration, providers, filters
│   ├── ExceptionMappers.java
│   ├── ObjectMapperCustomizer.java
│   └── ...
└── util/                # Pure utility classes
    ├── EtagGenerator.java
    ├── PaginationUtils.java
    └── DateUtils.java
```

### 2.2 Request Lifecycle  

```
HTTP Request
    │
    ▼
[CookieAuthFilter]        — Extract JWT from access_token cookie
    │
    ▼
[CsrfFilter]              — Validate X-CSRF-Token on POST/PUT/DELETE
    │
    ▼
[TenantResolver]          — Extract tenant_id from JWT, set Hibernate filter
    │
    ▼
[RateLimitFilter]         — Check per-user quota, emit headers
    │
    ▼
[JAX-RS Resource]         — Deserialize DTO, delegate to service
    │
    ▼
[Service Layer]           — Business logic, DMN evaluation, policy resolution
    │
    ▼
[Repository Layer]        — Panache queries (default) or native SQL (reports, calendar, bulk)
    │
    ▼
[Entity → Response DTO]   — MapStruct mapping
    │
    ▼
HTTP Response             — Add X-Request-Id, RateLimit-*, X-Unread-*, ETag headers
```

### 2.3 Dependency Rules  

```
Resource ──► Service ──► Repository ──► Entity
   │            │
   ▼            ▼
  DTO         DMN Engine
              Policy Engine
              Event Publisher
              Cache (Valkey)
```

- **Resources never touch entities or repositories directly.**  
- **Services are the only layer that calls DMN decisions, publishes events, and manages transactions.**  
- **Services may use Panache repositories (default) or native SQL repositories** for reports, calendar data, and bulk operations. A single service method uses one approach — never both.  
- **Native SQL repositories** still go through the `EntityManager` (for result set mapping) or raw `DataSource` (for batch operations). They are named with their purpose: `CalendarRepository`, `ReportRepository`, `BulkOperationRepository`.
- **DTOs are immutable** (Java records where possible). Request DTOs use `@Valid` for nested validation.  
- **Mappers (MapStruct) are interfaces** — implementation generated at compile time, zero runtime overhead.  

### 2.4 Persistence Strategy: When to Use What  

This project uses a **hybrid persistence model**. The default is Panache, but native SQL is used deliberately for specific operations.  

| Operation | Tool | Rationale |
|-----------|------|-----------|
| **CRUD** (create, update, delete single entity) | Panache entity `.persist()`, `.update()` | Automatic tenant filtering, `@PreUpdate` audit hooks, optimistic locking via `@Version` |
| **Simple queries** (find by ID, find by status) | Panache `.find()`, `.list()` | Type-safe, tenant-filtered automatically, no manual JOINs |
| **Relationship navigation** (employee → manager → leaves) | Panache `@ManyToOne`, `@OneToMany` | Hibernate fetches related entities in one query (or with `JOIN FETCH`) |
| **Multi-entity queries with filters** (leave list with employee name, type, status) | PanacheRepository with `PanacheQuery` | Criteria-based predicate building, pagination built-in, tenant filter automatic |
| **Calendar data** (all team leaves in a date range) | Native SQL via `EntityManager#createNativeQuery` | 3-table JOIN with date range conditions — easier to write, optimize, and read as SQL. Maps to DTO directly. |
| **Reports** (vacation balances, absence rates, sick leave stats) | Native SQL via `EntityManager#createNativeQuery` | Complex aggregations (`SUM`, `COUNT`, window functions, CTEs) are impractical in JPQL |
| **Bulk insert** (company-wide closure creating 50+ leave requests) | Raw JDBC batch via `DataSource#getConnection` | Hibernate would issue N individual INSERTs; JDBC `addBatch()` / `executeBatch()` sends one round trip |
| **Bulk update** (HR updating balances for 100 employees) | Native SQL `UPDATE ... WHERE tenant_id = ? AND employee_id IN (...)` | Single SQL statement vs N Hibernate entity loads + modifications |
| **Audit log queries** (keyset pagination on append-only table) | Native SQL via `EntityManager#createNativeQuery` | Simple query pattern but performance-critical on millions of rows; full control over index usage |

**The rule**: If the operation fits naturally into "load entity, modify, save" or "find entities matching criteria", use Panache. If it's a report, a bulk operation, or a query with complex aggregations/CTEs, use native SQL. Never mix: a service method uses either Panache OR native SQL, not both.

**Tenant safety with native SQL**: Every native query MUST include `WHERE tenant_id = :tenantId`. The `TenantContext` CDI bean provides the tenant ID. A custom `@TenantFiltered` annotation on native query methods triggers a code review checklist item.

---

## 3. Caching Strategy — Valkey  

### 3.1 Why Valkey  

The API handles expensive operations that benefit from caching:  

| Operation | Cost | Cache Benefit |
|-----------|------|--------------|
| DMN eligibility evaluation | 50–200ms (decision table evaluation) | Reduce to <1ms |
| Policy resolution (convenio + priority matching) | 10–50ms (multi-table query) | Reduce to <1ms |
| Leave type list | 5–10ms | Reduce to <1ms, shared across all employees |
| Eligible leave types per employee | 100–300ms (DMN × 30+ types) | Reduce to <5ms |
| User permissions set | 10–30ms (role + permission join) | Reduce to <1ms |
| Holiday calendar data | 5–10ms | Reduce to <1ms, cache for hours |
| Rate limit counters | Must be sub-ms | Valkey atomic counters |

Without caching, a dashboard load (3–5 API calls) could take 500ms–1s. With Valkey, <50ms.

### 3.2 Cache Topology  

```
┌────────────────┐     ┌────────────────┐
│  Quarkus API   │────▶│    Valkey      │
│  Instance 1    │     │  (primary)     │
└────────────────┘     └───────┬────────┘
                               │
┌────────────────┐             │ Replication (optional)
│  Quarkus API   │─────────────┤
│  Instance 2    │             │
└────────────────┘     ┌───────┴────────┐
                        │    Valkey      │
                        │  (replica)     │
                        └────────────────┘
```

- **Single Valkey instance** is sufficient for this scale (<1000 req/s).  
- **Optional replica** for high-availability deployments.  
- Quarkus Redis client connects to Valkey via TCP (internal Docker network or Kubernetes service).  
- Keyspace is shared; tenant isolation is enforced by key naming (see §3.4).  

### 3.3 Cache Layers  

#### L1: Quarkus `@CacheResult` on Service Methods  

Declarative, annotation-driven caching for service-layer methods:  

```java
@ApplicationScoped
public class LeaveTypeService {

    @CacheResult(cacheName = "leave-types")
    public List<LeaveTypeDto> getActiveLeaveTypes(TenantId tenantId) {
        return repository.findActiveByTenant(tenantId);
    }
}
```

- Uses **Caffeine** (in-process, near-zero latency) as L1.  
- Caffeine is configured with `maximumSize=500`, `expireAfterWrite=5m`.  
- Automatic eviction on size limit; TTL ensures eventual consistency.  

#### L2: Valkey (Distributed Cache)  

For data shared across API instances or too large for Caffeine:  

```java
@ApplicationScoped
public class EligibilityService {

    @CacheResult(cacheName = "eligible-leave-types")
    public List<EligibleLeaveTypeDto> getEligibleTypes(EmployeeId employeeId) {
        // Expensive: evaluates DMN for each leave type
        return leaveTypes.stream()
            .map(lt -> evaluateEligibility(employeeId, lt))
            .toList();
    }
}
```

- Valkey handles entries that need to survive instance restarts or are shared across instances.  
- Caffeine is the first layer; Valkey backs it.  

### 3.4 Cache Key Design — Tenant-Scoped  

**Critical**: Every cache key includes the tenant ID to prevent cross-tenant data leakage.  

```
Pattern: {tenant_id}:{resource}:{identifier}

Examples:
  "01J1234:leave-types:active"
  "01J1234:policies:vacation:metal"
  "01J1234:eligible-types:01J5678"
  "01J1234:user-permissions:01JABCD"
  "01J1234:holiday-calendar:madrid:2026"
```

The key generator extracts `tenant_id` from the current security context (JWT claim):  

```java
@ApplicationScoped
public class TenantAwareCacheKeyGenerator implements CacheKeyGenerator {
    
    @Inject
    SecurityIdentity identity;
    
    @Override
    public Object generate(CacheInvocationContext ctx) {
        String tenantId = identity.getAttribute("tenant_id");
        Object[] params = ctx.getParameters();
        return "%s:%s:%s".formatted(tenantId, ctx.getMethodName(), Arrays.toString(params));
    }
}
```

### 3.5 Cache Invalidation Strategy  

#### Pattern 1: Write-Through Invalidation  

When a resource is modified, invalidate its cache entries:  

```java
@Transactional
@CacheInvalidate(cacheName = "leave-types")
public LeaveTypeDto updateLeaveType(UUID id, UpdateLeaveTypeDto dto) {
    var entity = repository.findById(id);
    mapper.update(dto, entity);
    return mapper.toDto(entity);
}
```

#### Pattern 2: Event-Driven Invalidation  

For cross-cutting invalidation (e.g., a policy change affects many employees' eligibility):  

```java
// Publisher — after publishing a new policy
@Outgoing("policy-events")
public void onPolicyPublished(Policy policy) {
    emitter.send(Message.of(new PolicyActivatedEvent(policy)));
}

// Consumer — invalidates eligibility cache for affected employees
@Incoming("policy-events")
public void invalidateEligibilityCache(PolicyActivatedEvent event) {
    // Delete all eligible-leave-types cache entries for employees under this convenio
    String pattern = "%s:eligible-types:*".formatted(event.tenantId());
    redisClient.del(pattern); // Valkey SCAN + DEL for pattern matching
}
```

#### Pattern 3: TTL-Based Expiry  

Every cache entry has a TTL. Even if invalidation is missed, data is never stale indefinitely:  

| Cache | TTL | Rationale |
|-------|-----|-----------|
| `leave-types` | 5 min | Rarely changes; invalidated on update |
| `eligible-types` | 5 min | Expensive to compute; invalidated on policy/balance change |
| `user-permissions` | 8 hours | Matches JWT expiry; invalidated on role change |
| `policies` | 1 hour | Versioned; invalidated on publish |
| `holiday-calendars` | 1 hour | Rarely changes; invalidated on update |
| `balances` | 1 min | Changes with every leave submission; short TTL for consistency |
| `rate-limits` | Per window | Valkey TTL matches the rate window (1 min) |

### 3.6 Data That MUST NOT Be Cached  

| Data | Reason |
|------|--------|
| **Medical documents / certificates** | Special category GDPR data. Never stored in cache. Always fetched from encrypted filesystem. |
| **Audit log entries** | Append-only; must be 100% consistent. Cache would serve stale data. |
| **Pending approval steps** | Real-time data; stale cache could cause duplicate approvals. |
| **Leave request drafts** | Owned by a single user; rarely read by others. Caching adds complexity for no benefit. |
| **Refresh tokens** | Must be checked against DB for revocation. |

### 3.7 Consistency Guarantees  

| Scenario | Behavior |
|----------|----------|
| **Employee submits leave, then checks balance** | Balance cache TTL is 1 min. Balance is also computed fresh in the submit response → the response is always correct. The next GET may be up to 1 min stale (acceptable for dashboards). |
| **HR publishes new policy** | Event-driven invalidation clears eligibility cache for all employees under the affected convenio. Next request recomputes eligibility with the new policy. |
| **Two employees view the same leave type list** | Both get the same cached value (within TTL). Immutable for the TTL duration. |
| **Valkey restarts** | All caches are cold. Next requests hit the database. No data loss — cache is ephemeral by design. |
| **Cache key collision across tenants** | Impossible — tenant ID is part of every key. |

### 3.8 Rate Limiting with Valkey  

Valkey provides atomic counters with TTL, ideal for rate limiting:  

```java
@ApplicationScoped
public class RateLimitService {
    
    private final ReactiveRedisDataSource redis;
    
    public boolean tryConsume(String userId, String tier, int limit, Duration window) {
        String key = "ratelimit:%s:%s:%d".formatted(userId, tier, Instant.now().getEpochSecond() / window.getSeconds());
        return redis.withConnection(conn ->
            conn.value(Long.class).get(key)
                .flatMap(val -> {
                    if (val != null && val >= limit) return Uni.createFrom().item(false);
                    return conn.value(Long.class).incr(key)
                        .flatMap(v -> conn.value(Long.class).expire(key, window.getSeconds()))
                        .map(v -> true);
                })
        ).await().indefinitely();
    }
}
```

Counters auto-expire when the window elapses. No cleanup job needed.

---

## 4. Multi-Tenancy Implementation  

### 4.1 Hibernate Tenant Resolver  

Quarkus provides `TenantResolver` for discriminator-column multi-tenancy:  

```java
@ApplicationScoped
public class HibernateTenantResolver implements TenantResolver {
    
    @Inject
    SecurityIdentity identity;
    
    @Override
    public String getDefaultTenantId() {
        return "PUBLIC"; // Fallback for unauthenticated requests (login page)
    }
    
    @Override
    public String resolveTenantId() {
        if (identity.isAnonymous()) return getDefaultTenantId();
        return identity.getAttribute("tenant_id");
    }
}
```

This adds `WHERE tenant_id = ?` to every Hibernate query automatically. No entity code needs to manage tenant filtering.

### 4.2 Tenant Context Propagation  

The JWT `tenant_id` claim is available throughout the request:  

```java
@RequestScoped
public class TenantContext {
    
    @Inject
    SecurityIdentity identity;
    
    public UUID getTenantId() {
        return UUID.fromString(identity.getAttribute("tenant_id"));
    }
    
    public UUID getUserId() {
        return UUID.fromString(identity.getPrincipal().getName());
    }
}
```

Injected into services via `@Inject TenantContext ctx`. This avoids passing tenant ID as a method parameter throughout the call chain.

### 4.3 Tenant Bootstrap  

When `POST /admin/tenants` creates a new tenant:  

1. Insert `tenants` row.  
2. Create default roles (`Employee`, `System Administrator`).  
3. Create default leave types (all standard codes).  
4. Create a "General" convenio.  
5. Create default Spain policy set for all leave types (DMN + BPMN references).  
6. Create a default Spain holiday calendar (national holidays).  

All in a single `@Transactional` method. If any step fails, the entire tenant creation rolls back.

---

## 5. Security Implementation  

### 5.1 JWT Cookie Extraction  

SmallRye JWT is configured to read from a cookie instead of the `Authorization` header:  

```properties
# application.properties
quarkus.smallrye-jwt.token-header=Cookie
quarkus.smallrye-jwt.token-cookie=access_token
mp.jwt.verify.publickey.location=/etc/keys/publicKey.pem
mp.jwt.verify.issuer=hr-leave-management
```

Quarkus validates the JWT signature, expiry, and issuer automatically.

### 5.2 CSRF Filter  

```java
@Provider
@Priority(Priorities.AUTHENTICATION + 100)
public class CsrfFilter implements ContainerRequestFilter {
    
    @Override
    public void filter(ContainerRequestContext ctx) {
        if (isStateChanging(ctx.getMethod())) {
            String headerToken = ctx.getHeaderString("X-CSRF-Token");
            String cookieToken = extractCookie(ctx, "csrf_token");
            
            if (headerToken == null || !MessageDigest.isEqual(
                    headerToken.getBytes(StandardCharsets.UTF_8),
                    cookieToken.getBytes(StandardCharsets.UTF_8))) {
                throw new WebApplicationException(
                    Response.status(403)
                        .entity(new ProblemDetail("csrf-mismatch", "CSRF token mismatch"))
                        .type("application/problem+json")
                        .build()
                );
            }
        }
    }
}
```

Uses `MessageDigest.isEqual()` for timing-attack-safe comparison.

### 5.3 Permission Evaluator  

```java
@ApplicationScoped
public class PermissionEvaluator {
    
    @Inject
    SecurityIdentity identity;
    
    @CacheResult(cacheName = "user-permissions")
    public Set<String> getPermissions(UUID userId) {
        // Loaded from DB on first call, cached in Valkey with TTL = JWT lifetime
        return userRoleRepository.findPermissionsByUserId(userId);
    }
    
    public boolean hasPermission(String permission) {
        return getPermissions(getCurrentUserId()).contains(permission);
    }
    
    public void requirePermission(String permission) {
        if (!hasPermission(permission)) {
            throw new ForbiddenException("Missing permission: " + permission);
        }
    }
}
```

### 5.4 Medical Data Access Logging  

```java
@Interceptor
@MedicalDataAccess
@Priority(Interceptor.Priority.APPLICATION)
public class MedicalDataAccessInterceptor {
    
    @Inject
    AuditLogService auditLog;
    
    @AroundInvoke
    public Object logAccess(InvocationContext ctx) throws Exception {
        Object result = ctx.proceed();
        auditLog.record(
            AuditAction.DOCUMENT_ACCESSED,
            getDocumentId(ctx.getParameters()),
            getCurrentUserId()
        );
        return result;
    }
}
```

Annotation `@MedicalDataAccess` on document download endpoints triggers mandatory audit logging.

---

## 6. DMN & BPMN Integration  

### 6.1 DMN Decision Evaluation  

DMN files are stored in `src/main/resources/dmn/` and loaded by Kogito at startup:  

```java
@ApplicationScoped
public class EligibilityService {
    
    @Inject
    @Named("leave-eligibility")
    DMNRuntime dmnRuntime;
    
    public EligibilityResult evaluate(UUID employeeId, String leaveTypeCode) {
        DMNContext ctx = dmnRuntime.newContext();
        ctx.set("Employee ID", employeeId.toString());
        ctx.set("Leave Type Code", leaveTypeCode);
        ctx.set("Seniority Months", getEmployeeSeniorityMonths(employeeId));
        ctx.set("Contract Type", getEmployeeContractType(employeeId));
        // ... set all input parameters ...
        
        DMNResult result = dmnRuntime.evaluateAll(
            dmnRuntime.getModels().get(0),
            ctx
        );
        
        boolean eligible = (boolean) result.getDecisionResultByName("Eligible").getResult();
        String reason = (String) result.getDecisionResultByName("Ineligibility Reason").getResult();
        
        return new EligibilityResult(eligible, reason);
    }
}
```

### 6.2 Policy Resolution  

```java
@ApplicationScoped
public class PolicyResolutionService {
    
    @CacheResult(cacheName = "policy-resolution")
    public Policy resolvePolicy(UUID leaveTypeId, UUID convenioId, LocalDate date) {
        // Priority-ordered query: convenio-specific first, then general fallback
        return policyRepository.findBestMatch(leaveTypeId, convenioId, date)
            .orElseThrow(() -> new PolicyNotFoundException(leaveTypeId, convenioId));
    }
}
```

### 6.3 BPMN Workflow Start  

```java
@ApplicationScoped
public class ApprovalWorkflowService {
    
    @Inject
    @Named("approval-standard")
    Process<? extends BpmnProcess> approvalProcess;
    
    public List<ApprovalStep> startWorkflow(LeaveRequest leaveRequest, Policy policy) {
        BpmnProcess process = approvalProcess.createInstance(policy.getBpmnProcessId());
        process.start();
        
        // Kogito creates ApprovalStep entities via the BPMN process
        // Return the created steps for the API response
        return approvalStepRepository.findByLeaveRequest(leaveRequest.getId());
    }
}
```

---

## 7. Pagination & Filtering Implementation  

### 7.1 Offset Pagination — Panache  

Used for entity-returning list endpoints (leave requests, employees, approvals):  

```java
@ApplicationScoped
public class LeaveRequestRepository implements PanacheRepository<LeaveRequest> {
    
    public PageResponse<LeaveRequestDto> findWithFilters(
            LeaveRequestFilter filter, int page, int limit) {
        
        var query = find("tenantId = ?1", TenantContext.getTenantId());
        
        // Dynamic filter building — type-safe, no SQL injection risk
        if (filter.status() != null) {
            query.filter("status", filter.status());
        }
        if (filter.leaveType() != null) {
            query.filter("leaveType.code", filter.leaveType());
        }
        
        query.page(Page.of(page - 1, limit));
        var panachePage = query.list();
        
        return PageResponse.of(
            panachePage.list().stream().map(mapper::toDto).toList(),
            page, limit, panachePage.count(), panachePage.pageCount()
        );
    }
}
```

### 7.2 Offset Pagination — Native SQL  

Used for report queries, calendar data, and any query returning DTOs directly (no entity mapping needed):  

```java
@ApplicationScoped
public class CalendarRepository {
    
    @Inject
    EntityManager em;
    
    @Inject
    TenantContext tenant;
    
    public List<CalendarEntryDto> findTeamLeaves(
            UUID managerId, LocalDate from, LocalDate to) {
        
        // Native SQL for a 3-table JOIN with date range — cleaner than JPQL here
        var query = (Query) em.createNativeQuery("""
            SELECT lr.id, e.first_name, e.last_name, lt.code AS leave_type_code,
                   lt.calendar_color, lr.start_date, lr.end_date, lr.status
            FROM leave_requests lr
            JOIN employees e ON lr.employee_id = e.id
            JOIN leave_types lt ON lr.leave_type_id = lt.id
            WHERE lr.tenant_id = :tenantId
              AND e.manager_id = :managerId
              AND lr.status = 'APPROVED'
              AND lr.start_date <= :to
              AND lr.end_date >= :from
            ORDER BY lr.start_date
            """, "CalendarEntryMapping");
        
        query.setParameter("tenantId", tenant.getTenantId());
        query.setParameter("managerId", managerId);
        query.setParameter("from", from);
        query.setParameter("to", to);
        
        @SuppressWarnings("unchecked")
        List<Object[]> rows = query.getResultList();
        return rows.stream().map(CalendarEntryDto::fromRow).toList();
    }
}
```

**Result set mapping** defined once on a package-level annotation or `@SqlResultSetMapping`:  

```java
@SqlResultSetMapping(
    name = "CalendarEntryMapping",
    classes = @ConstructorResult(
        targetClass = CalendarEntryDto.class,
        columns = {
            @ColumnResult(name = "id"),
            @ColumnResult(name = "first_name"),
            // ...
        }
    )
)
```

### 7.3 Keyset Pagination — Native SQL  

Used for append-only tables (audit log) where performance on millions of rows matters:  

```java
@ApplicationScoped
public class AuditLogRepository {
    
    @Inject
    EntityManager em;
    
    public PageResponse<AuditLogDto> findAfter(String cursor, int limit) {
        Instant after = cursor != null ? decodeCursor(cursor) : Instant.MIN;
        
        var query = em.createNativeQuery("""
            SELECT id, entity_type, entity_id, action, changes,
                   performed_by, performed_at, ip_address
            FROM audit_log
            WHERE tenant_id = :tenantId
              AND performed_at > :after
            ORDER BY performed_at ASC, id ASC
            LIMIT :limit + 1
            """, AuditLogDto.class);
        
        query.setParameter("tenantId", tenant.getTenantId());
        query.setParameter("after", after);
        query.setParameter("limit", limit + 1); // Fetch one extra to detect hasMore
        
        @SuppressWarnings("unchecked")
        List<AuditLogDto> results = query.getResultList();
        
        boolean hasMore = results.size() > limit;
        if (hasMore) results.removeLast();
        
        String nextCursor = hasMore
            ? encodeCursor(results.getLast().performedAt())
            : null;
        
        return PageResponse.keyset(results, limit, nextCursor, hasMore);
    }
}
```

### 7.4 Rich Filter Parsing  

For Panache queries, a filter parser translates `?status=in:APPROVED,PENDING&start_date=gte:2026-01-01` into JPA Criteria predicates. For native SQL queries, the same parser appends `AND column_name >= :startDate` to the SQL string with bound parameters. Same parser, two output strategies:  

```java
public sealed interface FilterClause permits JpaClause, SqlClause {
    // JpaClause: adds to CriteriaBuilder predicates
    // SqlClause: appends "AND col op :param" + binds parameter value
}

public class FilterParser {
    
    public FilterClause parse(String field, String rawValue) {
        var parsed = parseOperator(rawValue); // Returns (operator, value)
        
        return switch (parsed.operator()) {
            case EQ   -> new EqClause(field, parsed.value());
            case IN   -> new InClause(field, parsed.valueList());
            case GT   -> new GtClause(field, parsed.value());
            case GTE  -> new GteClause(field, parsed.value());
            case LT   -> new LtClause(field, parsed.value());
            case LTE  -> new LteClause(field, parsed.value());
            case LIKE -> new LikeClause(field, parsed.value());
            case BETWEEN -> new BetweenClause(field, parsed.valueRange());
        };
    }
}
```

---

## 8. ETags & Optimistic Concurrency  

### 8.1 ETag Generation  

```java
public class EtagGenerator {
    
    public static String forEntity(Object entity) {
        // Combine updated_at and version into a hash
        String fingerprint = entity.getClass().getSimpleName() + ":" +
            ((HasTimestamps) entity).getUpdatedAt() + ":" +
            (entity instanceof HasVersion v ? v.getVersion() : "0");
        return "\"" + DigestUtils.sha256Hex(fingerprint).substring(0, 16) + "\"";
    }
}
```

### 8.2 If-Match Validation  

```java
@Provider
public class EtagFilter implements ContainerResponseFilter {
    
    @Override
    public void filter(ContainerRequestContext req, ContainerResponseContext res) {
        if (req.getMethod().equals("GET") && res.getEntity() != null) {
            String etag = EtagGenerator.forEntity(res.getEntity());
            res.getHeaders().add("ETag", etag);
        }
    }
}
```

Service-layer validation on PUT:  

```java
public LeaveRequestDto updateLeaveRequest(UUID id, UpdateLeaveRequestDto dto, String ifMatch) {
    var entity = repository.findById(id);
    
    if (ifMatch != null && !EtagGenerator.forEntity(entity).equals(ifMatch)) {
        throw new PreconditionFailedException("Resource was modified. Reload and try again.");
    }
    
    mapper.update(dto, entity);
    return mapper.toDto(entity);
}
```

---

## 9. Event-Driven Architecture  

### 9.1 Outbox Pattern (Recommended)  

For reliable event publication, write events to an `outbox` table in the same transaction as the business operation:  

```java
@Transactional
public void approveLeave(UUID stepId, String comment) {
    var step = approvalRepository.findById(stepId);
    step.approve(comment);
    
    // Write event to outbox table in the same DB transaction
    outboxRepository.save(new OutboxEvent(
        "leave.request.approved",
        new LeaveApprovedPayload(step.getLeaveRequest()).toJson()
    ));
}
```

A scheduled job (`@Scheduled(every="1s")`) polls the outbox and publishes to RabbitMQ:  

```java
@Scheduled(every = "1s")
public void publishOutboxEvents() {
    List<OutboxEvent> events = outboxRepository.findUnpublished(100);
    for (var event : events) {
        emitter.send(Message.of(event.getPayload())
            .withAck(() -> outboxRepository.markPublished(event.getId()))
            .withNack(() -> outboxRepository.markFailed(event.getId()))
        );
    }
}
```

This guarantees **at-least-once delivery** and avoids dual-write problems (DB commit + RabbitMQ publish).

### 9.2 Event Schema Registry  

All events follow a consistent envelope:  

```json
{
  "eventId": "01J1234...",
  "eventType": "leave.request.approved",
  "tenantId": "01J1234...",
  "timestamp": "2026-06-23T10:35:00Z",
  "payload": { ... },
  "metadata": {
    "version": "1.0",
    "source": "hr-leave-management-api",
    "correlationId": "01J1234..."
  }
}
```

### 9.3 Consumer Idempotency  

Consumers check `eventId` before processing to prevent duplicate handling:  

```java
@Incoming("notification-queue")
public void onLeaveApproved(Message<String> msg) {
    var event = parseEvent(msg.getPayload());
    
    if (notificationRepository.existsByEventId(event.getEventId())) {
        return; // Already processed — idempotent
    }
    
    notificationService.sendEmail(event);
    notificationRepository.markProcessed(event.getEventId());
}
```

---

## 10. Error Handling — RFC 7807  

### 10.1 Exception Mapper Hierarchy  

```java
@Provider
public class ValidationExceptionMapper implements ExceptionMapper<ConstraintViolationException> {
    
    @Override
    public Response toResponse(ConstraintViolationException ex) {
        var problem = ProblemDetail.builder()
            .type("validation-error")
            .title("Validation Error")
            .status(400)
            .detail("The request body contains invalid fields.")
            .errors(ex.getConstraintViolations().stream()
                .map(v -> new FieldError(
                    v.getPropertyPath().toString(),
                    v.getMessage(),
                    v.getConstraintDescriptor().getAnnotation().annotationType().getSimpleName()
                ))
                .toList())
            .build();
        
        return Response.status(400)
            .type("application/problem+json")
            .entity(problem)
            .build();
    }
}
```

### 10.2 Problem Detail Builder  

```java
public record ProblemDetail(
    String type,
    String title,
    int status,
    String detail,
    String instance,
    String traceId,
    List<FieldError> errors
) {
    public static Builder builder() { return new Builder(); }
    
    public static class Builder {
        // ... fluent setter methods ...
        public ProblemDetail build() {
            if (traceId == null) traceId = MDC.get("traceId");
            if (instance == null) instance = RestEasyContext.getContextData(UriInfo.class).getPath();
            return new ProblemDetail(type, title, status, detail, instance, traceId, errors);
        }
    }
}

public record FieldError(String field, String message, String code) {}
```

### 10.3 I18n Error Messages  

Validation error messages are resolved from resource bundles based on `Accept-Language`:  

```java
@ApplicationScoped
public class MessageResolver {
    
    public String getMessage(String code, Locale locale) {
        return ResourceBundle.getBundle("i18n.ValidationMessages", locale)
            .getString(code);
    }
}
```

Field errors carry a machine-readable `code` (e.g., `date_range_invalid`) and a human-readable `message` (localized). The frontend can use the `code` for custom handling and `message` for display.

---

## 11. Observability  

### 11.1 Structured Logging  

```properties
quarkus.log.console.format=%d{HH:mm:ss} %-5p traceId=%X{traceId}, tenantId=%X{tenantId}, userId=%X{userId} [%c{2.}] (%t) %s%e%n
```

Every request gets a `traceId` (UUID v7) set in MDC by a filter:

```java
@Provider
public class TraceIdFilter implements ContainerRequestFilter {
    @Override
    public void filter(ContainerRequestContext ctx) {
        String traceId = ctx.getHeaderString("X-Request-Id");
        if (traceId == null) traceId = UUID.randomUUID().toString();
        MDC.put("traceId", traceId);
        ctx.getHeaders().add("X-Request-Id", traceId); // Echo back
    }
}
```

### 11.2 Metrics  

```properties
quarkus.micrometer.export.prometheus.path=/q/metrics
quarkus.micrometer.binder.cache.enabled=true
quarkus.micrometer.binder.jvm=true
quarkus.micrometer.binder.http-server.enabled=true
```

Key custom metrics:  

```java
@Metered(name = "eligibility.evaluations", description = "DMN eligibility evaluations")
public EligibilityResult evaluate(UUID employeeId, String leaveTypeCode) { ... }

@Timed(name = "policy.resolution", description = "Policy resolution time")
public Policy resolvePolicy(UUID leaveTypeId, UUID convenioId, LocalDate date) { ... }
```

### 11.3 Distributed Tracing  

```properties
quarkus.opentelemetry.enabled=true
quarkus.opentelemetry.tracer.exporter.otlp.endpoint=http://jaeger:4317
```

Traces span: HTTP request → Service → DMN evaluation → DB query → Valkey cache lookup. Correlates with `traceId` in logs.

### 11.4 Health Checks  

Quarkus auto-generates health endpoints. Add custom checks for Valkey and RabbitMQ:  

```java
@Liveness
@ApplicationScoped
public class ValkeyHealthCheck implements HealthCheck {
    @Inject RedisDataSource redis;
    
    @Override
    public HealthCheckResponse call() {
        try {
            redis.ping();
            return HealthCheckResponse.up("valkey");
        } catch (Exception e) {
            return HealthCheckResponse.down("valkey");
        }
    }
}
```

Available at `/q/health/live` (liveness), `/q/health/ready` (readiness — includes DB, Valkey, RabbitMQ).

---

## 12. Configuration & Secrets  

### 12.1 Configuration Hierarchy  

```
application.properties (defaults, in source control)
    ↓ overridden by
.env (local development, not in source control)
    ↓ overridden by
Environment variables (Docker/K8s)
    ↓ overridden by
Vault / AWS Secrets Manager (production secrets)
```

### 12.2 Key Configuration Properties  

```properties
# Database
quarkus.datasource.db-kind=postgresql
quarkus.datasource.jdbc.url=jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:hr_leave}
quarkus.datasource.username=${DB_USERNAME:hr_app}
quarkus.datasource.password=${DB_PASSWORD}

# Valkey
quarkus.redis.hosts=redis://${VALKEY_HOST:localhost}:${VALKEY_PORT:6379}
quarkus.redis.timeout=5s
quarkus.redis.max-pool-size=20

# RabbitMQ
quarkus.messaging.rabbitmq.url=amqp://${RABBITMQ_USER:hr_app}:${RABBITMQ_PASSWORD}@${RABBITMQ_HOST:localhost}:${RABBITMQ_PORT:5672}

# JWT
smallrye.jwt.sign.key.location=${JWT_PRIVATE_KEY_PATH:/etc/keys/privateKey.pem}
mp.jwt.verify.publickey.location=${JWT_PUBLIC_KEY_PATH:/etc/keys/publicKey.pem}

# SMTP
quarkus.mailer.from=${SMTP_FROM:noreply@company.com}
quarkus.mailer.host=${SMTP_HOST}
quarkus.mailer.port=${SMTP_PORT:587}
quarkus.mailer.username=${SMTP_USERNAME}
quarkus.mailer.password=${SMTP_PASSWORD}

# CORS
quarkus.http.cors.origins=${FRONTEND_URL}
quarkus.http.cors.access-control-allow-credentials=true

# Cache
quarkus.cache.caffeine.leave-types.expire-after-write=5M
quarkus.cache.caffeine.leave-types.maximum-size=100
quarkus.cache.redis.eligible-types.ttl=300000

# Logging
quarkus.log.level=INFO
quarkus.log.category."es.mylair".level=${LOG_LEVEL:INFO}

# Rate Limiting
rate-limit.auth.requests-per-minute=10
rate-limit.read.requests-per-minute=300
rate-limit.write.requests-per-minute=60
rate-limit.upload.requests-per-minute=20
rate-limit.export.requests-per-minute=5
```

### 12.3 Secrets Management  

In production, secrets (DB password, JWT keys, SMTP credentials, encryption keys) are never in environment variables directly. Use:  

- **HashiCorp Vault** with `quarkus-vault` extension.  
- Or **Kubernetes Secrets** mounted as files.  
- Or **AWS Secrets Manager** with `quarkus-amazon-secretsmanager`.  

Example Vault configuration:  

```properties
quarkus.vault.url=${VAULT_ADDR}
quarkus.vault.authentication.client-token=${VAULT_TOKEN}
quarkus.vault.secret-config-kv-path=secret/hr-leave-management
```

Secrets are loaded at startup and refreshed on rotation without restart.

---

## 13. Testing Strategy  

### 13.1 Test Pyramid  

```
        ╱ E2E ╲          Playwright (frontend), RestAssured (API full stack)
       ╱───────╲
      ╱Integration╲      @QuarkusIntegrationTest + Testcontainers
     ╱─────────────╲
    ╱   Unit Tests   ╲    JUnit 5 + Mockito
   ╱───────────────────╲
```

### 13.2 Unit Tests  

Unit test every service, mapper, validator, and utility in isolation:  

```java
@QuarkusTest
class LeaveRequestServiceTest {
    
    @InjectMock
    LeaveRequestRepository repository;
    
    @InjectMock
    PolicyResolutionService policyService;
    
    @Inject
    LeaveRequestService service;
    
    @Test
    void shouldRejectOverlappingLeave() {
        when(repository.findOverlapping(any(), any(), any()))
            .thenReturn(List.of(existingLeave));
        
        var dto = new CreateLeaveRequestDto(/* ... */);
        assertThrows(ConflictException.class, () -> service.create(dto));
    }
}
```

### 13.3 Integration Tests with Testcontainers  

```java
@QuarkusIntegrationTest
class LeaveRequestResourceIT {
    
    @Testcontainers
    PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");
    
    @Testcontainers
    GenericContainer<?> valkey = new GenericContainer<>("valkey/valkey:8-alpine")
        .withExposedPorts(6379);
    
    @Testcontainers
    RabbitMQContainer rabbitmq = new RabbitMQContainer("rabbitmq:3.13-management");
    
    @Test
    void shouldCreateAndSubmitVacationLeave() {
        given()
            .cookie("access_token", jwt)
            .header("X-CSRF-Token", csrfToken)
            .body(createDto)
        .when()
            .post("/api/v1/leave-requests")
        .then()
            .statusCode(201)
            .body("status", equalTo("DRAFT"))
            .body("totalDays", equalTo(15.0f));
    }
}
```

### 13.4 DMN Decision Tests  

```java
@QuarkusTest
class VacationEntitlementDmnTest {
    
    @Inject
    @Named("vacation-entitlement")
    DMNRuntime dmnRuntime;
    
    @Test
    void fullTimeEmployeeWithFiveYearsGetsThirtyDays() {
        var ctx = dmnRuntime.newContext();
        ctx.set("Seniority Years", 5);
        ctx.set("Contract Type", "FULL_TIME");
        
        var result = dmnRuntime.evaluateAll(
            dmnRuntime.getModels().get(0), ctx);
        
        assertEquals(30,
            result.getDecisionResultByName("Base Entitlement").getResult());
    }
}
```

### 13.5 Security Tests  

```java
@QuarkusTest
class SecurityTest {
    
    @Test
    void employeeCannotAccessHrEndpoints() {
        given()
            .cookie("access_token", employeeJwt)
        .when()
            .get("/api/v1/employees")  // Requires HR Admin
        .then()
            .statusCode(403);
    }
    
    @Test
    void managerCannotViewMedicalDocuments() {
        given()
            .cookie("access_token", managerJwt)
        .when()
            .get("/api/v1/documents/" + medicalDocId)
        .then()
            .statusCode(403);
    }
    
    @Test
    void csrfTokenRequiredForPost() {
        given()
            .cookie("access_token", employeeJwt)
            // No X-CSRF-Token header
            .body(createDto)
        .when()
            .post("/api/v1/leave-requests")
        .then()
            .statusCode(403)
            .body("type", equalTo("csrf-mismatch"));
    }
}
```

### 13.6 Test Coverage Targets  

| Layer | Coverage | Notes |
|-------|----------|-------|
| Service layer | ≥ 90% line | Core business logic |
| DMN decisions | 100% of paths | Every decision table path tested |
| Repository | ≥ 80% line | Custom queries only; skip trivial Panache ops |
| Mapper (MapStruct) | 0% (no tests) | Compile-time generated; correctness verified by type system |
| DTOs | 0% (no tests) | Records with validation annotations; tested via integration |
| Resources (JAX-RS) | Covered by integration | Integration tests cover the HTTP contract |
| Security filters | ≥ 90% line | Critical for auth |

---

## 14. Build & CI/CD  

### 14.1 Build Commands  

```bash
# Development
./mvnw quarkus:dev                         # Hot reload, debug port 5005

# Testing
./mvnw verify                              # Unit + integration tests
./mvnw test -Dtest="*DmnTest"             # DMN tests only

# Build
./mvnw package -Dquarkus.package.jar.type=uber-jar  # Fat JAR
./mvnw package -Pnative                    # Native image (GraalVM)

# Docker
docker build -f src/main/docker/Dockerfile.jvm -t hr-api .
```

### 14.2 CI Pipeline (GitHub Actions / GitLab CI)  

```yaml
stages:
  - build
  - test
  - quality
  - package

build:
  script: ./mvnw compile

unit-tests:
  script: ./mvnw test
  artifacts:
    reports:
      junit: target/surefire-reports/*.xml

integration-tests:
  services:
    - postgres:16
    - valkey/valkey:8-alpine
    - rabbitmq:3.13
  script: ./mvnw verify -DskipITs=false

quality:
  script:
    - ./mvnw sonar:sonar -Dsonar.coverage.jacoco.xmlReportPaths=target/jacoco-report/jacoco.xml
  coverage: 80% line coverage minimum

package:
  script:
    - ./mvnw package -Dquarkus.package.jar.type=uber-jar
    - docker build -t registry.example.com/hr-api:$CI_COMMIT_SHA .
    - docker push registry.example.com/hr-api:$CI_COMMIT_SHA
```

### 14.3 Database Migrations (Flyway)  

Flyway manages schema versioning for **both Panache entities and native SQL queries**. It is ORM-agnostic — it runs plain SQL files in version order.  

```sql
-- src/main/resources/db/migration/V1__initial_schema.sql
CREATE TABLE tenants (...);
CREATE TABLE users (...);
-- ... all initial tables ...

-- V2__add_bridge_days.sql
CREATE TABLE bridge_days (...);

-- V3__add_outbox.sql  
CREATE TABLE outbox_events (...);

-- V4__add_calendar_indexes.sql
-- Native SQL calendar queries benefit from these indexes
CREATE INDEX idx_lr_dates_tenant ON leave_requests(tenant_id, start_date, end_date);
CREATE INDEX idx_lr_employee_status ON leave_requests(employee_id, status);

-- R__report_views.sql (repeatable migration — re-applied when changed)
CREATE OR REPLACE VIEW vacation_balance_report AS
SELECT tenant_id, employee_id, year, SUM(entitlement) - SUM(consumed) AS remaining
FROM leave_balances GROUP BY tenant_id, employee_id, year;
```

- **Versioned migrations** (`V1__`, `V2__`, ...): Run once, in order. Checksum-verified — Flyway fails if a previously-applied migration was modified.  
- **Repeatable migrations** (`R__`): Re-applied whenever the file changes. Ideal for database views used by native SQL report queries.  
- Flyway runs automatically at startup (`quarkus.flyway.migrate-at-start=true`).  
- **Works identically** whether the application code uses Panache or raw JDBC — it's just SQL files.

---

## Appendix A: Valkey Cache Usage Summary  

| Cache Name | Layer | TTL | Invalidation Trigger | Key Pattern |
|-----------|-------|-----|---------------------|-------------|
| `leave-types` | Caffeine + Valkey | 5 min | `PUT/DELETE /leave-types/{id}` | `{tenant}:leave-types:active` |
| `eligible-types` | Valkey | 5 min | Policy publish, balance change, leave submit | `{tenant}:eligible:{employeeId}` |
| `user-permissions` | Valkey | 8 h | Role assignment change | `{tenant}:permissions:{userId}` |
| `policies` | Valkey | 1 h | Policy publish | `{tenant}:policy:{id}` |
| `holiday-calendars` | Valkey | 1 h | Calendar update | `{tenant}:holidays:{calendarId}` |
| `balances` | Caffeine | 1 min | Leave submit, cancel, balance edit | `{tenant}:balances:{employeeId}` |
| `rate-limits` | Valkey | Per window | Auto-expire (Valkey TTL) | `ratelimit:{userId}:{tier}:{window}` |

## Appendix B: Docker Compose (Development with Valkey)  

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: hr_leave
      POSTGRES_USER: hr_app
      POSTGRES_PASSWORD: dev_password
    ports: ["5432:5432"]
    volumes: [pgdata:/var/lib/postgresql/data]

  valkey:
    image: valkey/valkey:8-alpine
    command: valkey-server --save "" --appendonly no
    ports: ["6379:6379"]
    healthcheck:
      test: ["CMD", "valkey-cli", "ping"]
      interval: 5s

  rabbitmq:
    image: rabbitmq:3.13-management-alpine
    environment:
      RABBITMQ_DEFAULT_USER: hr_app
      RABBITMQ_DEFAULT_PASS: dev_password
    ports: ["5672:5672", "15672:15672"]

  api:
    build:
      context: .
      dockerfile: src/main/docker/Dockerfile.jvm
    environment:
      QUARKUS_DATASOURCE_JDBC_URL: jdbc:postgresql://postgres:5432/hr_leave
      QUARKUS_DATASOURCE_USERNAME: hr_app
      QUARKUS_DATASOURCE_PASSWORD: dev_password
      QUARKUS_REDIS_HOSTS: redis://valkey:6379
      QUARKUS_MESSAGING_RABBITMQ_URL: amqp://hr_app:dev_password@rabbitmq:5672
      JWT_PRIVATE_KEY_PATH: /etc/keys/privateKey.pem
      JWT_PUBLIC_KEY_PATH: /etc/keys/publicKey.pem
      ENCRYPTION_KEY: ${DEV_ENCRYPTION_KEY}
      FRONTEND_URL: http://localhost:3000
    ports: ["8080:8080"]
    volumes:
      - ./keys:/etc/keys:ro
      - docs:/var/lib/hr-leave/documents
    depends_on:
      postgres:
        condition: service_started
      valkey:
        condition: service_healthy
      rabbitmq:
        condition: service_started
```

---

> **End of API Implementation Technical Specification**  
> For questions, contact the architecture team.
