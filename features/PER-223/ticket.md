---
key: PER-223
type: story
status: todo
blocks: []
blocked_by: []
labels:
- repo:agent
repos:
- agent
summary: A rejected case always records the reviewer's reason
---

When a reviewer rejects a case at the final review, the reason is the only record of why the payroll change didn't go ahead. Today a rejection can get through without a real reason: a reason of only spaces is accepted, the command line accepts an empty one, and a rejection that arrives through the queue isn't checked at all. Those cases show up in the audit log as rejected with no explanation.

We want every rejection to carry a real reason, however it arrives.

**Acceptance criteria**

*Rejecting a case*

1. A rejection with no reason, an empty reason, or a reason of only spaces, tabs or line breaks is refused. The case stays waiting for the final review.
2. This holds whether the rejection comes from the console, the command line or the queue. A rejection refused from the queue leaves the case's queue status as paused, with a message saying a reason is needed.
3. The reason is saved without the spaces around it. Spaces inside it are kept.

*General*

4. Approving a case, with or without a comment, works as today.
5. Answering flags works as today. A rejection sent while a case is waiting for flag answers is still refused, as it is today.

**Examples**

* Asha rejects SO-1003 with "   " and is asked for a reason. The case is still waiting for review.
* Ben rejects it with "  Duplicate of SO-1002  ", and the case's history shows "Duplicate of SO-1002".
* A rejection with no reason arrives through the queue. The case doesn't move, and its queue status says a reason is needed.

**Out of scope:** a length limit on the reason, changes to how flags are answered, and fixing cases already rejected without a reason.
