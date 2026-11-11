# Requirements — PER-209

**Reviewers can see, filter and count every case**

## Overview

Today a reviewer can only open a case they already have the ID of. Nobody can see which cases are waiting, which are flagged, or how many are stuck, so cases that need review sit unnoticed until someone asks about them.

We want one list of every case that reviewers and leads can filter and sort, with counts per status.

## Acceptance Criteria

**Note on terminology:** The ticket uses the terms "waiting for input" and "waiting for review" to match business language. The implementation normalizes these to `needs_input` and `needs_review` (matching the existing `State['status']` values in `states.py`). The API accepts both forms in query parameters but always returns the normalized forms in responses.

### AC1: Case list endpoint returns all cases with status

**Given** the system has cases in various states (running, needs_input, needs_review, done, rejected, failed)  
**When** a reviewer requests the case list  
**Then** the list includes every case regardless of status  
**And** each case shows its service order ID, employee, case type, status, priority, flagged flag, and last changed timestamp

---

### AC2: Default sort is most recently changed first

**Given** cases exist with different last-changed timestamps  
**When** a reviewer requests the case list without specifying a sort order  
**Then** cases appear with the most recently changed first

---

### AC3: Pagination with configurable page size

**Given** more than 25 cases exist  
**When** a reviewer requests the case list without specifying a page size  
**Then** the response contains 25 cases per page  

**Given** a reviewer specifies a page size between 1 and 100  
**When** they request the case list  
**Then** the response contains the requested number of cases per page  

**Given** a reviewer requests a page size greater than 100  
**When** they request the case list  
**Then** the request is refused with a message stating the maximum allowed size

---

### AC4: Filter by status

**Given** cases exist with different statuses  
**When** a reviewer filters by one or more statuses  
**Then** only cases matching the specified status(es) appear in the list

---

### AC5: Filter by priority

**Given** cases exist with different priorities  
**When** a reviewer filters by one or more priorities  
**Then** only cases matching the specified priority/priorities appear in the list

---

### AC6: Filter by flagged state

**Given** some cases are flagged and some are not  
**When** a reviewer filters by flagged=true  
**Then** only flagged cases appear  

**When** a reviewer filters by flagged=false  
**Then** only non-flagged cases appear

---

### AC7: Filter by case type

**Given** cases exist with different case types  
**When** a reviewer filters by one or more case types  
**Then** only cases matching the specified type(s) appear in the list

---

### AC8: Combine multiple filters

**Given** multiple filter criteria are provided (e.g., status=needs_review AND flagged=true)  
**When** a reviewer applies them  
**Then** only cases matching ALL criteria appear

---

### AC9: Filter by specific flag type

**Given** cases have flags of different types (e.g., `netPayMismatch`, `missingData`)  
**When** a reviewer filters by a specific flag type  
**Then** only cases with at least one unresolved flag of that type appear  
**And** cases where that flag type was resolved or merged into another flag do not appear

---

### AC10: Sort by wait time for review

**Given** multiple cases are waiting for review with different wait durations  
**When** a reviewer sorts by wait time  
**Then** cases appear with the longest-waiting first

---

### AC11: Status counts endpoint

**Given** cases exist in various statuses  
**When** a lead requests counts  
**Then** the response shows the count of cases in each status  
**And** the count of flagged cases currently waiting  
**And** no case list is loaded

---

### AC12: Cases persist across restarts

**Given** cases were created before a system restart  
**When** the system restarts  
**Then** those cases still appear in the list with the same status they had before the restart  
**And** they are included in the counts

---

### AC13: Invalid filter values are rejected

**Given** a filter value does not exist (e.g., an unknown status or priority)  
**When** a reviewer applies that filter  
**Then** the request is refused with a message naming the allowed values

---

### AC14: Single case operations unchanged

**Given** the case list and filtering are implemented  
**When** a reviewer runs, resumes, or opens a single case by ID  
**Then** that operation works exactly as it does today

## Examples

- Asha filters by "waiting for review" and "flagged" and sees 3 cases, the one waiting longest at the top.
- Ben filters by flag type `netPayMismatch` and sees the two cases where that flag is still unresolved, not the one where it was merged into another flag.
- After a restart, the lead's counts still show 5 cases waiting for review.

---

## Out of Scope

- Search by free text
- Saved filters
- Exporting the list
- Any change to how cases are reviewed
