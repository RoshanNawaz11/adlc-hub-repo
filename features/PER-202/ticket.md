---
key: PER-202
type: story
status: todo
blocks: []
blocked_by: []
labels:
- repo:agent
repos:
- agent
summary: Reviewers can claim, hand over and track their cases
---

When the agents pause a payroll case for review, any reviewer can pick it up. Nothing shows who is working on what. Two reviewers sometimes answer the same case, cases get dropped when someone goes off shift, and team leads can't see which cases are waiting too long.

We want reviewers to claim a case, hand it over when they need to, see their own queue, and let leads spot cases nobody has picked up.

**Acceptance criteria**

*Claiming a case*

1. A reviewer can claim a case that is waiting for review. Everyone else then sees who has claimed it and since when.
2. Only the reviewer who claimed a case can answer its flags or approve or reject it. Anyone else gets a message saying who has it.
3. A reviewer can release their claim, and the case becomes unclaimed again.

*Handing over*

4. A reviewer can hand a claimed case to another named reviewer, with a short reason (required).
5. The case's history shows each claim, release and handover: who, when, and the reason.

*My cases*

6. A reviewer can see a list of the cases they have claimed, oldest waiting first. Each shows the employee, the case type and how long it has been waiting.
7. When a case they claimed finishes, it leaves their list.

*Waiting too long*

8. A lead can see every case that has waited more than 4 hours without being claimed, longest first.
9. The 4-hour limit can be changed without a release.

*General*

10. Cases that aren't waiting for review (running or finished) can't be claimed, and the reviewer is told why.
11. Claims, handovers and history are kept when the service restarts.
12. How cases run and how they're reviewed stays the same for teams that never claim anything.

**Examples**

* Asha claims a case with a `netPayMismatch` flag. Ben opens it, sees "Claimed by Asha, 10:42" and can't answer the flag.
* Asha's shift ends. She hands the case to Ben with "Waiting on client for bonus details". Ben sees it in his list, with the reason in the history.
* A case paused at 08:00 and still unclaimed at 12:30 appears in the lead's waiting list.

**Out of scope:** notifications, automatic assignment, and reviewer permissions or roles beyond "lead".
