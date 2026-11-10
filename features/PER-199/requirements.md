# Requirements — PER-199

**Source:** Ticket PER-199  
**Epic:** (none — backlog story)  
**Date:** 2025-01-24

## Ticket summary

Reviewers can leave notes on a case

As a payroll reviewer looking at a case the agents paused for review, I want to leave a note on it, and to see the notes earlier reviewers left, so the next person picking up the case knows what was already checked or decided and doesn't redo it.

Today, anything a reviewer works out about a case lives only in their head or in a chat message. When a case is handed over, paused again, or reopened the next day, that knowledge is lost.

## Acceptance criteria

### AC1: Add note while paused

**Given** a case is paused (for flags or for final review)  
**When** a reviewer adds a note  
**Then** the note is saved with the case, recording the note text (up to 2,000 characters), the reviewer who wrote it, and the timestamp when it was written

**Clarification:** "Paused" means `status: "needs_input"` or `status: "needs_review"` per `src/agents/graph/states.py`.

---

### AC2: Notes are immutable

**Given** a note has been added to a case  
**When** the note is saved  
**Then** it cannot be edited or deleted afterwards  
**And** a correction is made by adding a new note

---

### AC3: Display all notes

**Given** a case has one or more notes  
**When** anyone looks at the case  
**Then** all notes are displayed, oldest first, in the same place the case's flags and summary are shown

**Clarification:** "The same place" refers to the case's state structure returned by the graph and displayed in the review UI.

---

### AC4: Notes persist

**Given** a case has notes  
**When** the case is resumed, paused again, the service restarts, or the case finishes  
**Then** the notes are still present and displayed

---

### AC5: Notes only on paused cases

**Given** a case is in a state other than paused (running, done, rejected, failed)  
**When** a reviewer attempts to add a note  
**Then** the attempt is refused with a clear message saying why (e.g., "Cannot add notes to a running case" or "Cannot add notes to a finished case")

**Clarification:** Allowed states are `needs_input` and `needs_review`; disallowed states are `running`, `done`, `rejected`, `failed` per `src/agents/graph/states.py`.

---

### AC6: Validation of note content

**Given** a reviewer is adding a note  
**When** the note is empty or exceeds 2,000 characters  
**Then** the note is refused with a message saying what's wrong (e.g., "Note cannot be empty" or "Note exceeds 2,000 character limit")

---

### AC7: No disruption to existing functionality

**Given** the case review workflow today  
**When** notes are added to the system  
**Then** everything a case shows today (flags, summary, steps, results, pipeline) and how it's resumed stays the same  
**And** notes are added alongside, not in place of anything

---

## Examples

1. A reviewer pauses on a `netPayMismatch` flag and writes "Checked with client: bonus paid separately, mismatch expected." The next reviewer sees it before answering the flag.
2. After the case finishes, a support person opening it still sees both reviewers' notes.

---

## Out of scope

- Editing or deleting notes
- Attachments
- Mentions or notifications
- Showing notes outside the case (e.g., in a separate log or audit trail)

---

## Open questions

None.

---

## Constraints

- Note text limit: 2,000 characters
- Notes can only be added when the case is paused (`needs_input` or `needs_review`)
- Notes are immutable once saved
- No new dependencies or frameworks introduced (use existing Python stack and storage mechanism)
