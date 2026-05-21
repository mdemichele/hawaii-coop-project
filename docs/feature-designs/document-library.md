# Document Library — Design Document

**Status:** Draft  
**Scope:** v1  
**Feature area:** Document Management  

---

## 1. Goals

1. **Single source of truth.** Every governing document, meeting minutes file, and member agreement lives in one place — not scattered across email threads or personal Google Drives.
2. **Access-controlled.** Only members of a co-op can access that co-op's documents. No document is publicly accessible.
3. **Version-aware.** When a governing document is amended, the old version is preserved and the new version becomes current. Members can always see the full history of any document.
4. **Template-driven.** Hawaii-specific legal templates are available out of the box so co-ops don't start from a blank page.
5. **Formation-integrated.** Documents generated during the formation wizard are automatically placed in the library — no manual re-upload required.

---

## 2. Core Concepts

### 2.1 Document Types

| Type | Description | Who can upload |
|------|-------------|----------------|
| `articles_of_incorporation` | Filed document establishing the co-op as a legal entity | Admin |
| `bylaws` | Governing rules of the co-op | Admin |
| `membership_agreement` | Agreement signed by each incoming member | Admin |
| `minutes` | Meeting minutes | Admin / Secretary role |
| `financial` | Budgets, financial statements, equity summaries | Admin |
| `correspondence` | Letters to DCCA, IRS, banks, or other external parties | Admin |
| `other` | Any other document the co-op needs to store | Admin |

### 2.2 Document Lifecycle

```
            ┌───────────┐
            │  TEMPLATE │  ← platform-provided; read-only; not co-op-specific
            └─────┬─────┘
                  │ generate (formation wizard or manual)
                  ▼
            ┌───────────┐
            │   DRAFT   │  ← editable; not yet published to members
            └─────┬─────┘
                  │ publish
                  ▼
            ┌───────────┐
            │  CURRENT  │  ← the active version; visible to all members
            └─────┬─────┘
                  │ superseded by a new version
                  ▼
            ┌───────────┐
            │ SUPERSEDED│  ← retained in history; read-only
            └───────────┘
                  │ or archived directly
                  ▼
            ┌───────────┐
            │  ARCHIVED │  ← hidden from default view; retrievable by admin
            └───────────┘
```

Only one version of a document can be `CURRENT` at a time within its version chain. Uploading a new version of an existing document automatically transitions the previous `CURRENT` version to `SUPERSEDED`.

### 2.3 Version Chains

Documents that can be amended (bylaws, articles of incorporation, membership agreement) are grouped into version chains. Each chain has:
- A `document_type` and a display name (e.g., "Bylaws")
- One `CURRENT` version
- Zero or more `SUPERSEDED` versions in chronological order
- An optional `ARCHIVED` state that removes the chain from the default library view

Single-use documents (minutes from a specific meeting, a letter sent on a specific date) are not versioned — each is its own chain with a single entry.

---

## 3. Data Model

### 3.1 `document_chains` Table

Groups related document versions together under a single logical identity.

```sql
CREATE TABLE document_chains (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id  UUID NOT NULL REFERENCES organizations(id),

  type             TEXT NOT NULL CHECK (type IN (
                     'articles_of_incorporation','bylaws','membership_agreement',
                     'minutes','financial','correspondence','other'
                   )),
  display_name     TEXT NOT NULL,                    -- human-readable: "Bylaws", "March 2026 Meeting Minutes"

  -- The current active version (denormalized for fast lookup; kept in sync on every version change)
  current_version_id UUID REFERENCES documents(id),

  status           TEXT NOT NULL DEFAULT 'active'
                   CHECK (status IN ('active','archived')),

  created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  created_by       UUID NOT NULL REFERENCES users(id)
);
```

### 3.2 `documents` Table

One row per version of a document. Each row points to a file in object storage.

```sql
CREATE TABLE documents (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id  UUID NOT NULL REFERENCES organizations(id),
  chain_id         UUID NOT NULL REFERENCES document_chains(id),

  -- Version identity
  version_number   INT NOT NULL,                     -- 1, 2, 3... within the chain
  version_label    TEXT,                             -- optional: "v1.2", "Amended March 2026"
  version_notes    TEXT,                             -- what changed in this version

  -- File metadata
  file_key         TEXT NOT NULL,                    -- object storage key (not a public URL)
  file_name        TEXT NOT NULL,                    -- original uploaded filename
  file_size_bytes  BIGINT NOT NULL,
  mime_type        TEXT NOT NULL,

  -- Status within the chain
  status           TEXT NOT NULL DEFAULT 'draft'
                   CHECK (status IN ('draft','current','superseded')),

  -- Source: manually uploaded or generated by the platform
  source           TEXT NOT NULL DEFAULT 'upload'
                   CHECK (source IN ('upload','generated','template')),
  generated_from   TEXT,                             -- template slug if source = 'generated'

  -- Audit
  uploaded_by      UUID NOT NULL REFERENCES users(id),
  uploaded_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  published_at     TIMESTAMPTZ,                      -- when status moved to 'current'

  UNIQUE (chain_id, version_number)
);
```

### 3.3 `document_access_log` Table

Append-only. Records every download and preview for audit purposes.

```sql
CREATE TABLE document_access_log (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id  UUID NOT NULL REFERENCES organizations(id),
  document_id      UUID NOT NULL REFERENCES documents(id),
  member_id        UUID NOT NULL REFERENCES members(id),

  action           TEXT NOT NULL CHECK (action IN ('preview','download')),
  accessed_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  ip_address       INET
);
```

### 3.4 `templates` Table

Platform-managed. Not per-organization. Stores the Hawaii-specific document templates used during formation and for manual document generation.

```sql
CREATE TABLE templates (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  slug             TEXT UNIQUE NOT NULL,             -- e.g. 'articles_421h', 'bylaws_housing'
  display_name     TEXT NOT NULL,
  description      TEXT,
  coop_type        TEXT CHECK (coop_type IN ('housing','agricultural','both')),
  document_type    TEXT NOT NULL,                    -- maps to document_chains.type values
  file_key         TEXT NOT NULL,                    -- base template file in object storage
  variable_schema  JSONB NOT NULL DEFAULT '{}',     -- substitution variables and their types
  version          TEXT NOT NULL,                    -- platform template version, e.g. '2026-01'
  active           BOOLEAN NOT NULL DEFAULT true,
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## 4. File Storage

### 4.1 Object Storage Layout

Files are stored in an S3-compatible object store (e.g., AWS S3, Cloudflare R2). The bucket is **private** — no public access is permitted at the bucket level.

Key structure:
```
organizations/{org_id}/documents/{chain_id}/{document_id}/{file_name}
templates/{template_slug}/{version}/{file_name}
```

### 4.2 Access via Pre-signed URLs

Members never access object storage directly. Every file access goes through the API, which:

1. Verifies the requesting member belongs to the document's organization.
2. Checks that the document chain is not archived (or that the requester is an admin).
3. Generates a **pre-signed URL** with a short expiry (5 minutes for preview; 15 minutes for download).
4. Returns the pre-signed URL to the client.
5. Appends to `document_access_log`.

The pre-signed URL expires regardless of whether it was used. This prevents link sharing — a member cannot forward a document URL to a non-member.

### 4.3 Accepted File Types

| Format | MIME type | Max size |
|--------|-----------|----------|
| PDF | `application/pdf` | 25 MB |
| Word document | `application/vnd.openxmlformats-officedocument.wordprocessingml.document` | 25 MB |
| Image (for signed pages, scanned forms) | `image/jpeg`, `image/png` | 10 MB |

All uploads are virus-scanned before being made available. If a scan fails, the upload is rejected and the file is deleted from storage.

---

## 5. Document Flows

### 5.1 Uploading a New Document (Admin)

1. Admin clicks **+ Upload Document** in the library.
2. Selects document type from the dropdown.
3. Enters a display name (e.g., "2026 Annual Budget").
4. Optionally adds version notes.
5. Selects and uploads the file.
6. Chooses whether to publish immediately or save as draft.
   - **Publish immediately:** Document status → `current`; visible to all members.
   - **Save as draft:** Document status → `draft`; visible only to admins.
7. Platform creates a new `document_chains` row and a `documents` row at version 1.

### 5.2 Uploading a New Version of an Existing Document (Admin)

1. Admin opens an existing document chain (e.g., "Bylaws").
2. Clicks **Upload New Version**.
3. Enters version label (optional) and version notes describing what changed.
4. Selects and uploads the file.
5. Chooses publish immediately or draft.
6. On publish:
   - Previous `CURRENT` version → `SUPERSEDED`.
   - New version → `CURRENT`.
   - `document_chains.current_version_id` updated.
   - Both operations in a single database transaction.
7. All members are notified that the document has been updated (in-app; optional email per notification preferences).

### 5.3 Generating a Document from a Template

Available for admins outside of the formation wizard (e.g., generating a fresh membership agreement after the co-op changes its template).

1. Admin clicks **Generate from Template**.
2. Platform lists available templates filtered to the co-op's type (housing or agricultural).
3. Admin selects a template.
4. Platform pre-fills known substitution variables from `organizations` (name, island, HRS chapter, EIN, mission statement). Admin fills in any remaining variables in a simple form.
5. Platform generates a PDF and creates a `documents` row with `source = 'generated'` in a new chain (or a new version on an existing chain if the admin chose to update).
6. Admin reviews the generated document and publishes or saves as draft.

### 5.4 Archiving a Document Chain

When a document chain is no longer relevant (e.g., a superseded correspondence letter), an admin can archive it.

- Archived chains are hidden from the default library view.
- All versions remain in object storage and are retrievable via an "Archived" filter.
- Archiving does not delete anything.

### 5.5 Deleting a Document

Deletion is limited and intentionally difficult.

- **Draft documents:** Admins can delete a draft version that has never been published. The file is removed from object storage.
- **Published documents:** Cannot be deleted. They can only be archived (chain-level) or superseded (version-level). This preserves the audit trail.

If a document was generated during formation (source = `generated` or linked to `formation_workflows`), it cannot be deleted at all — only superseded.

---

## 6. Document Library UI

### 6.1 Library Overview Page

The main library page shows all active document chains for the co-op, grouped by type.

**Sections (collapsible):**
- Governing Documents (articles of incorporation, bylaws, membership agreement)
- Meeting Minutes
- Financial Documents
- Correspondence
- Other

Each row shows: document display name, current version label, last updated date, file size, and action buttons (Preview, Download, Version History, [Admin: Upload New Version, Archive]).

Members who haven't signed the membership agreement see a banner at the top of the library: *"You haven't signed the current membership agreement. [Review and Sign]"*

### 6.2 Document Detail Page

Clicking a document chain opens its detail view:

- **Current version panel:** Preview button, download button, version label, upload date, uploaded by, version notes.
- **Version history table:** All versions in reverse chronological order — version number, label, status badge (Current / Superseded), upload date, uploaded by, download link.
- **Linked references:** If this document is linked to a proposal (e.g., the bylaws amendment that passed), a link to that proposal is shown.

### 6.3 Template Browser

Accessible from the library via **Browse Templates**. Shows platform-provided templates with:
- Display name and description
- Applicable co-op type badge (Housing / Agricultural / Both)
- Document type
- A **Generate** button that launches the generation flow (section 5.3).

Templates are read-only — members and admins cannot edit the base templates, only the documents generated from them.

---

## 7. Member Agreement Signing

A specific flow for the membership agreement document type, since every member must sign it.

### 7.1 How It Works

When the admin publishes a new `membership_agreement` version, the platform marks all active members as having an unsigned current agreement. Members see a banner on their dashboard and in the library until they sign.

Signing in v1 is an **acknowledgement gate** — the member clicks "I have read and agree to the membership agreement" after reviewing the document. This is not a cryptographic e-signature. A proper e-signature integration (DocuSign, HelloSign) is deferred to v2.

### 7.2 Signature Records

```sql
CREATE TABLE membership_agreement_signatures (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id  UUID NOT NULL REFERENCES organizations(id),
  member_id        UUID NOT NULL REFERENCES members(id),
  document_id      UUID NOT NULL REFERENCES documents(id),  -- the version they signed

  signed_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  ip_address       INET,

  UNIQUE (member_id, document_id)
);
```

Admins can see which members have and haven't signed the current agreement from the document detail page and from the member roster.

---

## 8. Formation Wizard Integration

Documents generated during formation (Articles of Incorporation, Bylaws, Membership Agreement, Formation Meeting Agenda) are automatically placed in the library:

- The formation wizard calls the same document generation logic used by the manual generation flow.
- Generated documents are stored with `source = 'generated'` and `generated_from = '<template_slug>'`.
- They appear in the library immediately after generation (as drafts until the formation vote passes and the organizer publishes them).
- When the co-op reaches `OPERATIONAL`, the organizer is prompted to publish all formation documents so they are visible to all members.

This means no duplicate documents — the library is the single store, and the formation wizard is just one path for putting documents into it.

---

## 9. API Sketch

All endpoints scoped under `/api/v1/organizations/{org_id}/`.

### Document Chains

| Method | Path | Who | Description |
|--------|------|-----|-------------|
| `GET` | `documents` | Member | List all active document chains |
| `GET` | `documents?status=archived` | Admin | Include archived chains |
| `POST` | `documents` | Admin | Create a new document chain (upload flow) |
| `GET` | `documents/{chain_id}` | Member | Get chain detail with version history |
| `PATCH` | `documents/{chain_id}` | Admin | Update display name or archive |

### Document Versions

| Method | Path | Who | Description |
|--------|------|-----|-------------|
| `POST` | `documents/{chain_id}/versions` | Admin | Upload a new version |
| `GET` | `documents/{chain_id}/versions/{doc_id}` | Member | Get version metadata |
| `POST` | `documents/{chain_id}/versions/{doc_id}/publish` | Admin | Publish a draft version |
| `DELETE` | `documents/{chain_id}/versions/{doc_id}` | Admin | Delete an unpublished draft |

### File Access

| Method | Path | Who | Description |
|--------|------|-----|-------------|
| `GET` | `documents/{chain_id}/versions/{doc_id}/preview-url` | Member | Get a short-lived pre-signed URL for in-browser preview |
| `GET` | `documents/{chain_id}/versions/{doc_id}/download-url` | Member | Get a short-lived pre-signed URL for download |

### Templates

| Method | Path | Who | Description |
|--------|------|-----|-------------|
| `GET` | `templates` | Member | List available templates for this co-op's type |
| `POST` | `documents/generate` | Admin | Generate a document from a template |

### Signatures

| Method | Path | Who | Description |
|--------|------|-----|-------------|
| `GET` | `documents/{chain_id}/signatures` | Admin | List who has signed the current membership agreement version |
| `POST` | `documents/{chain_id}/versions/{doc_id}/sign` | Member | Record membership agreement acknowledgement |

---

## 10. Notifications

| Event | Who is notified | Channel |
|-------|----------------|---------|
| New governing document published | All active members | Email + in-app |
| Bylaws or articles updated (new version) | All active members | Email + in-app |
| New meeting minutes published | All active members | In-app |
| Membership agreement updated — signature required | All active members | Email + in-app |
| Document generated from formation wizard | Organizer | In-app |

---

## 11. Key UI Pages

| Page | Path | Audience |
|------|------|----------|
| Document library | `/documents` | All members |
| Document chain detail | `/documents/{chain_id}` | All members |
| Template browser | `/documents/templates` | All members / Admin |
| Upload flow | `/documents/upload` | Admin |
| Generate from template | `/documents/generate` | Admin |

---

## 12. Open Questions

1. **In-browser preview vs. download only.** Previewing PDFs in-browser requires either a PDF.js embed or an iframe pointing at the pre-signed URL. Do we want to support in-browser preview in v1, or is download-only acceptable? Preview improves accessibility (no download needed) but adds frontend complexity.

2. **Member-uploaded documents.** The current design restricts uploads to admins. Should members be able to upload documents to a personal folder or to a shared "member contributions" section? Risk: clutters the library; harder to keep organized.

3. **Document linking to proposals.** A `bylaw_amendment` proposal can link to a document in the library. Should we enforce that the linked document exists before the proposal can be submitted, or is this advisory?

4. **Template versioning.** Platform templates will need to be updated when Hawaii statutes change. When a template is updated, existing documents generated from the old template are not automatically revised. Should the platform notify admins when a template they used has been updated so they can review and re-generate if needed?

5. **Signature strength.** The v1 acknowledgement-gate approach (checkbox, IP address recorded) is not a legal e-signature. Is this sufficient for the membership agreements the co-op will use, or do we need to budget for a DocuSign-class integration sooner than v2?

6. **Document search.** Should members be able to full-text search document contents, or is browse-by-type sufficient for v1? Full-text search requires indexing file contents (extracting text from PDFs), which is non-trivial.

---

## 13. Out of Scope (v1)

- Cryptographic e-signatures (DocuSign, HelloSign, or equivalent)
- Full-text search of document contents
- Document collaboration or inline editing (Google Docs-style)
- Member-uploaded personal documents
- Document expiry or renewal reminders (e.g., "insurance certificate expires in 30 days")
- Integration with Hawaii DCCA or Bureau of Conveyances for direct filing
- Automatic template updates when Hawaii statutes change
- Folder / subfolder organization within document types
