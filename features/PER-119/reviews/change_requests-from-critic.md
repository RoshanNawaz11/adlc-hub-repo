# Course correction PER-119 (auto from critic exhaustion)

Authoritative for the next run. Human: review/edit this file, then re-run:
`adlc start PER-119 --skip-dev` or `adlc review request-changes PER-119 --note "..."` then `adlc resume PER-119`.

## Critic mismatches to resolve

**What would go wrong:**

1. **Seen-flags tracking gap creates incorrect model calls**  
   The HLD (AD-5) and Story 3 LLD both state that seen flags must be tracked on **both** the model hop path and the forced hop path. Story 3's Test 6 verifies this. However, the Requirements document (REQ-3 AC-3.4) still says "The orchestrator updates a new state field with the flag IDs **from each brief it receives**." That wording implies tracking only happens when the model receives a brief (i.e., only on the model hop path), which contradicts the clarification in AD-5 and the Story 3 implementation.

   A developer reading the Requirements would implement tracking only on the model hop path. When a forced hop occurs, flags would never be marked as seen. The next hop would incorrectly call the model because it thinks the flags are unseen, defeating the optimization goal.

   **Where they disagree:** Requirements REQ-3 AC-3.4 says "from each brief it receives" (model hop only). HLD AD-5, Story 3 LLD, and Story 3 Test 6 all specify tracking on both paths. The Requirements wording needs updating to match the clarified design.

2. **Stage agent identification underspecified in all documents**  
   Requirements REQ-1 AC-1.3 says candidates include "every non-empty set of stage agents" but never defines what a stage agent is. The HLD (AD-4) clarifies that stage agents are "identified by examining the orchestrator's existing route-building code" and gives three patterns (constant, route logic, metadata), but the Requirements remain silent on this. Story 2 LLD repeats the HLD guidance but adds no new binding.

   A developer reading only the Requirements would not know how to identify stage agents. They might hard-code names, invent a new classification, or enumerate the wrong nodes. The code would pass the stated ACs but enumerate incorrect candidate routes.

   **Where they disagree:** Requirements say "stage agents" without defining them. HLD AD-4 and Story 2 LLD provide the definition and derivation rule. The Requirements need a clarification (not a new AC—just a note) pointing to the derivation rule so developers don't have to reverse-engineer it from the HLD.

---

**Note for the gate:**  
The second issue (stage agent identification) may be low-risk if the repo tree shows an obvious mechanism (e.g., a `STAGE_AGENTS` constant). You can check that when you see the code. The first issue (seen-flags tracking) is higher risk because it directly contradicts what the developer would build from the Requirements alone.

## Repair rules

- Keep ticket/repo labels (`repo:*`) and out-of-scope constraints.
- Prefer one complete story LLD per involved repo when possible; do not leave truncated mid-sentence files.
- Align AC counts, test counts, and file paths across requirements ↔ HLD ↔ breakdown ↔ stories.
- Rewrite incomplete artifacts fully (no cutoffs).
