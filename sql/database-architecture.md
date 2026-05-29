# Data Trust Foundation (DTF) – SQL Database Architecture

# 1. dbo.tbl_reports

## Purpose
Master governance table storing report inventory, lifecycle, approvals, and governance metadata.

```sql
CREATE TABLE dbo.tbl_reports (

    id INT IDENTITY(1,1) PRIMARY KEY,

    report_id VARCHAR(100) UNIQUE NOT NULL,

    report_name VARCHAR(500) NOT NULL,

    workspace_name VARCHAR(500),

    report_url VARCHAR(MAX),

    report_status VARCHAR(100) DEFAULT 'Draft',

    notes VARCHAR(MAX),

    updated_by VARCHAR(255),

    last_updated DATETIME DEFAULT GETDATE(),

    submitted_for_approval_by VARCHAR(255),

    submitted_date DATETIME,

    client_decision VARCHAR(50),

    client_notes VARCHAR(MAX),

    client_decided_by VARCHAR(255),

    client_decision_date DATETIME,

    assigned_client_email NVARCHAR(255)

);
```

## Status Constraint

```sql
ALTER TABLE dbo.tbl_reports
ADD CONSTRAINT chk_report_status
CHECK (
    report_status IN (
        'Draft',
        'Under Review',
        'Validated',
        'Pending Client Approval',
        'Certified',
        'Rejected'
    )
);
```

## Client Decision Constraint

```sql
ALTER TABLE dbo.tbl_reports
ADD CONSTRAINT chk_client_decision
CHECK (
    client_decision IN (
        'Approved',
        'Rejected'
    )
    OR client_decision IS NULL
);
```

# 2. dbo.tbl_validation_results

## Purpose
Stores validation and reconciliation records for reports.

```sql
CREATE TABLE dbo.tbl_validation_results (

    validation_id INT IDENTITY(1,1) PRIMARY KEY,

    report_id VARCHAR(100) NOT NULL,

    rule_id INT NULL,

    validation_name VARCHAR(500),

    source_system VARCHAR(255),

    source_value DECIMAL(18,2),

    powerbi_value DECIMAL(18,2),

    variance DECIMAL(18,2),

    variance_percent DECIMAL(18,4),

    validation_result VARCHAR(50),

    validation_notes VARCHAR(MAX),

    validated_by VARCHAR(255),

    validated_date DATETIME DEFAULT GETDATE(),

    client_comment NVARCHAR(MAX) NULL,

    client_suggested_result NVARCHAR(50) NULL

);
```

# 3. dbo.tbl_certification

## Purpose
Stores certification lifecycle records.

```sql
CREATE TABLE dbo.tbl_certification (

    certification_id INT IDENTITY(1,1) PRIMARY KEY,

    report_id VARCHAR(100) NOT NULL,

    certification_status VARCHAR(100),

    approved_by VARCHAR(255),

    approval_date DATETIME,

    certification_notes VARCHAR(MAX),

    certification_version VARCHAR(50),

    created_date DATETIME DEFAULT GETDATE(),

    flagged_validations NVARCHAR(MAX) NULL

);
```

# 4. dbo.tbl_users

## Purpose
Role-based access and workflow authorization.

```sql
CREATE TABLE dbo.tbl_users (

    user_email VARCHAR(255) PRIMARY KEY,

    user_name VARCHAR(255) NOT NULL,

    role VARCHAR(100) NOT NULL,

    is_active BIT DEFAULT 1,

    created_date DATETIME DEFAULT GETDATE()

);
```

## Role Constraint

```sql
ALTER TABLE dbo.tbl_users
ADD CONSTRAINT chk_user_role
CHECK (
    role IN (
        'Developer',
        'Client'
    )
);
```

# 5. dbo.tbl_user_workspaces

## Purpose
Maps client users to the workspaces they are authorised to access, and defines their action role within each workspace (Approver or Viewer).

```sql
CREATE TABLE dbo.tbl_user_workspaces (

    id INT IDENTITY(1,1) PRIMARY KEY,

    client_email VARCHAR(255) NOT NULL,

    workspace_name VARCHAR(500) NOT NULL,

    access_role VARCHAR(50) NOT NULL,

    is_active BIT DEFAULT 1,

    created_date DATETIME DEFAULT GETDATE()

);
```

## Access Role Constraint

```sql
ALTER TABLE dbo.tbl_user_workspaces
ADD CONSTRAINT chk_access_role
CHECK (
    access_role IN (
        'Approver',
        'Viewer'
    )
);
```

## Foreign Key — Client Email → Users

```sql
ALTER TABLE dbo.tbl_user_workspaces
ADD CONSTRAINT FK_user_workspaces_users
FOREIGN KEY (client_email)
REFERENCES dbo.tbl_users(user_email);
```

## Notes
- A client can have multiple rows — one per workspace they can access
- `access_role = 'Approver'`: can Approve and Reject reports in that workspace
- `access_role = 'Viewer'`: can view validation results only; no action buttons
- The app loads this table into `colMyWorkspaces` on screen load filtered by `User().Email` and `is_active = 1`
- Developers should NOT have rows here — having rows would not affect developer behaviour (guarded by `!IsDeveloper` in app logic) but keeping the table clean avoids confusion

# 6. dbo.tbl_report_history

## Purpose
Permanent audit/history table for lifecycle tracking.

```sql
CREATE TABLE dbo.tbl_report_history (

    history_id INT IDENTITY(1,1) PRIMARY KEY,

    report_id VARCHAR(100) NOT NULL,

    report_name VARCHAR(500) NOT NULL,

    from_status VARCHAR(100) NOT NULL,

    to_status VARCHAR(100) NOT NULL,

    changed_by VARCHAR(255) NOT NULL,

    changed_by_role VARCHAR(100) NOT NULL,

    changed_date DATETIME NOT NULL DEFAULT GETDATE(),

    notes VARCHAR(MAX) NULL

);
```

# Foreign Key Constraints

## Report History → Reports

```sql
ALTER TABLE tbl_report_history
ADD CONSTRAINT FK_history_report
FOREIGN KEY (report_id)
REFERENCES tbl_reports(report_id);
```

## Validation Results → Reports

```sql
ALTER TABLE dbo.tbl_validation_results
ADD CONSTRAINT FK_validation_results_reports
FOREIGN KEY (report_id)
REFERENCES dbo.tbl_reports(report_id);
```

## Certification → Reports

```sql
ALTER TABLE dbo.tbl_certification
ADD CONSTRAINT FK_certification_reports
FOREIGN KEY (report_id)
REFERENCES dbo.tbl_reports(report_id);
```

# Final Governance Lifecycle

Draft
↓
Under Review
↓
Validated
↓
Pending Client Approval
↓
Certified / Rejected

# Final Database Architecture

tbl_reports = Current Governance State

tbl_validation_results = Validation/Reconciliation Records

tbl_certification = Certification Records

tbl_users = Security & Role Management

tbl_user_workspaces = Client Workspace Access & Action Roles

tbl_report_history = Full Audit Timeline
