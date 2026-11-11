# requirements.md — PER-223

**Epic:** (none — standalone backlog story)  
**Story:** A rejected case always records the reviewer's reason  
**Derived from:** ticket acceptance criteria

---

## Requirements

### REQ-1: Reject with no reason → refused

**Given** a case is waiting for final review  
**When** the reviewer rejects it with no reason, an empty reason, or a reason of only spaces, tabs, or line breaks  
**Then** the rejection is refused and the case stays waiting for final review

**Technical notes:**

- "Empty reason" means the `reason` field is absent, `None`, or the empty string `""`.
- "Only spaces, tabs, or line breaks" means the reason contains only characters from the set `{' ', '\t', '\n', '\r'}`.
- The refusal applies whether the rejection arrives through the console, the command line, or the queue.

---

### REQ-2: Console entry point — validation

**Given** a case is waiting for final review  
**When** the reviewer rejects it through the console with no reason, an empty reason, or a reason of only spaces, tabs, or line breaks  
**Then** the rejection is refused and the case stays waiting for final review

**Technical notes:**

- The console is the interactive UI path where the reviewer types the action and reason.
- The same validation logic from REQ-1 applies.

---

### REQ-3: Command-line entry point — validation

**Given** a case is waiting for final review  
**When** the reviewer rejects it through the command line with no reason, an empty reason, or a reason of only spaces, tabs, or line breaks  
**Then** the rejection is refused and the case stays waiting for final review

**Technical notes:**

- The command line is the CLI where the rejection is submitted as arguments.
- The same validation logic from REQ-1 applies.

---

### REQ-4: Queue entry point — validation and status update

**Given** a case is waiting for final review  
**When** a rejection with no reason, an empty reason, or a reason of only spaces, tabs, or line breaks arrives through the queue  
**Then** the rejection is refused, the case stays waiting for final review, the case's queue status is set to paused, and a message saying a reason is needed is recorded

**Technical notes:**

- The queue is the asynchronous message path where rejections arrive as jobs.
- The same validation logic from REQ-1 applies.
- The queue status field is updated to `paused` when the rejection is refused.
- The message recorded should be clear that a non-empty reason is required.

---

### REQ-5: Reason is trimmed before saving

**Given** a case is rejected with a valid reason  
**When** the reason has spaces, tabs, or line breaks at the start or end  
**Then** those surrounding whitespace characters are removed before the reason is saved, and spaces inside the reason are kept

**Technical notes:**

- "Valid reason" means the reason passes the validation from REQ-1 — it has at least one non-whitespace character.
- "Trimmed" means leading and trailing whitespace are removed using the same character set as REQ-1: `{' ', '\t', '\n', '\r'}`.
- Spaces, tabs, and line breaks inside the reason (between non-whitespace characters) are preserved.
- Example: `"  Duplicate of SO-1002  "` is saved as `"Duplicate of SO-1002"`.

---

## Out of scope

- A length limit on the reason.
- Changes to how flags are answered.
- Fixing cases already rejected without a reason.
- Changes to how approvals work.
- Changes to how rejections work when a case is waiting for flag answers.

---

## Examples

- Asha rejects SO-1003 with `"   "` (three spaces) and is asked for a reason. The case is still waiting for review.
- Ben rejects SO-1003 with `"  Duplicate of SO-1002  "` and the case's history shows `"Duplicate of SO-1002"`.
- A rejection with no reason arrives through the queue. The case doesn't move, and its queue status says a reason is needed.
