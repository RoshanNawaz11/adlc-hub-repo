---
key: PER-222
type: story
status: todo
blocks: []
blocked_by: []
labels:
- repo:agent
repos:
- agent
summary: Reviewers can withdraw a paused case
---

When the agents pause a case, waiting for answers to its flags or for the final review, a reviewer can only carry on. If the order was raised by mistake or replaced by a newer one, the only way out is to reject it at the final review. A case waiting for flag answers can't be stopped at all: someone has to make up answers just to reach the review, and those answers then sit in the audit trail as if they were real.

We want reviewers to withdraw a paused case, with a reason, so it stops cleanly and its history says why.

**Acceptance criteria**

*Withdrawing a case*

1. A reviewer can withdraw a case that is waiting for answers to its flags, or waiting for the final review.
2. A withdrawal needs a reason of up to 500 characters. A reason that is empty, or only spaces, tabs or line breaks, is refused, and the case stays paused.
3. Withdrawing a case doesn't run any agent again and doesn't carry out any part of the payroll change. Answers already given to its flags stay as they were.

*After withdrawing*

4. The case shows as withdrawn, with the reason and when it was withdrawn. It no longer shows as paused.
5. The case's history ends with the withdrawal: by the reviewer, when, and the reason.
6. A withdrawn case can't be resumed, have its flags answered, be approved or rejected, be started again or be withdrawn a second time. The reviewer is told it was withdrawn.
7. A withdrawn case stays withdrawn when the service restarts.

*Cases that can't be withdrawn*

8. A case that is still running can't be withdrawn. The reviewer is told to wait until it pauses.
9. A case that is done, rejected or failed can't be withdrawn, and the reviewer is told its status. A case that doesn't exist gives a not-found message.

*Everywhere a case is handled*

10. A case can be withdrawn from the command line too, with the same rules.
11. When a queued request to resume a case arrives after the case was withdrawn, nothing resumes and the case's queue status shows it as withdrawn.
12. The console's activity feed is told about the withdrawal ("Case withdrawn", with the employee's name when the order has one). If the console can't be reached, the withdrawal still goes through.

*General*

13. Starting cases, answering flags, approving and rejecting work exactly as today for cases that are never withdrawn.

**Examples**

* SO-1004 duplicates SO-1003, which is already being processed, and is waiting for answers to two flags. Asha withdraws it with "Duplicate of SO-1003". It shows as withdrawn, and neither flag gets an answer.
* Ben withdraws a case at the final review with "Client asked to stop the termination". When he then tries to approve it, he is told it was withdrawn.
* Asha tries to withdraw a case with a reason of only spaces and is asked for a reason. The case is still waiting for review.
* After a restart, Ben opens a case withdrawn the day before. It still shows as withdrawn, with its reason.

**Out of scope:** stopping a case while it is running or still waiting in the queue to start, undoing a withdrawal, who is allowed to withdraw, notifications, and changes to the console itself.
