# Hawaii Co-op Project — Feature Specification (v1)

## 1. Co-op Formation Wizard

The core differentiator. A step-by-step guided flow that takes a group from "we want to form a co-op" to a legally registered organization with governing documents in hand.

### Steps

| Step | Name | Description |
|---|---|---|
| 1 | **Intent & Type** | Select co-op type (housing or agricultural), island, and confirm the group has 3+ founding members. |
| 2 | **Legal Structure** | Platform recommends the applicable HRS chapter(s) based on co-op type. Explains implications in plain language. |
| 3 | **Name & Purpose** | Enter the co-op's proposed name and mission statement. Platform checks DCCA name availability (or links to the lookup). |
| 4 | **Founding Members** | Invite founding members. Each must accept before the formation can proceed. |
| 5 | **Governing Documents** | Generate draft Articles of Incorporation and Bylaws from Hawaii-specific templates. Organizer can edit before finalizing. |
| 6 | **Formation Vote** | Founding members vote to adopt the governing documents. Requires unanimous consent. |
| 7 | **DCCA Filing** | Step-by-step instructions for filing with Hawaii DCCA. Platform tracks filing status (manual confirmation). |
| 8 | **EIN & Bank Account** | Instructions for obtaining an IRS EIN and opening a co-op bank account. |
| 9 | **First Meeting** | Platform generates a formation meeting agenda. Admin records minutes and marks the co-op operational. |

Each step shows its status. Blocked steps display a clear explanation of what's needed to proceed.

---

## 2. Membership Management

### Member Registry
- List of all members with status (`pending` | `active` | `exited`), join date, and role.
- Admins can approve pending members, change roles, or mark a member as exited.
- Members can view their own profile and equity balance.

### Equity Tracking
- Each member has an equity account (denominated in USD).
- Admins record equity contributions and withdrawals manually (v1 does not process payments).
- Members can see their equity history with a transaction log.

### Membership Admission
- New member applications trigger an admission proposal (if bylaws require a vote).
- Application form is customizable per co-op.

---

## 3. Governance

### Proposals
- Any member can create a proposal (subject to co-op bylaws configuration).
- Proposal types: `bylaw_amendment`, `membership_admission`, `budget_approval`, `general`.
- Each proposal specifies: description, voting period, quorum requirement (% of active members), and pass threshold.

### Voting
- Members receive a notification when a new proposal is open.
- Votes are recorded as `yes`, `no`, or `abstain`.
- Votes are final and cannot be changed after submission.
- Tally is revealed only after the voting period closes (to prevent bandwagon effects).
- Results are stored in the immutable audit log.

### Quorum & Passage Rules
- Default rules are generated from the chosen model bylaws.
- Admins can customize quorum % and pass threshold per proposal type.
- A proposal that fails quorum is automatically marked `failed_quorum` (not passed or failed on merits).

---

## 4. Document Management

### Document Library
- Secure file storage per co-op. Only members can access their co-op's documents.
- Documents tagged by type with a human-readable display name.
- Admins can upload, replace, or archive documents.

### Version Control
- When a governing document is amended (e.g., bylaws update), the old version is retained and the new version becomes current.
- Members can view the full version history of any document.

### Templates
Hawaii-specific templates included out of the box:

| Template | Notes |
|---|---|
| Articles of Incorporation (HRS 421) | General cooperative |
| Articles of Incorporation (HRS 421H) | Housing cooperative |
| Model Bylaws — Housing Co-op | Includes member equity, share transfer restrictions |
| Model Bylaws — Agricultural Co-op | Includes patronage dividend provisions |
| Membership Application & Agreement | Customizable |
| Formation Meeting Agenda | Used in Step 9 of formation wizard |
| Annual Meeting Agenda | Recurring governance |

All templates carry a disclaimer: *"This is a starting-point document. Have an attorney licensed in Hawaii review before filing or distributing."*

---

## 5. Meeting Management

### Meeting Creation
- Admins schedule a meeting with date, time, location (or video link), and agenda items.
- Members are notified on creation and 24 hours before the meeting.

### Agenda Builder
- Drag-and-drop agenda items with estimated durations.
- Standard items (call to order, quorum check, adjournment) are pre-populated.
- Proposals can be linked to agenda items directly.

### Minutes
- Admin (or designated secretary) records minutes during or after the meeting using a structured form.
- Minutes are submitted for member review and then stored in the Document Library as type `minutes`.

---

## 6. Notifications

- Email notifications for: proposal opened, vote reminder (24h before close), proposal result, meeting scheduled, meeting reminder, membership application received, formation step completed.
- In-app notification center for the same events.
- Members can configure notification preferences per category.

---

## 7. Advisor Access

- An advisor (lawyer, accountant, USDA agent) can be granted read-only or read-write access to one or more co-ops.
- Advisor dashboard shows a list of their co-ops and each co-op's formation status.
- Advisors can leave notes on formation workflow steps (visible to co-op admins).
- Advisors do not count toward quorum and cannot vote on proposals.

---

## Out of Scope for v1

The following are explicitly deferred to future versions:

- **Payment processing** — equity contributions tracked manually; no Stripe/ACH integration.
- **Public directory** — co-ops are private by default; no public search.
- **Land title tools** — no integration with Bureau of Conveyances or title companies.
- **Loan origination** — no connection to USDA FSA, CDFI lenders, or similar.
- **Mobile apps** — web only; responsive design is required but no native apps.
- **Multi-language support** — English only in v1; Hawaiian language support is a v2 consideration.
