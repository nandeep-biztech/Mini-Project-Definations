# Bulk Dynamic Email Campaign System (With Data Validation Engine)

---

## 1️⃣ Core Objective

Build a production-style bulk email campaign system with:

- Authentication
- Dynamic CSV upload
- Row-level validation
- Duplicate detection & removal
- Template editor with placeholder validation
- SMTP email sending
- Background processing
- Logging & failure handling

This is not a mail sender.  
It is a controlled data-processing + campaign engine.

---

## 2️⃣ Tech Stack

- **Frontend**: Next.js
- **UI**: shadcn/ui or Tailwind CSS
- **Editor**: CKEditor 5 or Tiptap
- **APIs**: Node.js (Express.js)
- **Database**: PostgreSQL
- **CSV processing**: Python

---

## 3️⃣ User Journey

### Flow (after login)

1. **Login** — User authenticates.

2. **Dashboard** — Single "Upload CSV" button. User uploads CSV file.

3. **CSV Processing** — Full Data Validation Layer (Section 5) runs on upload. Validation summary shown before data table.

4. **Data Table** — After validation, display table with CSV data and columns:
   - Pagination
   - Action column: allow delete row per row

5. **Campaign Name** — User provides a campaign name to identify and maintain the campaign.

6. **Template Create/Edit** — User creates or edits email template.

7. **Placeholder Status Table** — While editing template, show a table alongside the editor:
   - Column 1: Placeholder name (from CSV columns)
   - Column 2: ✅ or ❌ — indicates whether placeholder is valid (exists in CSV and available for replacement)

8. **Submit Button** — Enabled only when all placeholders in template are available in CSV column titles for replacement. On click → create campaign in database (persist campaign name, template, CSV data). If invalid → display readable error below editor.

9. **Send Mail** — When campaign created, "Send Mail" button enabled. On click → process send (Section 7).

10. **Campaign History** — Page listing campaigns created by the logged-in user only. Table with pagination; columns: campaign title, emails sent, action. Action column has "View details" button. On click: display template used; below that, display all recipient/CSV data for that campaign.

---

## 4️⃣ Authentication

All campaign operations require authenticated users. No anonymous access.

---

### A. Authentication Flow

#### Registration

- Email + password signup
- Validate email format before submission
- Password strength requirements (min 8 chars, at least one letter + one number)
- Reject duplicate email registration
- Optional: email verification before first login

#### Login

- Email + password credentials
- Return session/token on success
- Clear error messages for invalid credentials (avoid "user exists" vs "wrong password" enumeration)
- Lock account after N failed attempts (e.g., 5) — optional but recommended

#### Logout

- Invalidate session/token server-side
- Clear client-side storage

---

### B. Token/Session Strategy

Intern must choose and justify:

- **JWT** (stateless, good for API-first, needs refresh token strategy)
- **Session cookies** (stateful, simpler, works well with SSR)

Requirements:

- Token/session expiry (e.g., 24h access, 7d refresh)
- Secure storage (httpOnly cookies preferred over localStorage for web)
- Refresh flow if using JWT

---

### C. Security Requirements

- **Password hashing**: bcrypt — never store plain text
- **HTTPS only** in production
- **Rate limiting** on `/login` and `/register` (e.g., 5 req/min per IP)
- **CSRF protection** for state-changing requests (if using cookies)
- **No sensitive data in JWT** if chosen (e.g., no password hash)

---

### D. Protected Routes & API

- All campaign pages require authentication
- All API endpoints (CSV upload, campaign create, send, etc.) must validate token/session
- Return 401 Unauthorized for missing/invalid auth
- Redirect unauthenticated users to login (or return 401 for API)

---

### E. Users Table (Schema Reference)

Full schema for `users` table (required for auth):

| Column       | Type          | Notes                    |
|--------------|---------------|--------------------------|
| id           | UUID/PK       |                          |
| email        | VARCHAR(255)  | UNIQUE, NOT NULL         |
| password     | VARCHAR(255)  | NOT NULL                 |
| firstName    | VARCHAR(100)  | Optional                 |
| lastName     | VARCHAR(100)  | Optional                 |
| profileImage | VARCHAR(500)  | Optional, URL or path    |
| createdAt    | TIMESTAMP     |                          |
| updatedAt    | TIMESTAMP     |                          |
| lastLoginAt  | TIMESTAMP     | Optional                 |

---

### F. Profile Update Flow

Update Profile screen allows user to edit:

- **profileImage** — upload or change avatar/image
- **firstName** — editable
- **lastName** — editable
- **email** — read-only, not editable (identity field)

Validation:

- firstName/lastName: trim whitespace, optional max length
- profileImage: validate file type (e.g., jpg, png, webp), size limit

---

## 5️⃣ CSV Processing -- Full Data Validation Layer

This is where Python becomes serious.

---

### A. File-Level Validation

When CSV is uploaded:

System must validate:

- File type (only .csv)
- File not empty
- Header row exists
- No duplicate column names
- Column names must be snake_case (e.g., `first_name`, `email`, `created_at`) — no spaces allowed in column headers; these names are bound to template placeholders (Section 6)
- Required column `email` exists
- File size limit enforcement

If any of these fail → reject upload.

---

### B. Row-Level Validation (Critical)

Every row must be validated.

#### Required Checks

For each row:

- Email format valid (regex)
- No empty required fields
- Trim whitespace
- Normalize case where needed
- Remove leading/trailing spaces
- Detect malformed CSV row
- Detect missing column values

#### Data Type Validation

If column like:

date  
amount

Validate:

- Date format (YYYY-MM-DD or defined format)
- Numeric fields are numeric
- Boolean fields are boolean

Intern must implement flexible validation.

---

### C. Duplicate Handling (Very Important)

System must:

- Detect duplicate emails inside CSV
- Detect duplicate rows
- Detect duplicates against previously uploaded campaigns (optional advanced)

Provide strategy:

- Remove duplicates automatically  
  OR
- Flag duplicates and show report

Intern must justify decision.

---

### D. Data Cleaning & Normalization

Before storing:

- Normalize emails to lowercase
- Standardize date format
- Remove invisible characters
- Strip special characters where needed

---

### E. Validation Output Summary

After upload:

System must generate report:

- Total rows
- Valid rows
- Invalid rows
- Duplicate rows removed
- Clean rows remaining

UI must show this summary before enabling template creation.

This forces real system feedback loop.

---

## 6️⃣ Placeholder Validation Logic (Improved)

Placeholders are **bound to CSV column names**. Template placeholders must use snake_case to match the CSV headers.

Example — CSV has columns `first_name`, `email`, `meeting_date`:

Hello {first_name}, your meeting is on {meeting_date}

System must:

- Extract placeholders using regex
- Placeholder names must be snake_case (bound to CSV column names)
- Match placeholders against CSV headers — each `{column_name}` maps to CSV column `column_name`
- Reject unknown placeholders (not present in CSV)
- Reject unused required fields (optional strict mode)

Edge cases to handle:

- `{ first_name }` — trim whitespace, normalize to `first_name`
- `{FIRST_NAME}` — normalize to snake_case `first_name` for matching
- `{first_name1}` — invalid if no such CSV column
- Nested braces `{{first_name}}` — reject or unwrap per implementation

If mismatch → disable Send button.

Must validate both:

- Frontend (UX)
- Backend (security)

---

## 7️⃣ Campaign Execution Flow

When user clicks "Send Email":

1. Lock campaign status → "processing"
2. Generate personalized email per valid row
3. Push into background queue
4. Send via SMTP
5. Track per-recipient status:

   - pending
   - sent
   - failed
   - retried

6. Retry failed emails up to 3 times
7. Final campaign summary

Must prevent:

- Double submission
- Re-trigger while running

---

## 8️⃣ Python Responsibilities

Python must handle:

- CSV parsing using pandas
- Row validation engine
- Duplicate detection logic
- Data normalization
- Placeholder replacement engine
- Batch processing
- Logging per row
- Retry mechanism

This ensures Python is not cosmetic.

---

## 9️⃣ Database Schema (Guidelines)

Intern must design the schema to support:

- User authentication and profile
- CSV upload and validated data storage
- Validation summary (total, valid, invalid, duplicate counts)
- Campaign metadata (including campaign name), template, send status; campaigns linked to creating user
- Per-recipient send tracking
- Email audit trail

Use camelCase for column names. Create indexes and justify choices.

---

## 🔟 Failure Handling Requirements

Intern must answer:

- What if SMTP crashes mid-campaign?
- What if Python worker stops?
- What if DB commit fails after sending?
- How to ensure idempotency?
- How to resume failed campaign?

If they ignore this, they don't understand production systems.

---

## 1️⃣1️⃣ UI Requirements (Next.js)

Must include:

- **Auth pages**: Login, Register (and Logout in nav/header)
- **Profile/Update Profile page**: Edit profileImage, firstName, lastName; email displayed read-only
- Redirect to login when accessing protected routes while unauthenticated
- **Dashboard**: Single "Upload CSV" button; after upload, validation summary
- **Campaign name**: Input field to name the campaign (for identification and maintenance)
- **Data table**: CSV data with columns, pagination, action column for delete row
- **Template editor**: Create/edit template; placeholder status table alongside (placeholder name | ✅/❌)
- **Campaign history page**: Table with pagination; columns: campaign title, emails sent, action (View details button). On View details: show template used; below, show all recipient/CSV data for that campaign. Display only campaigns created by the logged-in user.
- **Submit/Send button**: Enabled only when all placeholders valid; readable error below editor when invalid
- CSV validation report page
- Campaign progress dashboard
- Error breakdown table
- Logs viewer
- SSR for dashboard page
- Loading + error states

---

## 1️⃣2️⃣ Non-Functional Requirements

- Centralized error handling
- Structured logging (JSON logs preferred)
- Environment-based config
- Clean folder separation
- Proper Git workflow
- No business logic inside controllers
- No blocking email sending inside request handler