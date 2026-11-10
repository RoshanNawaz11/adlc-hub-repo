---
key: PER-199
type: story
status: todo
blocks: []
blocked_by: []
labels:
- repo:agent
repos:
- agent
summary: Reviewers can leave notes on a case
---

As a payroll reviewer looking at a case the agents paused for review, I want to leave a note on it, and to see the notes earlier reviewers left, so the next person picking up the case knows what was already checked or decided and doesn't redo it.

Today, anything a reviewer works out about a case lives only in their head or in a chat message. When a case is handed over, paused again, or reopened the next day, that knowledge is lost.

**Acceptance criteria**

1. While a case is paused, for flags or for the final review, a reviewer can add a note: free text up to 2,000 characters.
2. Each note records who wrote it and when. Notes can't be edited or deleted afterwards; a correction is a new note.
3. Anyone looking at the case sees all its notes, oldest first, in the same place they see the case's flags and summary.
4. Notes are kept when the case is resumed, paused again, or the service restarts. A case that's finished still shows its notes.
5. A note can't be added to a case that is running or finished. The reviewer gets a clear message saying why.
6. An empty note, or one over the limit, is refused with a message saying what's wrong.
7. Everything a case shows today, and how it's resumed, stays the same. Notes are added alongside, not in place of anything.

**Examples**

* A reviewer pauses on a `netPayMismatch` flag and writes "Checked with client: bonus paid separately, mismatch expected." The next reviewer sees it before answering the flag.
* After the case finishes, a support person opening it still sees both reviewers' notes.

**Out of scope:** editing or deleting notes, attachments, mentions or notifications, and showing notes outside the case.
