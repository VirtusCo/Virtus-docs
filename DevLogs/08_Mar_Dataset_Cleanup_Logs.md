# 08 Mar 2026 — Tool-Use Dataset Cleanup & Full 12K Validation

## Scope
- Clean and validate AI training data quality issues in JSONL datasets.
- Prioritize high-confidence semantic mismatches in tool-use records, then scale to full-dataset verification.

## Files Covered
- `src/porter_ai_assistant/data/tool_use/train_cleaned.jsonl` (3000)
- `src/porter_ai_assistant/data/tool_use/eval_cleaned.jsonl` (600)
- `src/porter_ai_assistant/data/conversational/train.jsonl` (7000)
- `src/porter_ai_assistant/data/conversational/eval.jsonl` (1400)

## Work Completed

### 1) Manual + Hybrid Start (quality-first)
- Began with line-level manual corrections for early problematic samples.
- Fixed mismatches between:
  - tool arguments and returned tool output,
  - tool output and final assistant response,
  - departed-flight status and gate-escort recommendations.
- Preserved strict JSONL validity after each batch.

### 2) Automated Cleanup (speed + consistency)
- Implemented deterministic auto-fix rules and executed full-file sweeps.
- Applied these high-confidence rules:
  1. `get_transport_options`: if tool returns multiple options, force `transport_type` to `"any"`.
  2. `get_flight_status`: if status is `Departed`, remove/replace any suggestion to go to gate.
  3. Gate mismatch safeguard: if assistant final text references wrong gate vs tool output, replace with tool gate.

## Fix Statistics
- Historical changes applied in this session flow:
  - 156 records changed in tool-use train split.
  - 125 `transport_type_to_any`
  - 31 `departed_gate_advice_fixed`
  - 0 `gate_mismatch_corrected` (rule checked; no remaining matches after prior fixes)

## Validation Results
- JSON parse validation passed for all splits:
  - tool_use train: 3000/3000 valid
  - tool_use eval: 600/600 valid
  - conversational train: 7000/7000 valid
  - conversational eval: 1400/1400 valid
- Remaining high-confidence heuristic issues after full sweep: **0** across all 12,000 records.

## Audit Artifacts
- Per-run and global audit logs generated:
  - `src/porter_ai_assistant/data/tool_use/auto_fix_audit.csv`
  - `src/porter_ai_assistant/data/global_auto_fix_audit.csv`

## Final Status
- Dataset cleanup phase (targeted high-confidence semantic classes) is complete.
- Full 12K dataset was scanned and validated.
- Ready for downstream training/evaluation pipeline usage.
