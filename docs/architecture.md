# Hawaii Co-op Project — System Architecture

## Architectural Style

Hawaii Co-op Project is a **multi-tenant SaaS** application. Each co-op is a tenant. Members belong to one or more co-ops. Advisors can have read access across multiple tenants.

The architecture prioritizes:
- **Simplicity over premature scale.** v1 serves tens of co-ops, not thousands. Avoid distributed systems complexity until required.
- **Data isolation.** Co-op financial and membership data is sensitive. Tenant isolation must be enforced at the data layer, not just the application layer.
- **Auditability.** Governance decisions (votes, membership changes, financial entries) need an immutable audit trail.

---

## High-Level Modules

```
┌─────────────────────────────────────────────────────┐
│                     Web Client                       │
│         (SPA or SSR — TBD at stack selection)       │
└──────────────────────┬──────────────────────────────┘
                       │ HTTPS / REST or GraphQL
┌──────────────────────▼──────────────────────────────┐
│                   API Layer                          │
│  Auth │ Co-op Mgmt │ Members │ Governance │ Docs     │
└──┬────────┬─────────────┬──────────┬────────┬───────┘
   │        │             │          │        │
   ▼        ▼             ▼          ▼        ▼
 AuthN   Co-op DB     Member DB   Vote DB   File Store
 (JWT/   (primary     (per-       (immut-   (docs +
 OAuth)   relational)  tenant)    able log) templates)
```

---

## Core Entities

### `Organization`
Represents a single co-op. Holds type (`housing` | `agricultural`), formation status, island, and legal metadata (HRS chapter, EIN once filed).

### `Member`
A user within an organization. Has a role (`organizer` | `member` | `admin`), equity balance, join date, and status (`pending` | `active` | `exited`).

### `Proposal`
A governance item put to a vote. Linked to the `Organization`. Has a type (`bylaw_amendment` | `membership_admission` | `budget_approval` | `general`), quorum rules, and deadline.

### `Vote`
An immutable record linking a `Member` to a `Proposal` with their choice and timestamp. Never deleted or updated — only appended.

### `Document`
A file stored in object storage. Tagged with type (`articles_of_incorporation` | `bylaws` | `membership_agreement` | `minutes` | `other`). Has a version chain for amendments.

### `FormationWorkflow`
A state machine tracking a co-op's progress through the formation checklist. Each step has a status (`not_started` | `in_progress` | `complete` | `blocked`) and optional advisor notes.

---

## Tenant Isolation Strategy

All relational tables include an `organization_id` foreign key. Application-level middleware enforces that every authenticated request binds queries to the requesting user's organization(s).

Row-level security (RLS) enforced at the database layer provides a second line of defense — a misconfigured query cannot leak cross-tenant data.

Advisor accounts hold an explicit `OrganizationAccess` join table entry with a scope (`read` | `write`) per organization.

---

## Formation Workflow State Machine

```
NOT_STARTED
    │
    ▼
ORGANIZING          ← Members are recruited, intent established
    │
    ▼
LEGAL_STRUCTURE     ← Co-op type & HRS chapter selected
    │
    ▼
DOCUMENTS_DRAFTED   ← Articles, bylaws generated from templates
    │
    ▼
REGISTERED          ← Filed with Hawaii DCCA (manual step, platform tracks)
    │
    ▼
EIN_OBTAINED        ← IRS EIN acquired
    │
    ▼
BANK_ACCOUNT        ← Operating account opened
    │
    ▼
FIRST_MEETING       ← Formation meeting held, minutes recorded
    │
    ▼
OPERATIONAL         ← Co-op is live
```

Each transition can be blocked (e.g., waiting on DCCA filing) and unblocked by an admin or advisor.

---

## Authentication & Authorization

- Authentication: email/password with email verification, plus optional OAuth (Google).
- Sessions: short-lived JWTs with refresh tokens.
- Authorization model: Role-based per organization (`organizer`, `member`, `admin`) + a global `advisor` role with explicit per-org access grants.
- Sensitive operations (vote tallying, financial entry, document deletion) require re-authentication or a signed action token.

---

## Document Generation

Hawaii-specific legal templates are stored as structured templates (variable substitution, not full legal authoring). On formation, the platform generates:

- Articles of Incorporation (HRS 414D or 421/421H as applicable)
- Model Bylaws
- Membership Application & Agreement
- Meeting Agenda template

Generated documents are stored in the file store and linked to the `Document` entity. They are starting points — members are reminded to have an attorney review before filing.

---

## Data Storage Summary

| Data type | Store | Notes |
|---|---|---|
| Relational entities | PostgreSQL | Organizations, members, proposals, votes |
| File blobs | S3-compatible object store | Documents, templates, meeting attachments |
| Auth sessions | Redis or DB-backed | Short-lived JWT refresh tokens |
| Audit log | Append-only DB table or WAL | Votes, financial entries, role changes |

---

## Deployment Model (v1)

Single-region deployment (likely US-West given Hawaii's timezone and latency). A managed cloud platform (Render, Railway, Fly.io, or AWS) is preferred over self-hosted infrastructure to minimize operational burden while the user base is small.

Environments: `development` → `staging` → `production`. CI runs tests and lint on every PR; deployment to staging is automatic on merge to main.
