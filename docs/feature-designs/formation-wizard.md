# Co-op Formation Wizard — Design Document

**Status:** Draft  
**Scope:** v1  
**Feature area:** Co-op Formation  

---

## 1. Goals

1. **Guided from zero.** A group with no legal or business background can start the formation process on day one without needing to research Hawaii statutes first.
2. **Step-by-step progress.** The wizard makes it obvious what's done, what's next, and what's blocking progress. No one gets lost.
3. **Document generation.** The platform produces ready-to-review Hawaii-specific legal documents automatically from information the group has already entered.
4. **Multi-member coordination.** Formation requires agreement from all founding members — the platform handles invites, signatures, and votes rather than leaving that to email chains.
5. **Advisor-compatible.** A lawyer or USDA agent can be invited at any point to review progress and leave notes without disrupting the flow.

---

## 2. Core Concepts

### 2.1 Formation Lifecycle

```
NOT_STARTED
    │
    ▼
ORGANIZING          ← Co-op type chosen; founding members recruited
    │
    ▼
LEGAL_STRUCTURE     ← HRS chapter confirmed; plain-language summary shown
    │
    ▼
NAME_AND_PURPOSE    ← Co-op name and mission statement entered
    │
    ▼
DOCUMENTS_DRAFTED   ← Articles and bylaws generated from templates
    │
    ▼
FORMATION_VOTE      ← Founding members vote to adopt governing documents
    │
    ▼
DCCA_FILING         ← Filed with Hawaii DCCA (manual; platform tracks status)
    │
    ▼
EIN_OBTAINED        ← IRS EIN acquired
    │
    ▼
BANK_ACCOUNT        ← Operating bank account opened
    │
    ▼
FIRST_MEETING       ← Formation meeting held; minutes recorded
    │
    ▼
OPERATIONAL         ← Co-op is live
```

Each stage is a discrete state. The organizer cannot skip forward — a stage must be explicitly marked complete before the next unlocks. Any stage can be in one of four sub-states: `not_started`, `in_progress`, `complete`, or `blocked`. Blocked stages display a plain-language explanation of what is needed to unblock them.

### 2.2 Who Does What

| Role | Formation responsibilities |
|------|---------------------------|
| **Organizer** | Drives the wizard. Enters co-op details, generates documents, manages the formation vote, and marks filing milestones complete. |
| **Founding Member** | Accepts their invite, reviews governing documents, casts a formation vote. |
| **Advisor** | Read-only access to all stages. Can leave notes on any step. Cannot vote or mark steps complete. |

The organizer is the first user; they become the initial `admin` once the co-op reaches `OPERATIONAL`.

---

## 3. Data Model

### 3.1 `organizations` Table

```sql
CREATE TABLE organizations (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  slug                TEXT UNIQUE NOT NULL,             -- URL-friendly identifier
  name                TEXT,                             -- set in NAME_AND_PURPOSE step
  coop_type           TEXT CHECK (coop_type IN ('housing', 'agricultural')),
  island              TEXT,                             -- oahu, maui, hawaii, kauai, molokai
  hrs_chapter         TEXT,                             -- '421', '421H', or both
  mission_statement   TEXT,
  ein                 TEXT,                             -- set in EIN_OBTAINED step
  formation_status    TEXT NOT NULL DEFAULT 'NOT_STARTED',
  created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  created_by          UUID NOT NULL REFERENCES users(id)
);
```

### 3.2 `formation_workflows` Table

One row per organization. Tracks stage-level state, timestamps, and metadata for each step.

```sql
CREATE TABLE formation_workflows (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id  UUID NOT NULL UNIQUE REFERENCES organizations(id),

  -- Stage statuses (one column per stage for simplicity at v1 scale)
  organizing_status        TEXT NOT NULL DEFAULT 'not_started',
  legal_structure_status   TEXT NOT NULL DEFAULT 'not_started',
  name_purpose_status      TEXT NOT NULL DEFAULT 'not_started',
  documents_status         TEXT NOT NULL DEFAULT 'not_started',
  formation_vote_status    TEXT NOT NULL DEFAULT 'not_started',
  dcca_filing_status       TEXT NOT NULL DEFAULT 'not_started',
  ein_status               TEXT NOT NULL DEFAULT 'not_started',
  bank_account_status      TEXT NOT NULL DEFAULT 'not_started',
  first_meeting_status     TEXT NOT NULL DEFAULT 'not_started',

  -- Stage completion timestamps
  organizing_completed_at      TIMESTAMPTZ,
  legal_structure_completed_at TIMESTAMPTZ,
  name_purpose_completed_at    TIMESTAMPTZ,
  documents_completed_at       TIMESTAMPTZ,
  formation_vote_completed_at  TIMESTAMPTZ,
  dcca_filing_completed_at     TIMESTAMPTZ,
  ein_completed_at             TIMESTAMPTZ,
  bank_account_completed_at    TIMESTAMPTZ,
  first_meeting_completed_at   TIMESTAMPTZ,

  -- Advisor notes (keyed by stage name)
  advisor_notes    JSONB NOT NULL DEFAULT '{}',

  updated_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Status values per stage: `not_started` | `in_progress` | `complete` | `blocked`.

### 3.3 `founding_member_invites` Table

Tracks each founding member invitation and their acceptance status.

```sql
CREATE TABLE founding_member_invites (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id  UUID NOT NULL REFERENCES organizations(id),
  invited_by       UUID NOT NULL REFERENCES users(id),

  email            TEXT NOT NULL,
  name             TEXT,
  token            TEXT NOT NULL UNIQUE,               -- time-limited invite token
  expires_at       TIMESTAMPTZ NOT NULL,

  status           TEXT NOT NULL DEFAULT 'pending'
                   CHECK (status IN ('pending', 'accepted', 'declined', 'expired')),
  accepted_at      TIMESTAMPTZ,
  user_id          UUID REFERENCES users(id),          -- set on acceptance

  sent_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 3.4 `formation_votes` Table

A specialized vote table for the formation document adoption vote — distinct from the general `votes` table used for ongoing governance proposals.

```sql
CREATE TABLE formation_votes (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id  UUID NOT NULL REFERENCES organizations(id),
  member_id        UUID NOT NULL REFERENCES members(id),

  choice           TEXT NOT NULL CHECK (choice IN ('yes', 'no', 'abstain')),
  voted_at         TIMESTAMPTZ NOT NULL DEFAULT now(),

  UNIQUE (organization_id, member_id)               -- one vote per founding member
);
```

### 3.5 `formation_audit_log` Table

Append-only record of every action taken during formation.

```sql
CREATE TABLE formation_audit_log (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id  UUID NOT NULL REFERENCES organizations(id),
  actor_id         UUID REFERENCES users(id),

  event_type       TEXT NOT NULL,                      -- 'stage_advanced', 'document_generated', 'invite_sent', 'vote_cast', etc.
  stage            TEXT,
  details          JSONB,
  occurred_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## 4. Formation Steps — Detail

### Step 1: Intent & Type (`ORGANIZING`)

**What happens here:**  
The organizer establishes the fundamental identity of the co-op.

**Fields collected:**
- Co-op type: `Housing` or `Agricultural`
- Island: Oahu, Maui, Hawaii Island, Kauai, Molokai, or Other
- Founding member count confirmation: the organizer confirms there are 3+ people ready to form (HRS minimum)

**Platform behavior:**  
After submission, the platform explains in plain language what type of co-op the organizer has indicated they want to form and what that means legally. No legalese — one short paragraph, written at a 10th grade reading level.

**Completion condition:** Type, island, and 3+ confirmation all set.

---

### Step 2: Legal Structure (`LEGAL_STRUCTURE`)

**What happens here:**  
The platform recommends the applicable HRS chapter and explains it.

**Logic:**
- Housing co-op → HRS Chapter 421H
- Agricultural co-op → HRS Chapter 421
- If the organizer's description doesn't cleanly fit either, the platform flags this and recommends they consult an advisor before proceeding

**Displayed to organizer:**
- Name of the statute
- What it means for their co-op (liability, governance requirements, member equity rights)
- A link to the full statute text on the Hawaii Legislature website
- A callout: *"Not sure this is the right structure? Invite an advisor to review before moving on."*

**Completion condition:** Organizer clicks "This looks right, continue." No data entry required — this step is informational with an acknowledgement gate.

---

### Step 3: Name & Purpose (`NAME_AND_PURPOSE`)

**What happens here:**  
The co-op gets its official name and mission statement.

**Fields collected:**
- Proposed co-op name (required)
- Mission statement (required; 50–500 characters)

**Platform behavior:**
- Displays a link to the Hawaii DCCA Business Name Search so the organizer can check availability before committing. The platform does not automate this check in v1 — it links out.
- Validates that the name ends in "Cooperative," "Co-op," or a legally recognized variant per HRS (enforced as a soft warning, not a hard block — the organizer may know something the platform doesn't).

**Completion condition:** Name and mission statement saved.

---

### Step 4: Founding Members (`ORGANIZING` sub-task, runs in parallel)

**What happens here:**  
The organizer invites the other founding members. This is technically part of the `ORGANIZING` stage but can proceed in parallel with steps 2 and 3.

**Flow:**
1. Organizer enters email addresses (one at a time or comma-separated bulk entry).
2. Platform sends each invitee a personalized email with a 7-day invite link.
3. Invitees click the link, create an account (or sign in), and land on the co-op's formation page showing current progress.
4. The organizer's dashboard shows each invite with status: `pending`, `accepted`, `declined`, `expired`.
5. Admins can resend expired invites or revoke pending ones.

**Completion condition:** At least 2 additional members have accepted (organizer + 2 = 3 minimum required by HRS). The platform allows continuing with fewer in `in_progress` state but marks the stage `blocked` at the document adoption vote if the minimum isn't met.

---

### Step 5: Governing Documents (`DOCUMENTS_DRAFTED`)

**What happens here:**  
The platform generates draft Articles of Incorporation and Bylaws from Hawaii-specific templates, pre-filled with the co-op's details.

**Generated documents:**
| Document | Template basis |
|----------|----------------|
| Articles of Incorporation | HRS 421 or 421H template |
| Model Bylaws | Housing or Agricultural variant |
| Membership Application & Agreement | Standard Hawaii co-op form |

**Document generation flow:**
1. Platform substitutes co-op name, island, co-op type, founding member names, and mission statement into templates.
2. Generated PDFs are stored in the Document Library.
3. Organizer can download each document for offline review.
4. Organizer can request edits — in v1, this opens the document in a simple text editor (Markdown-to-PDF) for manual revisions. Full legal authoring is out of scope.

**Important disclaimer** shown prominently on this step:  
> *"These documents are starting points based on Hawaii law. Have an attorney licensed in Hawaii review them before filing or distributing to members."*

**Completion condition:** Organizer clicks "Documents are ready for member review." This locks the documents for the formation vote — no edits allowed after this point without restarting the vote.

---

### Step 6: Formation Vote (`FORMATION_VOTE`)

**What happens here:**  
All founding members vote to adopt the governing documents. Hawaii co-op law requires unanimous consent for initial formation.

**Vote flow:**
1. Organizer opens the vote. All active founding members are notified by email and in-app.
2. Each member reviews the documents (links provided in the vote UI) and casts `yes`, `no`, or `abstain`.
3. Votes are recorded immediately but the tally is hidden until all members have voted or the deadline passes.
4. Deadline: organizer sets a close date (minimum 48 hours; default 7 days).

**Outcome rules:**
- **All yes:** Vote passes. Stage marked `complete`. Formation advances.
- **Any no:** Vote fails. Organizer is notified. Documents can be revised and a new vote initiated (previous vote is archived in the audit log).
- **Abstain counts as a non-yes** for the purposes of unanimous consent — i.e., any abstain also fails the vote.
- **Deadline reached with missing votes:** Vote fails. Organizer is notified with a list of who has not voted.

**Completion condition:** All founding members vote `yes` before the deadline.

---

### Step 7: DCCA Filing (`DCCA_FILING`)

**What happens here:**  
The co-op files its Articles of Incorporation with the Hawaii DCCA. The platform does not automate this filing — it provides step-by-step instructions and tracks status via manual confirmation.

**What the platform provides:**
- A checklist of required DCCA filing materials (what to bring, what fees to expect)
- Direct link to the DCCA online filing portal
- A filing status field the organizer updates manually: `not_filed` → `filed_pending` → `approved` → `rejected`
- If `rejected`: a notes field for the rejection reason, and a prompt to consult an advisor

**Completion condition:** Organizer marks filing status as `approved` and uploads the DCCA confirmation (PDF or image). Upload stored in the Document Library tagged as `dcca_confirmation`.

---

### Step 8: EIN (`EIN_OBTAINED`)

**What happens here:**  
The co-op obtains an Employer Identification Number from the IRS.

**What the platform provides:**
- Link to the IRS online EIN application
- Explanation of why an EIN is needed (bank account, taxes, payroll if applicable)
- A field to enter the EIN once obtained

**Completion condition:** Organizer enters the EIN (validated as 9-digit format `XX-XXXXXXX`) and saves. The EIN is stored on the `organizations` row.

---

### Step 9: Bank Account (`BANK_ACCOUNT`)

**What happens here:**  
The co-op opens an operating bank account. The platform cannot do this for them, but provides guidance.

**What the platform provides:**
- A checklist of what most banks require (EIN, articles of incorporation, bylaws, resolution authorizing signers)
- A pre-filled "Banking Resolution" template the co-op can bring to the bank (identifies authorized signers)
- A field to record the bank name and account opening date

**Completion condition:** Organizer enters bank name and account open date and clicks "Bank account opened."

---

### Step 10: First Meeting (`FIRST_MEETING`)

**What happens here:**  
The co-op holds its inaugural meeting. This is both a legal and cultural milestone — the organization officially becomes operational.

**What the platform provides:**
- A pre-filled formation meeting agenda (generated document):
  - Call to order
  - Quorum check
  - Election of officers
  - Adoption of any initial resolutions (bank signers, fiscal year)
  - Adjournment
- A minutes recording form the secretary fills out during or after the meeting
- On completion, the meeting minutes are stored in the Document Library tagged as `minutes`

**Completion condition:** Organizer marks the meeting as held and submits minutes. The platform transitions `organizations.formation_status` to `OPERATIONAL`.

---

## 5. Progress Dashboard

The main formation UI is a progress tracker visible to all founding members and advisors.

**Layout:**
- A vertical stepper on the left showing all 9 stages with icons: complete (checkmark), in progress (spinner), blocked (warning), not started (circle).
- Clicking any stage navigates to its detail panel on the right.
- At the top: co-op name, type, island, and a "Days since formation started" counter.

**Blocking states:**  
When a stage is `blocked`, the detail panel explains specifically what is needed:
- "Waiting for 2 more founding members to accept their invitations. (3 accepted, 0 pending)"
- "The formation vote did not pass. Marcus voted No. Revise the governing documents and restart the vote."

**Advisor note panel:**  
Each stage has an expandable "Advisor Notes" section. Notes are timestamped and attributed. Advisors can add notes; members can read them but not reply in-line (reply by messaging the advisor directly).

---

## 6. Invite Link & Access Model

| Who | How they access the formation wizard |
|-----|--------------------------------------|
| Organizer | Creates the co-op; gets full admin access immediately |
| Founding Member | Receives an email invite link; gains access on acceptance |
| Advisor | Organizer grants access via Settings → Advisors; advisor gets read-only view of all stages |

Invite tokens expire after 7 days. Expired tokens show a clear error with an instruction to contact the organizer for a new link.

---

## 7. API Sketch

All endpoints scoped under `/api/v1/organizations/{org_id}/formation/`.

### Workflow State

| Method | Path | Who | Description |
|--------|------|-----|-------------|
| `GET` | `` (root) | Organizer / Member / Advisor | Get full formation workflow state |
| `PATCH` | `stages/{stage}` | Organizer | Advance or update a stage's status |

### Founding Members

| Method | Path | Who | Description |
|--------|------|-----|-------------|
| `POST` | `invites` | Organizer | Send founding member invite(s) |
| `GET` | `invites` | Organizer | List all invites and their status |
| `DELETE` | `invites/{id}` | Organizer | Revoke a pending invite |
| `POST` | `invites/{token}/accept` | Invitee | Accept an invite (creates member row) |

### Documents

| Method | Path | Who | Description |
|--------|------|-----|-------------|
| `POST` | `documents/generate` | Organizer | Trigger document generation |
| `GET` | `documents` | Organizer / Member / Advisor | List generated formation documents |
| `PATCH` | `documents/{id}` | Organizer | Update document content (before vote opens) |

### Formation Vote

| Method | Path | Who | Description |
|--------|------|-----|-------------|
| `POST` | `vote/open` | Organizer | Open the formation vote |
| `GET` | `vote` | Organizer / Member | Get vote status and (post-close) tally |
| `POST` | `vote/cast` | Founding Member | Cast a vote |

### Filing Milestones

| Method | Path | Who | Description |
|--------|------|-----|-------------|
| `PATCH` | `stages/dcca_filing` | Organizer | Update DCCA filing status |
| `POST` | `stages/dcca_filing/upload` | Organizer | Upload DCCA confirmation document |
| `PATCH` | `stages/ein` | Organizer | Record EIN |
| `PATCH` | `stages/bank_account` | Organizer | Record bank account info |

### Advisor Notes

| Method | Path | Who | Description |
|--------|------|-----|-------------|
| `POST` | `stages/{stage}/notes` | Advisor | Add a note to a stage |
| `GET` | `stages/{stage}/notes` | Organizer / Advisor | List notes for a stage |

---

## 8. Key UI Pages

| Page | Path | Audience |
|------|------|----------|
| Formation dashboard | `/formation` | Organizer, Founding Members, Advisors |
| Stage detail (any step) | `/formation/{stage}` | Organizer, Founding Members, Advisors |
| Invite acceptance landing | `/invite/formation/{token}` | Invitee |
| Document viewer/editor | `/formation/documents/{id}` | Organizer |
| Formation vote | `/formation/vote` | All founding members |

---

## 9. Notifications

| Event | Who is notified | Channel |
|-------|----------------|---------|
| Founding member invite sent | Invitee | Email |
| Founding member accepts invite | Organizer | Email + in-app |
| Documents ready for review | All founding members | Email + in-app |
| Formation vote opened | All founding members | Email + in-app |
| Formation vote reminder | Members who haven't voted | Email (24h before close) |
| Formation vote passed | All founding members | Email + in-app |
| Formation vote failed | Organizer | Email + in-app |
| DCCA filing approved | Organizer | (manual — no auto-notify) |
| Formation complete (OPERATIONAL) | All members | Email + in-app |

---

## 10. Open Questions

1. **Document editing in v1.** The current design proposes a simple Markdown/text editor for document revisions. Is this sufficient, or do we need inline PDF annotation? A proper document editor is significantly more work.

2. **Unanimous vote requirement.** HRS 421H requires unanimous consent for initial formation. Should the platform hard-code this, or make it configurable (some organizers may want to test with a lower threshold during drafting)? Recommendation: hard-code unanimous for the formation vote; the ongoing governance proposals system handles configurable thresholds.

3. **Re-vote after failure.** When the formation vote fails, the current design lets the organizer revise documents and open a new vote. Should there be a limit on how many re-votes are allowed, or any cooling-off period?

4. **Co-op type: both?** Can a co-op be both housing and agricultural (e.g., a community land trust with farming)? The current model picks one. If we need to support both, the HRS chapter selection and template logic gets more complex.

5. **DCCA filing automation.** The Hawaii DCCA has an online portal. Is a future integration with their API worth scoping? v1 is manual-only, but the data model should not make this hard to add later.

6. **Organizer vs. admin after formation.** The design transitions the organizer to `admin` on OPERATIONAL. Should other founding members also be offered the option to become admins at that point, or does the organizer set roles manually?

---

## 11. Out of Scope (v1)

- Automated DCCA filing or status polling via DCCA API
- Legal document authoring (full rich-text editor, clause library, redlining)
- Multi-type co-ops (housing + agricultural)
- Formation wizard for co-op types other than housing and agricultural
- EIN application automation via IRS
- Integration with Hawaii DHHL for Native Hawaiian land trust formation
- Formation milestone reminders / nudge emails beyond the vote deadline reminder
