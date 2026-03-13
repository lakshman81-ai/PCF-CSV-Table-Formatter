# PCF Data Pipeline Benchmark Plan (BM_Plan.md)

This plan details the exact procedure for running the calculation, validation, smart-fixing, and export benchmarks. Every benchmark must be run one by one to ensure zero errors and that fixes are legitimately applied in the data, not just reported in the UI logs.

## 1. Execution Methodology

For each benchmark test in the suite, the pipeline will perform the following steps:

1.  **Ingestion & Parsing:** Load the input points (PTE) or elements (CSV) to generate the initial Data Table.
2.  **Validation Check (Pre-Fix):** Run the Validator engine (`validator.js`). Assert that the correct errors/warnings are flagged.
3.  **Basic/Smart Fix (Action):** Run the Fixer engine (`basicFixer.js` / `smartFixer`). Assert that the "Fixing Action" log is generated *and* the underlying Data Table values are correctly mutated according to the rule constraint.
4.  **Export PCF:** Generate the final PCF text string from the fixed Data Table.
5.  **Re-Validation Check (Post-Fix):** Run the Validator engine *again* on the newly exported/re-imported data. Assert that **zero** errors remain.
6.  **Output Matching:** Verify the final exported string exactly matches the expected strings/conditions in `benchmark_data.json` without missing components, broken coordinates, or invalid syntax.

**Crucial Validation Requirement:**
We are testing *implementation*, not just log strings. If a benchmark expects a "Fix", we must explicitly assert `row.cp.x !== old_x` (for example) to confirm the data table was physically altered.

## 2. Benchmark Categories & Targets

### Group A: PTE Pipeline (BM-PTE-01 to BM-PTE-30)
**Focus:** Converts raw input points into valid pipeline components.
- Validate that the correct geometry (Flanges, Pipes, Supports, Branches) is created based on the source columns (`Point`, `PPoint`, `Line_Key`, etc.).
- Example: `BM-PTE-01` must explicitly discard physical component generation for chain-starting branches.

### Group B: Data Table Syntax Fixes (BM-SF-01 to BM-SF-20)
**Focus:** Correcting missing identifiers, coordinate normalization, and bore synchronization.
- **BM-SF-13 (SUPPORT Bore):** Ensure missing or `null` bores are explicitly set to `0`. Assert `row.bore === 0`.
- **BM-SF-09 (TEE Bore Sync):** Ensure the TEE's Center-Point (CP) bore perfectly matches the End-Point (EP) bore.
- **BM-SF-Tee-CP (Corner/Perpendicular Intersections):** Ensure axis-aligned 90-degree TEEs do not simply take the midpoint `(ep1+ep2)/2`, but calculate the correct orthogonal intersection with the Branch-Point (BP).

### Group C: Smart Chaining & Gaps (BM-SF-21 to BM-SF-55)
**Focus:** Micro-pipe removal, foldback penalties, and connection tolerances.
- Validate that pipes under 6mm are absorbed into adjacent components without breaking the chain sequence.
- Validate that foldbacks (overlapping pipes $> 25mm$ with reversed EP2s) are accurately flagged and deleted.

### Group D: PCF Export (BM-PCF-01 to BM-PCF-05)
**Focus:** Final text formatting.
- Ensure all line endings are strictly CRLF (`\r\n`).
- Ensure component names map to correct `<SKEY>` identifiers (e.g., `FLANGE` -> `FLWN`).
- Ensure no Data Table columns are randomly dropped if `config.pte.sequentialData` is enabled.

## 3. Implementation Steps

1.  **Refactor Test Runner:** Update `src/tests/benchmarks.js` to execute the explicit Validation -> Fix -> Re-Validation -> Export cycle described above. Currently, it runs the engine but does not systematically re-validate the final state to guarantee "zero errors".
2.  **Add Data Assertions:** Inject specific `expect(fixedRow.property).toBe(expectedValue)` checks into the benchmark definitions instead of solely relying on the `"action": "FIX"` string check.
3.  **Execute & Verify:** Run `npx vitest run benchmarks.test.js`. If any test fails the post-fix re-validation phase, the logic implementation is flawed and must be corrected in the core engine before proceeding.
