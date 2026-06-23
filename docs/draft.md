# Goal

Create a web application that can handle the different kinds of company leave request for a Spanish company

# Leave types
The leave types have different sources:

- Legal leave types (mandatory by law)
- Collective bargaining agreement (Convenio colectivo)


The application should be designed so that leave rules are configurable, because many details depend on the employee's collective agreement and company policies.

## Employee master data required

The system should maintain:

- Employment start date
- Contract type
- Working schedule
  - Full-time
  - Part-time
  - Shift worker
- Work center
- Collective bargaining agreement (Convenio)
- Professional category
- Annual working hours
- Annual vacation entitlement
- Family status (where relevant for leave eligibility)
- Seniority

These attributes affect leave eligibility and calculations.

# Vacation Leave (Vacaciones)

## Legal requirements

According to the Spanish Workers' Statute:

- Minimum: 30 calendar days per year (or equivalent)
- Cannot be replaced by payment except on termination
- Must be agreed between employer and employee
- Vacation calendar should be known in advance
- Vacation periods during:
  - temporary disability (IT)
  - maternity/paternity leave
  - pregnancy-related leave

must be recoverable later.

## System requirements
- Annual entitlement management
- Prorated entitlement for new hires
- Carry-over rules (configurable)
- Vacation calendar
- Balance tracking
- Partial-day vacations
- Conflict detection
- Team absence visibility
- Approval workflow

# Temporary Disability / Sick Leave (Incapacidad Temporal - IT)

Types:

## Common illness

- Non-work-related sickness
- Non-work accident

## Professional contingencies

- Work accident
- Occupational disease

## System requirements

- Register medical leave periods
- Store medical certificates
- Track start/end dates
- Manage extensions
- Payroll integration support
- Prevent vacation consumption during active sick leave
- Audit trail of medical documents

## Data protection

Medical information is special category personal data under GDPR.

The system should:

- Restrict access
- Log access
- Encrypt stored documents
- Apply retention policies

# Birth and Childcare Leave

(Current "permiso por nacimiento y cuidado del menor")

## Requirements

Track:

- Expected birth date
- Actual birth date
- Leave start
- Leave end
- Mandatory periods
- Optional periods
- Multiple births

## System requirements

- Legal duration configurable
- Suspension of work contract tracking
- Document management


# Pregnancy and Breastfeeding Related Leave

Types:

Risk during pregnancy
Risk during breastfeeding
Prenatal examinations
Childbirth preparation classes

## Requirements
- Medical documentation
- Protected leave periods
- Special workflow

# Paid Leave (Permisos Retribuidos)

he application should support configurable leave categories because the exact duration may vary by legal updates and collective agreements.

Typical categories include:

- Marriage or registered partnership
- Death of relatives
- Serious illness of relatives
- Hospitalization of relatives
- Surgery requiring home rest
- Moving house
- Public duties

Examples:

  - Jury duty
  - Electoral duties

- Trade union activities
- Exams
- Medical appointments
- Family emergencies
- Force majeure leave
- Blood donation
- Organ donation

## System requirements

For every category:

- Configurable duration
- Eligibility rules
- Required documentation
- Relationship verification
- Approval workflow
- Expiration deadlines for document submission


# Unpaid Leave (Permisos no retribuidos)

Requirements depend largely on collective agreements.

The system should support:

- Employee request
- Manager approval
- HR approval
- Payroll impact
- Contract suspension indicators

# Parental Leave (Permiso Parental)

Current Spanish regulations include parental leave rights.

Requirements:

- Child identification
- Child age validation
- Requested periods
- Multiple periods
- Balance tracking if applicable

# Family Care Leave

Care of child
Care of family member
Care of dependent person

Possible forms:

- Full leave
- Reduced working hours
- Flexible schedule adaptations

System should support:

- Percentage reduction
- Effective dates
- Supporting documentation
- Impact on payroll calculations

# Leave of Absence (Excedencias)

The system should support:

Voluntary leave (Excedencia voluntaria)

Requirements:

- Seniority validation
- Minimum and maximum durations
- Reinstatement requests
Family care leave
Child care
Family member care
Public office leave
Trade union leave
Forced leave (Excedencia forzosa)

System requirements:

- Eligibility validation
- Duration tracking
- Reservation rights
- Reinstatement workflow
- Alerts for return deadlines

# Reduced Working Hours (Reducción de Jornada)

Although not strictly leave, it is usually managed together.

Examples:

- Childcare
- Care of dependent family members

Requirements:

- Percentage reduction
- Effective dates
- Schedule definition
- Legal documentation

# Public Holidays

The system should support:

- National holidays
- Autonomous community holidays
- Local municipality holidays

For example, employees in:

- Madrid
- Barcelona

may have different holiday calendars.

Requirements:

- Multiple calendars
- Automatic entitlement calculations
- Integration with leave planning

# Collective Agreement (Convenio) Support

This is one of the most important requirements.

The system should allow configuration of:

- Additional vacation days
- Additional paid leave categories
- Different durations
- Seniority benefits
- Sector-specific rights
- Documentation requirements
- Approval rules

Examples:

- Metal industry
- Consulting
- Banking
- Retail
- Healthcare
- Public sector

Never hard-code leave rules.

# Approval Workflow Requirements

Configurable workflow:

Employee → Manager → HR

Possible variants:

- Auto-approved
- Manager only
- Manager + HR
- Multi-level approval

Requirements:

- Delegation during absence
- Escalations
- Reminders
- SLA tracking
- Rejection reasons

# Compliance and Audit Requirements

The system should maintain:

- Request timestamp
- Approval timestamp
- Approver identity
- Supporting documents
- Change history
- Cancellation history
- Restored leave history

Requirements:

- Immutable audit trail
- Legal evidence export
- Reporting

# GDPR Requirements

The application will process personal data and, for some leave types, health data.

Requirements:

## Access control
- RBAC
- HR-only access to sensitive data

## Data minimization

Managers should only see:

- Leave type
- Dates
- Status

They should not necessarily see medical details.

## Audit logs

Track:

- Who viewed data
- Who modified data
- When

## Retention

Configurable retention periods for:

- Leave records
- Medical certificates
- Employment records

## Security
- Encryption at rest
- Encryption in transit
- MFA support
- Backup and recovery

# Reporting Requirements

Reports typically required by HR:

- Vacation balances
- Upcoming absences
- Sick leave statistics
- Absence rates
- Leave by department
- Leave by leave type
- Excedencia tracking
- Legal leave utilization
- Documentation pending
- Payroll-impacting absences

# Recommended architecture

Instead of modeling leave types directly in code, create a rules engine:

## Leave Type

- Code
- Name
- Paid/Unpaid
- Requires approval
- Requires documentation
- Duration calculation method

## Eligibility Rules

- Seniority requirements
- Family relationship requirements
- Contract requirements

## Workflow Rules

- Approvers
- Escalation rules

## Entitlement Rules

- Annual balance
- Accrual method
- Carry-over rules


# Core Concept

A leave request should go through a generic engine:

1. Employee requests leave
2. System determines applicable policy
3. Eligibility rules are evaluated
4. Duration rules are evaluated
5. Documentation requirements are evaluated
6. Approval workflow is determined
7. Balance impact is calculated
8. Request is approved/rejected

The leave type merely selects which rules apply.

## Domain Model

### LeaveType

leaveType
---------
id
code
name
country
paid
active

Examples:

VACATION
MARRIAGE
BIRTH
PARENTAL
MEDICAL_APPOINTMENT
FORCE_MAJEURE
EXCEDENCIA_VOLUNTARIA

Notice that there is almost no behavior here.

### Policy

A policy contains the actual rules.

Policy
------
id
name
effectiveFrom
effectiveTo
country
convenio

Examples:

Spain General Labor Law 2026

Spain Consulting Agreement 2026

Spain Metal Industry Agreement 2026

### Rule Categories

I would separate rules into five groups.

1. Eligibility Rules

Determines whether the employee can request the leave.

Example:

{
  "type": "MIN_SENIORITY",
  "months": 12
}

Example:

{
  "type": "EMPLOYMENT_STATUS",
  "allowed": [
      "FULL_TIME",
      "PART_TIME"
  ]
}

Example:

{
  "type": "RELATIVE_TYPE",
  "allowed": [
      "SPOUSE",
      "CHILD",
      "PARENT"
  ]
}

2. Duration Rules

Determine the maximum duration.

Marriage leave:

{
  "type": "FIXED_DAYS",
  "days": 15
}

Vacation:

{
  "type": "ANNUAL_ENTITLEMENT",
  "days": 30
}

Parental leave:

{
  "type": "MAX_WEEKS",
  "weeks": 8
}
3. Documentation Rules

Example:

{
  "type": "REQUIRES_DOCUMENT"
}

Marriage leave:

{
  "type": "DOCUMENT_TYPE",
  "accepted": [
      "MARRIAGE_CERTIFICATE"
  ]
}

Hospitalization leave:

{
  "type": "DOCUMENT_TYPE",
  "accepted": [
      "HOSPITAL_REPORT"
  ]
}
4. Workflow Rules

Example:

{
  "type": "APPROVAL_CHAIN",
  "steps": [
      "MANAGER",
      "HR"
  ]
}

Vacation:

Manager approval

Sick leave:

Auto-approved

Excedencia:

Manager -> HR -> Legal
5. Accounting Rules

How balances are affected.

Vacation:

{
  "type": "DEDUCT_BALANCE",
  "balance": "VACATION"
}

Marriage leave:

{
  "type": "NO_BALANCE_IMPACT"
}

Unpaid leave:

{
  "type": "PAYROLL_IMPACT"
}

# Example: Marriage Leave

Instead of coding:

if (leaveType == MARRIAGE)

Store:

{
  "leaveType": "MARRIAGE",
  "rules": [
    {
      "type": "FIXED_DAYS",
      "days": 15
    },
    {
      "type": "REQUIRES_DOCUMENT",
      "document": "MARRIAGE_CERTIFICATE"
    },
    {
      "type": "APPROVAL_CHAIN",
      "steps": ["HR"]
    }
  ]
}

The engine evaluates those rules.

# Rule Evaluation Pipeline
Leave Request
       |
       V
Policy Resolver
       |
       V
Eligibility Engine
       |
       V
Duration Engine
       |
       V
Document Engine
       |
       V
Workflow Engine
       |
       V
Balance Engine
       |
       V
Result

Each engine is independent.


# Policy Resolution

This is where the real value appears.

Suppose you have:

Spain
 ├─ General Law
 ├─ Consulting Convenio
 ├─ Metal Convenio
 └─ Banking Convenio

An employee requests marriage leave.

The resolver chooses:

Employee
    ↓
Convenio = Consulting
    ↓
Consulting Marriage Policy

No code changes.

Just configuration.

# Effective Dating

Every rule should be versioned.

Marriage Leave

Version A
2020-01-01 → 2026-12-31

Version B
2027-01-01 →

When labor law changes:

new policy version

not

code deployment

This is critical because HR may need to review requests from previous years under the rules that existed at that time.

# Rule Expressions

Eventually you will need more complex rules.

Example:

Employee has more than 5 years seniority
AND
works full time
AND
belongs to Consulting Convenio

Store as expressions:

{
  "expression":
    "seniorityYears >= 5 && contractType == 'FULL_TIME'"
}

or use a decision table.

Example:

Condition	Result
Seniority < 1 year	22 days
1-5 years	25 days
>5 years	30 days

Business users understand tables better than code.

# Event-Driven Design

I would also emit events:

LeaveRequested
LeaveApproved
LeaveRejected
LeaveCancelled
LeaveStarted
LeaveEnded

This allows:

payroll integration
calendar integration
reporting
notifications

without coupling those systems to the leave engine.

# What I would actually build

For a modern SaaS leave-management system, I would use:

Leave Type
        |
        V
Policy
        |
        V
Rule Sets
        |
        V
Rule Evaluators

with rules stored in the database as configuration.

I would avoid embedding business rules in application code except for the generic rule evaluators themselves.

This gives you:

- Support for any Spanish convenio
- Future labor-law changes
- Multiple countries later
- No code changes for most HR policy modifications
- Full auditability of which rule version was applied to each request

# DMN-only Approach (My Preferred Option)

If I were starting today with Quarkus, I would seriously consider avoiding a traditional rule engine entirely.

Instead:

Store policies in database
LeaveType
Policy
Rule
RuleVersion
Use DMN for calculations

Examples:

Vacation entitlement
Eligibility
Documentation requirements
Use BPMN for workflows

Examples:

Manager approval
HR approval
Escalations
Use Quarkus services for orchestration
REST API
      |
      V
Leave Service
      |
      +--> Policy Repository
      |
      +--> DMN Engine
      |
      +--> Workflow Engine

This approach is easier to explain to HR departments than a large DRL rule base.

Quarkus
   |
   +-- PostgreSQL
   |
   +-- Policy Metadata
   |
   +-- DMN Decisions
   |
   +-- BPMN Approval Flows
   |
   +-- REST API

# i18n

The application should support multiple languages. The user should be able con configure the default language in the user profile
If the language is not set in the profile the language should match the browser preferred language if supported, otherwise use english as the default language
Spanish and English are minimum number of languages that should be supported.

## 2. User Roles

Users can hold **multiple roles simultaneously**. Permissions are additive — for example, a user with `system-admintrator` and `employee` roles has all permissions of both. The most common case is a system-administrator who is also an employee receiving asset assignments.

Available roles:

For a leave management application, I would avoid having a single "HR" role. In practice, HR responsibilities are usually split across different people and separating them improves security, auditability, and compliance (especially for medical leave data under GDPR).

A good model is to define **business roles** first and then map them to application permissions.

# 1. Employee

The employee is the owner of the leave request.

## Responsibilities

* Submit leave requests
* Upload supporting documentation
* Cancel requests
* View balances
* View leave history
* View approval status
* Request corrections

## Permissions

Can view:

* Own leave requests
* Own balances
* Own documents

Cannot view:

* Other employees' requests
* Medical documents of others
* Company-wide reports

---

# 2. Line Manager

The employee's direct supervisor.

## Responsibilities

* Approve operational leave requests
* Verify team coverage
* Plan workload around absences
* Reject requests with justification

## Permissions

Can view:

* Leave calendar of direct reports
* Team leave balances
* Approval queues

Should not view:

* Medical certificates
* Health information
* Detailed medical diagnoses

For example:

Manager sees:

```text
Sick Leave
01/02/2026 - 15/02/2026
Approved
```

Manager does not see:

```text
Medical diagnosis
Hospital report
Doctor notes
```

This is important for GDPR compliance.

---

# 3. HR Administrator

The most common HR role.

## Responsibilities

* Manage leave policies
* Review supporting documents
* Approve HR-controlled leave
* Resolve conflicts
* Correct balances
* Configure holiday calendars
* Manage collective agreements

## Permissions

Can:

* View all employee leave records
* View documentation
* Modify balances
* Override approvals
* Create company calendars

Cannot:

* Modify audit logs
* Access system administration

---

# 4. HR Specialist - Medical Leave

In larger organizations this should be separated.

## Responsibilities

Handle:

* Sick leave
* Occupational accidents
* Pregnancy-related leave
* Disability-related leave

## Permissions

Can access:

* Medical certificates
* Health-related documents
* Disability information

Cannot access:

* System administration

This role exists primarily because medical information is special category personal data under GDPR.

---

# 5. HR Policy Administrator

Responsible for maintaining leave policies.

This becomes particularly useful if you implement the rules engine we discussed.

## Responsibilities

Configure:

* Leave types
* Eligibility rules
* Approval workflows
* Collective agreements
* Leave entitlements

## Permissions

Can:

* Create policies
* Version policies
* Activate future policies

Cannot:

* Modify historical leave records

Example:

```text
Marriage Leave

2026 Policy:
15 days

2027 Policy:
20 days
```

HR Policy Administrator creates the new version.

---

# 6. Payroll Officer

Often separate from HR.

## Responsibilities

Process leave impacts affecting salary.

Examples:

* Unpaid leave
* Reduced working hours
* Contract suspensions
* Excedencias

## Permissions

Can view:

* Payroll-relevant leave data
* Leave balances
* Effective dates

Should not view:

* Medical diagnoses

---

# 7. Compliance / Legal Officer

Needed in larger organizations.

## Responsibilities

* Investigate disputes
* Review audit trails
* Handle labor inspections
* Produce legal evidence

## Permissions

Can view:

* Audit logs
* Historical decisions
* Policy versions

Cannot modify:

* Leave requests
* Policies

Read-only access is usually sufficient.

---

# 8. Department Manager

Above line managers.

## Responsibilities

* Department-wide planning
* Resource management
* Escalated approvals

## Permissions

Can view:

* Department calendar
* Department balances
* Team statistics

Should not view:

* Medical documentation

---

# 9. Leave Administrator

Common in large companies.

Focused only on leave operations.

## Responsibilities

* Validate requests
* Check documents
* Follow up with employees
* Resolve workflow issues

## Permissions

Can:

* Process leave requests
* Request missing documents

Cannot:

* Change policies
* Manage users

---

# 10. System Administrator

Technical role.

## Responsibilities

* User provisioning
* Security configuration
* Backups
* Integrations

## Permissions

Can:

* Manage users
* Configure SSO
* Configure integrations

Should not automatically have access to HR data.

This is a very common security mistake.

---

# 11. Auditor

Read-only role.

## Responsibilities

* Internal audits
* Compliance reviews

## Permissions

Can view:

* Requests
* Decisions
* Audit logs
* Policy versions

Cannot modify anything.

---

# Recommended Permission Model

Instead of assigning permissions directly to roles, use:

```text
Role
   |
   V
Permissions
```

Example:

```text
HR Administrator
    |
    +-- Leave.View.All
    +-- Leave.Approve.HR
    +-- Leave.Override
    +-- Document.View
```

```text
Line Manager
    |
    +-- Leave.View.Team
    +-- Leave.Approve.Team
```

```text
HR Medical Specialist
    |
    +-- Medical.View
    +-- Medical.Approve
```

---

# Additional Role for a Rules-Driven System

If you implement policy versioning and configurable rules, I would add:

## Leave Policy Owner

Responsibilities:

* Own labor-law configuration
* Maintain convenio-specific rules
* Review future policy changes

Permissions:

```text
Policy.Create
Policy.Edit
Policy.Publish
Policy.Version
```

but not:

```text
Leave.Approve
Leave.Override
Payroll.Process
```

This separation prevents the same person from both defining the rules and approving exceptions to them, which is a strong governance and audit-control practice.


# Create a permissions Matix


## 3. Authentication & User Management

### SSO (Single Sign-On)
- Users authenticate via Google or Zoho OAuth 2.0 / OIDC
- No separate passwords — users log in with existing Google Workspace or Zoho accounts
- On first login, user record is auto-created in the database
- Subsequent logins match by email and update user info

### Google SSO
- OAuth 2.0 with OpenID Connect
- Scopes: `openid`, `email`, `profile`
- Requires: Google Cloud project with OAuth client ID configured
- Redirect URI: `https://<app-domain>/api/auth/google/callback`

### Zoho SSO
- OpenID Connect (OIDC)
- Scopes: `openid email profile`
- Userinfo endpoint: `{accountsBaseUrl}/oauth/user/info`

### Session Management
- JWT tokens issued after successful SSO authentication
- Tokens include: user ID (sub), email (upn), name, userId, groups (roles array)
- Signing: RSA-256 (RS256) via SmallRye JWT with matching key pair (privateKey.pem / publicKey.pem)
- Verification: SmallRye JWT extension validates signature, expiry, and issuer
- Token expiry: configurable (default 8 hours)
- Logout clears token from localStorage
- No server-side logout endpoint; sessions are stateless (JWT-based)
- OAuth callback redirects to `{frontendUrl}/login?token=<jwt>` where the frontend extracts and stores the token

### User Resolution
- Users matched by email across both SSO providers
- Users in state "Terminated" are not allowed to login.
- Admin can manually set status for users not managed by SSO

### Role Assignment
- First user to log in is auto-assigned both `system administrator` and `employee` roles
- Subsequent users default to `employee` role
- Super Admin can manage user roles.
- A user must always have at least one role
- There must always be at least on system administrator in the platform.


