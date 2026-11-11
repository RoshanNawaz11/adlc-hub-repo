# ADLC spec critic re-check

I am re-checking the documents after the fixer's pass. I will examine each earlier item against the current files and raise new items only within changed sections or where changed sections now disagree with other documents.

## Reading the earlier items against current files

### C1: CLI validator propagation and assignment

**Current text in `stories/01-agent.md` › ## Interfaces › `src/agents/runtime/server.py` (modified):**

```python
@model_validator(mode="after")
def _reason_on_reject(self) -> "ReviewDecision":
    if self.action == "reject":
        # validate_rejection_reason raises ValueError if invalid
        self.reason = validate_rejection_reason(self.reason)
    return self
```

**Current text in `stories/01-agent.md` › ## Interfaces (before pseudo-code section):**

> **Error handling:** Pydantic catches `ValueError` and FastAPI returns 422 with the error message.

The interface section now explicitly states the error handling in a separate paragraph, and the comment in the pseudo-code says "raises ValueError if invalid". The assignment is shown. This is consistent: the validator assigns the trimmed value and lets the `ValueError` propagate (Pydantic catches it). The two parts of the document agree. **C1 is fixed.**

### C3: validation.py location

**Current file plan in `stories/01-agent.md` › ## File Plan:**

```
| src/agents/graph/validation.py | **New file.** Validation function and error message constant. | ~20 |
```

**Current interfaces section in `stories/01-agent.md` › ## Interfaces:**

```python
### `src/agents/graph/validation.py` (new)
```

**Decision note from critic-items.md:**

> decided: The validation module belongs in `src/agents/runtime/` rather than `src/agents/graph/` because two of the three files that import it (`server.py` and `orders.py`) are in the `runtime` package...

The fixer recorded a decision but did not apply it. The file plan and interfaces still say `src/agents/graph/validation.py`. The decision says the file should be in `src/agents/runtime/validation.py`. **This is still a problem: the documents contradict the decision.**

However, I check the code to confirm what the fixer checked. The rejection says "would drop sections: src/agents/graph/validation.py` (new)". That suggests the fixer's tool would remove the entire section if it tried to move the file. The decision is recorded, but the documents are unchanged.

**Current state:** The file plan, interfaces and all pseudo-code sections still use `src/agents/graph/validation.py`. The decision in `critic-items.md` says it should be `src/agents/runtime/validation.py`. The documents and the decision disagree.

But I re-read the rejection reason: "why cannot fix: fix not applied: would drop sections". This is the fixer explaining why it could not act on the item, not evidence from the code. The code does not yet have this file — it is new. So there is no code evidence to check.

**Should I raise C3 again?** The problem is unchanged: the file location is still `src/agents/graph/validation.py` in every document, and the decision recorded by the fixer says it should be `src/agents/runtime/validation.py`. The documents must agree with the decision, or the decision must be removed and the location explained.

I check `changed-since-last-review.md`: "The fixer changed nothing since your last review." No sections were rewritten. So C3 is not inside a changed section, and nothing changed that would make it newly disagree with another document. Under the re-check instructions, I raise it again only if the current text still has the problem (it does) and I quote from the current files. I do.

But the instructions also say: "When an earlier item shows `decided:`, the fixer settled a choice there. Treat it as settled unless the documents still disagree with it."

The documents disagree with the decision. The decision says `runtime`, the documents say `graph`. That is a disagreement. I raise it.

## Checking for new problems in changed sections

`changed-since-last-review.md` says no sections were changed. I raise no new items.

## What would go wrong

### C3
- file: stories/01-agent.md
- section: ## File Plan
- quote: src/agents/graph/validation.py
- match: critic-items.md › C3 › decided
- severity: blocking
- blocks because: The file plan and every interface and pseudo-code section name the new file as `src/agents/graph/validation.py`, but the decision recorded in `critic-items.md` says it belongs in `src/agents/runtime/validation.py`. Dev would create the file in `graph` as the plan says, and the imports in `server.py` and `orders.py` would fail because they are told to import from `graph` but the decision says the file should be in `runtime`.
- problem: The decision says the file belongs in `src/agents/runtime/validation.py`, but the file plan and all references still say `src/agents/graph/validation.py`.
- change: Either change every mention of the file path to `src/agents/runtime/validation.py` (file plan table, interfaces heading, and all import statements in pseudo-code), or remove the `decided:` note from `critic-items.md` C3 and add a note in the file plan explaining why `graph` is the right package despite two of three importers being in `runtime`.

VERDICT: FAIL
