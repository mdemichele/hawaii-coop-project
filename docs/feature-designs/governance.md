# Governance — Design Document

**Status:** Draft  
**Scope:** v1  
**Feature area:** Governance (Proposals & Voting)  

---

## 1. Goals

1. **Democratic by default.** Any active member can raise a proposal. The platform enforces quorum and passage rules so outcomes are legitimate, not just whoever shouted loudest.
2. **Transparent outcomes, blind process.** Members can see who voted after a vote closes, but the running tally is hidden during the voting period to prevent bandwagon effects.
3. **Bylaw-aligned rules.** Quorum thresholds and passage requirements are configurable per co-op and per proposal type, reflecting the actual governing documents the co-op adopted.
4. **Immutable audit trail.** Every vote cast is a permanent record. Nothing can be deleted or altered after the fact.
5. **Low friction for members.** Casting a vote should take under 60 seconds. Notification, one click to the proposal, vote cast. Done.

---

## 2. Core Concepts

### 2.1 Proposal Lifecycle

```
              ┌─────────┐
              │  DRAFT  │  ← author editing, not yet visible to members
              └────┬────┘
                   │ submitted
                   ▼
              ┌─────────┐
              │  OPEN   │  ← voting period active; tally hidden
              └────┬────┘
                   │ deadline reached or all members voted
                   ▼
              ┌──────────┐
              │  CLOSED  │  ← tally revealed; outcome determined
              └────┬─────┘
                   │
         ┌─────────┼──────────┐
         ▼         ▼          ▼
      PASSED   FAILED    FAILED_QUORUM
```

| Status | Meaning |
|--------|---------|
| `draft` | Author is still writing. Invisible to other members. |
| `open` | Voting period is active. Members can cast votes. Tally is hidden. |
| `closed` | Voting period has ended. Tally is public. Outcome determined. |
| `passed` | Closed + quorum met + pass threshold met. |
| `failed` | Closed + quorum met + pass threshold not met. |
| `failed_quorum` | Closed + quorum not met. Outcome is neither passed nor failed on the merits. |

`PASSED` and `FAILED` are terminal. A failed proposal can be resubmitted as a new proposal — there is no "reopen" operation. A `FAILED_QUORUM` result carries no precedent on the merits.

### 2.2 Proposal Types

| Type | Typical use | Notes |
|------|-------------|-------|
| `bylaw_amendment` | Changing the co-op's governing rules | Higher pass threshold typical (e.g., 2/3) |
| `membership_admission` | Admitting a new member when bylaws require a vote | Linked to a `membership_applications` row |
| `budget_approval` | Approving an operating budget or major expenditure | Often lower threshold (simple majority) |
| `general` | Any other member-initiated proposal | Default thresholds apply |

Proposal types exist primarily to allow different quorum and threshold defaults. Admins can override thresholds per proposal regardless of type.

### 2.3 Quorum & Passage Rules

**Quorum** — the minimum fraction of active members who must cast a vote (yes, no, or abstain) for the result to be valid. If quorum is not met when the voting period closes, the result is `failed_quorum` regardless of the yes/no split.

**Pass threshold** — the fraction of votes cast that must be `yes` for the proposal to pass. Abstentions count toward quorum but do not count toward the pass threshold numerator or denominator.

Example: 10 active members, quorum = 60%, pass = simple majority.
- At least 6 members must vote for the result to be valid.
- Of those who voted yes or no, more than half must vote yes.
- If 7 vote (quorum met), 3 yes / 3 no / 1 abstain → tally on yes/no votes: 3 yes out of 6 = 50% → fails (not strictly above 50%).

---

## 3. Data Model

### 3.1 `proposals` Table

```sql
CREATE TABLE proposals (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id  UUID NOT NULL REFERENCES organizations(id),
  author_id        UUID NOT NULL REFERENCES members(id),

  -- Identity
  title            TEXT NOT NULL,
  description      TEXT NOT NULL,                    -- Markdown supported
  type             TEXT NOT NULL
                   CHECK (type IN ('bylaw_amendment','membership_admission','budget_approval','general')),

  -- Voting rules (set when proposal is submitted; immutable after)
  quorum_pct       NUMERIC(5,2) NOT NULL,            -- e.g. 60.00 means 60%
  pass_threshold   TEXT NOT NULL
                   CHECK (pass_threshold IN ('simple_majority','two_thirds','unanimous')),
  eligible_voters  INT NOT NULL,                     -- snapshot of active member count at open time

  -- Timing
  voting_opens_at  TIMESTAMPTZ NOT NULL,
  voting_closes_at TIMESTAMPTZ NOT NULL,

  -- State
  status           TEXT NOT NULL DEFAULT 'draft'
                   CHECK (status IN ('draft','open','closed','passed','failed','failed_quorum')),

  -- Resolution (populated on close)
  yes_count        INT,
  no_count         INT,
  abstain_count    INT,
  outcome_notes    TEXT,                             -- plain-language summary of result

  -- Links
  linked_application_id UUID REFERENCES membership_applications(id),  -- for membership_admission type
  linked_document_id    UUID REFERENCES documents(id),                 -- for bylaw_amendment type

  created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  submitted_at     TIMESTAMPTZ,                      -- when status moved from draft → open
  closed_at        TIMESTAMPTZ
);
```

### 3.2 `votes` Table

Immutable. Rows are never updated or deleted.

```sql
CREATE TABLE votes (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id  UUID NOT NULL REFERENCES organizations(id),
  proposal_id      UUID NOT NULL REFERENCES proposals(id),
  member_id        UUID NOT NULL REFERENCES members(id),

  choice           TEXT NOT NULL CHECK (choice IN ('yes','no','abstain')),
  voted_at         TIMESTAMPTZ NOT NULL DEFAULT now(),

  UNIQUE (proposal_id, member_id)                   -- one vote per member per proposal
);

-- Prevent updates and deletes at the database layer
CREATE RULE no_update_votes AS ON UPDATE TO votes DO INSTEAD NOTHING;
CREATE RULE no_delete_votes AS ON DELETE TO votes DO INSTEAD NOTHING;
```

### 3.3 `proposal_comments` Table

Members can leave comments on open proposals (discussion, not binding). Comments are not votes.

```sql
CREATE TABLE proposal_comments (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  proposal_id      UUID NOT NULL REFERENCES proposals(id),
  author_id        UUID NOT NULL REFERENCES members(id),
  body             TEXT NOT NULL,
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  edited_at        TIMESTAMPTZ,
  deleted_at       TIMESTAMPTZ                       -- soft delete; body replaced with "[removed]"
);
```

### 3.4 `governance_settings` Table

One row per organization. Stores default thresholds per proposal type, overridable per proposal.

```sql
CREATE TABLE governance_settings (
  organization_id       UUID PRIMARY KEY REFERENCES organizations(id),

  -- Default quorum percentages per type
  quorum_bylaw_amendment      NUMERIC(5,2) NOT NULL DEFAULT 75.00,
  quorum_membership_admission NUMERIC(5,2) NOT NULL DEFAULT 60.00,
  quorum_budget_approval      NUMERIC(5,2) NOT NULL DEFAULT 60.00,
  quorum_general              NUMERIC(5,2) NOT NULL DEFAULT 50.00,

  -- Default pass thresholds per type
  threshold_bylaw_amendment      TEXT NOT NULL DEFAULT 'two_thirds',
  threshold_membership_admission TEXT NOT NULL DEFAULT 'simple_majority',
  threshold_budget_approval      TEXT NOT NULL DEFAULT 'simple_majority',
  threshold_general              TEXT NOT NULL DEFAULT 'simple_majority',

  -- Who can submit proposals
  proposal_submission    TEXT NOT NULL DEFAULT 'any_member'
                         CHECK (proposal_submission IN ('any_member','admin_only')),

  -- Minimum voting period in hours
  min_voting_period_hours INT NOT NULL DEFAULT 48,

  updated_at             TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 3.5 `governance_audit_log` Table

Append-only. Records every state transition and vote event.

```sql
CREATE TABLE governance_audit_log (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id  UUID NOT NULL REFERENCES organizations(id),
  proposal_id      UUID REFERENCES proposals(id),
  actor_id         UUID REFERENCES members(id),

  event_type       TEXT NOT NULL,                    -- 'proposal_submitted', 'vote_cast', 'proposal_closed', etc.
  details          JSONB,
  occurred_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## 4. Proposal Flows

### 4.1 Creating a Proposal

1. Member clicks **+ New Proposal** from the Governance section.
2. Selects type. The form auto-populates quorum and pass threshold from the co-op's `governance_settings` defaults for that type.
3. Fills in:
   - Title (required; max 120 characters)
   - Description (required; Markdown editor with preview)
   - Voting opens: immediately on submit, or a scheduled future date
   - Voting closes: date and time (must be at least `min_voting_period_hours` after opens)
   - Optional: override quorum % and/or pass threshold (admin only; member proposals use defaults)
4. Member saves as **draft** (visible only to them) or **submits** (opens the voting period immediately or on scheduled date).
5. On submit, `eligible_voters` is snapshotted from the current active member count. Members who join after a proposal opens are not eligible to vote on it.

For `membership_admission` proposals, the system auto-generates the proposal from the application review flow — the author selects the applicant and the title/description are pre-filled. The member can edit the description before submitting.

For `bylaw_amendment` proposals, the author can link a document from the Document Library (e.g., a revised bylaws PDF). This link is displayed prominently in the proposal view.

### 4.2 Voting

1. When a proposal opens, all eligible members are notified (email + in-app).
2. Member opens the proposal. They see:
   - Title, description, type
   - Who submitted it and when
   - Voting deadline and time remaining
   - Quorum requirement and pass threshold (no live tally shown)
   - Any linked documents
   - A comment thread (read + write)
   - Three buttons: **Vote Yes**, **Vote No**, **Abstain**
3. Member casts a vote. The choice is recorded immediately. A confirmation message is shown: *"Your vote has been recorded. You cannot change it."*
4. Votes are final. There is no edit or retract option.
5. A reminder notification is sent to members who have not voted 24 hours before the deadline.

### 4.3 Closing a Proposal

Proposals close automatically when `voting_closes_at` passes. A background job runs every 15 minutes to check for proposals past their close time and processes them.

**Close processing:**
1. Count `yes_count`, `no_count`, `abstain_count` from the `votes` table.
2. Calculate participation: `(yes_count + no_count + abstain_count) / eligible_voters`.
3. If participation < `quorum_pct`: status → `failed_quorum`.
4. Otherwise, calculate the pass ratio: `yes_count / (yes_count + no_count)`.
   - `simple_majority`: pass ratio > 0.50
   - `two_thirds`: pass ratio >= 0.667
   - `unanimous`: yes_count == (yes_count + no_count) (no no-votes; abstains allowed)
5. If pass ratio meets threshold: status → `passed`. Otherwise: status → `failed`.
6. Write `yes_count`, `no_count`, `abstain_count`, `outcome_notes`, and `closed_at` to the proposal row.
7. Tally becomes visible to all members.
8. Append to `governance_audit_log`.
9. Notify all eligible voters of the outcome (email + in-app).

**Side effects on pass:**
- `membership_admission` proposal passes → linked application auto-approved, member onboarding email sent.
- `bylaw_amendment` proposal passes → admin is prompted to upload the amended document to the Document Library.

Admins can manually close a proposal before its deadline (e.g., if all members have voted early). Manual close is recorded in the audit log with the actor's ID.

---

## 5. Proposal List & Detail UI

### 5.1 Proposal List

The Governance page shows three tabs:

| Tab | Contents |
|-----|----------|
| **Active** | All open proposals, sorted by deadline (soonest first). Shows title, type, time remaining, and whether the current member has voted. |
| **Past** | All closed proposals sorted by close date descending. Shows title, type, outcome badge (Passed / Failed / Failed Quorum), and final vote tally. |
| **My Drafts** | Current member's own draft proposals. |

Each row in the Active tab shows a **"Vote Now"** button if the member hasn't voted, or a muted **"Voted"** indicator if they have. No tally is shown for open proposals on the list view.

### 5.2 Proposal Detail

**Header:** Title, type badge, status badge, author, submitted date.

**Voting rules strip:** Quorum requirement, pass threshold, deadline, eligible voter count.

**Body:** Full description (rendered Markdown). Linked documents (if any).

**Vote panel (right sidebar or bottom card):**
- If open and member hasn't voted: three vote buttons.
- If open and member has voted: shows their choice; greyed-out buttons.
- If closed: full tally — yes/no/abstain counts, participation %, outcome, and a list of who voted what (name + choice). The who-voted-what list is visible to all members after close.

**Comments section:** Below the proposal body. Members can post comments while the proposal is open. Comments remain visible after close as a record of the discussion.

---

## 6. Governance Settings (Admin)

Admins configure governance rules under **Settings → Governance**.

| Setting | Description | Default |
|---------|-------------|---------|
| Who can submit proposals | `Any member` or `Admins only` | Any member |
| Minimum voting period | Hours from open to earliest possible close | 48 hours |
| Quorum — bylaw amendment | % of active members required | 75% |
| Quorum — membership admission | % of active members required | 60% |
| Quorum — budget approval | % of active members required | 60% |
| Quorum — general | % of active members required | 50% |
| Pass threshold — bylaw amendment | simple majority / two-thirds / unanimous | Two-thirds |
| Pass threshold — membership admission | simple majority / two-thirds / unanimous | Simple majority |
| Pass threshold — budget approval | simple majority / two-thirds / unanimous | Simple majority |
| Pass threshold — general | simple majority / two-thirds / unanimous | Simple majority |

Changes to governance settings apply to proposals submitted after the change. Proposals already open are not affected — their rules were snapshotted at open time.

---

## 7. Tally Visibility Rules

This is a critical design decision. The table below is the single source of truth.

| Viewer | Proposal open | Proposal closed |
|--------|--------------|-----------------|
| Any member — own vote | Visible (their choice shown) | Visible |
| Any member — tally | **Hidden** | Visible (all counts + who voted what) |
| Admin — tally | **Hidden** | Visible |
| Advisor | **Hidden** | Visible |

No exceptions. Admins do not see the live tally. This is intentional — if admins could see the live tally, they could time a manual close to lock in a favorable outcome.

---

## 8. Eligibility Rules

**Who is eligible to vote on a proposal:**
- Must be an `active` member at the time the proposal opens.
- Membership status is snapshotted (`eligible_voters` count) at open time.
- Members who go on `leave` after a proposal opens retain their eligibility (they were active when it opened).
- Members who become `active` after a proposal opens are not eligible to vote on it.
- Advisors are never eligible to vote.

**Who is eligible to submit a proposal:**
- Any `active` member (or `admin` only, depending on governance settings).
- Members in `pending` or `leave` status cannot submit.

---

## 9. API Sketch

All endpoints scoped under `/api/v1/organizations/{org_id}/`.

### Proposals

| Method | Path | Who | Description |
|--------|------|-----|-------------|
| `GET` | `proposals` | Member | List proposals (filterable by status, type) |
| `POST` | `proposals` | Member / Admin | Create a new proposal (draft) |
| `GET` | `proposals/{id}` | Member | Get proposal detail |
| `PATCH` | `proposals/{id}` | Author (draft only) | Edit a draft proposal |
| `POST` | `proposals/{id}/submit` | Author | Submit draft → open voting |
| `POST` | `proposals/{id}/close` | Admin | Manually close an open proposal early |
| `DELETE` | `proposals/{id}` | Author (draft only) | Delete a draft |

### Votes

| Method | Path | Who | Description |
|--------|------|-----|-------------|
| `POST` | `proposals/{id}/votes` | Eligible member | Cast a vote |
| `GET` | `proposals/{id}/votes` | Member (post-close) / Admin (post-close) | Get vote records (hidden until closed) |

### Comments

| Method | Path | Who | Description |
|--------|------|-----|-------------|
| `GET` | `proposals/{id}/comments` | Member | List comments |
| `POST` | `proposals/{id}/comments` | Member | Post a comment |
| `DELETE` | `proposals/{id}/comments/{cid}` | Author or Admin | Soft-delete a comment |

### Settings

| Method | Path | Who | Description |
|--------|------|-----|-------------|
| `GET` | `settings/governance` | Admin | Get governance settings |
| `PUT` | `settings/governance` | Admin | Update governance settings |

---

## 10. Notifications

| Event | Who is notified | Channel |
|-------|----------------|---------|
| Proposal submitted (voting opens) | All eligible voters | Email + in-app |
| Proposal opens on scheduled date | All eligible voters | Email + in-app |
| Vote reminder | Members who haven't voted | Email (24h before close) |
| Proposal closed — passed | All eligible voters | Email + in-app |
| Proposal closed — failed | All eligible voters | Email + in-app |
| Proposal closed — failed quorum | All eligible voters + author | Email + in-app |
| New comment on a proposal | Proposal author + members who commented | In-app |
| Bylaw amendment passed (action needed) | Admins | Email + in-app |
| Membership admission passed | Applicant + admins | Email + in-app |

---

## 11. Key UI Pages

| Page | Path | Audience |
|------|------|----------|
| Governance hub (proposal list) | `/governance` | All members |
| Proposal detail & vote | `/governance/{proposal_id}` | All members |
| New proposal form | `/governance/new` | Member / Admin |
| Governance settings | `/admin/settings/governance` | Admin |

---

## 12. Open Questions

1. **Proposal amendments.** If a proposal is open and the author spots an error in the description, can it be edited? The current design says no — once open, immutable. Editing a live proposal is problematic because members may have already read and voted based on the original. Is a "withdraw and resubmit" flow sufficient?

2. **Who-voted-what visibility.** The current design makes individual votes public post-close. Some co-ops prefer secret ballots even after the vote. Should this be a per-co-op setting, or should we standardize on open votes (which better supports accountability)?

3. **Comment moderation.** Only authors and admins can delete comments (soft delete). Should there be a "flag for review" mechanism, or is admin deletion sufficient for v1?

4. **Scheduled proposal open dates.** The current design allows proposals to be set to open on a future date. Is this used in practice often enough to warrant the scheduling complexity, or should v1 require proposals to open immediately on submit?

5. **Linked proposal chains.** A `bylaw_amendment` might logically depend on a `budget_approval` passing first. Should proposals support blocking dependencies, or is this over-engineering for v1? Recommendation: defer; members can manage sequencing by not opening the dependent proposal until the prerequisite passes.

6. **Quorum based on eligible voters vs. total active members.** Currently quorum is calculated against `eligible_voters` (snapshotted at open time). An alternative is to recalculate against current active membership at close time. The snapshot approach is more predictable — is it what co-op bylaws typically specify?

---

## 13. Out of Scope (v1)

- Ranked-choice or multi-option voting (all proposals are binary yes/no/abstain)
- Proxy voting (a member voting on behalf of another)
- Weighted voting by equity share
- Real-time vote tally updates via websocket
- Automated bylaw document amendment (admin must upload revised doc manually after passage)
- Email-based voting (votes must be cast through the platform UI)
- Proposal templates library
