# Open items from the critic for PER-223

Written by the critic, not approved by a person. Delete the items you disagree with and edit the rest; what is left is applied on the next run:
`adlc resume PER-223` (or `adlc start PER-223 --skip-dev`).

### C3 (cannot_fix)
- file: stories/01-agent.md
- section: ## File Plan
- quote: src/agents/graph/validation.py
- match: critic-items.md › C3 › decided
- problem: The decision says the file belongs in `src/agents/runtime/validation.py`, but the file plan and all references still say `src/agents/graph/validation.py`.
- change: Either change every mention of the file path to `src/agents/runtime/validation.py` (file plan table, interfaces heading, and all import statements in pseudo-code), or remove the `decided:` note from `critic-items.md` C3 and add a note in the file plan explaining why `graph` is the right package despite two of three importers being in `runtime`.
- decided: The validation module belongs in `src/agents/runtime/validation.py` rather than `src/agents/graph/validation.py` because two of the three files that import it (`server.py` and `orders.py`) are in the `runtime` package, and separating data validation from graph execution logic keeps responsibilities clear.
- blocks because: The file plan and every interface and pseudo-code section name the new file as `src/agents/graph/validation.py`, but the decision recorded in `critic-items.md` says it belongs in `src/agents/runtime/validation.py`. Dev would create the file in `graph` as the plan says, and the imports in `server.py` and `orders.py` would fail because they are told to import from `graph` but the decision says the file should be in `runtime`.
- why cannot fix: fix not applied: would drop sections: src/agents/graph/validation.py` (new)
- history:
  - round 1: raised (blocking)
  - round 1: the fixer decided: The validation module belongs in `src/agents/runtime/` rather than `src/agents/graph/` because two of the three files that import it (`server.py` and `orders.py`) are in the `runtime` package, and separating data validation from graph execution logic keeps responsibilities clear. The `graph` package handles state transitions and node orchestration; the `runtime` package handles HTTP and queue entry points where input validation occurs.
  - round 1: cannot fix: fix not applied: would drop sections: src/agents/graph/validation.py` (new)
  - round 2: raised again (blocking)
  - round 2: the fixer decided: The validation module belongs in `src/agents/runtime/validation.py` rather than `src/agents/graph/validation.py` because two of the three files that import it (`server.py` and `orders.py`) are in the `runtime` package, and separating data validation from graph execution logic keeps responsibilities clear.
  - round 2: cannot fix: fix not applied: would drop sections: src/agents/graph/validation.py` (new)
