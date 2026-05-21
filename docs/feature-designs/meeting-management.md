# Meeting Management — Design Document

**Status:** Draft  
**Scope:** v1  
**Feature area:** Meeting Management  

---

## 1. Goals

1. **Structured meetings.** Every meeting has an agenda built before the call — not improvised on the spot. This is especially important for co-ops that must demonstrate proper governance to external parties (lenders, regulators, advisors).
2. **Proposal integration.** Active governance proposals can be linked directly to agenda items so members arrive knowing what decisions are on the table.
3. **Accurate minutes.** Minutes are recorded in a structured form tied to the agenda, stored in the Document Library, and subject to member review before they are finalized.
4. **Member awareness.** Members are notified of upcoming meetings with enough lead time to prepare, and can access past meeting records at any time.
5. **Low coordination overhead.** Scheduling, notifying, and running a meeting should not require a separate email thread or spreadsheet — the platform handles it end to end.

---

## 2. Core Concepts

### 2.1 Meeting Lifecycle

```
            ┌──────────────┐
            │   SCHEDULED  │  ← created; agenda being built; members notified
            └──────┬───────┘
                   │ meeting time arrives
                   ▼
            ┌──────────────┐
            │   IN_PROGRESS│  ← optional soft state; minutes recording unlocked
            └──────┬───────┘
                   │ admin marks meeting held
                   ▼
            ┌──────────────┐
            │     HELD     │  ← meeting occurred; minutes draft in progress
            └──────┬───────┘
                   │ admin submits minutes for review
                   ▼
            ┌──────────────┐
            │ MINUTES_REVIEW│ ← members can read and comment on draft minutes
            └──────┬───────┘
                   │ admin finalizes
                   ▼
            ┌──────────────┐
            │  FINALIZED   │  ← minutes published to Document Library; record closed
            └──────────────┘
                   │ or cancelled before meeting time
            ┌──────────────┐
            │  CANCELLED   │  ← members notified; no minutes produced
            └──────────────┘
```

| Status | Meaning |
|--------|---------|
| `scheduled` | Meeting is upcoming. Agenda is editable. Members are notified. |
| `in_progress` | Meeting time has passed; admin has opened minutes recording. |
| `held` | Admin has confirmed the meeting occurred. Minutes draft active. |
| `minutes_review` | Draft minutes submitted for member review. No further agenda edits. |
| `finalized` | Minutes approved and published to the Document Library. Record is complete. |
| `cancelled` | Meeting will not occur. Members notified. |

### 2.2 Meeting Types

| Type | Typical cadence | Notes |
|------|----------------|-------|
| `annual` | Once per year | Required by most co-op bylaws (HRS 421-11). Agenda must include officer elections, financial report, and member vote on any annual matters. |
| `special` | As needed | Called for a specific purpose (e.g., bylaw amendment vote). Agenda scope is limited to the stated purpose. |
| `board` | Monthly or quarterly | Board/committee meeting; may not require all members depending on bylaws. |
| `committee` | Varies | Sub-group meeting (e.g., finance committee, land committee). |
| `general` | Ad hoc | Catch-all for regular member meetings. |

### 2.3 Agenda Items

Each agenda item is an ordered entry in the meeting's agenda. Items have a type that determines their behavior:

| Item type | Description |
|-----------|-------------|
| `standard` | Pre-defined procedural item (call to order, quorum check, adjournment). Auto-populated; cannot be removed. |
| `report` | Informational item (treasurer's report, committee update). No vote. |
| `discussion` | Open discussion topic. No formal vote outcome. |
| `vote` | A governance proposal is being voted on in this meeting. Links to a `proposals` row. |
| `action` | An action item assigned to a member. Tracked through to completion. |
| `other` | Catch-all for items that don't fit the above. |

---

## 3. Data Model

### 3.1 `meetings` Table

```sql
CREATE TABLE meetings (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id  UUID NOT NULL REFERENCES organizations(id),

  title            TEXT NOT NULL,                    -- e.g. "2026 Annual General Meeting"
  type             TEXT NOT NULL
                   CHECK (type IN ('annual','special','board','committee','general')),
  status           TEXT NOT NULL DEFAULT 'scheduled'
                   CHECK (status IN ('scheduled','in_progress','held','minutes_review','finalized','cancelled')),

  -- Scheduling
  scheduled_at     TIMESTAMPTZ NOT NULL,             -- date and time of meeting
  duration_minutes INT,                              -- estimated duration
  location         TEXT,                             -- physical address or video link
  location_type    TEXT CHECK (location_type IN ('in_person','virtual','hybrid')),

  -- Optional: video conference link (separate from location text for structured display)
  video_link       TEXT,

  -- Notes to members (shown in invite notification)
  description      TEXT,

  -- Who called the meeting
  created_by       UUID NOT NULL REFERENCES users(id),
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),

  -- Minutes author (set when minutes recording begins)
  minutes_author_id UUID REFERENCES users(id),

  -- Cancellation
  cancelled_at     TIMESTAMPTZ,
  cancellation_reason TEXT,

  -- Finalization
  finalized_at     TIMESTAMPTZ,
  minutes_document_id UUID REFERENCES documents(id)  -- link to published minutes in Document Library
);
```

### 3.2 `agenda_items` Table

```sql
CREATE TABLE agenda_items (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  meeting_id       UUID NOT NULL REFERENCES meetings(id),
  organization_id  UUID NOT NULL REFERENCES organizations(id),

  type             TEXT NOT NULL
                   CHECK (type IN ('standard','report','discussion','vote','action','other')),
  title            TEXT NOT NULL,
  description      TEXT,

  -- Ordering
  position         INT NOT NULL,                     -- 1-based display order within the meeting

  -- Estimated and actual duration
  estimated_minutes INT,
  actual_minutes    INT,                             -- filled in during minutes recording

  -- For 'vote' type items: link to the proposal being decided
  proposal_id      UUID REFERENCES proposals(id),

  -- For 'action' type items: assigned member and due date
  assigned_to      UUID REFERENCES members(id),
  due_date         DATE,

  -- Standard items cannot be removed; other items can
  is_removable     BOOLEAN NOT NULL DEFAULT true,

  created_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 3.3 `meeting_attendance` Table

Records which members were present at the meeting. Used to calculate quorum and attribute minutes.

```sql
CREATE TABLE meeting_attendance (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  meeting_id       UUID NOT NULL REFERENCES meetings(id),
  organization_id  UUID NOT NULL REFERENCES organizations(id),
  member_id        UUID NOT NULL REFERENCES members(id),

  status           TEXT NOT NULL CHECK (status IN ('present','absent','excused')),
  arrived_late     BOOLEAN NOT NULL DEFAULT false,
  left_early       BOOLEAN NOT NULL DEFAULT false,
  proxy_for        UUID REFERENCES members(id),      -- if this member is representing another (v1: informational only)

  recorded_by      UUID NOT NULL REFERENCES users(id),
  recorded_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

  UNIQUE (meeting_id, member_id)
);
```

### 3.4 `meeting_minutes` Table

Stores structured notes tied to each agenda item. One row per agenda item per meeting — the full minutes record is assembled from these rows.

```sql
CREATE TABLE meeting_minutes (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  meeting_id       UUID NOT NULL REFERENCES meetings(id),
  agenda_item_id   UUID NOT NULL REFERENCES agenda_items(id),
  organization_id  UUID NOT NULL REFERENCES organizations(id),

  notes            TEXT,                             -- Markdown; what was discussed or decided
  outcome          TEXT,                             -- for vote items: 'passed', 'failed', 'tabled', 'withdrawn'
  motioned_by      UUID REFERENCES members(id),
  seconded_by      UUID REFERENCES members(id),

  updated_at       TIMESTAMPTZ NOT NULL DEFAULT now(),

  UNIQUE (meeting_id, agenda_item_id)
);
```

### 3.5 `meeting_rsvps` Table

Optional. Tracks member intent to attend before the meeting. Not a binding commitment — attendance is recorded separately after the meeting.

```sql
CREATE TABLE meeting_rsvps (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  meeting_id       UUID NOT NULL REFERENCES meetings(id),
  member_id        UUID NOT NULL REFERENCES members(id),

  response         TEXT NOT NULL CHECK (response IN ('attending','not_attending','maybe')),
  responded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),

  UNIQUE (meeting_id, member_id)
);
```

### 3.6 `meeting_audit_log` Table

Append-only. Records status transitions and key actions.

```sql
CREATE TABLE meeting_audit_log (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id  UUID NOT NULL REFERENCES organizations(id),
  meeting_id       UUID NOT NULL REFERENCES meetings(id),
  actor_id         UUID REFERENCES users(id),

  event_type       TEXT NOT NULL,                    -- 'meeting_created', 'agenda_item_added', 'minutes_submitted', 'finalized', etc.
  details          JSONB,
  occurred_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## 4. Meeting Flows

### 4.1 Scheduling a Meeting (Admin)

1. Admin clicks **+ New Meeting** in the Meetings section.
2. Fills in:
   - Title (required)
   - Type (required)
   - Date and time (required)
   - Estimated duration (optional)
   - Location type: in-person, virtual, or hybrid
   - Location text (address, room, or video link)
   - Description / notes for members (optional)
3. Clicks **Create Meeting**. Platform:
   - Creates the `meetings` row with status `scheduled`.
   - Inserts the three non-removable standard agenda items: *Call to Order*, *Quorum Check*, *Adjournment*, in positions 1, 2, and (last).
   - Sends a meeting notification to all active members (email + in-app).
   - Appends to `meeting_audit_log`.

### 4.2 Building the Agenda (Admin)

After creation, the admin is taken directly to the Agenda Builder.

**Adding items:**
- Admin clicks **+ Add Item** between any two existing items.
- Selects the item type, fills in title and description, sets estimated duration, and (for `vote` items) searches for and links an open proposal.
- Item is inserted at the chosen position. All subsequent positions shift down.

**Reordering items:**
- Items are reordered via drag-and-drop (or up/down arrow buttons for accessibility).
- Standard items (*Call to Order*, *Quorum Check*, *Adjournment*) are locked in their positions and cannot be moved or removed.

**Editing items:**
- Any non-standard item can be edited or removed up until the meeting is marked `in_progress`.
- After the meeting begins, the agenda is locked — changes require the admin to add a note in the minutes instead.

**Previewing:**
- Admin can preview the agenda as a formatted document (suitable for sharing outside the platform if needed).

### 4.3 Running the Meeting

When the scheduled time arrives, the admin clicks **Start Meeting** (or it can be done retroactively). This transitions status to `in_progress` and unlocks minutes recording.

**During the meeting:**
- The admin (or designated secretary) works through agenda items in order.
- For each item, they fill in the minutes notes field in real time.
- For `vote` items, they record outcome (`passed`, `failed`, `tabled`, `withdrawn`), motioned-by, and seconded-by.
- Attendance is recorded: the admin marks each member as present, absent, or excused. The platform calculates quorum from this list and displays it (e.g., "8 of 12 active members present — quorum met at 60%").

**After the meeting:**
- Admin clicks **Mark as Held**. Status → `held`.
- Any remaining minutes fields can be filled in after the fact (common — notes are usually rough during the meeting and cleaned up afterward).

### 4.4 Submitting Minutes for Review

1. Admin reviews all minutes entries, fills in any gaps, and clicks **Submit for Review**.
2. Status → `minutes_review`.
3. All active members are notified that draft minutes are available.
4. Members can view the draft and leave comments (a lightweight comment thread, not a formal vote).
5. A 7-day review period is suggested (not enforced — the admin finalizes at their discretion).

### 4.5 Finalizing Minutes

1. Admin addresses any member comments and clicks **Finalize Minutes**.
2. Platform generates a formatted PDF of the full minutes document (meeting title, date, attendees, agenda with notes, vote outcomes, action items).
3. The PDF is stored in the Document Library as a new `documents` row in a new or existing `minutes` chain. The chain display name is set to the meeting title.
4. `meetings.minutes_document_id` is updated to point to this document.
5. Status → `finalized`.
6. Members are notified that finalized minutes are available in the Document Library.
7. Any linked `vote` agenda items with outcome `passed` trigger the same side effects as a passed governance proposal (membership admission, bylaw amendment prompt, etc.).

### 4.6 Cancelling a Meeting

Admin clicks **Cancel Meeting**, enters a cancellation reason, and confirms. Status → `cancelled`. All members are notified immediately. Cancelled meetings remain visible in the meeting history with the cancellation reason shown.

---

## 5. Agenda Builder UI

The agenda builder is the central UI of this feature. It should feel structured but not bureaucratic.

**Layout:**
- Left panel: ordered list of agenda items, each showing position number, type badge, title, and estimated duration.
- Right panel: detail editor for the selected item (title, description, duration, linked proposal if applicable).
- Drag handle on each non-standard item for reordering.
- "+ Add Item" button appears between items on hover.
- Estimated total duration shown at the bottom of the list (sum of all item durations).
- "Preview Agenda" button generates a clean printable view.

**Agenda item type badges use distinct colors:**
- Standard — grey
- Report — blue
- Discussion — teal
- Vote — orange (with proposal title shown inline)
- Action — purple
- Other — grey outline

---

## 6. Minutes Recording UI

The minutes view mirrors the agenda structure exactly. Each agenda item expands into an editable notes section.

**Per-item panel:**
- Item title and type (read-only)
- Notes field (Markdown editor)
- For `vote` items: outcome dropdown (passed / failed / tabled / withdrawn), motioned-by member picker, seconded-by member picker
- For `action` items: assigned-to member picker, due date field
- Actual duration field

**Attendance panel (sidebar):**
- Full member list with present / absent / excused toggle per member
- Live quorum calculation displayed as members are marked: "6 / 10 present — quorum met (≥60%)"

**Draft state:**
- Minutes auto-save every 30 seconds while the admin is editing.
- A "Last saved" timestamp is shown at the top of the page.
- The admin can close and return — draft is preserved.

---

## 7. Past Meetings & Meeting History

### 7.1 Meetings List

The Meetings section shows two tabs:

| Tab | Contents |
|-----|----------|
| **Upcoming** | All `scheduled` meetings sorted by date ascending. Shows title, type, date/time, location, and RSVP button. |
| **Past** | All `held`, `finalized`, and `cancelled` meetings sorted by date descending. Shows title, type, date, status badge, and link to minutes if finalized. |

### 7.2 Meeting Detail (Past)

- Meeting metadata: title, type, date, location, duration (actual if recorded).
- Attendance summary: who was present, absent, excused. Quorum result.
- Agenda with notes (read-only once finalized).
- Link to finalized minutes PDF in the Document Library.
- Linked proposals and their outcomes.
- Action items from this meeting with current completion status.

---

## 8. Action Item Tracking

Action items assigned during a meeting are tracked across the platform so they don't fall through the cracks.

- Action items appear on the assignee's dashboard under "My Action Items."
- Admins see all open action items from all meetings in an "Action Items" view under the Meetings section.
- Assignees can mark their action items complete. Completion is noted in the meeting record.
- Action items do not expire automatically — they stay open until marked complete or explicitly dismissed by an admin.

---

## 9. RSVP Flow (Optional)

Members can RSVP to a meeting from the notification or the meeting detail page.

- Options: Attending / Not Attending / Maybe
- RSVPs are visible to admins (to estimate attendance) but not to other members.
- RSVPs are not binding — attendance is recorded separately after the fact.
- An admin can see the RSVP summary on the meeting detail: "7 attending, 2 not attending, 1 maybe, 2 no response."

---

## 10. Quorum Calculation

Quorum for meetings is calculated separately from governance proposal quorum, since meetings use attendance (bodies in the room) rather than votes cast.

The platform calculates meeting quorum using the co-op's configured meeting quorum threshold (default: 50% of active members) applied to the attendance record.

```
quorum_met = (present_count / active_member_count_at_meeting_time) >= meeting_quorum_pct
```

Members marked `excused` do not count as present for quorum purposes. The platform displays the quorum calculation clearly in the attendance panel and on the finalized minutes.

A separate `meeting_quorum_pct` setting is added to `governance_settings` (default 50%) — meeting quorum and proposal vote quorum are often different in co-op bylaws.

---

## 11. Formation Meeting Integration

The First Meeting step of the Formation Wizard creates a meeting record automatically using the platform's meeting system.

- Meeting type is `annual` (formation meeting qualifies as the first annual meeting under most bylaw templates).
- A pre-built agenda is generated with standard formation items:
  1. Call to Order
  2. Quorum Check
  3. Election of Officers
  4. Ratification of Formation Documents
  5. Authorization of Bank Signatories
  6. Selection of Fiscal Year
  7. Any Other Business
  8. Adjournment
- The organizer can edit the agenda before the meeting.
- Minutes are recorded and finalized through the same flow as any other meeting.
- Finalized minutes are stored in the Document Library and the formation wizard step is marked `complete`.

---

## 12. API Sketch

All endpoints scoped under `/api/v1/organizations/{org_id}/`.

### Meetings

| Method | Path | Who | Description |
|--------|------|-----|-------------|
| `GET` | `meetings` | Member | List meetings (filterable by status) |
| `POST` | `meetings` | Admin | Create a new meeting |
| `GET` | `meetings/{id}` | Member | Get meeting detail |
| `PATCH` | `meetings/{id}` | Admin | Update meeting metadata (title, time, location) |
| `POST` | `meetings/{id}/cancel` | Admin | Cancel a meeting |
| `POST` | `meetings/{id}/start` | Admin | Mark meeting in progress |
| `POST` | `meetings/{id}/hold` | Admin | Mark meeting as held |
| `POST` | `meetings/{id}/submit-minutes` | Admin | Submit draft minutes for review |
| `POST` | `meetings/{id}/finalize` | Admin | Finalize minutes and publish to library |

### Agenda Items

| Method | Path | Who | Description |
|--------|------|-----|-------------|
| `GET` | `meetings/{id}/agenda` | Member | Get full agenda |
| `POST` | `meetings/{id}/agenda` | Admin | Add an agenda item |
| `PATCH` | `meetings/{id}/agenda/{item_id}` | Admin | Edit an agenda item |
| `DELETE` | `meetings/{id}/agenda/{item_id}` | Admin | Remove a non-standard agenda item |
| `PUT` | `meetings/{id}/agenda/order` | Admin | Reorder all agenda items (array of IDs in new order) |

### Minutes

| Method | Path | Who | Description |
|--------|------|-----|-------------|
| `GET` | `meetings/{id}/minutes` | Member | Get minutes (draft visible to members in review stage) |
| `PATCH` | `meetings/{id}/minutes/{item_id}` | Admin | Update minutes for a specific agenda item |

### Attendance

| Method | Path | Who | Description |
|--------|------|-----|-------------|
| `GET` | `meetings/{id}/attendance` | Admin | Get attendance record |
| `PUT` | `meetings/{id}/attendance` | Admin | Set attendance for all members (bulk upsert) |
| `PATCH` | `meetings/{id}/attendance/{member_id}` | Admin | Update attendance for one member |

### RSVPs

| Method | Path | Who | Description |
|--------|------|-----|-------------|
| `POST` | `meetings/{id}/rsvp` | Member | Submit or update an RSVP |
| `GET` | `meetings/{id}/rsvps` | Admin | Get all RSVPs for a meeting |

### Action Items

| Method | Path | Who | Description |
|--------|------|-----|-------------|
| `GET` | `action-items` | Admin | List all open action items across meetings |
| `GET` | `meetings/{id}/action-items` | Member | List action items from a specific meeting |
| `PATCH` | `meetings/{id}/action-items/{item_id}` | Admin / Assignee | Mark complete or update due date |

---

## 13. Notifications

| Event | Who is notified | Channel |
|-------|----------------|---------|
| Meeting scheduled | All active members | Email + in-app |
| Meeting details updated (time/location change) | All active members | Email + in-app |
| Meeting reminder | All active members | Email (24h before meeting) |
| Meeting cancelled | All active members | Email + in-app |
| Draft minutes available for review | All active members | Email + in-app |
| Minutes finalized | All active members | In-app |
| Action item assigned | Assignee | Email + in-app |
| Action item due tomorrow | Assignee | Email |

---

## 14. Key UI Pages

| Page | Path | Audience |
|------|------|----------|
| Meetings hub | `/meetings` | All members |
| Meeting detail (upcoming) | `/meetings/{id}` | All members |
| Agenda builder | `/meetings/{id}/agenda` | Admin |
| Minutes recording | `/meetings/{id}/minutes` | Admin |
| Meeting detail (past, finalized) | `/meetings/{id}` | All members |
| Action items dashboard | `/meetings/action-items` | Admin |

---

## 15. Open Questions

1. **Secretary role.** The current design lets any admin record minutes. Some co-ops have a designated secretary role. Should we add a `secretary` role that can record minutes but doesn't have other admin permissions, or is the `admin` role sufficient for v1?

2. **Meeting quorum threshold.** The design proposes a `meeting_quorum_pct` separate from proposal quorum. Should this be configurable per meeting type (annual vs. special) or a single flat setting?

3. **Draft minutes visibility.** During `minutes_review`, draft minutes are visible to all members. Some co-ops only share draft minutes with officers before wider distribution. Should there be a "board-only preview" stage before full member review?

4. **Agenda PDF format.** The agenda preview and minutes PDF are generated by the platform. Should we support custom formatting (co-op name, logo, letterhead), or is a clean platform-branded format sufficient for v1?

5. **Recurring meetings.** Annual and board meetings often recur on a predictable schedule. Should we support a recurring meeting template (create once, auto-generate for each cycle), or is manual creation each time acceptable for v1?

6. **Online voting during meetings.** If a meeting is virtual, members might want to cast governance votes through the platform in real time during the meeting, rather than through the asynchronous proposal system. Is this a meaningful use case for this co-op context, or is async always sufficient?

---

## 16. Out of Scope (v1)

- Video conferencing integration (Zoom, Google Meet links are entered as plain text)
- Automated quorum check during a live meeting (recorded manually)
- In-platform live voting during a meeting session
- Meeting recording storage
- Recurring meeting templates
- Calendar sync (Google Calendar, iCal export)
- Multi-language minutes (English only)
- Parliamentary procedure enforcement (Robert's Rules of Order)
