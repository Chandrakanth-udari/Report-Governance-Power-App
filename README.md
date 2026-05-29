# Data Trust Foundation — Report Governance App

A Power Apps Canvas app that manages the full **report validation and certification lifecycle** — from a developer running data validations to a client formally approving (or rejecting) a report as certified.

> **Client engagement.** Built as a production solution for a client in the commercial door & hardware manufacturing industry. Report names, workspaces, and user labels shown in the screenshots are anonymized sample data.

## The Problem

Business reports drive decisions — but "can I trust this number?" usually had no real answer. A report would be built, shared, and quietly relied upon with no record of whether anyone had checked it against the source system, who signed off, or when. When a figure later turned out to be wrong, there was no trail: no validations, no approver, no history. Developers and clients also had no shared, controlled space to disagree — a client who doubted a number had nowhere formal to flag it, and the developer had no structured way to see and resolve that objection.

## The Solution

Give "is this report trustworthy?" a defined, auditable workflow with two clearly separated audiences:

1. **Developers** register a report and run **reconciliation validations** — source-system value vs. report value, with variance, variance %, and a PASS / FAIL / WARN result recorded per check.
2. Once validated, a developer **submits** the report for client approval; an assigned client is notified by a **Power Automate** email.
3. **Clients** review the validations behind the report and formally **Approve** (→ Certified) or **Reject** it. On rejection, the client **flags which validation** looked wrong (or picks "Other" and gives a reason), which the developer sees in a dedicated detail view.
4. **Every status change is written to an audit history table** — who, when, from-status → to-status, and notes — so the report's trust journey is permanently recorded.

Access is role- and workspace-aware: clients only see reports in workspaces they're assigned to, with **Approver vs. Viewer** rights controlling who can actually approve or reject.

## Status Lifecycle

```
Draft → Under Review → Validated → Pending Client Approval → Certified / Rejected
```

## How It Works

```
 Developer (internal)                      Client (external)
 ┌──────────────────────────────┐         ┌──────────────────────────────┐
 │ register report               │         │ review validations            │
 │ run validations (src vs PBI)  │         │ Approve  → Certified          │
 │ submit for approval ──────────┼────────►│ Reject   → flag validation    │
 └──────────────┬───────────────┘  email  └──────────────┬───────────────┘
                │  (Power Automate notifies client)       │
                ▼                                          ▼
   ┌───────────────────────────────────────────────────────────────┐
   │ SQL: tbl_reports · tbl_validation_results · tbl_certification   │
   │      tbl_users · tbl_user_workspaces · tbl_report_history       │
   │      (every status change is logged to the history table)       │
   └───────────────────────────────────────────────────────────────┘
```

## Key Features

- **Role-based experience** — Developer vs. Client views are driven by a user registry; navigation and actions adapt to the signed-in role.
- **Validation center** — record reconciliation checks (source value vs. report value, variance, variance %, PASS / FAIL / WARN) per report.
- **Certification workflow** — developers submit validated reports; assigned clients approve or reject with reasons.
- **Rejection with flagged validation** — when a client rejects, they flag which validation looked wrong (or "Other" + a reason); developers get a detail view of the rejection.
- **Workspace-scoped access** — clients see only reports in workspaces they're assigned to, with Approver vs. Viewer action rights.
- **Audit history** — every status change is logged (who, when, from → to, notes).
- **Email notifications** — a Power Automate flow notifies the assigned client when a report is submitted for approval.

## Tech Stack

- **Power Apps** (Canvas app, role-aware multi-screen)
- **SQL Server** (governance tables + audit history)
- **Power Automate** (approval notification emails)

## Screenshots

| Dashboard | Reports | Validation Center |
|---|---|---|
| ![Dashboard](screenshots/01-dashboard.png) | ![Reports](screenshots/02-reports.png) | ![Validation Center](screenshots/03-validation-center.png) |

| Certification — pending approval | Certification — validated |
|---|---|
| ![Certification pending](screenshots/04-certification-pending.png) | ![Certification validated](screenshots/05-certification-validated.png) |

## Data Model

Full schema in [`sql/database-architecture.md`](sql/database-architecture.md). Core tables:

| Table | Purpose |
|---|---|
| `tbl_reports` | Report inventory, lifecycle status, approval metadata |
| `tbl_validation_results` | Reconciliation records (source vs. report values, variance, result) |
| `tbl_certification` | Certification records, including flagged validations on rejection |
| `tbl_users` | Role-based access (Developer / Client) |
| `tbl_user_workspaces` | Client → workspace access and Approver/Viewer action role |
| `tbl_report_history` | Full audit timeline of status transitions |

---

*Screenshots use anonymized sample data. Internal connection identifiers have been omitted.*
