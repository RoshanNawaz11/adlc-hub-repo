# Requirements — PER-202

**Feature:** Reviewers can claim, hand over and track their cases

**Source:** PER-202 ticket

---

## Acceptance Criteria

### AC1: Claiming a case (unclaimed → claimed)
**Given** a case is waiting for review and unclaimed  
**When** a reviewer claims it  
**Then** the case is marked as claimed by that reviewer with a timestamp, and all other reviewers see who claimed it and when

### AC2: Claim exclusivity
**Given** a case is claimed by Reviewer A  
**When** Reviewer B attempts to answer flags, approve, or reject the case  
**Then** Reviewer B receives a message indicating the case is claimed by Reviewer A and cannot proceed

### AC3: Releasing a claim (claimed → unclaimed)
**Given** a reviewer has claimed a case  
**When** they release their claim  
**Then** the case becomes unclaimed again and available to any reviewer

### AC4: Handing over a claim (claimed by A → claimed by B)
**Given** Reviewer A has claimed a case  
**When** Reviewer A hands the case to Reviewer B with a reason  
**Then** the case is now claimed by Reviewer B, the reason is required and recorded, and Reviewer A no longer has claim

### AC5: Claim history tracking
**Given** a case has had claims, releases, and handovers  
**When** the history is viewed  
**Then** each claim, release, and handover is recorded showing who performed the action, when, and the reason (for handovers)

### AC6: Reviewer's queue view
**Given** a reviewer has claimed one or more cases  
**When** they view their queue  
**Then** they see all cases they have claimed, sorted oldest waiting first, showing employee name, case type, and waiting duration

### AC7: Completed cases leave the queue
**Given** a reviewer's queue contains a claimed case  
**When** that case finishes (approved, rejected, or otherwise completed)  
**Then** the case is removed from the reviewer's queue

### AC8: Lead view of unclaimed waiting cases
**Given** cases are waiting for review  
**When** a lead views the unclaimed waiting list  
**Then** they see every case that has waited more than the configured threshold (default 4 hours) without being claimed, sorted longest waiting first

### AC9: Configurable waiting threshold
**Given** the waiting threshold is currently set to 4 hours  
**When** the threshold is changed to a different value  
**Then** the lead's waiting list reflects the new threshold without requiring a service restart or code release

### AC10: Claim state validation
**Given** a case is not in "waiting for review" state (e.g., still running or already finished)  
**When** a reviewer attempts to claim it  
**Then** the claim is rejected with a message explaining why the case cannot be claimed

### AC11: Persistence across restarts
**Given** cases have claims, handovers, and history recorded  
**When** the service restarts  
**Then** all claim states, handover records, and history are preserved and available

### AC12: Backward compatibility
**Given** a team or environment that does not use the claim workflow  
**When** cases are processed  
**Then** case execution and review behavior remains unchanged for teams that never claim cases

---

## Examples

**Example 1 (AC1, AC2):**  
- Asha claims a case with a `netPayMismatch` flag at 10:42.
- Ben opens the same case and sees "Claimed by Asha, 10:42" and cannot answer the flag.

**Example 2 (AC4, AC5):**  
- Asha's shift ends.
- She hands the case to Ben with reason "Waiting on client for bonus details".
- Ben sees the case in his queue with the handover reason in the history.

**Example 3 (AC8, AC9):**  
- A case paused at 08:00 and is still unclaimed at 12:30.
- The lead's waiting list shows this case (4.5 hours > 4-hour threshold).
- Threshold is changed to 6 hours; the case no longer appears in the lead's list until 14:00.

---

## Out of Scope

- Notifications (push, email, or in-app) when cases are claimed, handed over, or waiting too long
- Automatic assignment of cases to reviewers
- Reviewer permissions or roles beyond distinguishing "lead" from "reviewer"

---

## Open Questions

*None at this time. All requirements are derived directly from the ticket.*
