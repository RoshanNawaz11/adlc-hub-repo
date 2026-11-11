# Requirements — PER-222

**Status:** Draft  
**Source:** Ticket PER-222  
**Epic:** _(none — standalone ticket)_

## Overview

Add a withdrawal operation for paused cases: a reviewer can stop a case waiting for flag answers or waiting for final review, with a mandatory reason, cleanly ending the case without running agents or carrying out the payroll change.

## Acceptance Criteria

### AC 1: Withdrawing a case

**Given** a case is paused waiting for answers to its flags  
**When** a reviewer withdraws it with a reason  
**Then** the case stops without running any agent, without carrying out the payroll change, and the answers already given to its flags remain as they were

---

**Given** a case is paused waiting for the final review  
**When** a reviewer withdraws it with a reason  
**Then** the case stops without running any agent, without carrying out the payroll change

---

**Given** a reviewer is withdrawing a case  
**When** the reason is empty, or contains only spaces, tabs or line breaks  
**Then** the withdrawal is refused, and the case remains paused

---

**Given** a reviewer is withdrawing a case  
**When** the reason is 500 characters or fewer  
**Then** the withdrawal proceeds

---

**Given** a reviewer is withdrawing a case  
**When** the reason exceeds 500 characters  
**Then** the withdrawal is refused, and the case remains paused

### AC 2: After withdrawing

**Given** a case has been withdrawn  
**When** a reviewer views the case  
**Then** the case shows as withdrawn, with the reason and the timestamp of when it was withdrawn, and no longer shows as paused

---

**Given** a case has been withdrawn  
**When** a reviewer views the case's history  
**Then** the history ends with the withdrawal entry: by which reviewer, when, and the reason

---

**Given** a case has been withdrawn  
**When** a reviewer attempts to resume it, answer its flags, approve it, reject it, start it again, or withdraw it a second time  
**Then** the operation is refused, and the reviewer is told it was withdrawn

---

**Given** a case has been withdrawn  
**When** the service restarts  
**Then** the case still shows as withdrawn, with its reason

### AC 3: Cases that can't be withdrawn

**Given** a case is still running (status `running`)  
**When** a reviewer attempts to withdraw it  
**Then** the withdrawal is refused, and the reviewer is told to wait until it pauses

---

**Given** a case is done, rejected, or failed  
**When** a reviewer attempts to withdraw it  
**Then** the withdrawal is refused, and the reviewer is told the case's status

---

**Given** a case does not exist  
**When** a reviewer attempts to withdraw it  
**Then** a not-found message is returned

### AC 4: Everywhere a case is handled

**Given** a case is paused  
**When** a reviewer withdraws it from the command line with a valid reason  
**Then** the withdrawal proceeds with the same rules as the other interfaces

---

**Given** a case has been withdrawn  
**When** a queued request to resume the case arrives  
**Then** nothing resumes, and the case's queue status shows it as withdrawn

---

**Given** a case has been withdrawn  
**When** the console's activity feed is notified  
**Then** the feed receives a "Case withdrawn" event, with the employee's name when the order has one

---

**Given** the console cannot be reached when a case is withdrawn  
**When** the withdrawal proceeds  
**Then** the withdrawal still completes, and the case shows as withdrawn

### AC 5: General

**Given** a case is never withdrawn  
**When** it is started, has its flags answered, or is approved or rejected  
**Then** these operations work exactly as they do today

## Examples

**Example 1:**  
SO-1004 duplicates SO-1003, which is already being processed, and is waiting for answers to two flags. Asha withdraws it with "Duplicate of SO-1003". It shows as withdrawn, and neither flag gets an answer.

**Example 2:**  
Ben withdraws a case at the final review with "Client asked to stop the termination". When he then tries to approve it, he is told it was withdrawn.

**Example 3:**  
Asha tries to withdraw a case with a reason of only spaces and is asked for a reason. The case is still waiting for review.

**Example 4:**  
After a restart, Ben opens a case withdrawn the day before. It still shows as withdrawn, with its reason.

## Out of Scope

- Stopping a case while it is running
- Stopping a case still waiting in the queue to start
- Undoing a withdrawal
- Authorization rules (who is allowed to withdraw)
- Notifications beyond the console activity feed
- Changes to the console itself

## Notes

- The ticket states that a case waiting for flag answers currently "can't be stopped at all", and a case at final review can only be rejected. Withdrawal provides a cleaner exit for mistaken or superseded orders.
- The withdrawal reason is mandatory, validated (non-empty after trimming whitespace), and capped at 500 characters.
- Withdrawal does not re-run agents or execute payroll changes; it is a terminal state.
- The case's history must record the withdrawal with: reviewer identity, timestamp, and reason.
- Queue integration: a resume message arriving after withdrawal must be handled gracefully (no-op, status reflects withdrawn).
- Console notification is best-effort: withdrawal proceeds even if the console is unreachable.
