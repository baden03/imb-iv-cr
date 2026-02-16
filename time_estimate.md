# Time Estimates For README Changes

These estimates cover implementation and basic verification in the current codebase. They exclude deployment, data backup, and client decision time. Ranges reflect unknowns until deeper code review.

## 2. System-wide Date Normalization (Foundational Change)
- **2.1 Canonical storage (Unix timestamps everywhere):**
- **2.2 German display format (UI only):**
- **2.3 Migration strategy + scripts + dry-run + verification:**
8 hours

## 4. Participant Immutability & Name Changes
- **4.1 Enforce no edits after approval (GK):**
- **4.2 IMB member name change flow reflects in IV:**
- **4.3 Visiting participant name change by chamber + logging:** 
2 hours

## 5. IMB → Non-IMB Transitions
- **No linking between IMB and visiting records + split reporting:**
- **Prevent manual conversion of approved attendee type:**
- **Freeze approved attendee details (admins included):** 
6 hours

## 6. Intervision Points & Inactive Members
- **6.1 Keep historical points for inactive members:**
- **6.2 Chamber-only PDF generation for inactive members:**
- **6.3 IV meeting points check for inactive status + log:**
2 hours

## 7. Printing Attendance & Points for Visiting Participants
- **7.2 Chamber ability to print for any visiting participant:** 
- **7.3 Admin UI metabox, tabs, and PDF handoff:**
2 hours

## 9. IV Meetings UI – List & Pagination Rules
- **9.1 Meetings list pagination + 6-year window:**
- **9.3 Attendees view pagination + 6-year window:**
3 hours

## 10. Logs vs Displayed Data
- **Immutable logs + canonical name display everywhere:**
1 hour

## Total (Implementation Only, Excluding Decision Items)
- **Approximate range:** 24 hours

