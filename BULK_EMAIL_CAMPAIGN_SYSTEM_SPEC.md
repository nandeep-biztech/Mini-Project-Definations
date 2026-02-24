Bulk Dynamic Email Campaign System (With Data Validation Engine)
================================================================

* * * * *

1️⃣ Core Objective
==================

Build a production-style bulk email campaign system with:

- Authentication

- Dynamic CSV upload

- Row-level validation

- Duplicate detection & removal

- Template editor with placeholder validation

- SMTP email sending

- Background processing

- Logging & failure handling

This is not a mail sender.\
It is a controlled data-processing + campaign engine.

* * * * *

2️⃣ CSV Processing -- Full Data Validation Layer
===============================================

This is where Python becomes serious.

* * * * *

A. File-Level Validation
------------------------

When CSV is uploaded:

System must validate:

- File type (only .csv)

- File not empty

- Header row exists

- No duplicate column names

- Required column `email` exists

- File size limit enforcement

If any of these fail → reject upload.

* * * * *

B. Row-Level Validation (Critical)
----------------------------------

Every row must be validated.

### Required Checks

For each row:

- Email format valid (regex)

- No empty required fields

- Trim whitespace

- Normalize case where needed

- Remove leading/trailing spaces

- Detect malformed CSV row

- Detect missing column values

### Data Type Validation

If column like:

date\
amount

Validate:

- Date format (YYYY-MM-DD or defined format)

- Numeric fields are numeric

- Boolean fields are boolean

Intern must implement flexible validation.

* * * * *

C. Duplicate Handling (Very Important)
--------------------------------------

System must:

- Detect duplicate emails inside CSV

- Detect duplicate rows

- Detect duplicates against previously uploaded campaigns (optional advanced)

Provide strategy:

- Remove duplicates automatically\
    OR

- Flag duplicates and show report

Intern must justify decision.

* * * * *

D. Data Cleaning & Normalization
--------------------------------

Before storing:

- Normalize emails to lowercase

- Standardize date format

- Remove invisible characters

- Strip special characters where needed

* * * * *

E. Validation Output Summary
----------------------------

After upload:

System must generate report:

- Total rows

- Valid rows

- Invalid rows

- Duplicate rows removed

- Clean rows remaining

UI must show this summary before enabling template creation.

This forces real system feedback loop.

* * * * *

3️⃣ Placeholder Validation Logic (Improved)
===========================================

Template contains:

Hello {name}, your meeting is on {date}

System must:

- Extract placeholders using regex

- Normalize placeholder names

- Match against CSV headers

- Case-insensitive matching

- Reject unknown placeholders

- Reject unused required fields (optional strict mode)

Edge cases to handle:

- `{ name }`

- `{NAME}`

- `{name1}`

- `{name_1}`

- Nested braces `{{name}}`

If mismatch → disable Send button.

Must validate both:

- Frontend (UX)

- Backend (security)

* * * * *

4️⃣ Campaign Execution Flow (Improved)
======================================

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

* * * * *

5️⃣ Python Responsibilities (Now Clear & Strong)
================================================

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

* * * * *

6️⃣ Database Schema (Refined)
=============================

Tables:

### users

### uploads

### upload_validation_summary

### campaigns

### campaign_recipients

### email_logs

Indexes:

- email index

- campaign_id index

- status index

- created_at index

Intern must justify indexing choices.

* * * * *

7️⃣ Failure Handling Requirements
=================================

Intern must answer:

- What if SMTP crashes mid-campaign?

- What if Python worker stops?

- What if DB commit fails after sending?

- How to ensure idempotency?

- How to resume failed campaign?

If they ignore this, they don't understand production systems.

* * * * *

8️⃣ UI Requirements (Next.js)
=============================

Must include:

- CSV validation report page

- Template editor

- Placeholder validation indicator

- Campaign progress dashboard

- Error breakdown table

- Logs viewer

- SSR for dashboard page

- Loading + error states

* * * * *

9️⃣ Non-Functional Requirements
===============================

- Centralized error handling

- Structured logging (JSON logs preferred)

- Environment-based config

- Clean folder separation

- Proper Git workflow

- No business logic inside controllers

- No blocking email sending inside request handler

* * * * *

🔟 Hard Mode Enhancements
=========================

If intern is strong:

- Add rate limiting per domain

- Add scheduled sending

- Add bounce tracking simulation

- Add unsubscribe logic

- Add HTML sanitization

- Add throttling logic
