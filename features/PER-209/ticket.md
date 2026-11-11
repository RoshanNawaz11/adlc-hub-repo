---
key: PER-209
type: story
status: todo
blocks: []
blocked_by: []
labels:
- repo:agent
repos:
- agent
summary: Reviewers can see, filter and count every case
---

Today a reviewer can only open a case they already have the ID of. Nobody can see

which cases are waiting, which are flagged, or how many are stuck, so cases that

need review sit unnoticed until someone asks about them.

We want one list of every case that reviewers and leads can filter and sort, with

counts per status.

**Acceptance criteria**

*The case list*

1. A reviewer can see a list of every case: running, waiting for input, waiting

   for review, done, rejected and failed.
2. Each case in the list shows its service order ID, the employee, the case type,

   its status, its priority, whether it is flagged, and when it last changed.
3. The list shows the most recently changed cases first unless asked otherwise.
4. The list comes in pages. A page has 25 cases unless the reviewer asks for a

   different size, up to 100.

*Filtering and sorting*

5. A reviewer can filter the list by status, by priority, by flagged or not, and

   by case type. Filters can be combined.
6. A reviewer can filter by flag type, such as `netPayMismatch`, and see only the

   cases with at least one unresolved flag of that type.
7. A reviewer can sort cases waiting for review by how long they have been

   waiting, longest first.

*Counts*

8. A lead can see how many cases are in each status, and how many waiting cases

   are flagged, without loading the list.

*General*

 9. Cases that started before a restart still appear in the list and the counts,

    with the same status they had.
10. A filter value that doesn't exist (an unknown status or priority) is refused

    with a message naming the allowed values.
11. Running, resuming and opening a single case work exactly as today.

**Examples**

* Asha filters by "waiting for review" and "flagged" and sees 3 cases, the one

  waiting longest at the top.
* Ben filters by flag type `netPayMismatch` and sees the two cases where that flag

  is still unresolved, not the one where it was merged into another flag.
* After a restart, the lead's counts still show 5 cases waiting for review.

**Out of scope:** search by free text, saved filters, exporting the list, and any

change to how cases are reviewed.
