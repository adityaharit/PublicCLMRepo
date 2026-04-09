# Contract Lifecycle Management — MVP Product Narrative

---

## The Contract Management Problem

Most organisations manage contracts across a fragmented landscape: email threads, shared drives, approval chains that live in someone's inbox, and no reliable way to answer basic questions about their own agreements.

- *"Where is the signed MSA with Vendor X?"*
- *"Who approved this contract, and when?"*
- *"Which agreements are expiring in the next 60 days?"*
- *"Has anyone changed this document since it was approved?"*

The CLM platform exists to make these questions trivially answerable — and to make the governance behind every contract visible, auditable, and enforceable.

---

## The Contract Lifecycle: End to End

Every contract moves through five stages. The MVP delivers the highest-leverage four natively.

```text
┌────────────────┐    ┌─────────────────────┐    ┌──────────────────┐    ┌─────────────────┐    ┌────────────────┐
│  1. INTAKE     │───▶│ 2. GENERATION &     │───▶│ 3. GOVERNANCE &  │───▶│ 4. EXECUTION &  │───▶│ 5. REPOSITORY  │
│  & REQUEST     │    │    NEGOTIATION      │    │    APPROVALS     │    │    OBLIGATION   │    │ & REPORTING    │
└────────────────┘    └─────────────────────┘    └──────────────────┘    └─────────────────┘    └────────────────┘

MVP Scope:
  Intake              Template Builder           State Machine             [eSign deferred]    Searchable Library
  Document Upload     Token Substitution         Approval Workflows                            Vendor Dashboard
                      [Redlining in Word]        RBAC & Audit Log                              Expiry Tracking

Key Integrations:
  [Commercial Estimating] ──▶ Pre-fills contract metadata at intake
  [Procurement / PO System] ──▶ Triggered when contract reaches Executed
  [Palantir RAID Register] ◀── Receives governance exceptions and execution decisions from CLM
```

The platform handles Intake, Generation, Governance, and Repository natively. In-document negotiation and redlining remain in Microsoft Word for this release; final agreed versions are uploaded to the platform. E-signature integration is deferred — executed PDFs are uploaded post-signature.

## Product Foundations: Making the MVP Robust

To ensure the MVP delivers measurable business value and operates reliably from Day 1, the following foundations dictate how the system is built and measured.

### Contract Owner

The **Contract Owner** is the user who uploads a contract record to the platform. Ownership can be reassigned by the current owner or an Admin after creation. The Admin defines which user roles are eligible to hold contract ownership. The Contract Owner receives all state-change notifications, workflow rejection alerts, RAID resolution notifications, and approval summary emails for that contract.

### Target User Personas
- **The Viewer (Business Stakeholder):** Requires read-only access limited strictly to contracts they have been explicitly granted. Their goal is quick information retrieval without the risk of accidentally modifying or approving a document.
- **The Procurement Manager:** Needs to upload contracts, edit metadata, and generate standard agreements via a web form. Their goal is to initiate the contract lifecycle and trigger downstream Purchase Orders once execution is reached.
- **Legal / Legal Ops:** Needs to govern approved templates, initiate approval workflows, and manage lifecycle exceptions. Their goal is to maintain the organisation's legal language and enforce compliance. Template management (upload, version, archive) is available to the Legal role and to the Legal Ops Lead designation.
- **The Administrator:** Requires full access to manage user provisioning, configure approval workflows, and handle overrides (e.g., unlocking approved contracts). Their goal is to maintain system continuity and an immutable audit trail.

### Non-Functional Requirements (NFRs)
- **Performance:** The Contract Detail page must load within 3 seconds for documents up to 25MB. State transitions and notifications must execute within 60 seconds.
- **Security & Identity:** Access must be gated via SSO with session timeouts for inactivity. Role boundaries must be enforced at the server API level, not merely obscured in the UI.
- **Data Integrity:** The platform must maintain a permanently uneditable audit log across all roles.

### Risks & Mitigations
- **Risk:** Low user adoption if the system feels slower than email.
  **Mitigation:** Provide a focused "Approver Inbox" sorted by urgency and fast Out-of-Office (OOO) delegation.
- **Risk:** Integration failures with Palantir RAID drop critical governance data.
  **Mitigation:** Implement robust queuing and retry mechanisms for all API calls; fallback alerts logged in the Admin audit trail.
- **Risk:** "Shadow IT" redlining loses negotiation history.
  **Mitigation:** Mandate metadata tagging and immutable audit logs to ensure the final uploaded version is indisputably locked, even if drafted externally.

---

## Delivery Plan: Eight Self-Contained Capabilities

Each Epic delivers a usable, testable increment of the platform. No Epic leaves a partial product — every one closes with a distinct capability the organisation can operate.

| # | Epic | What You Have at the End | Depends On |
|---|---|---|---|
| 1 | Identity & Access Control | Secured platform, user roles enforced | — |
| 2 | Contract Repository | Store, find, and access all contracts | E1 |
| 3 | Contract Lifecycle & Audit | Tracked states, immutable audit trail | E1, E2 |
| 4 | Template Library & Builder | Generate standard contracts from approved templates | E1, E2 |
| 5 | Core Approval Workflows | Formal, configured approval chains | E1, E2, E3 |
| 6 | Approver Experience & Delegation | Managed inbox, OOO coverage, chain visibility | E1–E5 |
| 7 | Portfolio Dashboard | Live portfolio exposure and renewal planning | E1, E2, E3 |
| 8 | RAID Integration | Contract exceptions in the enterprise risk register | E1–E5 |

**Sizing:** 1 point = 1 hour of developer effort. Maximum 15 points per story. Maximum 200 points per Epic.

---

## Epic 1 — Identity & Access Control `36 pts`

#### Story 1.1 — Secure User Login `8 pts`
**User Story:** As a **Platform User**, I want to log in securely via SSO or email and password, so that I can reach my contracts without creating access risk for the organisation.
**Acceptance Criteria:**
- Login via SSO or email/password. Session expires after configurable inactivity; default timeout is 15 minutes; Admin can adjust the duration.
- Five consecutive failed login attempts locks the account; email notification sent to user. User can request unlock from Admin on the platform. Admin can reset login which also resets the failed attempt counter.
- Password reset via a time-limited email link that expires after 2 hours.
- Every login and logout recorded in the audit log with timestamp and IP address.
**Edge Cases:**
- *SSO Outage:* System falls back to email/password authentication if SSO provider times out.

#### Story 1.2 — Role-Based Permission Enforcement `10 pts`
**User Story:** As an **Admin**, I want the system to enforce strict role-based access limits, so that no one accidentally edits, approves, or shares a contract beyond their authority.
**Acceptance Criteria:**
- Viewer: read-only access to explicitly granted contracts only.
- Procurement: Viewer permissions + upload contracts and edit metadata.
- Legal: Procurement permissions + initiate approval workflows + manage and govern templates (upload, version, archive). Template management also available to the Legal Ops Lead designation.
- Admin: full access including user provisioning, workflow configuration, and system overrides.
- Forbidden actions are entirely hidden in the UI and blocked at the server API level — UI hiding alone is not sufficient.
**Edge Cases:**
- *Role Downgrade:* If a user's role is downgraded while logged in, their session automatically refreshes to enforce the new permissions immediately.

#### Story 1.3 — Contract-Level Permission Overrides `8 pts`
**User Story:** As a **Legal User or Owner**, I want to grant temporary View or Edit access to a specific colleague on a single contract, so that I can collaborate precisely and revoke access when done.
**Acceptance Criteria:**
- Owners with Legal role or above can add specific users as View or Edit collaborators. The Contract Owner is the user who uploaded the record; ownership can be reassigned by the current owner or Admin.
- Grants and revocations take effect immediately.
- All actions logged in the audit trail.
**Edge Cases:**
- *Deactivated Collaborator:* If a collaborator's account is deactivated, their contract-specific grants are gracefully ignored by the system.

#### Story 1.4 — User Account Provisioning & Off-boarding `6 pts`
**User Story:** As an **Admin**, I want to provision or deactivate user accounts, so that system access stays synchronised with employment status.
**Acceptance Criteria:**
- Admin creates accounts with name, email, and role; 48-hour invite link sent.
- Deactivated users cannot log in; their names persist on historical records.
- All account lifecycle events logged.
**Edge Cases:**
- *Expired Invite:* If an invite link expires, the system displays a clear message directing the user to request a new link from their Admin.

---

## Epic 2 — Contract Repository `44 pts`

#### Story 2.1 — Upload Contract Document `8 pts`
**User Story:** As a **Procurement User or above**, I want to upload a final agreement to the platform, so that it is accessible to colleagues and securely stored.
**Acceptance Criteria:**
- Drag-and-drop or file browser accepts PDF and DOCX formats only.
- Maximum file size is 25 MB; Admin can configure this limit.
- File type and size validated client-side before upload begins; clear error message on failure.
- Progress bar displayed during upload.
- Successfully uploaded files are attached to the contract record and the upload action is logged in the audit trail.
**Edge Cases:**
- *0-Byte or Malware File:* System rejects files that are 0 bytes or fail basic security scanning, displaying a distinct error.

#### Story 2.2 — Contract Metadata Capture `8 pts`
**User Story:** As a **Procurement User or above**, I want to attach structured metadata to a contract, so that agreements are consistently tagged and searchable.
**Acceptance Criteria:**
- Required fields: Name, Type, Status (defaults to Draft), Counterparty, Start Date, Total Value.
- Optional fields: End Date, Currency.
- Contract Type is selected from an Admin-configured list; default list is NDA, MSA, SOW, Employee Contract, Other Contract. Admin can add or remove values at any time.
- Status defaults to Draft on record creation and cannot be set directly to any other state by the user.
- Every metadata edit is individually logged: field name, old value, new value, user, and timestamp.
**Edge Cases:**
- *Chronology Error:* Form validation blocks submission if End Date is chronologically before Start Date.

#### Story 2.3 — Contract Library Search & Sorting `10 pts`
**User Story:** As a **Platform User**, I want to search and filter my accessible contracts, so that I can find specific documents within seconds.
**Acceptance Criteria:**
- Full-text real-time search across Name and Counterparty fields.
- Combinable filters (Type, Status, Counterparty, Date Range) displayed as removable tags.
- Sortable columns.
- Only role-permitted or explicitly granted contracts are visible.
**Edge Cases:**
- *Special Characters:* Search input sanitises complex regex strings to prevent query errors or slow database loads.

#### Story 2.4 — Parent-Child Contract Linking `6 pts`
**User Story:** As a **Procurement User or above**, I want to link new amendments or renewals to a parent contract, so that the full family of agreements is visible in one place.
**Acceptance Criteria:**
- Any contract can be designated a child (Amendment, Renewal, Addendum).
- Parent shows "Related Agreements" list; child shows back-link to parent.
- Linking is audited.
**Edge Cases:**
- *Circular Links:* System prevents a contract from being linked as its own parent, or forming an infinite loop (A -> B -> A).

#### Story 2.5 — Contract Detail View `12 pts`
**User Story:** As a **Platform User**, I want a single unified page showing the document, metadata, and lifecycle status, so that I can act on the contract without navigating multiple screens.
**Acceptance Criteria:**
- Displays metadata, in-browser PDF preview / DOCX download, and status badge.
- Shows valid state transition buttons based on RBAC.
- Displays state history, audit log, collaborators, and related agreements.
- Loads in <3 seconds.
**Edge Cases:**
- *Corrupted Preview:* If the PDF viewer fails to render, a fallback "Download Document" button remains functional.

---

## Epic 3 — Contract Lifecycle & Audit `48 pts`

#### Story 3.1 — The Six-State Lifecycle `8 pts`
**User Story:** As the **System**, I want to enforce strict lifecycle phases (Draft → In Review → Approved → Executed → Expired), so that no contract bypasses required stages.
**Acceptance Criteria:**
- Linear path enforced.
- Invalid transitions blocked and hidden in UI.
- Terminated/Cancelled reachable from any active state.
- All state changes logged (from-state, to-state, user, time).
**Edge Cases:**
- *Concurrency:* If two authorised users transition a state simultaneously, the system processes the first and returns a "State already updated" message to the second.
**Open Question (P0-5):** Which role triggers the Approved → Executed transition? Is it a manual button action by a specific role, or is it triggered automatically by an external signal (e.g., PO system confirmation)? *Owner: S-AI*

#### Story 3.2 — Terminated / Cancelled State `6 pts`
**User Story:** As a **Legal Ops User or Admin**, I want to transition a contract to Terminated/Cancelled with a recorded reason, so that I can remove it from active workflows without losing its history.
**Acceptance Criteria:**
- Reachable from Draft, In Review, Approved, or Executed.
- Mandatory reason note (min 20 chars) required.
- Record locks immediately on transition.
**Edge Cases:**
- *Already Expired:* System prevents Terminated/Cancelled transitions on contracts already in the Expired terminal state.

#### Story 3.3 — Automatic Expiry on End Date `6 pts`
**User Story:** As the **System**, I want to transition contracts to Expired automatically when their end date passes, so that the portfolio is always accurate without manual tracking.
**Acceptance Criteria:**
- Daily CRON job runs at midnight UTC, checking all contracts in Executed state.
- Contracts with End Date on or before today's UTC date are automatically transitioned to Expired.
- Contract owner receives both an in-app notification and an email on automatic expiry; the notification includes the contract name, expiry date, and a direct link.
**Edge Cases:**
- *Job Failure:* If the daily job fails, it automatically retries every hour until successful.

#### Story 3.4 — Lock Approved and Closed Contracts `6 pts`
**User Story:** As an **Auditor/System**, I want Approved, Executed, Terminated, and Expired contracts permanently locked against edits, so that agreed terms are preserved intact.
**Acceptance Criteria:**
- All metadata fields and document edit controls are locked immediately upon reaching Approved, Executed, Terminated/Cancelled, or Expired state.
- Admin can unlock any locked contract back to Draft state; a mandatory justification text is required before the unlock is saved.
- The unlock justification is permanently and immutably recorded in the audit log alongside the Admin's identity and timestamp.
**Edge Cases:**
- *Admin Unlock Alerts:* Unlocking a contract triggers an automatic email to Legal Ops and the contract owner to prevent silent administrative overrides.

#### Story 3.5 — Stakeholder Notifications on State Change `6 pts`
**User Story:** As a **Contract Owner**, I want automatic notifications when my contract changes state, so that I can keep workflows moving efficiently.
**Acceptance Criteria:**
- In-app and email notification sent within 60s of state change.
- Includes previous/new state, actor, and direct link.
**Edge Cases:**
- *Email Bounce:* If the email provider fails, the in-app notification is still reliably delivered.

#### Story 3.6 — Immutable Contract Audit Log `10 pts`
**User Story:** As a **Compliance Officer**, I want an uneditable, permanent log of every contract action, so that I can demonstrate compliance during audits.
**Acceptance Criteria:**
- Logs the following action types: create, upload, edit (field-level), state transition, approval decision, collaborator grant/revoke, document download. View-only page loads are explicitly excluded to prevent log noise.
- Each log entry is written in plain language using the format: `[User] [action] on [contract/field] — changed from [old value] to [new value] — [timestamp UTC]`.
- The audit log is read-only for all roles including Admin; no entry can be edited or deleted.
- Log is filterable by date range and action type; exportable to CSV with columns: Contract ID, Contract Name, Type, Counterparty, Action, Changed Field, Old Value, New Value, User, Timestamp (UTC).
**Edge Cases:**
- *Massive Export:* If a global log export exceeds standard row limits, the system paginates the CSV or delivers it asynchronously via email link.

---

## Epic 4 — Template Library & Contract Builder `54 pts`

#### Story 4.1 — Version-Controlled Template Library `8 pts`
**User Story:** As **Legal Ops**, I want to manage version-controlled contract templates, so that the business always uses current approved language.
**Acceptance Criteria:**
- Upload DOCX template with name, version label, and category selected from an Admin-managed fixed list; templates without a category are listed under "Uncategorized."
- Uploading a new version automatically archives the previous version; the archived version is still viewable but unavailable for generation.
- Only active (non-archived) templates are available in the template library for contract generation.
**Edge Cases:**
- *In-Flight Generation:* If Legal Ops archives a template while a user is filling out its form, the user can finish, but a warning is logged on the contract record.

#### Story 4.2 — Browse & Preview Templates `6 pts`
**User Story:** As a **Business User**, I want to browse and preview active templates, so that I choose the correct legal form for my transaction.
**Acceptance Criteria:**
- Grouped by category, real-time search.
- Previews document alongside metadata.
**Edge Cases:**
- *Preview Failure:* Displays a generic placeholder and "Download to Preview" button if rendering engine fails.

#### Story 4.3 — Define Template Variables `8 pts`
**User Story:** As **Legal Ops**, I want to embed variables (e.g. `{{vendor_name}}`) in templates, so that the system can automatically build data-entry forms.
**Acceptance Criteria:**
- Extracts `{{variable}}` placeholders on upload.
- Displays extracted list for author verification.
- Flags malformed placeholders as warnings.
**Edge Cases:**
- *Nested Variables:* The system blocks template upload with a clear error message if complex or nested bracket syntax is detected (e.g., `{{var_{{name}}}}`). Silent ignoring is not permitted.

#### Story 4.4 — Generate a Contract from a Template `12 pts`
**User Story:** As a **Procurement User**, I want to fill in a web form to generate a contract, so that I produce accurate, compliant documents in minutes.
**Acceptance Criteria:**
- Form dynamically generated based on template variables.
- Live preview shows where data lands.
- On submit: substitutes variables, creates PDF/DOCX, creates Draft record.
**Edge Cases:**
- *Generation Timeout:* If document compilation takes >30s, the UI shows a friendly loading state rather than a crash screen.
**Open Question (P2-5):** What are the required metadata fields for template upload? *Owner: Product*

#### Story 4.5 — Builder Validation & Error Handling `8 pts`
**User Story:** As a **Business User**, I want instant validation on my form inputs, so that I can fix errors before generation fails.
**Acceptance Criteria:**
- Real-time inline validation (dates, currencies, required fields).
- Failed generation preserves form entries so work isn't lost.
**Edge Cases:**
- *Extreme Inputs:* Numeric currency fields enforce maximum limits to prevent integer overflow errors in the database.

## Epic 5 — Core Approval Workflows `62 pts`

#### Story 5.1 — Configure Sequential & Parallel Workflows `10 pts`
**User Story:** As an **Admin**, I want to build approval routing templates, so that specific contract types route correctly and automatically.
**Acceptance Criteria:**
- Build named workflow templates with ordered stages; each stage is designated as sequential or parallel.
- Workflow templates are associated with specific contract types; one active template per contract type at a time.
- A workflow template is active if at least one in-flight contract is currently using it; active templates cannot be edited and require a new version to be created. Inactive templates (no in-flight contracts) can be edited directly.
- Each stage must have at least one approver assigned before the template can be saved.
**Edge Cases:**
- *Empty Stage:* The UI prevents saving a workflow template if a stage is created but no approvers are assigned.

#### Story 5.2 — Approval Stage SLA Configuration `6 pts`
**User Story:** As an **Admin**, I want to set SLAs on approval stages, so that the system tracks bottlenecks automatically.
**Acceptance Criteria:**
- SLA duration is configurable per stage and is measured in business hours (not calendar hours); e.g., a value of 48 means 48 business hours.
- Stages that exceed their SLA are visually marked with an amber warning indicator.
- SLA breach triggers an in-app and email notification to the contract owner identifying the blocked stage and overdue party.
**Edge Cases:**
- *0-Day SLA:* Form validation requires SLAs to be >0.

#### Story 5.3 — Submit a Contract for Approval `8 pts`
**User Story:** As a **Legal User**, I want to trigger the approval chain from the contract page, so that stakeholders are formally engaged.
**Acceptance Criteria:**
- "Submit for Approval" button for Legal/Admin roles.
- Verifies workflow exists for contract type.
- Transitions to In Review and notifies first approvers.
**Edge Cases:**
- *No Workflow Configured:* Blocks submission with a clear error: "No approval workflow configured for this Contract Type. Contact Admin."

#### Story 5.4 — Approve or Reject with Mandatory Comment `8 pts`
**User Story:** As an **Approver**, I want to record my decision in the platform, so that my sign-off or objection is officially captured.
**Acceptance Criteria:**
- Approve records decision immediately.
- Reject requires mandatory text comment.
- Decisions written to audit log.
**Edge Cases:**
- *Role Change Mid-Flight:* If an approver loses their role while a contract is pending in their queue, the system flags the stage as "Unassigned" for Admin reassignment.

#### Story 5.5 — Automatic Chain Advancement `12 pts`
**User Story:** As the **System**, I want to automatically advance contracts to the next stage upon approval, so that the workflow never stalls manually.
**Acceptance Criteria:**
- Sequential: completion of one stage immediately and automatically triggers the next.
- Parallel: all approvers in the stage must approve before the chain advances. A single rejection from any parallel approver immediately halts the entire chain; the contract reverts to Draft and the owner and submitter are notified with the rejection comment. The owner must add a mandatory re-initiation comment before the chain can be restarted from Stage 1.
- Final stage approval automatically transitions the contract to Approved state with no manual action required.
**Edge Cases:**
- *Simultaneous Parallel Action:* If two parallel approvers reject at the exact same microsecond, both rejection comments are preserved in the audit log.

#### Story 5.6 — Rejection & Resubmission Flow `10 pts`
**User Story:** As a **Contract Owner**, I want to be notified of rejections and easily resubmit, so that I can resolve issues without losing the historical chain.
**Acceptance Criteria:**
- Owner notified within 60s of rejection with full comment.
- Contract reverts to Draft.
- Prior approval records retained in audit.
- Resubmitting restarts chain from Stage 1.
**Edge Cases:**
- *No Document Change:* The UI allows resubmission even if the document wasn't changed, as the blocker may have been resolved off-system (e.g., a budget clearance).

---

## Epic 6 — Approver Experience & Delegation `46 pts`

#### Story 6.1 — Approver Inbox & Priority Queue `8 pts`
**User Story:** As an **Approver**, I want a personal inbox of pending tasks sorted by urgency, so that I can manage my workload efficiently.
**Acceptance Criteria:**
- Live count badge in navigation.
- Shows contract details, submitter, and SLA due date.
- Sorted by SLA (urgent first).
**Edge Cases:**
- *High Volume:* Inbox implements lazy loading/pagination if a user has >50 pending approvals.
**Assumption (P2-6):** When SLA due dates are equal, secondary sort is Contract Name A–Z, then oldest submission date first.

#### Story 6.2 — Approval Chain Visibility `8 pts`
**User Story:** As a **Stakeholder**, I want to view the real-time status of the approval chain, so that I know who is currently blocking progress.
**Acceptance Criteria:**
- Visual timeline on contract detail page.
- Highlights active stage and groups parallel stages.
**Edge Cases:**
- *Extremely Long Chains:* UI handles chains of 10+ stages via horizontal scrolling or clean vertical wrapping.

#### Story 6.3 — Final Approval Locks Contract & Notifies Owner `6 pts`
**User Story:** As a **Contract Owner**, I want immediate notification and locking upon final approval, so that I can confidently proceed to execution.
**Acceptance Criteria:**
- Auto-transitions to Approved.
- Locks against edits.
- Sends summary email of the full approval chain.
**Edge Cases:**
- *Self-Approval:* If the final approver happens to be the contract owner, the system still formally logs the approval and sends the completion summary.

#### Story 6.4 — Out-of-Office Delegation `12 pts`
**User Story:** As an **Approver**, I want to designate an OOO deputy, so that my queue does not bottleneck while I am on leave.
**Acceptance Criteria:**
- Select delegate and date range.
- Auto-redirects requests during period.
- Audit trail explicitly notes delegated actions ("User B on behalf of User A").
- No nested sub-delegation allowed.
**Edge Cases:**
- *Circular Delegation:* System prevents User A from delegating to User B if User B has already delegated to User A.
**Open Question (P2-7):** If delegate User B also goes OOO and cannot sub-delegate, who resolves stalled approvals in their queue — Admin manual reassignment (Story 6.5), an automatic escalation alert, or another mechanism? *Owner: Client*

#### Story 6.5 — Admin Reassignment of Stuck Approvals `6 pts`
**User Story:** As an **Admin**, I want to reassign stuck approvals manually, so that workflows can bypass sudden employee departures.
**Acceptance Criteria:**
- Admin overrides pending approver with a new user.
- Requires mandatory justification text.
- Logs original, new user, admin, and reason.
**Edge Cases:**
- *Invalid Target:* System prevents reassigning to a user who has *already* approved a previous stage in the same chain.

---

## Epic 7 — Portfolio Dashboard `32 pts`

#### Story 7.1 — Vendor Exposure Dashboard `8 pts`
**User Story:** As an **Executive**, I want to see aggregated contract data grouped by counterparty, so that I can instantly assess supplier liability.
**Acceptance Criteria:**
- Summary cards per counterparty (count, total value, status).
- Click navigates to filtered library.
- Honors RBAC visibility.
**Edge Cases:**
- *Massive Counterparty List:* Dashboard paginates or lazy-loads if the user has access to >100 counterparties.

#### Story 7.2 — Contract Status Summary `6 pts`
**User Story:** As a **Legal Leader**, I want a visual count of contracts in each lifecycle state, so that I can monitor portfolio health.
**Acceptance Criteria:**
- Status summary chart and data table.
- Clickable states navigate to filtered library.
- Updates in real-time.
**Edge Cases:**
- *Zero States:* States with zero contracts still display on the chart so the visual structure remains consistent.

#### Story 7.3 — Expiry Timeline `8 pts`
**User Story:** As **Legal Ops**, I want an automated tracker for expiring contracts, so that I can initiate renewals proactively.
**Acceptance Criteria:**
- Contracts bucketed by days to expiry: 0–30, 31–60, 61–90.
- Filterable by counterparty and contract type within the timeline view.
- Contracts with no end date are excluded from the timed buckets but are surfaced in a dedicated "No End Date" filter so that open-ended obligations are never silently invisible.
**Edge Cases:**
- *Past Due Active Contracts:* Contracts that have technically passed their expiry date but remain in an active state (e.g., Executed) are flagged in red in the 0–30 day bucket.

#### Story 7.4 — Dashboard Filters & Export `10 pts`
**User Story:** As a **Business User**, I want to filter and export dashboard views, so that I can share accurate reports with external stakeholders.
**Acceptance Criteria:**
- Global filters update all dashboard widgets simultaneously.
- Export to CSV or PDF (with user name, date, active filters).
- Export action is audited.
**Edge Cases:**
- *Empty Export:* If filters yield zero results, the export button is disabled to prevent generating blank PDFs.

---

## Epic 8 — RAID Integration `34 pts`

#### Story 8.1 — Approval SLA Breach Creates RAID Issue `8 pts`
**User Story:** As the **System**, I want to post an Issue to Palantir RAID when an SLA is breached, so that governance blockages escalate to leadership.
**Acceptance Criteria:**
- A RAID Issue is created when an approval stage exceeds its configured SLA duration; the trigger threshold equals the configured SLA and is not a hardcoded value.
- The Issue payload includes: contract name, overdue approver, SLA due date, and a direct link to the contract.
- One Issue is created per breach incident; if the breach resolves and the SLA is subsequently exceeded again, a new Issue is created for the new incident.
- If the Palantir API call fails, the system retries up to 5 times at 15-minute intervals. After 5 failures, an escalation alert is raised to the Admin; a reminder alert is sent after 48 business hours if the integration remains unresolved.
**Edge Cases:**
- *API Timeout:* Each failed attempt is individually logged as a "System Integration Failure" entry in the Admin audit trail.

#### Story 8.2 — Contract Execution Creates RAID Decision `6 pts`
**User Story:** As the **System**, I want to record major contract executions as a RAID Decision, so that the wider team receives an official signal to proceed.
**Acceptance Criteria:**
- Executed contracts above a configurable value threshold create a RAID Decision.
- Includes counterparty name, contract value, execution date, and link.
**Edge Cases:**
- *Missing Value:* If the executed contract lacks a total value (e.g., zero-dollar NDA), it bypasses this specific RAID creation logic unless overridden by Admin settings.

#### Story 8.3 — Approaching Expiry Without Renewal Creates RAID Risk `6 pts`
**User Story:** As the **System**, I want to raise a RAID Risk for contracts 30 days from expiry without a renewal, so that unmanaged exposure is highlighted.
**Acceptance Criteria:**
- A RAID Risk is created when a contract reaches 30 days before its End Date and has no linked child "Renewal" record.
- Linking a Renewal record to the contract automatically closes the open RAID Risk.
- If the contract remains unrenewed, escalation reminder emails are sent to the contract owner (CC: Admin) at 20 days, at 10 days, and on the expiry date itself. The final email explicitly states that no further automatic reminders will be sent.
**Edge Cases:**
- *Terminated Contracts:* Contracts manually moved to Terminated/Cancelled bypass the expiry risk logic entirely.

#### Story 8.4 — Admin Contract Unlock Creates RAID Risk `6 pts`
**User Story:** As the **System**, I want to create a RAID Risk when an Admin unlocks a locked contract, so that post-approval modifications are visible to leadership.
**Acceptance Criteria:**
- Unlocking any locked contract (Approved, Executed, Terminated/Cancelled, or Expired) back to Draft state automatically creates a RAID Risk in Palantir.
- The RAID Risk payload includes: Admin name, justification text entered at unlock, and the date of the action.
**Edge Cases:**
- *Immediate Relock:* Even if the Admin relocks the contract 5 seconds later, the RAID Risk persists for audit review.

#### Story 8.5 — RAID Resolution Notifies CLM Contract Owner `8 pts`
**User Story:** As a **Contract Owner**, I want to be notified when my linked RAID item is resolved, so that I can unblock my workflow.
**Acceptance Criteria:**
- Palantir sends a webhook to a CLM-hosted endpoint when a linked RAID item is resolved; CLM processes the webhook and delivers both an in-app and email notification to the contract owner within 60 seconds of receipt.
- The notification includes the resolution note text and a direct link to the relevant contract.
**Edge Cases:**
- *Reopened RAID Item:* If a resolved item is reopened in Palantir, the CLM contract owner receives a corresponding "RAID Item Reopened" alert via the same webhook mechanism.

