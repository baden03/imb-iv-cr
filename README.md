# IMB Intervision (IV) Module Change Request 
## System-wide Date Normalization & IV Enhancements – Concept & Current State

**Status:** Agreed direction with open questions  
**Audience:** Client, chamber representatives, implementation team  
**Scope:** System-wide (IMB core, IV module, Fortbildung, reporting)  
**Key decision:** Unix timestamp storage for all date/time values

---

## 1. Context & Overall Direction

During ongoing development and operation of the IMB system, multiple structural limitations were identified around:

- date and time handling
- participant identity consistency
- historical attendance and point reporting
- auditability and legal retention requirements
- UI scalability for long-running IV groups

Following client feedback, a **system-wide decision** has been made:

> **All date and time values in the IMB system will be stored as Unix timestamps.**

This change goes beyond the IV module and therefore requires:
- a separate project scope
- a controlled migration
- a temporary system freeze during production rollout

The IV-related changes described below are designed **with this new timestamp-based foundation as a prerequisite**.

---

## 2. System-wide Date Normalization (Foundational Change)

### 2.1 Canonical Storage
- All dates and times are stored internally as **Unix timestamps (UTC)**.
- This applies system-wide:
  - IV meetings
  - Fortbildung events
  - reporting ranges
  - point calculations
  - any other date-based logic

### 2.2 Display Format
- Dates are displayed in the UI using the German format:
  - `dd.mm.yyyy` (and time where applicable)
- Display formatting is purely presentational and never used for logic.

### 2.3 Migration Strategy
- All changes are developed and tested on a **development/staging system**.
- Production rollout:
  1. System freeze (no reads, no writes) late night / early morning
  2. Deployment of updated code
  3. Execution of migration scripts
  4. Verification tests
  5. System unfreeze

This migration is treated as a **separate project**, with its own plan and risk management.

---

## 3. Roles & Terminology

### IMB Member
- Has an IMB user account (`user_id`)
- Eligible for automatic point calculation
- Can become inactive (not deleted)

### Visiting Participant (Non-IMB)
- No IMB user account
- External participant (other chambers, Ärztekammer, etc.)
- Attendance tracked, but no automatic IMB points

### IV Participant Record
- Represents a concrete group membership
- Status:
  - pending
  - approved
  - inactive / archived
- Approval is **per IV group**, not global

### Group Coordinator (GK)
- Manages group composition
- Submits participants
- **Cannot modify participants after approval**

### Chamber / Chamber Admin
- Approves participants
- Manages name changes for visiting participants
- Can generate historical reports
- Responsible for audit and compliance

---

## 4. Participant Immutability & Name Changes

### 4.1 General Rule
Once a participant is **approved**, **no changes** may be made by Group Coordinators.

This applies to:
- IMB members
- visiting participants
- all participant data fields

### 4.2 IMB Members
- Name changes are handled exclusively via the **IMB name-change workflow**.
- Once approved:
  - the updated name is reflected everywhere (current and historical views).
- Logs remain unchanged and record the original name at the time of each action.

### 4.3 Visiting Participants
- Name changes may **only** be performed by:
  - chamber / chamber_admin users
- Group Coordinators must relay change requests to the chamber.
- Name changes:
  - update the canonical participant name
  - are reflected retroactively in historical attendance views
  - **do not modify logs**
- Logs will show:
  - original name at time of action
  - explicit “name changed by chamber user X” events

---

## 5. IMB → Non-IMB Transitions

When an IMB member leaves the chamber and later participates as a visiting participant:

- **No linking** is performed between the IMB record and the visiting record.
- The system treats them as **two distinct participant identities**.
- Reporting is therefore split:
  - IMB reports (points-relevant)
  - visiting participant reports (attendance-focused)
- The chamber **cannot manually convert** a PTK-Member into a Visitor by adjusting an approved attendee.
- Once approved, the attendee record is **frozen**; details cannot be edited.
- If a PTK-Member has a name change, it is updated in IMB and reflected in IV because the `user_id` is the same.
- The attendee **type** (PTK-Member vs Visitor) cannot be changed, even by chamber admins.
- Open question to client: can a visiting participant's name be changed manually?

This separation is intentional and explicit.

---

## 6. Intervision Points & Inactive Members

### 6.1 Core Rule
When a member becomes inactive:
- previously earned Intervision points **do not disappear**
- historical points remain:
  - stored
  - auditable
  - printable

This supports cases where former members request point statements for submission to a new chamber.

### 6.2 Permissions
- Only the **chamber** may generate point statements (PDFs) for **inactive members**.

---

## 7. Printing Attendance & Points for Visiting Participants

### 7.1 Current State
- Group Coordinators can print reports for **active visiting participants** via the frontend.
- Chambers can already print reports for **any IMB member** (active or inactive).

### 7.2 Required Enhancement
- Add the ability for the **chamber** to generate:
  - attendance reports
  - points reports (where applicable)
  - for **any visiting participant** (active or inactive)

### 7.3 Admin UI Placement
- WordPress admin → Edit IV Group
- Add a metabox that:
  - lists all visiting participants
  - provides tabs:
    - Active
    - Inactive
  - selecting a participant opens a new tab
  - passes data into the existing frontend date-range PDF generator

---

## 8. Open Privacy Question (Pending)

To reduce chamber workload, it has been proposed to also allow:

- Group Coordinators to print attendance/points reports
- for **inactive visiting participants**
- similar to what is already allowed for active visiting participants

This is currently under review by the client with respect to **German privacy regulations**.

**Status:** Open question – decision pending.

---

## 9. IV Meetings UI – List & Pagination Rules

### 9.1 IV Group Details → Meetings List
- Paginated list
- **10 meetings per page**
- Only meetings from the **last 6 years**

Pagination is believed to already exist in the GK view and must be verified.

### 9.2 Meetings Older Than 6 Years
**Open question (client checking):**
- Should meetings older than 6 years:
  - be hidden from the UI but retained in the system, or
  - be removed from the system entirely?

### 9.3 Attendees View
- Same rules must apply:
  - paginated
  - 10 per page
  - max 6 years
- This is **not currently implemented** and must be added.

---

## 10. Logs vs. Displayed Data

- Logs are **immutable** and always reflect:
  - the state at the time of the action
- UI views, reports, and printouts:
  - always use the **current canonical name**
- This ensures:
  - audit correctness
  - legal traceability
  - clean user-facing documents

---

## 11. Open Questions Summary

1. May Group Coordinators print reports for **inactive visiting participants** (privacy)?
2. Should IV meetings older than 6 years be **hidden** or **deleted**?
3. Final confirmation that system-wide timestamp migration is handled as a **separate project** with its own rollout.

---

## 12. Next Steps

1. Client confirmation of open questions
2. Separate concept & plan for system-wide timestamp migration
3. Implementation of IV- and UI-related changes on top of the new date model
4. Production rollout following freeze/migrate/test/unfreeze procedure

---