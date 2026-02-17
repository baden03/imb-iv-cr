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

## 3. Legacy Migration: Hamburg-Centric Meta Keys

While the change request focuses on moving the system forward, the existing system requires cleanup and migration. The codebase currently uses Hamburg-centric meta keys that should be updated to generic, chamber-agnostic keys.

### 3.1 Scope
- **System-wide**: All references to these keys across field creation, program logic, and database storage
- **Codebase**: Update field creation and program logic
- **Database**: Migrate existing `meta_key` values in post meta and related tables

### 3.2 Key Mappings (Examples)

| Current (Hamburg-centric) | Target (Generic) |
|---------------------------|------------------|
| `teil_ptkhh_yes`          | `teil_ptk_yes`   |
| `teilnehmer_ptkhh`        | `teilnehmer_ptk` |

*A full audit of all `ptkhh` / Hamburg-specific keys is required before migration.*

### 3.3 Requirements
- Identify all occurrences of Hamburg-centric keys in the codebase
- Update field creation and any program logic that reads/writes these keys
- Provide a database migration script to rename `meta_key` values
- Migration should follow the same freeze/deploy/migrate/verify/unfreeze procedure as other system-wide changes

---

## 4. Roles & Terminology

### 4.1 IMB Member
- Has an _active_ IMB user account (`user_id`)
- Eligible for automatic point calculation
- Can become inactive (not deleted)

### 4.2 Visiting Participant (Non-IMB)
- No _active_ IMB user account
- External participant (other chambers, Ärztekammer, etc.)
- Attendance tracked, but no automatic IMB points

### 4.3 IV Participant Record
- Represents a concrete group membership
- Status:
  - `pending` — in Prüfung (in review)
  - `approved` — Approved
  - `denied` — Denied
  - `archived` — (needs to be added) Archived (inactive); used when the participant has left (e.g. IMB user became inactive) and the coordinator should remove them and re-add as guest if they return
- Approval is **per IV group**, not global

### 4.4 Group Coordinator (GK)
- Manages group composition
- Submits participants
- **Cannot modify participants after approval**

### 4.5 Chamber / Chamber Admin
- Approves participants
- Manages name changes for visiting participants
- Can generate historical reports
- Responsible for audit and compliance

---

## 5. Participant Immutability & Name Changes

### 5.1 General Rule
Once a participant is **approved**, **no changes** may be made by Group Coordinators — with one exception: the **chamber** field (visiting chamber) may be updated by chamber admins and Group Coordinators when it was left blank (e.g. after automatic PTK→Visitor conversion).

This applies to:
- IMB members
- visiting participants
- all other participant data fields

### 5.2 IMB Members
- Name changes are handled exclusively via the **IMB name-change workflow**.
- Once approved:
  - the updated name is reflected everywhere (current and historical views).
- Logs remain unchanged and record the original name at the time of each action.

### 5.3 Visiting Participants
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

## 6. IMB → Non-IMB Transitions

When an IMB member leaves the chamber and later participates as a visiting participant:

- **No linking** is performed between the IMB record and the visiting record.
- The system treats them as **two distinct participant identities**.
- Reporting is therefore split:
  - IMB reports (points-relevant)
  - visiting participant reports (attendance-focused)
- The chamber **cannot manually convert** a PTK-Member into a Visitor by adjusting an approved attendee.
- Once approved, the attendee record is **frozen**; details cannot be edited (except the chamber field when left blank after automatic PTK→Visitor conversion — see §8.1).
- If a PTK-Member has a name change, it is updated in IMB and reflected in IV because the `user_id` is the same.
- The attendee **type** (PTK-Member vs Visitor) cannot be changed, even by chamber admins.
- Open question to PTK-HH (client): can a visiting participant's name be changed manually?

This separation is intentional and explicit.

---

## 7. Intervision Points & Inactive Members

### 7.1 Core Rule
When a member becomes inactive:
- previously earned Intervision points **do not disappear**
- historical points remain:
  - stored
  - auditable
  - printable

This supports cases where former members request point statements for submission to a new chamber.

### 7.2 Permissions
- Only the **chamber** may generate point statements (PDFs) for **inactive members**.

### 7.3 IV Meeting Points Check
When an IV meeting is created, points are automatically generated for each PTK member and require chamber approval.

- Add a check to ensure the member is **not** `user_status = inactive` before assigning points.
- If a user is inactive but still assigned as a PTK-Member, log this in the IV meeting log.

---

## 8. IV Group Membership When IMB User Becomes Inactive

### 8.1 Automatic Processing (Agreed)

When a PTK user is marked as inactive with an exit date (required), the system will **automatically** perform the following:

1. **Update `valid_to`** in the `group_membership` table for all IV groups the user belongs to — set to the exit date (timestamp).
2. **Create a new entry** in the `group_membership` table for each `group_id` the inactive user is part of: `valid_from` = today (timestamp), `valid_to` = 20 years in the future (or whatever timestamp is used for "far future").
3. **Remove** (not mark as inactive) the user completely from all IV groups' `group_member_details`.
4. **Create a new visiting participant** in each group's `group_member_details` with:
   - `teilnehmer_name` — from the former PTK member
   - `teilnehmer_ptkhh` (to be renamed to `teilnehmer_ptk`) — nein (visitor, not PTK member)
   - `teil_kammer_other` — **blank** (to be filled by the Group Coordinator)
   - `teilnehmer_type` — copied from the old PTK Member values
   - `teil_appurkunde` — blank (already approved as PTK member)
   - `status` — approved

5. **Update postmeta** for any `intervision_meeting` posts held after the user's exit date where `meta_key` = `'attended'` and `meta_value` = the inactive `user_id`: replace the `user_id` with the first and last name (used as the new `member_key` in `group_membership`).
6. **Purge points** assigned to the user that were created **after** the exit date.

The **chamber** field (`teil_kammer_other`) is the **only** field that may be changed by chamber admins and Group Coordinators for an approved participant, but only if it is blank. All other approved participant fields remain frozen.

### 8.2 Historical Cleanup: Existing Inactive Members in IV Groups

There is a backlog of members who have since gone inactive. Currently they have either:

- been manually changed to visitors (currently allowed by the chamber, but will be **locked** after this change), or
- remain listed in IV groups as PTK members, with points assigned, etc.

A utility job has been developed that exports all inactive members who belong to IV groups as a PTK member. The resulting CSV can be used to perform a cleanup.

**Open question to PTK-HH (client):**

For the clean up job, the visiting chamber they joined is unknown, but a column can be added to the CSV for manual update; this modified list would then drive the cleanup. This approach would allow all attended meetings **after** their exit date to be reassigned to the new visitor record, while keeping the legacy PTK attendance intact.

### 8.3 PTK Member Exiting as Group Coordinator

**Open question to PTK-HH (client):**

What if the PTK member who is exiting is a Group Coordinator? The ability to mark a GC as inactive should only be allowed if they have already transferred their GC role to another member.

**Proposed implementation (for now):** Always check for any open IV groups the user owns before allowing the chamber to mark the user as inactive. If the user owns open IV groups, block the inactivity action until ownership has been transferred.

---

## 9. Printing Attendance & Points for Visiting Participants

### 9.1 Current State
- Group Coordinators can print reports for **active visiting participants** via the frontend.
- Chambers can already print reports for **any IMB member** (active or inactive).

### 9.2 Required Enhancement
- Add the ability for the **chamber** to generate:
  - attendance reports
  - points reports (where applicable)
  - for **any visiting participant** (active or inactive)

### 9.3 Admin UI Placement
- WordPress admin → Edit IV Group
- Add a metabox that:
  - lists all visiting participants
  - provides tabs:
    - Active
    - Inactive
  - selecting a participant opens a new tab
  - passes data into the existing frontend date-range PDF generator

---

## 10. Open Privacy Question (Pending)

To reduce chamber workload, it has been proposed to also allow:

- Group Coordinators to print attendance/points reports
- for **inactive visiting participants**
- similar to what is already allowed for active visiting participants

This is currently under review by the client with respect to **German privacy regulations**.

**Status:** Open question – decision pending.

---

## 11. IV Meetings UI – List & Pagination Rules

### 11.1 IV Group Details → Meetings List
- Paginated list
- **10 meetings per page**
- Only meetings from the **last 6 years**

Pagination is believed to already exist in the GK view and must be verified.

### 11.2 Meetings Older Than 6 Years
**Open question (client checking):**
- Should meetings older than 6 years:
  - be hidden from the UI but retained in the system, or
  - be removed from the system entirely?

### 11.3 Attendees View
- Same rules must apply:
  - paginated
  - 10 per page
  - max 6 years
- This is **not currently implemented** and must be added.

---

## 12. Logs vs. Displayed Data

- Logs are **immutable** and always reflect:
  - the state at the time of the action
- UI views, reports, and printouts:
  - always use the **current canonical name**
- This ensures:
  - audit correctness
  - legal traceability
  - clean user-facing documents

---

## 13. Open Questions Summary

1. May Group Coordinators print reports for **inactive visiting participants** (privacy)?
2. Should IV meetings older than 6 years be **hidden** or **deleted**?
3. Final confirmation that system-wide timestamp migration is handled as a **separate project** with its own rollout.
4. ~~When an IMB user becomes inactive~~ **Resolved**: Automatic processing per §8.1 (remove from groups, create visitor, chamber field editable).
5. **Historical cleanup**: For existing inactive members still in IV groups as PTK members: can we create approved visitors without the normal approval process? Yes. We would use a CSV of existing inactive users that belong to IV groups as PTK members (with a manually filled out column for chamber) to drive cleanup, and reassign post-exit meetings to the new visitor while preserving legacy PTK attendance.
6. **PTK member exiting as Group Coordinator**: What if the exiting PTK member is a Group Coordinator? Proposed: block marking as inactive until they have transferred ownership of any open IV groups to another user. Confirm with client.

---

## 14. Next Steps

1. Client confirmation of open questions
2. Separate concept & plan for system-wide timestamp migration
3. Full audit of Hamburg-centric meta keys (`ptkhh`, etc.) and migration plan
4. Implementation of IV- and UI-related changes on top of the new date model
5. Production rollout following freeze/migrate/test/unfreeze procedure (including meta key migration)

---