---
key: PER-234
type: story
status: todo
blocks: []
blocked_by: []
labels:
- repo:agent
repos:
- agent
summary: Reviewers can set a case's priority
---

The agents decide a case's priority when they plan it, and raise it one level as soon as any flag is raised. Reviewers often know better: a termination with a final-pay deadline today is urgent even when the planner said medium, and a case waiting on the client can drop to low. Today nobody can change it, so urgent cases sit among routine ones.

We want reviewers to set a case's priority, with a reason, and have it stick.

**Acceptance criteria**

*Setting the priority*

1. A reviewer can set the priority of a case that is waiting for answers to its flags, or waiting for the final review, to high, medium or low.
2. Setting a priority needs a reason of up to 300 characters. A reason that is empty, or only spaces, tabs or line breaks, is refused, and the case keeps its priority.
3. A priority other than high, medium or low is refused with a message naming the allowed values.

*After setting it*

4. The case shows the new priority straight away, and that a reviewer set it, with the reason.
5. The priority the reviewer set stays when the case continues. A flag raised later no longer raises it a level, and planning the case again doesn't replace it.
6. The case's history shows each change: the old priority, the new one, and the reason.
7. Setting the priority again replaces the earlier one. The history keeps both changes.
8. The console's activity feed is told about the change ("Priority set to high", with the employee's name when the order has one).

*Cases that can't be changed*

 9. A case that is still running can't be changed. The reviewer is told to wait until it pauses.
10. A case that is done, rejected or failed can't be changed, and the reviewer is told its status. A case that doesn't exist gives a not-found message.

*General*

11. A priority set by a reviewer is kept when the service restarts.
12. Cases whose priority no reviewer has set work exactly as today: the planner's priority, raised a level once any flag is raised.

**Examples**

* SO-1004 is a termination whose final pay is due today. The planner said medium. Asha sets it to high with "Final pay due today". It shows as high, set by a reviewer.
* Later a compliance flag is raised on SO-1004. It stays high and doesn't change.
* Ben sets a case waiting on the client from medium to low with "Waiting on client for bonus details". The next day he sets it back to medium with "Client replied". The history shows both changes.
* Asha tries to set a priority of "urgent" and is told the allowed values are high, medium and low.

**Out of scope:** changing the priority from the command line, changing how the planner decides priority, ordering cases or queues by priority, and notifications.
