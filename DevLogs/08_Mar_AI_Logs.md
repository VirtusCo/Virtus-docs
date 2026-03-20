# Porter Robotics – Developer Log

### **Date:** 08 Mar 2026

### **Engineer:** Claude (AI) + Antony Austin (review)

### **Subsystem:** AI Assistant — Phase 4.5, Tasks 18d–18f + Virtue Rename

---

## 1. Summary

Completed all remaining AI Assistant subtasks (18d, 18e, 18f) and renamed the
AI persona from "Porter" to **"Virtue"**. The on-device AI assistant is now
fully functional with base GGUF model + modular LoRA adapters loaded at runtime.

**Key results:**
- **Task 18d — GGUF Quantization:** Base GGUF (Unsloth Q4_K_M, 241 MB) + separate GGUF LoRA adapters (7.3 MB each) loaded at runtime via llama-cpp-python `lora_path`
- **Task 18e — ROS 2 Service Node:** Added `/porter/ai_query` topic subscriber for GUI integration, unified model params
- **Task 18f — Benchmarks:** Conversational mean 497ms (100% <2s), tool_use mean 250ms (100% <2s), 351 MB RSS
- **Virtue Rename:** AI persona renamed across 16 files (system prompts, training data, source code, tests)
- **Critical lesson:** `merge_and_unload()` on 4-bit QLoRA models degrades weights — use base GGUF + runtime LoRA instead
- **All 20 unit tests pass**, 0 lint errors

---

## 2. Task 18d — GGUF Quantization

### 2.1 The merge_and_unload() Problem

Initial approach: merge QLoRA adapter into base model → convert to GGUF. This failed because
`merge_and_unload()` on a 4-bit quantized model:
1. Dequantizes weights (lossy — NF4 → bf16 is not lossless)
2. Merges LoRA deltas onto degraded weights
3. Resulting model produces worse output than base model

**Solution architecture:**
```
Base GGUF (Unsloth Q4_K_M, 241 MB)     ← Downloaded pre-quantized
  + LoRA adapter GGUF (7.3 MB each)    ← Converted from safetensors
  = Runtime loading via lora_path       ← llama-cpp-python handles merge in memory
```

### 2.2 GGUF LoRA Conversion

Used `llama.cpp/convert_lora_to_gguf.py` to convert safetensors LoRA adapters to GGUF format:
- `porter-conversational-lora-f16.gguf` — 7.3 MB
- `porter-tool_use-lora-f16.gguf` — 7.3 MB

Key issues fixed during conversion:
- **Vocab assertion:** Tokenizer had 262,145 tokens vs model 262,144 — fixed with embedding resize
- **Pre-tokenizer hash:** Missing `tokenizer.model` file — fixed with auto-copy from HF cache
- **llama.cpp CUDA build:** sm_120a (Blackwell RTX 5070) unsupported — built CPU-only (sufficient for convert/quantize)

### 2.3 Inference Engine Refactor

Refactored `inference_engine.py` for the new architecture:
- `ModelConfig.lora_dir` — directory containing GGUF LoRA adapters
- `_discover_lora_adapters()` — auto-discovers available adapters
- `switch_adapter()` — hot-swap LoRA adapters without reloading base model
- `query()` — auto-routes to conversational vs tool_use adapter based on keyword classification
- `n_ctx=768` unified across all config files

### 2.4 Files Under models/gguf/

| File | Size | Git Tracking |
|------|------|-------------|
| `gemma-3-270m-it-Q4_K_M.gguf` | 241 MB | gitignored (download via `download_model.py`) |
| `porter-conversational-lora-f16.gguf` | 7.3 MB | Git LFS tracked |
| `porter-tool_use-lora-f16.gguf` | 7.3 MB | Git LFS tracked |

---

## 3. Task 18e — ROS 2 Service Node Updates

Added `/porter/ai_query` subscriber (String) for topic-driven GUI queries:
- GUI publishes query string → node processes → publishes response on `/porter/ai_response`
- Unified params: `model_path`, `lora_dir`, `default_adapter` (replaces 4 separate params)
- Services: `~/query` (Trigger), `~/get_status` (Trigger)

---

## 4. Task 18f — Integration Benchmarks

Ran benchmarks with `--lora` flag for adapter-specific testing:

| Adapter | Mean Latency | P95 | P99 | Max | 100% <2s | RSS |
|---------|-------------|-----|-----|-----|----------|-----|
| Conversational | 497 ms | 816 ms | 965 ms | 1.1s | ✅ | 351 MB |
| Tool-Use | 250 ms | 390 ms | 422 ms | 450 ms | ✅ | 351 MB |

All within the <2s target for RPi 4 deployment. Tool-use is faster due to shorter structured output.

---

## 5. Virtue AI Rename

Renamed the AI assistant persona from "Porter" to "Virtue" across the entire codebase.

### What was renamed (AI persona → "Virtue"):
- **System prompts** (system_prompts.yaml): All 13 prompts — "You are Virtue, a smart airport assistant..."
- **Training data**: "Hey Porter" → "Hey Virtue" (574 occurrences), "I'm Porter" → "I'm Virtue" (196 occurrences)
- **Source code**: prompt_templates.py, inference_engine.py (fallback prompts)
- **Scripts**: benchmark.py, inference_test.py, generate_dataset.py
- **Tests**: 4 assertions checking for 'Virtue' instead of 'Porter'
- **Documentation**: data/README.md

### What was kept as "Porter" (robot product name):
- "Porter's built-in scale", "Porter's screen" — hardware references in tool descriptions
- "Porter robots around the terminal" — product references in assistant responses
- Package names (`porter_ai_assistant`) — this is the ROS 2 package name, not the AI persona

### Retraining Note
Existing LoRA adapters were trained on "Porter" data. The inference-time system prompt now says
"Virtue", which partially helps, but full retraining on updated data is recommended for the model
to consistently self-identify as "Virtue" in free-form responses.

---

## 6. Files Modified

### Task 18d (GGUF Quantization)
- `porter_ai_assistant/inference_engine.py` — lora_dir, adapter discovery, switch_adapter
- `porter_ai_assistant/assistant_node.py` — unified model_path + lora_dir params
- `porter_ai_assistant/config.py` — DEFAULT_LORA_DIR, DEFAULT_ADAPTER, DEFAULT_N_CTX
- `config/assistant_params.yaml` — updated param names and defaults
- `scripts/download_model.py` — output dir updated to models/gguf/
- `scripts/convert_to_gguf.py` — embedding resize fix, tokenizer.model auto-copy
- `test/test_assistant.py` — n_ctx assertions updated to 768

### Task 18e (ROS 2 Service Node)
- `porter_ai_assistant/assistant_node.py` — `/porter/ai_query` subscriber

### Task 18f (Benchmarks)
- `scripts/benchmark.py` — `--lora` flag, updated defaults

### Virtue Rename (16 files)
- `data/system_prompts.yaml`, `data/README.md`
- All 8 JSONL training data files
- `porter_ai_assistant/prompt_templates.py`, `porter_ai_assistant/inference_engine.py`
- `scripts/benchmark.py`, `scripts/inference_test.py`, `scripts/generate_dataset.py`
- `test/test_assistant.py`

---

## 7. Lessons Learned

| # | Lesson | Detail |
|---|--------|--------|
| 1 | Never merge_and_unload() on 4-bit QLoRA | Dequantization is lossy. Use base GGUF + runtime LoRA adapter loading instead. |
| 2 | GGUF vocab size must match | If tokenizer has N+1 tokens vs model N, resize embeddings before conversion. |
| 3 | llama.cpp sm_120a not supported | Blackwell (RTX 5070) too new for current llama.cpp CUDA — CPU-only build works for conversion/quantization. |
| 4 | Unify n_ctx across all configs | Had 768 in code but 512 in YAML and tests — caused confusion. Single source of truth in config.py. |
| 5 | AI persona name ≠ product name | "Virtue" is the AI assistant. "Porter" is the robot product. Keep these separate in training data. |
| 6 | DPO fails with fresh LoRA | Zero-initialized LoRA B matrix means policy = reference → zero gradients. Need pre-trained adapter or full fine-tune. |

---

## 8. Git Commits

| Commit | Message |
|--------|---------|
| `7e32559` | feat(ai-assistant): Task 18d — GGUF quantization with runtime LoRA adapters |
| `6035cfe` | feat(ai-assistant): Task 18e — add /porter/ai_query subscription |
| `f28d99e` | feat(ai-assistant): Task 18f — integration benchmarks with LoRA adapters |
| `201e1f1` | feat(ai-assistant): rename AI persona from Porter to Virtue |

---

## 9. Current State

- **Phase 4.5 complete** — AI assistant is functional with base GGUF + LoRA adapters
- **20/20 tests pass**, all benchmarks within targets
- **Next steps:**
  - Retrain LoRA adapters on Virtue-named data (optional but recommended)
  - LangChain orchestrator for GUI ↔ AI ↔ tools integration (Phase 5 prerequisite)
  - GUI display integration
