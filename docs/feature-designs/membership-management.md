# Membership Tracking — Design Document

**Status:** Draft  
**Scope:** v1  
**Feature area:** Membership Management  

---

## 1. Goals

1. **Easy to join.** An interested person can submit a membership application in under five minutes. Admins get a clear queue to act on.
2. **Full roster visibility.** Any member can see who is in the co-op, their role, and their status. Nothing is hidden from members about other members.
3. **Equity transparency.** Every member sees their own equity balance and a full transaction history. Admins see equity across all members.
4. **Auditability.** Every membership change — admission, role change, equity entry, exit — is recorded and cannot be altered retroactively.
5. **Bylaw-aligned admission.** The admission flow can be configured to require a membership vote or admin approval only, matching the co-op's governing rules.

---

## 2. Core Concepts

### 2.1 Member Lifecycle

```
                    ┌────────────┐
  (application) ──► │  PENDING   │
                    └─────┬──────┘
                          │ approved (admin or vote)
                          ▼
                    ┌────────────┐
                    │   ACTIVE   │◄─── (returned from leave)
                    └──┬──────┬──┘
                       │      │
               (leave) │      │ (resignation / removal)
                       ▼      ▼
                    ┌──────┐  ┌────────┐
                    │ LEAVE│  │ EXITED │
                    └──────┘  └────────┘
```

| Status | Meaning |
|--------|---------|
| `pending` | Application submitted; awaiting approval. No voting rights. No equity contributions yet. |
| `active` | Full member. Can vote, contribute equity, access all documents. |
| `leave` | Temporarily inactive (e.g., extended travel). Preserved equity. No voting rights during leave. |
| `exited` | Membership ended. Record retained for audit. Equity balance frozen pending payout per bylaws. |

### 2.2 Roles

Roles are separate from status and can change independently.

| Role | Permissions |
|------|-------------|
| `member` | Read roster, view own equity, vote on proposals, attend meetings. |
| `admin` | All member permissions + approve/reject applications, record equity entries, change roles, manage meetings. |
| `organizer` | Admin permissions during formation only; demoted to `member` or `admin` once operational. |

A co-op must always have at least one `admin`. The system enforces this — an admin cannot be demoted if they are the last one.

---

## 3. Data Model

### 3.1 `members` Table

```sql
CREATE TABLE members (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id   UUID NOT NULL REFERENCES organizations(id),
  user_id           UUID NOT NULL REFERENCES users(id),

  -- Status & role
  status            TEXT NOT NULL CHECK (status IN ('pending','active','leave','exited')),
  role              TEXT NOT NULL CHECK (role IN ('organizer','member','admin')),

  -- Key dates
  applied_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  joined_at         TIMESTAMPTZ,                -- set when status changes to 'active'
  exited_at         TIMESTAMPTZ,                -- set when status changes to 'exited'

  -- Equity summary (denormalized for fast reads; source of truth is equity_entries)
  equity_balance_cents  BIGINT NOT NULL DEFAULT 0,

  -- Application data
  application_data  JSONB,                      -- answers to the co-op's custom application questions

  UNIQUE (organization_id, user_id)
);
```

### 3.2 `equity_entries` Table

Equity is tracked as a ledger — each row is a line item. The balance in `members.equity_balance_cents` is a running total, kept in sync on every insert.

```sql
CREATE TABLE equity_entries (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id  UUID NOT NULL REFERENCES organizations(id),
  member_id        UUID NOT NULL REFERENCES members(id),

  -- Entry details
  type             TEXT NOT NULL CHECK (type IN ('contribution', 'withdrawal', 'dividend', 'adjustment')),
  amount_cents     BIGINT NOT NULL,             -- positive = credit, negative = debit
  description      TEXT NOT NULL,
  effective_date   DATE NOT NULL,

  -- Audit
  recorded_by      UUID NOT NULL REFERENCES users(id),
  recorded_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  notes            TEXT
);
```

### 3.3 `membership_applications` Table

Stores the full application before it becomes a `member` row, and links to the admission proposal if a vote was required.

```sql
CREATE TABLE membership_applications (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id  UUID NOT NULL REFERENCES organizations(id),
  applicant_user_id UUID NOT NULL REFERENCES users(id),

  status           TEXT NOT NULL CHECK (status IN ('submitted','under_review','approved','rejected','withdrawn')),

  -- Application content
  full_name        TEXT NOT NULL,
  email            TEXT NOT NULL,
  phone            TEXT,
  statement        TEXT,                        -- "why do you want to join?"
  custom_answers   JSONB,                       -- co-op-configured questions

  -- Resolution
  reviewed_by      UUID REFERENCES users(id),
  reviewed_at      TIMESTAMPTZ,
  rejection_reason TEXT,

  -- If bylaws require a vote, linked proposal
  admission_proposal_id UUID REFERENCES proposals(id),

  submitted_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 3.4 `member_audit_log` Table

Append-only. Every status change, role change, or equity entry writes a row here.

```sql
CREATE TABLE member_audit_log (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id  UUID NOT NULL REFERENCES organizations(id),
  member_id        UUID NOT NULL REFERENCES members(id),
  actor_id         UUID NOT NULL REFERENCES users(id),

  event_type       TEXT NOT NULL,               -- 'status_changed', 'role_changed', 'equity_recorded', 'application_approved', etc.
  previous_state   JSONB,
  new_state        JSONB,
  occurred_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## 4. Application & Onboarding Flow

### 4.1 How a New Member Joins

**Step 1 — Discover the co-op**  
The admin generates a shareable invite link (or the applicant arrives via a public application URL if the co-op has that enabled). No account required to start.

**Step 2 — Submit application**  
The applicant fills out the membership application form:
- Name, email, phone (required)
- "Why do you want to join this co-op?" statement (required; min 50 chars to encourage thoughtful responses)
- Any custom questions the co-op has configured (optional per question)

The form is one page. No multi-step wizard here — joining should feel light. Submitting creates a `membership_applications` row with status `submitted` and sends a confirmation email to the applicant.

**Step 3 — Admin review queue**  
Admins see a "Pending Applications" card on their dashboard with a count badge. Each application shows: name, email, statement, submission date, and a direct link to approve or reject.

**Step 4a — Admin-only approval (default)**  
Admin reviews the application and clicks Approve or Reject.
- **Approve:** Creates a `members` row with `status = 'pending'` and sends a welcome email with an account setup link. Status moves to `active` once the new member completes account setup and signs the membership agreement.
- **Reject:** Marks the application `rejected`, stores an optional rejection reason (shown to the applicant in their notification), and sends a respectful decline email.

**Step 4b — Membership vote required (bylaw-configured)**  
If the co-op's settings require a membership vote for admission:
- Admin clicks "Refer to Vote," which automatically creates an admission `Proposal` linked to the application.
- The vote proceeds through the standard governance flow.
- If the vote passes, the application is auto-approved. If it fails, it's marked `rejected`.

**Step 5 — Account setup & onboarding**  
The approved applicant receives an email with a time-limited setup link (72 hours). They:
1. Create a password (or connect via Google OAuth).
2. Review and e-sign the membership agreement (PDF, generated from the co-op's template).
3. Land on their member dashboard with a first-time checklist: view the roster, read the bylaws, see their equity balance (starts at $0.00).

Once they sign, `members.status` transitions from `pending` → `active` and `joined_at` is recorded.

---

## 5. Member Roster

### 5.1 What Members See

The roster is visible to all active members. It shows:

| Field | Visibility |
|-------|-----------|
| Full name | All members |
| Role | All members |
| Status | All members |
| Join date | All members |
| Equity balance | **Admin only** (member can only see their own) |
| Contact info (email/phone) | **Admin only** |

The roster page has:
- Search by name
- Filter by status (`active`, `pending`, `leave`, `exited`)
- Filter by role (`member`, `admin`)
- Sort by name, join date, or equity (admin only)

### 5.2 What Admins See

Admins see the full table including equity balances and contact info. An export button lets them download the roster as CSV (name, email, role, status, join date, equity balance). This is useful for external accounting or legal filings.

---

## 6. Equity Tracking

### 6.1 Overview

Equity is the financial stake a member holds in the co-op. In v1, admins record equity transactions manually — there is no payment processing. The platform is the ledger, not the payment rail.

Equity entry types:

| Type | Description | Sign |
|------|-------------|------|
| `contribution` | Member pays in (initial share purchase, dues, etc.) | + |
| `withdrawal` | Member withdraws equity (per bylaws, usually limited) | − |
| `dividend` | Patronage dividend allocated by the co-op | + |
| `adjustment` | Correction or allocation (e.g., depreciation, error fix) | ± |

### 6.2 Recording an Entry (Admin Flow)

From any member's profile or the Equity section of the admin panel, an admin clicks **+ Record Transaction**:

1. Select member (pre-filled if on member profile)
2. Select type
3. Enter amount (positive number; type determines direction)
4. Effective date (defaults to today)
5. Description (required; appears in the member's transaction log)
6. Optional notes (admin-only; for context not shown to member)

On save, the system:
- Inserts a row into `equity_entries`
- Updates `members.equity_balance_cents`
- Appends to `member_audit_log`

Both operations run in a database transaction. If either fails, neither commits.

### 6.3 Member Equity Dashboard

Each member has an Equity tab on their profile showing:

- Current balance (large, prominent)
- Total contributed (lifetime sum of contributions)
- Total dividends received (lifetime)
- Transaction history table: date, type, description, amount, running balance

The transaction history is read-only for members. Only admins can add or correct entries.

### 6.4 Co-op Equity Summary (Admin)

The admin Equity dashboard shows:

- Total member equity across the co-op (sum of all active member balances)
- A table: each active member, their balance, their % share of total equity, their last transaction date
- Sortable by name, balance, % share, last activity

This gives the board a one-page answer to "how invested is each member?"

---

## 7. Configuration Per Co-op

Admins can configure membership behavior under Settings → Membership:

| Setting | Options | Default |
|---------|---------|---------|
| Application mode | Public (anyone with the link) / Invite-only | Invite-only |
| Admission requires | Admin approval / Membership vote | Admin approval |
| Membership agreement | Use platform template / Upload custom PDF | Platform template |
| Custom application questions | Add/remove/reorder questions | None |
| Minimum equity contribution | Dollar amount (or $0 for no requirement) | $0 |
| Allow member self-exit | Yes / No | Yes |

---

## 8. API Sketch

All endpoints are scoped under `/api/v1/organizations/{org_id}/`.

### Applications

| Method | Path | Who | Description |
|--------|------|-----|-------------|
| `POST` | `applications` | Public / invited | Submit a new membership application |
| `GET` | `applications` | Admin | List applications (filterable by status) |
| `GET` | `applications/{id}` | Admin | Retrieve a single application |
| `POST` | `applications/{id}/approve` | Admin | Approve application (creates member row) |
| `POST` | `applications/{id}/reject` | Admin | Reject application |
| `POST` | `applications/{id}/refer-to-vote` | Admin | Create admission proposal for application |

### Members

| Method | Path | Who | Description |
|--------|------|-----|-------------|
| `GET` | `members` | Member | List all members (equity omitted for non-admins) |
| `GET` | `members/{id}` | Member (own) / Admin | Get member details |
| `PATCH` | `members/{id}` | Admin | Update role or status |
| `GET` | `members/{id}/equity` | Member (own) / Admin | Get equity balance and transaction history |

### Equity Entries

| Method | Path | Who | Description |
|--------|------|-----|-------------|
| `POST` | `members/{id}/equity/entries` | Admin | Record a new equity transaction |
| `GET` | `equity/summary` | Admin | Co-op-wide equity summary |

### Invite Links

| Method | Path | Who | Description |
|--------|------|-----|-------------|
| `POST` | `invites` | Admin | Generate a new invite link |
| `DELETE` | `invites/{id}` | Admin | Revoke an invite link |
| `GET` | `invites` | Admin | List active invite links |

---

## 9. Key UI Pages

| Page | Path | Audience |
|------|------|----------|
| Application form | `/apply/{org_slug}` or `/invite/{token}` | Public / invited |
| Application queue | `/admin/applications` | Admin |
| Member roster | `/members` | All members |
| Member profile | `/members/{id}` | All members (limited) + Admin (full) |
| My equity | `/me/equity` | Member (own view) |
| Equity admin | `/admin/equity` | Admin |
| Membership settings | `/admin/settings/membership` | Admin |

---

## 10. Open Questions

1. **Equity denominated in units vs. USD?** Housing co-ops sometimes use share units rather than dollar amounts. v1 defaults to USD; should we make this configurable?

2. **Member-initiated exit flow.** If a member resigns, do we require admin acknowledgement before marking them `exited`, or is self-exit immediate? Bylaws vary. Default proposed: admin must confirm within 30 days, or it auto-completes.

3. **Waiting list.** If a co-op has a size cap (common for housing co-ops), rejected-for-capacity applicants should go on a waitlist rather than getting a hard rejection. Worth building in v1 or defer?

4. **Equity payout on exit.** When a member exits, the platform records their final balance but does not process any payout. Should the admin dashboard include a "pending payout" state to track that the cash transfer still needs to happen externally?

5. **Member-to-member visibility of equity.** The current design hides individual balances from non-admins. Some co-ops prefer full transparency. Should this be a per-co-op setting?

---

## 11. Out of Scope (v1)

- Payment processing for equity contributions (manual entry only)
- Automated dues collection or recurring billing
- Equity transfer between members (share sales)
- Legal compliance enforcement for equity cap rules (HRS 421 limits)
- Bulk import of existing member rosters from CSV
