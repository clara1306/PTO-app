<!--
 @generated
 @context Define an implementation-ready specification for a simple standalone BMC Helix planned-time-off application.
 @decisions Scope v1 to full-day paid-time-off requests with a single manager approval step and no balance accounting.
 @references cookbook/13-requirement-gathering.md, cookbook/23-requirements-analysis.md, .cursor/rules/bmc-helix-requirements-analysis.mdc
 @modified 2026-10-01
-->

# PTO application specification

## Goals

Build a standalone BMC Helix Innovation Studio application that lets employees request planned paid time off (PTO) and lets their managers approve or reject those requests.

The first release must provide a reliable, auditable workflow for full-day PTO requests. It must make the current request status visible to both the employee and the assigned manager.

## Target users

- **Employee**: creates, views, and withdraws their own PTO requests.
- **Manager**: reviews pending requests submitted by their direct reports and approves or rejects them.
- **PTO administrator**: maintains employee-to-manager assignments and can view all requests for operational support.

Authentication uses the existing BMC Helix platform session. The application must not introduce its own login system.

## User stories

- As an employee, I can submit a planned PTO request with a start date, end date, and optional reason.
- As an employee, I can see the status and decision details for my own requests.
- As an employee, I can withdraw a request while it is pending.
- As a manager, I can see pending PTO requests from employees assigned to me.
- As a manager, I can approve or reject a pending request and include an optional decision note.
- As a PTO administrator, I can maintain the manager assignment used for approval routing.
- As a PTO administrator, I can view all PTO requests without changing employee or manager ownership.

## Features

### Request lifecycle

1. An employee creates a request in **Draft** status.
2. The employee submits the draft. The system validates it, resolves the employee's assigned manager, and changes the status to **Pending approval**.
3. The assigned manager approves or rejects the pending request.
4. The employee may withdraw a pending request. Withdrawn requests cannot be approved or rejected.

Allowed statuses:

- `Draft`
- `Pending approval`
- `Approved`
- `Rejected`
- `Withdrawn`

The application records who submitted, approved, rejected, or withdrew a request and when the action occurred. Status changes are append-only audit events; historical decision data must not be overwritten.

### Business rules

- Version 1 supports only full-day PTO requests.
- Start and end dates are inclusive and must be date-only values in the employee's configured time zone.
- The end date must be on or after the start date.
- Requests cannot begin in the past.
- An employee cannot submit a request that overlaps their own pending or approved request.
- Submission is blocked when no active manager is assigned to the employee.
- Only the assigned manager can approve or reject a request.
- A manager cannot approve or reject their own request, even if an invalid assignment exists.
- No PTO balance, accrual, entitlement, payroll calculation, holiday calculation, or calendar integration is included in version 1.

### Notifications

The application displays confirmation and error messages in the UI. In-app or email notifications are optional for version 1 and must not be required for the approval workflow to complete.

## Screens / UI

Use Angular 18 standalone components with OnPush change detection and BMC Adapt UI components. All displayed text must be localized.

### My PTO requests

An employee-facing list showing only the signed-in employee's requests. Include:

- New request action
- Status, start date, end date, submitted date, and manager decision columns
- Filters for status and date range
- Empty, loading, and safe error states
- A request-details link or action

### PTO request form

An employee-facing create/edit screen for drafts with:

- Start date
- End date
- Optional reason, limited to 1,000 characters
- Read-only request status and assigned manager once submitted
- Save draft, submit, and withdraw actions as allowed by the current status
- Inline validation explaining invalid dates, overlaps, and missing manager assignments

### Manager approval queue

A manager-facing list of only their direct reports' pending requests. Include:

- Employee name or login identifier
- Requested date range and reason
- Submitted date
- Approve and reject actions
- Optional decision-note input, limited to 1,000 characters
- Confirmation before recording a decision

### Request details

An employee, assigned manager, or administrator can view request details subject to authorization. The screen presents the date range, reason, lifecycle status, decision note, and audit history.

### PTO administration

An administrator-only screen to create and maintain employee-to-manager assignment records. It must provide search, create, edit, deactivate, and validation for distinct employee and manager identities.

## Data model

Create the following Regular Record Definitions. Exact FQNs and field IDs must be proposed, reviewed, and approved before `.def` generation.

### PTO employee assignment

Stores the approval relationship used for routing.

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| Employee login ID | String | Yes | Unique platform user identifier. |
| Manager login ID | String | Yes | Platform user identifier of the direct manager. |
| Active | Boolean | Yes | Defaults to true. |
| Effective from | Date | Yes | Date on which the assignment is usable. |
| Effective to | Date | No | Optional end date; must not precede Effective from. |
| Description | String | No | Administrative note. |

Only administrators may create or modify assignment records. An implementation may replace this record with an approved, authoritative Helix employee/manager source only after the integration and field mapping are explicitly specified.

### PTO request

Stores one employee's requested planned absence.

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| Request number | String | Yes | System-generated, immutable, human-readable identifier. |
| Employee login ID | String | Yes | Derived from the authenticated user; never accepted as an arbitrary client value. |
| Manager login ID | String | Yes after submission | Resolved from the active assignment and frozen for the request lifecycle. |
| Start date | Date | Yes | Date-only, inclusive. |
| End date | Date | Yes | Date-only, inclusive. |
| Reason | String | No | Maximum 1,000 characters. |
| Status | Enum | Yes | Draft, Pending approval, Approved, Rejected, Withdrawn. |
| Submitted at | Date/time | No | Stored as epoch milliseconds. |
| Submitted by | String | No | Authenticated user ID. |
| Decision at | Date/time | No | Stored as epoch milliseconds. |
| Decision by | String | No | Authenticated manager user ID. |
| Decision note | String | No | Maximum 1,000 characters. |
| Withdrawal at | Date/time | No | Stored as epoch milliseconds. |
| Withdrawal by | String | No | Authenticated employee user ID. |
| Status history | Child or related record | Yes | Immutable audit events: status, actor, time, optional note. |

Use the platform's required Status and Description fields where applicable. The implementation must identify the correct record schema shape from the record-definition cookbook; it must not invent `.def` JSON.

## API requirements

Expose APIs or process actions only after selecting the appropriate platform pattern from the BMC Helix cookbook. The client must not write record instances directly in ways that bypass authorization and business validation.

### Required operations

- `createDraft`: creates a draft owned by the authenticated employee.
- `updateDraft`: updates dates and reason only when the authenticated employee owns a draft.
- `submitRequest`: validates the draft, resolves the manager, checks for overlaps, and creates an audit event.
- `listMyRequests`: returns paginated requests owned by the authenticated employee.
- `getRequest`: returns a request only to its employee, assigned manager, or administrator.
- `withdrawRequest`: allows the owning employee to withdraw a pending request.
- `listManagerQueue`: returns paginated pending requests only for the authenticated manager.
- `decideRequest`: lets only the assigned manager approve or reject a pending request.
- `manageAssignment`: administrator-only create, update, list, and deactivate operations for employee-manager assignments.

### API security and validation

- Every REST endpoint and process activity has `@AccessControlledMethod(authorization = ValidUser, licensing = Application)`.
- Mutations verify ownership, manager assignment, or administrator authorization server-side; client-side visibility is not authorization.
- Truncate free-text inputs before use and validate date ranges and status transitions.
- Use `escapeQual()` for every user-controlled value inserted into an AR System qualification.
- Paginated queries are bounded to a maximum page size of 1,000; never use `DataPage.INFINITE_PAGE_SIZE`.
- Return generic, user-safe error messages. Log server-side exceptions without secrets or full credential-bearing request bodies.
- Do not log PTO reasons unless an approved audit policy requires it.

## AI prompt requirements

Any Cursor agent implementing this specification must:

1. Treat this document as the functional scope for version 1 and identify any contradiction before coding.
2. Read the BMC Helix requirements-analysis guidance first, present a concrete design, run the Grill Me checkpoint, and wait for explicit user approval before generating Helix code, components, records, processes, or definitions.
3. Read the applicable cookbook sections before each artifact type: record definitions, view components, Java services, and process definitions as applicable.
4. Propose exact record FQNs, custom fields, API names, component type strings, and process topology for approval before generating `.def` files.
5. Use `src/lib/actions/` for view actions; do not create new actions in legacy `view-actions/`.
6. Add required generation-context headers and co-located vibe-coding transcripts to substantive implementation artifacts.
7. Never hardcode credentials, tokens, manager mappings, URLs, or secrets.
8. Confirm the target server version and align SDK versions before deployment. Do not copy the existing `on-duty-mgmt` SDK version without verifying the target environment.
9. Before any build or deployment, complete the repository's mandatory pre-build review. Stop for critical issues or critical/high security findings.
10. Deploy Java services before process or record definitions that depend on them, and verify the deployed result with bounded, authorization-aware smoke tests.

## Non-functional requirements

### Security and privacy

- Enforce least-privilege access: employees see only their own data; managers see only requests assigned to them; administrators use an explicit administrator permission.
- Treat PTO reasons and dates as personal information. Restrict access, avoid unnecessary logging, and use existing tenant retention and audit policies.
- Preserve immutable status-history events and record the authenticated actor for every state transition.

### Performance and reliability

- Lists must be paginated and load a maximum of 100 records by default.
- The UI must show a loading state and a recoverable error state for every remote operation.
- Duplicate submission must be prevented through server-side status checks and idempotent handling where feasible.
- Timeouts, service failures, and concurrent manager decisions must not leave a request in an invalid status.

### Accessibility and usability

- Forms must have labels, keyboard-accessible controls, and actionable validation messages.
- Dates must be displayed using the user's Helix locale while stored in the platform-supported representation.
- UI controls must visibly reflect whether the current user can perform the action.

### Maintainability and observability

- Use localized UI strings, `RxLogService` for frontend errors, and server-side structured logging for failures and authorization denials.
- Add automated tests for validation, authorization, status transitions, and overlap detection.
- Include API/process smoke tests and an end-to-end employee-to-manager approval scenario.

## Tech stack

- **Platform**: BMC Helix Innovation Studio coded application.
- **Frontend**: Angular 18, TypeScript 5.5, RxJS 7.8, and BMC Adapt UI.
- **Backend**: Java 17, OSGi, JAX-RS/Spring services, and BMC Helix process activities where workflow orchestration adds value.
- **Definitions**: Regular Record Definitions for PTO requests and assignments; a BPMN process definition only if the approved design uses process orchestration.
- **Testing**: Unit tests for Java and Angular behavior, plus Playwright end-to-end coverage for the request/approval path.
- **Deployment**: Docker or Podman development container; Maven deployment using environment-supplied BMC credentials.

Before deployment, query and verify the target Helix server version, then align `rx-sdk.version`, `rx-sdk-plugin.version`, and local SDK installation with that server. Repository guidance currently documents a 26.1.00 target environment, while an unrelated existing application declares 26.2.00-SNAPSHOT; this must be resolved for the new PTO project before its first build.

## Success criteria

- A separate PTO application project exists and can be built and deployed against a verified, SDK-aligned target server.
- An authenticated employee can create a full-day date-range request, save it as a draft, and submit it.
- Invalid dates, past starts, missing manager assignments, and overlapping pending/approved requests are rejected with safe, useful messages.
- A submitted request routes to the employee's configured manager and appears only in that manager's queue.
- The manager can approve or reject a pending request, and the employee can see the final status and decision note.
- An employee cannot read or alter another employee's request.
- A manager cannot decide requests that are not assigned to them.
- An administrator can maintain employee-manager assignments and view all requests.
- Every lifecycle transition produces an immutable audit event with actor and timestamp.
- Automated tests and the employee-to-manager approval smoke test pass.
- The pre-build review passes with no unresolved critical issues or critical/high security findings.

## Out of scope

- PTO balance tracking, accruals, carry-over, and entitlement calculations.
- Vacation, sick, personal, unpaid, or other separately governed leave categories.
- Partial-day, hourly, or recurring leave.
- Multi-level, delegated, or conditional approval flows.
- Payroll, HRIS, timekeeping, or external calendar integration.
- Email, Teams, Slack, or push notification delivery.
- Holiday calendars, regional work schedules, and working-day calculations.
- Mobile-native application delivery.
- Historical data migration from another PTO system.
