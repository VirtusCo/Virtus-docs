# 08 Mar 2026 — AI Model Switch: Gemma 3 270M → Qwen 2.5 1.5B Instruct

## Scope
- Replace Gemma 3 270M IT with Qwen 2.5 1.5B Instruct across the entire `porter_ai_assistant` codebase.
- Update all config defaults, training scripts, download registry, tests, docs, and YAML configs.
- Prepare for retraining (code only — training not started yet).

## Motivation

The Gemma 3 270M IT model, while fast and tiny (~253 MB Q4_K_M), showed fundamental limitations during GUI integration testing:

| # | Issue | Root Cause |
|---|-------|------------|
| 1 | Mixed-language output (Spanish + Chinese bleed) | 270M can't suppress non-English tokens from multilingual pretraining |
| 2 | Fabricated specifics ("100 meters ahead", "Level 2") | No real airport data — hallucinated locations |
| 3 | Incorrect topic grounding (restroom → shower) | 270M parameter budget too small for fine-grained concept separation |
| 4 | Identity instability ("Virtus", "CiaoBlade", "LIDBot") | System prompt gets overridden by pretraining noise |
| 5 | No profanity handling | Model tries to be helpful instead of redirecting |
| 6 | Off-topic requests not rejected | Can't stay in airport domain scope |
| 7 | Hallucinated capabilities | Invents features not in system prompt |

**Decision:** Switch to **Qwen 2.5 1.5B Instruct** — community gold standard for sub-2B models, much stronger instruction following, native tool calling support, better multilingual control.

## Model Comparison

| Property | Gemma 3 270M IT (old) | Qwen 2.5 1.5B Instruct (new) |
|----------|----------------------|-------------------------------|
| Parameters | 270M | 1.5B (~5.5× larger) |
| Q4_K_M GGUF size | ~253 MB | ~1.0 GB |
| Estimated RSS | ~351 MB | ~1.1–1.5 GB |
| Max context | 8K (used 768) | 32K (using 2048) |
| HF repo (base) | `google/gemma-3-270m-it` | `Qwen/Qwen2.5-1.5B-Instruct` |
| GGUF source | `unsloth/gemma-3-270m-it-GGUF` | `Qwen/Qwen2.5-1.5B-Instruct-GGUF` |
| Chat template | Gemma (`<start_of_turn>`) | ChatML (`<\|im_start\|>`) |
| RPi 4 (4 GB) fit | ✅ Comfortable | ⚠️ Tight (~2.6 GB total with stack) |
| RPi 5 (8 GB) fit | ✅ Plenty | ✅ Comfortable |
| Instruction following | Poor | Strong |
| Tool calling | Needs LoRA | Native support + LoRA boost |

## Files Modified (15 files)

### Core Source (4 files)
| File | Changes |
|------|---------|
| `porter_ai_assistant/config.py` | `DEFAULT_MODEL_FILENAME` → `qwen2.5-1.5b-instruct-q4_k_m.gguf`, `DEFAULT_N_CTX` 768→2048, `DEFAULT_TOP_K` 64→50, `MODEL_SELECTION_NOTES` rewritten, rationale block updated |
| `porter_ai_assistant/inference_engine.py` | `ModelConfig` defaults: temp 1.0→0.7, top_p 0.95→0.9, top_k 64→50, n_ctx 768→2048. Removed "FunctionGemma" from docstrings |
| `setup.py` | Description → "Qwen 2.5 1.5B GGUF" |
| `package.xml` | Description → "Qwen 2.5 1.5B GGUF" |

### Training Scripts (5 files)
| File | Changes |
|------|---------|
| `scripts/finetune.py` | `base_model` → `Qwen/Qwen2.5-1.5B-Instruct`, docstring, CLI default, pad token comment |
| `scripts/convert_to_gguf.py` | `--base-model` default → Qwen, QUANT_TYPES descriptions updated for 1.5B sizes |
| `scripts/download_model.py` | Added `qwen2.5-1.5b` + `qwen2.5-0.5b` entries, kept `gemma-3-270m` as legacy, CLI default → `qwen2.5-1.5b` |
| `scripts/rl_train.py` | `base_model` → `Qwen/Qwen2.5-1.5B-Instruct` |
| `scripts/dpo_train.py` | `base_model` → Qwen, updated size comments (1.5B bf16 ≈ 3 GB) |

### Utility Scripts (3 files)
| File | Changes |
|------|---------|
| `scripts/ai_server.py` | Docstring model path updated |
| `scripts/benchmark.py` | Docstring model paths, removed "Unsloth Gemma 3" from temperature help |
| `scripts/inference_test.py` | Comment "270M model" → "small models" |
| `scripts/generate_dataset.py` | 4× "FunctionGemma" refs → model-agnostic phrasing |

### Config & Data (3 files)
| File | Changes |
|------|---------|
| `config/assistant_params.yaml` | model_path, generation params (temp=0.7, top_p=0.9, top_k=50), n_ctx=2048, source URL |
| `data/system_prompts.yaml` | Adapter comments → Qwen 2.5, chat template format → ChatML |
| `data/README.md` | Base model table → Qwen 2.5, "FunctionGemma" → model-agnostic |

### Tests (1 file)
| File | Changes |
|------|---------|
| `test/test_assistant.py` | Assertions: filename→`qwen2.5-1.5b`, ctx→2048, top_k→50, size→1000, temp→0.7. Renamed `test_gemma3_defaults`→`test_qwen25_defaults` |

## Key Config Changes Summary

| Setting | Old (Gemma) | New (Qwen) | Reason |
|---------|-------------|------------|--------|
| `DEFAULT_MODEL_FILENAME` | `gemma-3-270m-it-Q4_K_M.gguf` | `qwen2.5-1.5b-instruct-q4_k_m.gguf` | New model |
| `DEFAULT_N_CTX` | 768 | 2048 | Qwen supports 32K natively; 2048 gives better multi-turn |
| `DEFAULT_TEMPERATURE` | 0.7 | 0.7 | Already grounded (unchanged) |
| `DEFAULT_TOP_P` | 0.9 | 0.9 | Already balanced (unchanged) |
| `DEFAULT_TOP_K` | 64 | 50 | Balanced default for Qwen |
| `base_model` (HF) | `google/gemma-3-270m-it` | `Qwen/Qwen2.5-1.5B-Instruct` | New base for training |

## Intentionally Kept Gemma References

1. **`config.py` rationale comments** (lines 63, 70, 73) — historical context explaining *why* we switched.
2. **`download_model.py` legacy entry** — `gemma-3-270m` marked as "legacy (replaced by Qwen 2.5)" so old models can still be downloaded.
3. **LoRA adapter metadata** (`models/lora_adapters/*/`) — will be auto-regenerated when retraining; no manual edit needed.

## Test Results

```
20 passed in 0.02s
```

All 20 unit tests pass with updated Qwen values.

## Next Steps

1. **Download Qwen 2.5 1.5B GGUF** — `python3 scripts/download_model.py --model qwen2.5-1.5b --quant Q4_K_M`
2. **Retrain LoRA adapters** — `python3 scripts/finetune.py --adapter both` (on RTX 5070)
3. **Convert LoRA to GGUF** — `python3 scripts/convert_to_gguf.py --adapter both`
4. **Benchmark on dev machine** — `python3 scripts/benchmark.py --model <gguf_path>`
5. **Test on RPi 5** — validate RSS < 2 GB, latency < 3s
