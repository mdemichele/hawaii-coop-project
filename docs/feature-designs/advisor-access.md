# Advisor Access — Design Document

**Status:** Draft  
**Scope:** v1  
**Feature area:** Advisor Access  

---

## 1. Goals

1. **External experts can help without becoming members.** A lawyer, accountant, USDA agent, or community organization can be granted access to a co-op's platform without holding equity, voting rights, or a seat on any roster.
2. **Multi-co-op visibility.** An advisor who supports several co-ops sees all of them from a single dashboard — they are not required to log in and out of separate accounts.
3. **Scoped access.** The co-op controls exactly what an advisor can see and do. Access is per-co-op and can be revoked at any time without affecting the advisor's access to other co-ops.
4. **Formation-specific usefulness.** Advisors are most valuable during formation — reviewing documents, flagging issues with a step, leaving structured notes for the organizer. The platform makes this workflow explicit.
5. **No governance interference.** Advisors never count toward quorum, cannot vote on proposals, and are excluded from all membership calculations.

---

## 2. Core Concepts

### 2.1 What an Advisor Is

An advisor is a platform user with a global `advisor` role. Unlike members, advisors are not attached to a single organization — they hold an explicit per-organization access grant for each co-op they support. A user can be an advisor and a member simultaneously (e.g., a lawyer who is also a founding member of a different co-op), but within any single organization they hold only one role.

### 2.2 Access Scopes

Each advisor-organization relationship has a scope:

| Scope | What the advisor can do |
|-------|------------------------|
| `read` | View all co-op data: member roster (limited), formation progress, proposals, meeting agendas, documents, minutes. Cannot make any changes. |
| `write` | Everything in `read`, plus: leave notes on formation steps, upload documents, comment on proposals. Cannot approve members, cast votes, record equity, or change settings. |

`write` scope is intentionally narrow — advisors assist, they do not administer. If a co-op wants an advisor to have admin-level control they should add them as a member with the `admin` role instead.

### 2.3 What Advisors Cannot Do (Regardless of Scope)

- Vote on governance proposals
- Be counted in quorum calculations (proposals or meetings)
- Approve or reject membership applications
- Record equity transactions
- Change co-op settings
- Add or remove other advisors
- Access another advisor's notes on a different co-op

---

## 3. Data Model

### 3.1 `advisor_profiles` Table

Stores advisor-specific metadata. One row per user who holds the global `advisor` role.

```sql
CREATE TABLE advisor_profiles (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id           UUID NOT NULL UNIQUE REFERENCES users(id),

  -- Professional identity
  display_name      TEXT NOT NULL,
  organization_name TEXT,                            -- firm, agency, or institution name
  title             TEXT,                            -- "Attorney at Law", "USDA Farm Loan Officer", etc.
  advisor_type      TEXT NOT NULL
                    CHECK (advisor_type IN ('attorney','accountant','usda_agent','community_org','other')),
  bio               TEXT,                            -- short description shown to co-op admins
  website_url       TEXT,
  phone             TEXT,

  created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 3.2 `organization_advisor_access` Table

The join table that grants an advisor access to a specific co-op. Adding a row here gives access; removing it revokes it.

```sql
CREATE TABLE organization_advisor_access (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id  UUID NOT NULL REFERENCES organizations(id),
  advisor_user_id  UUID NOT NULL REFERENCES users(id),

  scope            TEXT NOT NULL DEFAULT 'read'
                   CHECK (scope IN ('read','write')),

  -- Who granted access and when
  granted_by       UUID NOT NULL REFERENCES users(id),
  granted_at       TIMESTAMPTZ NOT NULL DEFAULT now(),

  -- Optional expiry (useful for time-limited engagements)
  expires_at       TIMESTAMPTZ,

  -- Revocation
  revoked_by       UUID REFERENCES users(id),
  revoked_at       TIMESTAMPTZ,

  -- Active flag (soft delete; row is retained for audit)
  is_active        BOOLEAN NOT NULL DEFAULT true,

  UNIQUE (organization_id, advisor_user_id)
);
```

### 3.3 `advisor_notes` Table

Structured notes left by advisors on specific contexts within a co-op. Notes are associated with a context type and context ID (a formation stage, a proposal, a document, or a meeting) so they appear inline where relevant rather than in a separate inbox.

```sql
CREATE TABLE advisor_notes (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id  UUID NOT NULL REFERENCES organizations(id),
  advisor_user_id  UUID NOT NULL REFERENCES users(id),

  -- What the note is attached to
  context_type     TEXT NOT NULL
                   CHECK (context_type IN ('formation_stage','proposal','document','meeting','general')),
  context_id       TEXT,                             -- UUID of the formation stage, proposal, document, or meeting
                                                     -- null for 'general' notes

  body             TEXT NOT NULL,                    -- Markdown supported
  is_urgent        BOOLEAN NOT NULL DEFAULT false,   -- surfaces note prominently to admins

  created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  edited_at        TIMESTAMPTZ,
  resolved_at      TIMESTAMPTZ,                      -- admin marks the note as addressed
  resolved_by      UUID REFERENCES users(id)
);
```

### 3.4 `advisor_access_log` Table

Append-only audit trail of advisor activity within a co-op.

```sql
CREATE TABLE advisor_access_log (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id  UUID NOT NULL REFERENCES organizations(id),
  advisor_user_id  UUID NOT NULL REFERENCES users(id),

  event_type       TEXT NOT NULL,                    -- 'access_granted', 'access_revoked', 'note_added', 'document_viewed', etc.
  details          JSONB,
  occurred_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## 4. Advisor Flows

### 4.1 Granting an Advisor Access (Admin Flow)

1. Admin navigates to **Settings → Advisors → + Add Advisor**.
2. Admin enters the advisor's email address.
   - If the email matches an existing advisor profile: the advisor's name, title, and organization are shown for confirmation.
   - If the email does not match an existing user: an invite email is sent. The recipient creates an account and completes their advisor profile before access is active.
3. Admin selects scope: `Read only` or `Read + Write`.
4. Admin optionally sets an expiry date (e.g., for a fixed-term engagement).
5. Admin clicks **Grant Access**.
   - Inserts a row into `organization_advisor_access`.
   - Notifies the advisor by email.
   - Appends to `advisor_access_log`.

### 4.2 Advisor Account Setup

When an advisor is invited for the first time (no existing account):

1. They receive an email: "[Co-op Name] has invited you to advise them on Hawaii Co-op Project."
2. They click the link and create an account with email/password or Google OAuth.
3. They are prompted to complete their advisor profile: display name, title, advisor type, organization name, and optional bio.
4. On completion, their access to the inviting co-op is activated.

Subsequent invitations from other co-ops do not require profile re-entry — the existing profile is reused.

### 4.3 Revoking Advisor Access (Admin Flow)

1. Admin navigates to **Settings → Advisors**.
2. Finds the advisor in the active advisor list.
3. Clicks **Revoke Access** and confirms.
4. `organization_advisor_access.is_active` → false; `revoked_by` and `revoked_at` set.
5. Advisor's session for this co-op is invalidated immediately.
6. Advisor receives an email notification that their access has been removed.
7. The co-op disappears from the advisor's dashboard.

### 4.4 Scope Change (Admin Flow)

Admin can upgrade `read` → `write` or downgrade `write` → `read` from the advisor settings page. The change takes effect immediately. Appended to the audit log.

### 4.5 Advisor Leaving a Note

Available only to advisors with `write` scope.

1. On any supported page (formation stage, proposal detail, document detail, meeting detail), the advisor sees an **Advisor Notes** panel.
2. They click **+ Add Note**, type their note in a Markdown editor, and optionally toggle **Mark as Urgent**.
3. On save:
   - Inserts into `advisor_notes`.
   - If `is_urgent = true`: notifies all admins of this co-op immediately (email + in-app).
   - If not urgent: admins see the note inline when they next visit the page; no push notification.
4. The note appears in the Advisor Notes panel on that page, attributed to the advisor by name and timestamp.

### 4.6 Admin Resolving a Note

Admins can mark an advisor note as resolved once the issue has been addressed:

1. Admin clicks **Mark Resolved** on a note.
2. `advisor_notes.resolved_at` and `resolved_by` are set.
3. The note collapses to a muted "Resolved" state in the UI (still visible, just not prominent).
4. The advisor is notified in-app that their note was marked resolved.

---

## 5. Advisor Dashboard

The advisor's primary view is a multi-co-op dashboard — the main page they land on after logging in.

### 5.1 Dashboard Layout

**Header:** Advisor name and profile summary. Link to edit profile.

**Co-op list:** A card per co-op the advisor is actively advising, sorted by most recently active.

Each card shows:
- Co-op name and type (Housing / Agricultural)
- Island
- Formation status (stage name + status badge if still in formation; "Operational" if past formation)
- Access scope badge (Read Only / Read + Write)
- Unresolved advisor notes count (notes they left that haven't been marked resolved)
- Last activity date
- "Open" button → takes advisor into that co-op's view

**Filter/sort:** Filter by co-op type, formation status, island. Sort by name, last activity, or formation stage.

**Past engagements:** A collapsed section showing co-ops where access has been revoked or expired. Read-only; retained for the advisor's reference.

### 5.2 Advisor View Within a Co-op

When an advisor opens a co-op they see the same navigation as a member, but with visual indicators of their role:

- A persistent banner at the top: **"You are advising [Co-op Name] — [Read Only / Read + Write]"**
- Actions they cannot take are hidden (not greyed out) — they should not see a "Cast Vote" button at all.
- Their own notes appear inline in the relevant pages.

**What advisors can see:**

| Section | Advisor sees |
|---------|-------------|
| Formation wizard | All stages, completion status, advisor notes panel per stage |
| Member roster | Names and roles only — no contact info, no equity balances |
| Governance | All proposals (open and closed), full vote tallies post-close, comments |
| Document Library | All published documents; can download; no drafts unless write scope |
| Meetings | All scheduled and past meetings, agendas, finalized minutes |
| Settings | Read-only view of co-op name, type, island — no configuration access |

---

## 6. Formation Stage Notes (Detail)

The formation wizard is the highest-value surface for advisor involvement — this is where legal and financial expertise most directly prevents mistakes.

Each formation stage panel includes an **Advisor Notes** section at the bottom:

- All notes from all advisors for this stage are shown, attributed by name and timestamp.
- Urgent notes are shown with a warning indicator.
- Resolved notes are collapsed but accessible via "Show resolved."
- Admins can reply to a note by leaving their own note in the same context (there is no threaded reply — both parties just add notes to the same list).

**Specific value per stage:**

| Stage | What an advisor typically flags |
|-------|---------------------------------|
| Legal Structure | Wrong HRS chapter selected; DHHL applicability missed |
| Name & Purpose | Name conflicts; purpose clause too narrow for intended activities |
| Governing Documents | Non-standard provisions; missing required clauses; errors in member names |
| Formation Vote | Quorum issues; signature requirements |
| DCCA Filing | Incorrect filing fee; missing attachments; filing form version outdated |
| EIN | Wrong entity type selected on IRS form |
| Bank Account | Missing resolution language; wrong signatories |

---

## 7. Access Expiry

If an access grant has an `expires_at` date:

- 7 days before expiry: advisor and co-op admins are notified.
- On expiry: `is_active` is set to false by a background job. Advisor session for that co-op is invalidated.
- Admins can extend the expiry or re-grant access at any time.

Expired access is treated identically to revoked access from the system's perspective — the distinction is only in the audit log (`event_type: 'access_expired'` vs. `'access_revoked'`).

---

## 8. Tenant Isolation for Advisors

Advisors cross organizational boundaries — they are the one user type that can see data from multiple co-ops. This makes tenant isolation enforcement especially important.

**Rules enforced at the application middleware layer:**

1. Every request from an advisor to a co-op resource checks `organization_advisor_access` for an active, non-expired grant before proceeding.
2. An advisor's session token encodes their `user_id` — not any `organization_id`. The organization is resolved per-request from the URL parameter, then validated against the access table.
3. If no active grant exists for that `(advisor_user_id, organization_id)` pair, the request returns 403 — the same response as if the co-op didn't exist. No information leakage.

**Row-level security (database layer):**

The `organization_advisor_access` table is checked in all advisor-specific database policies. Co-op tables with RLS policies enforce that an advisor's query can only return rows for organizations where they hold an active grant — a second line of defense against misconfigured application code.

---

## 9. Advisor Type Considerations

The `advisor_type` field on the profile affects what contextual information the platform surfaces to co-op admins:

| Advisor type | Platform surfaces to admins |
|---|---|
| `attorney` | A reminder that this advisor can review governing documents for legal compliance |
| `accountant` | A reminder that this advisor can review equity tracking and financial documents |
| `usda_agent` | Links to relevant USDA programs (FSA farm loans, Rural Development) that the co-op may qualify for |
| `community_org` | Notes that this advisor may have experience connecting co-ops to local resources |
| `other` | No special context |

This is informational only — the platform does not restrict what any advisor type can view or do based on their type.

---

## 10. Notifications

| Event | Who is notified | Channel |
|-------|----------------|---------|
| Advisor access granted | Advisor | Email |
| Advisor invite (new user) | Invitee | Email |
| Advisor access revoked | Advisor | Email |
| Advisor access expiring in 7 days | Advisor + co-op admins | Email + in-app |
| Advisor access expired | Co-op admins | In-app |
| Urgent advisor note posted | All co-op admins | Email + in-app |
| Non-urgent advisor note posted | All co-op admins | In-app (next login) |
| Advisor note marked resolved | Note author (advisor) | In-app |
| Scope changed (read ↔ write) | Advisor | Email |

Advisor notifications use the same notification system defined in the notifications design doc. Advisor-specific events fall under a new `advisor` category in the notification preferences — this category is only shown to users who hold the advisor role.

---

## 11. API Sketch

### Advisor Profile (user-scoped)

| Method | Path | Who | Description |
|--------|------|-----|-------------|
| `GET` | `advisor/profile` | Advisor | Get own advisor profile |
| `PUT` | `advisor/profile` | Advisor | Update own advisor profile |
| `GET` | `advisor/organizations` | Advisor | List all co-ops the advisor has active access to |

### Access Grants (org-scoped)

| Method | Path | Who | Description |
|--------|------|-----|-------------|
| `GET` | `organizations/{org_id}/advisors` | Admin | List all advisors for a co-op |
| `POST` | `organizations/{org_id}/advisors` | Admin | Grant an advisor access |
| `PATCH` | `organizations/{org_id}/advisors/{id}` | Admin | Update scope or expiry |
| `DELETE` | `organizations/{org_id}/advisors/{id}` | Admin | Revoke advisor access |

### Advisor Notes

| Method | Path | Who | Description |
|--------|------|-----|-------------|
| `GET` | `organizations/{org_id}/advisor-notes` | Admin / Advisor | List all notes for a co-op (filterable by context type, resolved status) |
| `POST` | `organizations/{org_id}/advisor-notes` | Advisor (write) | Create a new note |
| `PATCH` | `organizations/{org_id}/advisor-notes/{id}` | Advisor (own notes) | Edit a note |
| `POST` | `organizations/{org_id}/advisor-notes/{id}/resolve` | Admin | Mark note resolved |
| `GET` | `organizations/{org_id}/advisor-notes?context_type=formation_stage&context_id={stage}` | Admin / Advisor | Get notes for a specific context |

### Advisor Dashboard (cross-org)

| Method | Path | Who | Description |
|--------|------|-----|-------------|
| `GET` | `advisor/dashboard` | Advisor | Aggregated summary of all active co-ops with key status fields |

---

## 12. Key UI Pages

| Page | Path | Audience |
|------|------|----------|
| Advisor dashboard | `/advisor` | Advisor |
| Advisor profile settings | `/advisor/profile` | Advisor |
| Co-op view (as advisor) | `/organizations/{org_id}/...` | Advisor (same routes as member, scoped) |
| Advisor settings for a co-op | `/admin/settings/advisors` | Admin |
| All advisor notes for a co-op | `/admin/advisor-notes` | Admin |

---

## 13. Open Questions

1. **Can an advisor invite another advisor?** The current design restricts advisor grants to admins only. A senior advisor might want to bring in a specialist (e.g., an attorney bringing in an accountant for the financial sections). Should advisors with `write` scope be able to invite other advisors, or is this always an admin action?

2. **Advisor note visibility to members.** Currently only admins can see advisor notes. Should active members also be able to read advisor notes — at least non-urgent ones? Transparency is a co-op value, but some advisors may be more candid knowing their notes are admin-only.

3. **Advisor commenting on proposals.** The current `write` scope allows advisors to comment on proposals (in the proposal comment thread). Should advisor comments be visually distinguished from member comments, so members know they are reading outside expert input rather than a peer member's view?

4. **Advisor-to-advisor communication.** If two advisors are advising the same co-op, they can read each other's notes but cannot communicate directly through the platform. Is this a gap worth filling in v1, or is out-of-band communication (email) acceptable?

5. **Advisor directory.** Should there be a discoverable directory of advisors that co-op organizers can browse when looking for help? This would require advisors to opt in to being listed. Deferred to v2 in the current plan — does that still hold?

6. **Billing model for advisors.** If the platform is eventually monetized, do advisors pay separately, are they included in the co-op's plan, or is advisor access always free? This doesn't affect v1 design but should be considered before the access grant model is finalized.

---

## 14. Out of Scope (v1)

- Advisor directory / public search for advisors
- Advisors inviting other advisors
- Threaded replies on advisor notes (both parties add to the same note list)
- Advisor billing or subscription management
- Advisor-specific document upload permissions beyond the write scope
- Integration with professional licensing databases (e.g., Hawaii State Bar) to verify advisor credentials
- Time tracking or engagement logging for billable advisory hours
- Advisor-generated reports or exports scoped to their co-op portfolio
