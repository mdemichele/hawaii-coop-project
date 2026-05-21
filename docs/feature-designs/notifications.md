# Notifications — Design Document

**Status:** Draft  
**Scope:** v1  
**Feature area:** Notifications  

---

## 1. Goals

1. **Members always know what needs their attention.** A vote is open, a meeting is tomorrow, their application was approved — the platform surfaces these without requiring members to check in manually.
2. **Two reliable channels.** Email reaches members who haven't logged in. In-app notifications reach members who are active. Both channels carry the same events.
3. **Member-controlled preferences.** Members can silence categories they don't care about without opting out of everything. No one should feel spammed.
4. **No duplicate noise.** A single event fires one notification per channel — not one per feature module. The notification system is the single place where delivery is coordinated.
5. **Delivery visibility.** Admins can see whether critical notifications (membership admission, formation steps) were sent, without exposing member preferences to each other.

---

## 2. Core Concepts

### 2.1 Notification Categories

Events are grouped into categories. Members can toggle each category independently for email and in-app delivery.

| Category | Description |
|----------|-------------|
| `membership` | Application status changes, admission results, member exits |
| `governance` | Proposal opened, vote reminders, proposal results |
| `meetings` | Meeting scheduled, agenda updated, minutes available, action items |
| `documents` | New documents published, membership agreement signature required |
| `formation` | Formation step completed or blocked (relevant during formation only) |
| `account` | Password change, email change, login from new device (always on; cannot be disabled) |

### 2.2 Notification Priority

Each event has a priority that affects delivery behavior:

| Priority | Meaning | Email behavior | In-app behavior |
|----------|---------|----------------|-----------------|
| `critical` | Requires immediate action (e.g., membership agreement to sign, formation blocked) | Always delivered; ignores category preferences | Always shown; persists until dismissed |
| `normal` | Standard event notification | Delivered if category is enabled | Shown; dismissed on read |
| `digest` | Low-urgency updates that can be batched | Eligible for daily digest (if enabled) | Shown inline; no badge increment |

In v1, digest delivery is the preference setting — members can choose per-category whether they want immediate emails or a daily digest email. The in-app notification center always shows all events in real time regardless of digest preference.

### 2.3 Delivery Channels

| Channel | Mechanism | Provider |
|---------|-----------|----------|
| Email | Transactional email via third-party provider | Postmark (recommended) or SendGrid |
| In-app | Persistent notification center; badge count in the nav bar | Platform-native; no third-party |

Push notifications (mobile browser or native app) and SMS are out of scope for v1.

---

## 3. Data Model

### 3.1 `notifications` Table

One row per notification per recipient. The in-app notification center reads directly from this table.

```sql
CREATE TABLE notifications (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id  UUID NOT NULL REFERENCES organizations(id),
  recipient_id     UUID NOT NULL REFERENCES users(id),

  -- Content
  category         TEXT NOT NULL,
  event_type       TEXT NOT NULL,                    -- e.g. 'proposal_opened', 'meeting_scheduled'
  priority         TEXT NOT NULL DEFAULT 'normal'
                   CHECK (priority IN ('critical','normal','digest')),
  title            TEXT NOT NULL,                    -- short headline shown in notification center
  body             TEXT,                             -- one or two sentence detail
  action_url       TEXT,                             -- deep link to the relevant page in the app

  -- Source context (for filtering/grouping; at most one of these is set)
  proposal_id      UUID REFERENCES proposals(id),
  meeting_id       UUID REFERENCES meetings(id),
  member_id        UUID REFERENCES members(id),
  document_id      UUID REFERENCES documents(id),

  -- In-app state
  read_at          TIMESTAMPTZ,                      -- null = unread
  dismissed_at     TIMESTAMPTZ,                      -- null = not dismissed

  -- Email delivery state
  email_sent       BOOLEAN NOT NULL DEFAULT false,
  email_sent_at    TIMESTAMPTZ,
  email_skipped    BOOLEAN NOT NULL DEFAULT false,   -- true if preference said don't send
  email_provider_id TEXT,                            -- message ID from email provider for delivery tracking

  created_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notifications_recipient_unread
  ON notifications (recipient_id, organization_id)
  WHERE read_at IS NULL AND dismissed_at IS NULL;
```

### 3.2 `notification_preferences` Table

One row per member per organization. Stores their per-category, per-channel preferences.

```sql
CREATE TABLE notification_preferences (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id          UUID NOT NULL REFERENCES users(id),
  organization_id  UUID NOT NULL REFERENCES organizations(id),

  -- Per-category email preference: 'immediate', 'digest', 'off'
  email_membership  TEXT NOT NULL DEFAULT 'immediate' CHECK (email_membership  IN ('immediate','digest','off')),
  email_governance  TEXT NOT NULL DEFAULT 'immediate' CHECK (email_governance  IN ('immediate','digest','off')),
  email_meetings    TEXT NOT NULL DEFAULT 'immediate' CHECK (email_meetings    IN ('immediate','digest','off')),
  email_documents   TEXT NOT NULL DEFAULT 'immediate' CHECK (email_documents   IN ('immediate','digest','off')),
  email_formation   TEXT NOT NULL DEFAULT 'immediate' CHECK (email_formation   IN ('immediate','digest','off')),

  -- Per-category in-app preference: 'on' or 'off'
  inapp_membership  BOOLEAN NOT NULL DEFAULT true,
  inapp_governance  BOOLEAN NOT NULL DEFAULT true,
  inapp_meetings    BOOLEAN NOT NULL DEFAULT true,
  inapp_documents   BOOLEAN NOT NULL DEFAULT true,
  inapp_formation   BOOLEAN NOT NULL DEFAULT true,

  -- Digest delivery time (if any category uses 'digest')
  digest_hour      INT NOT NULL DEFAULT 8            -- hour of day in user's local time (0-23)
                   CHECK (digest_hour BETWEEN 0 AND 23),

  updated_at       TIMESTAMPTZ NOT NULL DEFAULT now(),

  UNIQUE (user_id, organization_id)
);
```

`account` category notifications are always delivered and have no preference row — they are hardcoded on.

### 3.3 `email_digest_queue` Table

Holds digest-eligible notifications that have not yet been batched into a digest email.

```sql
CREATE TABLE email_digest_queue (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  notification_id  UUID NOT NULL REFERENCES notifications(id),
  user_id          UUID NOT NULL REFERENCES users(id),
  organization_id  UUID NOT NULL REFERENCES organizations(id),
  scheduled_for    TIMESTAMPTZ NOT NULL,             -- next digest send time for this user
  sent_at          TIMESTAMPTZ                       -- null until digest is dispatched
);
```

---

## 4. Event Catalog

All notification events in the system. Each row defines what triggers it, who receives it, the category, default priority, and which channels carry it.

### 4.1 Membership Events

| Event type | Trigger | Recipients | Category | Priority |
|------------|---------|------------|----------|----------|
| `application_received` | New membership application submitted | Admins | `membership` | `normal` |
| `application_approved` | Admin approves application | Applicant | `membership` | `critical` |
| `application_rejected` | Admin rejects application | Applicant | `membership` | `normal` |
| `member_onboarding_link` | Approved applicant needs to set up account | Applicant | `account` | `critical` |
| `membership_agreement_required` | New membership agreement version published | All active members who haven't signed | `membership` | `critical` |
| `member_admitted` | New member completes onboarding (status → active) | All active members | `membership` | `digest` |
| `member_exited` | Member status changed to exited | Admins | `membership` | `normal` |

### 4.2 Governance Events

| Event type | Trigger | Recipients | Category | Priority |
|------------|---------|------------|----------|----------|
| `proposal_opened` | Proposal submitted and voting begins | All eligible voters | `governance` | `normal` |
| `vote_reminder` | 24 hours before voting closes | Eligible voters who haven't voted | `governance` | `normal` |
| `proposal_passed` | Voting closes; outcome is passed | All eligible voters | `governance` | `normal` |
| `proposal_failed` | Voting closes; outcome is failed | All eligible voters | `governance` | `normal` |
| `proposal_failed_quorum` | Voting closes; quorum not met | All eligible voters + proposal author | `governance` | `normal` |
| `bylaw_amendment_action_needed` | Bylaw amendment proposal passed | Admins | `governance` | `critical` |
| `proposal_comment_posted` | New comment on a proposal | Proposal author + prior commenters | `governance` | `digest` |

### 4.3 Meeting Events

| Event type | Trigger | Recipients | Category | Priority |
|------------|---------|------------|----------|----------|
| `meeting_scheduled` | Admin creates a new meeting | All active members | `meetings` | `normal` |
| `meeting_updated` | Time, location, or agenda changed | All active members | `meetings` | `normal` |
| `meeting_reminder` | 24 hours before meeting time | All active members | `meetings` | `normal` |
| `meeting_cancelled` | Admin cancels a meeting | All active members | `meetings` | `normal` |
| `minutes_available_for_review` | Admin submits draft minutes | All active members | `meetings` | `normal` |
| `minutes_finalized` | Admin finalizes minutes | All active members | `meetings` | `digest` |
| `action_item_assigned` | Action item created and assigned | Assignee | `meetings` | `normal` |
| `action_item_due_tomorrow` | Action item due date is tomorrow | Assignee | `meetings` | `normal` |

### 4.4 Document Events

| Event type | Trigger | Recipients | Category | Priority |
|------------|---------|------------|----------|----------|
| `document_published` | Admin publishes a new governing document | All active members | `documents` | `normal` |
| `document_updated` | Admin publishes a new version of an existing document | All active members | `documents` | `normal` |
| `meeting_minutes_published` | Minutes finalized and placed in Document Library | All active members | `documents` | `digest` |

### 4.5 Formation Events

| Event type | Trigger | Recipients | Category | Priority |
|------------|---------|------------|----------|----------|
| `formation_invite_sent` | Organizer invites a founding member | Invitee | `formation` | `critical` |
| `founding_member_accepted` | Invited founding member accepts | Organizer | `formation` | `normal` |
| `formation_step_completed` | Any formation stage transitions to complete | Organizer | `formation` | `normal` |
| `formation_step_blocked` | Any formation stage transitions to blocked | Organizer | `formation` | `critical` |
| `formation_vote_opened` | Organizer opens the document adoption vote | All founding members | `formation` | `normal` |
| `formation_vote_reminder` | 24 hours before formation vote closes | Founding members who haven't voted | `formation` | `normal` |
| `formation_vote_passed` | Formation vote passes unanimously | All founding members | `formation` | `normal` |
| `formation_vote_failed` | Formation vote fails | Organizer | `formation` | `critical` |
| `formation_complete` | Co-op reaches OPERATIONAL status | All founding members | `formation` | `normal` |

### 4.6 Account Events (Always On)

| Event type | Trigger | Recipients | Category | Priority |
|------------|---------|------------|----------|----------|
| `email_verification` | New account created | New user | `account` | `critical` |
| `password_reset` | Password reset requested | Requesting user | `account` | `critical` |
| `password_changed` | Password successfully changed | User | `account` | `critical` |
| `email_changed` | Email address updated | Old and new email address | `account` | `critical` |
| `role_changed` | Admin changes a member's role | Affected member | `account` | `normal` |

---

## 5. Notification Dispatch Flow

### 5.1 How a Notification is Created

Every feature module emits events — it does not send notifications directly. A central notification service consumes those events and handles delivery. This separation means feature code stays clean and notification logic (preferences, deduplication, templates) lives in one place.

```
Feature module (e.g., governance)
    │
    │ emits event: { type: 'proposal_opened', proposal_id, organization_id }
    ▼
Notification service
    ├── Looks up recipients for this event type
    ├── For each recipient:
    │     ├── Inserts a row into `notifications`
    │     ├── Checks `notification_preferences`
    │     ├── If email: immediate → enqueue email job
    │     │           digest   → insert into `email_digest_queue`
    │     │           off      → set email_skipped = true
    │     └── If in-app disabled: mark notification as pre-dismissed
    └── Done
```

Feature modules call a single internal function:

```
notify(event_type, context_ids, organization_id)
```

They never touch the `notifications` table, email providers, or preference checks directly.

### 5.2 Email Delivery

**Immediate emails** are placed on a job queue (e.g., Sidekiq, BullMQ, or pg_cron depending on stack) and processed within seconds. The job:

1. Renders the email template for the event type.
2. Sends via the email provider API.
3. Records `email_sent = true`, `email_sent_at`, and `email_provider_id` on the `notifications` row.
4. On provider error: retries with exponential backoff up to 3 times. After 3 failures, marks `email_sent = false` and logs the error — does not raise to the user.

**Digest emails** are batched by a scheduled job that runs every hour:

1. Finds all `email_digest_queue` rows where `scheduled_for <= now()` and `sent_at IS NULL`.
2. Groups by `user_id`.
3. For each user: pulls all pending digest notifications, renders a single digest email listing all items, sends it, and marks rows `sent_at = now()`.
4. Schedules the next digest for each user based on their `digest_hour` preference.

**Unsubscribe:** Every email (except account/security emails) includes a one-click unsubscribe link that sets the relevant category preference to `off` without requiring the member to log in. This is required by CAN-SPAM and good practice regardless.

### 5.3 Reminder Jobs

Time-based reminders (vote reminder at 24h before close, meeting reminder at 24h before meeting, action item due tomorrow) are scheduled when the parent record is created or updated.

A scheduled job table tracks these:

```sql
CREATE TABLE scheduled_notification_jobs (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  event_type       TEXT NOT NULL,
  fire_at          TIMESTAMPTZ NOT NULL,
  context          JSONB NOT NULL,                   -- ids needed to generate the notification
  fired_at         TIMESTAMPTZ,                      -- null until executed
  cancelled_at     TIMESTAMPTZ                       -- set if the parent record is cancelled/deleted
);
```

When a meeting is rescheduled, the existing reminder job is cancelled and a new one is inserted for the new time. When a proposal voting period is extended, the old reminder job is cancelled and replaced.

The job processor runs every 5 minutes, finds all unfired jobs with `fire_at <= now()`, and emits the corresponding notification events.

---

## 6. In-App Notification Center

### 6.1 UI Structure

A bell icon in the navigation bar shows an unread badge count. Clicking it opens a slide-out panel (or dropdown).

**Panel layout:**
- "Mark all as read" button at the top.
- Notifications in reverse chronological order.
- Each item shows: category icon, title, body snippet, relative timestamp ("2 hours ago"), and the co-op name (since a user may belong to multiple co-ops in the future).
- Unread items have a distinct background. Read items are muted.
- Clicking an item: marks it read, navigates to `action_url`.
- A "Dismiss" button (×) on each item removes it from the panel without navigating.
- A "See all" link at the bottom opens a full notifications page with filtering by category and date range.

**Badge count:** Sum of `notifications` rows for the current user where `read_at IS NULL` and `dismissed_at IS NULL`.

### 6.2 Full Notifications Page

`/notifications`

- Full list of all notifications for the user, paginated.
- Filter by category (all, membership, governance, meetings, documents, formation).
- Filter by status (unread, read, all).
- "Mark all read" button.
- Each item is a full-width card with the complete title and body.

---

## 7. Notification Preferences UI

`/settings/notifications`

A simple grid: categories as rows, channels (email, in-app) as columns.

```
                   Email              In-App
Membership         [Immediate ▾]      [Toggle]
Governance         [Immediate ▾]      [Toggle]
Meetings           [Immediate ▾]      [Toggle]
Documents          [Digest    ▾]      [Toggle]
Formation          [Immediate ▾]      [Toggle]
Account & Security Always on          Always on
```

Email options per category: Immediate / Daily Digest / Off

If the member selects "Daily Digest" for any category, a time picker appears: "Send digest at [8:00 AM ▾] your local time."

Changes save immediately (no submit button). A confirmation toast appears: "Preferences saved."

---

## 8. Admin Notification Visibility

Admins can see a notification delivery summary for critical system events — not the full preferences of individual members.

From **Settings → Notifications → Delivery Log**, admins see:

- A table of the last 90 days of critical-priority notifications sent by the organization.
- Columns: event type, recipient name, sent at, email delivered (yes / no / skipped).
- No preference details — admins cannot see what categories a member has turned off.

This is primarily useful for confirming that formation invites and membership admission emails were sent.

---

## 9. Email Template Structure

All emails share a consistent layout:

```
[Co-op name + logo placeholder]
─────────────────────────────
[Headline — same as notification title]

[Body — 1-3 sentences of context]

[Primary action button — e.g., "View Proposal", "Cast Your Vote"]

─────────────────────────────
[Co-op name] · Sent via Hawaii Co-op Project
[Unsubscribe from {category} notifications] · [Manage all preferences]
```

In v1, templates are plain HTML/text — no drag-and-drop email builder. Each event type has a corresponding template file. Templates are parameterized with: co-op name, recipient first name, event-specific fields (proposal title, meeting date, etc.), and action URL.

### Email subject line conventions

| Category | Format |
|----------|--------|
| Governance | `[{Co-op name}] {Proposal title} — voting is open` |
| Meetings | `[{Co-op name}] Meeting scheduled: {Meeting title}, {Date}` |
| Membership | `[{Co-op name}] Your membership application has been {approved/reviewed}` |
| Formation | `[{Co-op name}] Formation update: {step name} complete` |
| Account | `Hawaii Co-op Project: {action description}` |

---

## 10. API Sketch

All endpoints scoped under `/api/v1/`.

### Notifications (user-scoped, not org-scoped)

| Method | Path | Who | Description |
|--------|------|-----|-------------|
| `GET` | `notifications` | Authenticated user | List notifications (filterable by org, category, read status) |
| `GET` | `notifications/unread-count` | Authenticated user | Get unread badge count |
| `PATCH` | `notifications/{id}` | Authenticated user | Mark read or dismissed |
| `POST` | `notifications/mark-all-read` | Authenticated user | Mark all notifications read |

### Preferences

| Method | Path | Who | Description |
|--------|------|-----|-------------|
| `GET` | `organizations/{org_id}/notification-preferences` | Member | Get own preferences for this org |
| `PUT` | `organizations/{org_id}/notification-preferences` | Member | Update own preferences |

### Admin Delivery Log

| Method | Path | Who | Description |
|--------|------|-----|-------------|
| `GET` | `organizations/{org_id}/notifications/delivery-log` | Admin | List critical notification delivery records |

### Unsubscribe (no auth required)

| Method | Path | Who | Description |
|--------|------|-----|-------------|
| `GET` | `unsubscribe/{token}` | Anyone with a valid token | One-click unsubscribe from a category |

---

## 11. Key UI Pages

| Page | Path | Audience |
|------|------|----------|
| Notification center (panel) | Accessible from nav bar | All authenticated users |
| Full notification list | `/notifications` | All authenticated users |
| Notification preferences | `/settings/notifications` | All authenticated users |
| Admin delivery log | `/admin/settings/notifications` | Admin |

---

## 12. Open Questions

1. **Multi-org notifications.** In v1 a user belongs to one co-op. If future versions allow multi-co-op membership (advisor role already implies this), the notification center will need to group or filter by organization. Should the data model support this now? The current design includes `organization_id` on each `notifications` row, so the model is ready — the UI just needs a filter.

2. **Email provider choice.** Postmark is recommended for transactional email (strong deliverability, simple API, clear bounce/complaint handling). SendGrid is an alternative with more features but more complexity. This choice has minor but real cost implications. Decide before implementation begins.

3. **Digest email timing.** The current design allows members to choose the hour for digest delivery. Should the digest be daily only, or should we also support weekly digests for low-volume categories like `documents`?

4. **In-app notification retention.** How long should notifications be retained in the panel and full list? Proposed: 90 days rolling window. After 90 days, `digest` and read `normal` notifications are soft-deleted. `critical` notifications are retained indefinitely.

5. **Notification grouping.** If five members accept formation invites within an hour, the organizer gets five separate `founding_member_accepted` notifications. Should these be grouped into one ("5 founding members accepted their invitations")? Grouping is a UX improvement but adds complexity to the rendering logic.

6. **Bounce and complaint handling.** When the email provider reports a hard bounce or spam complaint, the platform should automatically set that category preference to `off` for that member and surface a warning to admins. Is this required at launch, or can it wait for v2?

---

## 13. Out of Scope (v1)

- Push notifications (mobile browser or native)
- SMS notifications
- Slack or other chat integrations
- Webhook delivery for external integrations
- Per-event-type granularity in preferences (category-level only)
- Rich HTML email templates with co-op branding / custom logo
- Notification scheduling by admins (e.g., "send this announcement at 9 AM Monday")
- Read receipts or delivery confirmation visible to senders
- In-app real-time push via WebSocket (notifications appear on next page load or manual refresh)
