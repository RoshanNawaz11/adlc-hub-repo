# Critic notes for PER-234

Not blocking. For whoever reviews the specs at the gate.

## Outdated: about text that has changed since (3)

- **C1** · `specs/1.md` › ## File Plan: The spec prescribes `graph.aupdate_state(config, state)` to write the modified state back to the checkpoint, but LangGraph graphs do not expose an `aupdate_state` method. _(its quote is in no spec file)_
- **C2** · `specs/1.md` › ## Algorithm Notes: The spec acknowledges uncertainty about how to write the checkpoint without resuming the run ("Investigation needed … verify in LangGraph docs"), prescribes a method (`aupdate_state`) that may not exist, and offers only … _(its quote is in no spec file)_
- **C4** · `stories/01-agent.md` › ## Tests: The test code has a typo: `monke ypatch.setattr` instead of `monkeypatch.setattr`. _(its quote is in no spec file)_
