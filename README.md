# Email to Task

An n8n automation that turns an inbox into a triaged, trackable job-search workflow: incoming email is classified by an LLM, routed into the right Notion database, and followed up on automatically.

## What it does

Every 5 minutes, an Outlook trigger polls for new mail. Each message is classified by Claude Haiku (Basic LLM Chain + Structured Output Parser) into one of seven categories: `ACTION_NEEDED`, `JOB_CONFIRMATION`, `JOB_REJECTION`, `RECRUITER_REPLY`, `JOB_ALERT`, `FYI`, `IGNORE`. Every classification is logged to a metrics database, then gated on confidence: results below 80 fall through to a manual review queue instead of auto-filing. Confident results are routed by category to the right Notion database. Two scheduled workflows follow up: a daily follow-up scheduler and a weekly digest email.

## Workflows

**`workflows/email-classifier.json`** — Outlook Trigger (5 min poll) → Classify Email → log to Email Metrics → confidence gate (≥80) → switch on category:
- `ACTION_NEEDED` → Daily Planner (tagged `Category = Email`)
- `RECRUITER_REPLY` → Job Search Tasks
- `JOB_CONFIRMATION` → Job Application Tracker (new row, `Status = Applied`)
- `JOB_REJECTION` → find the matching tracker row and update it (`Status = Rejected`, `Reply Received = today`)

**`workflows/follow-up-scheduler.json`** — runs daily at 08:00. Pulls tracker rows and, in a Code node, flags rows where `Applied Date` was **exactly** yesterday with no outreach sent, or outreach was sent **exactly** 6 days ago with no reply — then creates a Job Search Tasks entry per flagged row. Exact-day matching, not "on or before", ensures each condition fires once per row, ever, instead of re-firing daily until acted on.

**`workflows/monday-digest.json`** — runs Mondays at 08:00. Pulls all four databases, aggregates the trailing 7 days (applications submitted, outreach sent, replies received, tasks overdue per task database, classification volume and precision), and emails the summary via Outlook.

## Notion schema

| Database | Key properties |
|---|---|
| Job Application Tracker | Company (title), Role, Status (select), Applied Date, Outreach Sent, Reply Received (dates), Job Link, Notes |
| Email Metrics Log | Summary (title), Timestamp, Category, Confidence, Suggested Task, Corrected, Auto-filed |
| Job Search Tasks | Task (title), Date, Done, Company, Notes |
| Daily Planner | Tasks (title), Date, Category (select, includes `Email`), Priority Level, Done, Notes |

A filtered view on Email Metrics Log (`Confidence < 80`) doubles as the manual review queue.

## Setup

1. Create the four Notion databases above and share them with your n8n Notion integration.
2. In each JSON, replace the `YOUR_..._DATABASE_ID` placeholders with your real database IDs, and `you@example.com` with your own address.
3. Import each JSON into n8n (Workflows → Import from File).
4. Re-attach credentials: Outlook, Anthropic, Notion — stripped from the exported JSON.
5. Activate all three workflows.

## Design notes

- **Notion's simplified output** flattens every property to `property_` + lowercase name (spaces → underscores); the title property also appears as top-level `name`. Code nodes read these keys, not the human-readable property names.
- **Date properties** come back either as a plain ISO string or as `{ start, end, time_zone }`; a small `dval()` helper normalizes both.
- **Fan-in caution**: wiring multiple Notion `getAll` nodes into one Code node's input makes it (and everything downstream) execute once per branch. The digest avoids this with a serial chain (`executeOnce` + `alwaysOutputData` on each getAll node), pulling each dataset by node name inside the Code node.
- **Exact-day matching** in the follow-up scheduler prevents duplicates — a `≤` comparison would re-fire the same condition every later run.
- **Known quirk**: the digest email lands in the same monitored inbox, so the classifier files it (as `FYI`) — adding one classification per week to the following digest's volume count. A subject-line filter in the classifier is the planned fix. Property names matter: the Code nodes read the flattened keys (e.g. `property_applied_date`), so the Notion property must be named exactly `Applied Date`.
