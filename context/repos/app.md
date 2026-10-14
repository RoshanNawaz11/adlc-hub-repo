# App repo (`app`)

## Stack

- Language: **Python 3.12**
- Layout: `src/greeter.py`, `tests/test_greeter.py`
- Tests: `python -m pytest -q`
- Lint/compile: `python -m compileall -q src`

## Do

- Extend `greet(name: str) -> str` in `src/greeter.py` when changing greeting behavior.
- Keep unit tests in `tests/test_greeter.py`.

## Don't

- Do **not** introduce TypeScript/JavaScript (`*.ts`, `*.tsx`, `package.json` app code).
- Do **not** invent alternate module names (e.g. `greeting.ts`) — use `greeter.py`.
