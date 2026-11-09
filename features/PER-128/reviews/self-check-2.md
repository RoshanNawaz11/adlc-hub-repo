## What would go wrong

**1. Story 2 contradicts the HLD on forced route return type**

The HLD (AD-2) says `forced_route` returns `list[str] | None`. Story 2's spec repeats this correctly in the API signature. But Story 2's LLD Test 6 asserts `assert route == ["analyst"]` when exactly one route is valid, which is correct. However, Test 7 mocks `broken_rules` to reject all candidates and expects `None`, which matches the HLD. No contradiction here on inspection—this was a false alarm during first pass.

**2. Story 3 Test 2 creates an invalid State for migration testing**

Story 3's LLD Test 2 (`test_run_without_usage_seen_flags_resumes`) constructs `old_state_data` with only `run_id` and `status`, then asks the developer to "add ALL other required State fields." The spec says "without all required fields, the test will fail with schema validation error instead of testing the migration logic." But the test's purpose is to verify the loader handles *missing* `usage` and `seen_flags`. If the developer adds every required field, the test works. But the comment warns that missing other required fields breaks the test before the migration check runs, which means the spec is internally consistent but relies on the developer doing the scan correctly. This is flagged as a risk but not a blocker—if the developer follows the instruction ("Replace with ALL required State fields from schema scan"), the test works. The spec is sound; the risk is in execution.

**3. Story 3 Test 6 risks circular mocking**

Story 3's LLD Test 6 (`test_forced_hop_status_matches_model_hop`) compares forced hop and model hop outcomes. The spec explicitly warns against circular mocking: "This test does NOT mock `_decide` to return forced hop values (which would be circular)." The test design runs two paths: forced (route forced, no model call) and model (route None, `_decide` called). The model path mocks `_decide` to call `_compute_status_for_route`, `_compute_priority_for_route`, `_compute_confidence_for_route`. But those functions are the same ones `_record_forced_hop` calls (per Story 3 spec section "Forced Hop Recording"). So the test compares `forced_result.status` (from `_compute_status_for_route(["planner"], initial_state)`) with `model_result.status` (from `_compute_status_for_route(["planner"], initial_state)`). They will always match because they call the same function with the same arguments. The test proves nothing—it's a tautology disguised as a comparison. A developer following this spec would write a test that always passes but never catches a bug where forced hops compute status differently from model hops.

**4. Story 3's API response extension lacks a concrete nesting decision**

The HLD (AD-5) says usage totals go "in the response payload under a `total_usage` key" and "The run status API includes the same totals in the response payload." Story 3's spec says totals nest in `values.total_usage`, but then hedges: "Developer must verify the existing response structure before nesting `total_usage`." The spec provides a placeholder implementation with `return {"values": {"total_usage": {...}}}`, but Test 4 asserts `data["values"]["total_usage"]` without resolving the open question. If the actual API response has `values` at a different level or uses a different structure, the test and implementation will both be wrong. The spec defers the decision to the developer but writes tests that assume one structure, so the developer would either rewrite the test after discovering the real structure (wasted work) or implement the wrong structure to match the test (wrong behavior).

**5. Breakdown and stories disagree on file touch counts**

Breakdown Story 1 says "Allowed paths (see LLD for exact locations once repo-tree is mapped)" and lists 8 file categories: State schema, State loader, Skill client, Tool bindings, Model gateway, Node update logic, UsageStats definition, Test file. Story 1's LLD "File Plan" lists the same 8 under "Files to Create" (2) and "Files to Modify" (6), total 8. But the breakdown says Story 1 modifies files in "State schema definition (add `usage` field, implement reducer)," which is one file, then separately "State loader (handle missing `usage`)," which might be the same file or a different one depending on the codebase structure. The breakdown treats "State schema definition" and "State loader" as distinct touch points, but the LLD lists "State schema module" and "State loader module" as separate placeholders, assuming they're different files. If they're the same file in the actual codebase, the count is off by one. This is a minor discrepancy, but it means the developer might mark the story done after touching 7 files instead of 8, missing the loader change.

## Notes for the gate

Story 3 Test 6 is a tautology—it will always pass because both paths compute status/priority/confidence using the same functions with the same arguments. The test should either:
- Compare outcomes from different initial states where the functions produce different results, OR
- Verify the *call pattern* (assert `_compute_status_for_route` was called with expected args) rather than comparing outputs.

The API response nesting question (Story 3, open question 1) is resolved in the spec's placeholder implementation but not in the HLD. The architect should either confirm `values.total_usage` is the right structure or update Story 3's tests and implementation to match the actual API response after a codebase scan. Leaving it as "TBD" with tests that assume one structure creates a trap.

VERDICT: FAIL
