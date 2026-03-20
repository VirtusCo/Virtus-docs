
# Porter Robotics – Developer Log

### **Date:** 07 Mar 2026

### **Engineer:** Claude (AI) + Antony Austin (review)

### **Subsystem:** AI Assistant — Phase 4.5, Tasks 18a–18c

---

## 1. Summary

Built the `porter_ai_assistant` ROS 2 package scaffolding and completed the first
two subtasks of Phase 4.5: dataset curation (18a) and model selection (18b).
Completed Task 18c: QLoRA fine-tuning of both conversational and tool_use LoRA adapters.
The package provides an on-device AI assistant for airport passengers using a
small language model (Gemma 3 270M IT) running via llama-cpp-python on RPi 4/5.

**Key results:**
- **12,000 training examples** curated: 7K conversational (14 categories) + 3K tool-use (13 tool types) + 2K eval
- **Model selected:** Gemma 3 270M IT from `unsloth/gemma-3-270m-it-GGUF` — Q4_K_M (253 MB)
- **2 LoRA adapters trained** via QLoRA on RTX 5070 Laptop GPU
  - **Conversational:** 26.7 min, eval_loss 0.152, token accuracy 95.0%
  - **Tool_use:** 23.2 min, eval_loss 0.0001, token accuracy 100%
- **22/22 tests pass** (20 unit tests + flake8 + pep257), 0 lint errors
- **5 automation scripts** ready: finetune.py, convert_to_gguf.py, download_model.py, benchmark.py, generate_dataset.py
- **ROS 2 service node** coded: `~/query`, `~/get_status`, `/porter/ai_response` topic
- **Docker updated:** porter_ai_assistant included in build, AI venv with llama-cpp-python 0.3.16

---

## 2. Task 18a — Dataset Curation

### 2.1 Dataset Generator (`scripts/generate_dataset.py`)

Built a comprehensive dataset generator (1897 lines) that produces airport-domain
Q&A examples for fine-tuning. Generates JSONL files in Gemma 3 chat format with
`<start_of_turn>` / `<end_of_turn>` markers.

**Conversational categories (14):**
- Gates & terminals, flight information, directions & wayfinding, dining & restaurants
- Shopping & retail, transportation, lounges & clubs, services & facilities
- Security & customs, accessibility, weather & travel tips, hotel & accommodation
- Entertainment & activities, emergency & safety

**Tool-use categories (13):**
- check_flight_status, find_gate, get_directions, search_restaurants
- search_shops, check_lounge_access, find_service, get_weather
- search_hotels, find_transport, check_accessibility, emergency_contact, search_entertainment

**Output structure:**
```
data/
├── conversational/
│   ├── train.jsonl      (7,000 examples)
│   └── eval.jsonl       (1,400 examples)
├── tool_use/
│   ├── train.jsonl      (3,000 examples)
│   └── eval.jsonl       (600 examples)
└── combined/
    ├── train.jsonl      (10,000 examples)
    └── eval.jsonl       (2,000 examples)
```

### 2.2 Supporting Data Files

- `data/system_prompts.yaml` — 4 system prompt variants (standard, concise, detailed, multilingual)
- `data/tool_schemas.json` — 13 tool definitions with JSON Schema parameters

---

## 3. Task 18b — Model Selection

### 3.1 Decision: Gemma 3 270M IT

| Criterion | Gemma 3 270M IT | Gemma 2B |
|-----------|----------------|----------|
| Parameters | 270M | 2B |
| GGUF Q4_K_M | 253 MB | ~1.2 GB |
| RAM at inference | ~400 MB | ~2 GB |
| RPi 4 feasible | ✅ Comfortable | ⚠ Tight on 4GB model |
| Fine-tune VRAM | ~2 GB (QLoRA) | ~6 GB (QLoRA) |
| Latency target | < 2s on RPi 4 | > 2s on RPi 4 |

**Rationale:** The 270M model fits comfortably in RPi 4's 4 GB RAM alongside ROS 2 + Nav2.
With airport-domain LoRA fine-tuning, it should match the 2B base model for our narrow domain.

### 3.2 Download Script (`scripts/download_model.py`)

Automated downloader using `huggingface_hub` — downloads GGUF variants:
- Q4_K_M (253 MB) — primary deployment target
- Q8_0 (283 MB) — higher quality fallback
- F16 (540 MB) — maximum quality for benchmarking

### 3.3 Benchmark Script (`scripts/benchmark.py`)

Performance benchmark tool measuring:
- Token generation latency (mean, p50, p95, p99)
- Tokens per second throughput
- Memory usage (RSS, peak)
- First-token latency vs subsequent
- Exports JSON report for comparison

---

## 4. Package Architecture

### 4.1 Core Modules

| File | Lines | Purpose |
|------|-------|---------|
| `assistant_node.py` | 350 | ROS 2 node: `~/query` service, `~/get_status`, `/porter/ai_response` |
| `inference_engine.py` | 469 | llama-cpp-python wrapper, model loading, LoRA support, health checks |
| `config.py` | 121 | Gemma 3 270M defaults, generation params (Unsloth-recommended) |
| `prompt_templates.py` | 142 | System prompts, conversation formatting, adapter routing |

### 4.2 ROS 2 Interfaces

| Interface | Type | Description |
|-----------|------|-------------|
| `~/query` | Service (Trigger) | Query the AI — request text in, response text + latency out |
| `~/get_status` | Service (Trigger) | Model status: loaded, memory, avg latency |
| `/porter/ai_response` | Topic (String) | Published response for GUI display |
| `/diagnostics` | Topic (DiagnosticArray) | Model health, inference stats |

### 4.3 Generation Parameters (Unsloth-recommended)

```yaml
temperature: 1.0
top_k: 64
top_p: 0.95
min_p: 0.0
max_tokens: 256
repeat_penalty: 1.0
```

---

## 5. Fine-Tuning Pipeline (ready, not yet executed)

### 5.1 `scripts/finetune.py` (515 lines)

QLoRA fine-tuning pipeline using:
- **Base model:** `google/gemma-3-270m-it` from HuggingFace
- **Method:** QLoRA 4-bit (bitsandbytes NF4)
- **LoRA config:** r=16, alpha=32, dropout=0.05, target modules: q_proj, k_proj, v_proj, o_proj
- **Training:** SFTTrainer from TRL, 3 epochs, batch=4, gradient_accum=4
- **Hardware target:** RTX 5070 (8 GB VRAM, Blackwell sm_120)
- **Produces:** LoRA adapter checkpoints in `models/lora-<adapter_name>/`

### 5.2 `scripts/convert_to_gguf.py` (390 lines)

Post-training pipeline:
1. Merge LoRA adapter into base model
2. Convert to GGUF format via llama.cpp
3. Quantize to Q4_K_M / Q8_0 / F16
4. Validate output file

### 5.3 Training Environment

```
Hardware:  RTX 5070 (8 GB VRAM), Ryzen 9 8940HX, 30 GB RAM
OS:        Ubuntu 24.04
PyTorch:   Nightly cu128 (for Blackwell sm_120 support)
Venv:      .venv-finetune (gitignored via .venv-*/ pattern)
Packages:  torch, transformers, peft, trl, bitsandbytes, datasets, accelerate
```

---

## 6. Docker Integration

### 6.1 Changes Made

- **Dockerfile.dev:** Added `porter_ai_assistant/package.xml` to COPY manifest section,
  added `--skip-keys="ament_python"` to rosdep, AI venv with llama-cpp-python 0.3.16
- **Dockerfile.prod:** Same package.xml and rosdep fixes, AI venv in runtime stage
- **docker-compose.dev.yml:** Added `ai_models` named volume for persistent GGUF storage
- **docker-entrypoint.sh:** Added AI venv PATH setup (`PORTER_AI_VENV` env var)

### 6.2 Verified

```
✅ docker compose -f docker/docker-compose.dev.yml build    — 5 packages, 0 errors
✅ All 5 packages discoverable: ydlidar_driver, porter_lidar_processor,
   porter_orchestrator, porter_esp32_bridge, porter_ai_assistant
✅ 8 node executables registered across all packages
✅ llama-cpp-python 0.3.16 importable in AI venv
✅ 22/22 porter_ai_assistant tests pass inside container
```

---

## 7. Test Results

```
porter_ai_assistant: 22 tests
  - test_assistant.py: 20 unit tests (config, prompts, engine mock, node mock)
  - test_flake8.py: 1 test (0 violations)
  - test_pep257.py: 1 test (0 violations)

Total project tests: 269
  - 209 colcon tests (5 packages)
  - 60 Ztest unit tests (native_sim)
  - 0 failures, 12 skipped
```

---

## 8. Remaining Work (Phase 4.5)

| Task | Status | Description |
|------|--------|-------------|
| 18a — Dataset curation | ✅ Done | 12K examples across 14+13 categories |
| 18b — Model selection | ✅ Done | Gemma 3 270M IT, Q4_K_M primary |
| 18c — LoRA fine-tune | ⬜ Next | Execute finetune.py on RTX 5070 |
| 18d — Quantization | ⬜ Blocked | Needs 18c output (LoRA adapter) |
| 18e — ROS 2 node | ✅ Code done | Needs trained model to function |
| 18f — Integration | ⬜ Blocked | Needs 18d output (GGUF model) |

**Next step:** Run `finetune.py` with `.venv-finetune` active to produce LoRA adapters,
then `convert_to_gguf.py` to produce deployable GGUF files for RPi 4.

---

## 9. Files Created / Modified

### New Files (33 total)

```
src/porter_ai_assistant/
├── package.xml
├── setup.py
├── setup.cfg
├── resource/porter_ai_assistant
├── porter_ai_assistant/
│   ├── __init__.py
│   ├── assistant_node.py
│   ├── inference_engine.py
│   ├── config.py
│   └── prompt_templates.py
├── scripts/
│   ├── finetune.py
│   ├── convert_to_gguf.py
│   ├── download_model.py
│   ├── benchmark.py
│   └── generate_dataset.py
├── data/
│   ├── system_prompts.yaml
│   ├── tool_schemas.json
│   ├── conversational/train.jsonl
│   ├── conversational/eval.jsonl
│   ├── tool_use/train.jsonl
│   ├── tool_use/eval.jsonl
│   ├── combined/train.jsonl
│   └── combined/eval.jsonl
├── models/.gitkeep
├── launch/assistant_launch.py
├── config/assistant_params.yaml
└── test/
    ├── test_assistant.py
    ├── test_flake8.py
    └── test_pep257.py
```

### Modified Files

```
docker/Dockerfile.dev          — Added AI assistant package.xml, AI venv, --skip-keys
docker/Dockerfile.prod         — Added AI assistant package.xml, AI venv, --skip-keys
docker/docker-compose.dev.yml  — Added ai_models volume
docker/docker-entrypoint.sh    — Added AI venv PATH setup
.gitignore                     — Added .venv-*/ pattern
.gitattributes                 — Added LFS tracking for model files
CLAUDE.md                      — Updated repo layout, task status, references
OBJECTIVES.md                  — Updated Phase 4.5, tech decisions 18-21
```
---

## 6. Task 18c — QLoRA Fine-Tuning

### 6.1 Setup

- **GPU:** NVIDIA GeForce RTX 5070 Laptop GPU, 8.1 GB VRAM
- **CUDA:** 13.0, Driver 580.126.09
- **PyTorch:** 2.12.0.dev20260307+cu128 (nightly for Blackwell sm_120)
- **Stack:** transformers 5.3.0, peft 0.18.1, trl 0.29.0, bitsandbytes 0.49.2
- **Venv:** `.venv-finetune` (gitignored)
- **HuggingFace Auth:** Required for gated `google/gemma-3-270m-it` model

### 6.2 QLoRA Configuration

| Parameter | Value |
|-----------|-------|
| Quantization | 4-bit NF4, double quant, bf16 compute |
| LoRA rank | 16 |
| LoRA alpha | 32 |
| LoRA dropout | 0.05 |
| Target modules | q/k/v/o_proj, gate/up/down_proj |
| Trainable params | 3.80M / 222M total (1.71%) |
| Optimizer | paged_adamw_8bit |
| LR schedule | Cosine, 2e-4 peak, 3% warmup |
| Effective batch | 16 |
| Max seq length | 512 |
| Epochs | 3 |

### 6.3 Issues Resolved

| # | Issue | Fix |
|---|-------|-----|
| 1 | TRL 0.29.0 removed `max_seq_length` from `SFTConfig` | Changed to `max_length` |
| 2 | TRL 0.29.0 removed `tokenizer` from `SFTTrainer` | Changed to `processing_class` |
| 3 | Tool_use dataset `tool` role breaks Gemma 3 chat template | Merged `assistant→tool→assistant` into single assistant message with `<tool_response>` tags |
| 4 | CUDA OOM with batch=8, no gradient checkpointing | Reverted to batch=4+grad_checkpoint for conversational; batch=2+grad_checkpoint for tool_use (262K vocab creates 2 GB logits) |
| 5 | `device_map="auto"` caused CPU offloading | Changed to `{"": 0}` to force all layers on GPU 0 |

### 6.4 Training Results

#### Conversational Adapter

| Metric | Value |
|--------|-------|
| Training time | 26.7 min |
| Steps | 1314 |
| Batch config | 4 × 4 = 16 effective |
| Final train loss | 0.316 (avg all epochs) |
| Best eval loss | **0.152** |
| Final token accuracy | **95.0%** |
| Loss trajectory | 4.88 → 0.65 → 0.19 → 0.14 |
| Eval trajectory | 0.31 → 0.22 → 0.19 → 0.15 |

Learning curve: Rapid improvement in epoch 1 (88% → 95%), gradual refinement in epochs 2-3. No overfitting — eval loss tracked train loss throughout.

#### Tool_use Adapter

| Metric | Value |
|--------|-------|
| Training time | 23.2 min |
| Steps | 564 |
| Batch config | 2 × 8 = 16 effective |
| Final train loss | 0.041 (avg all epochs) |
| Best eval loss | **0.0001** |
| Final token accuracy | **100%** |
| Loss trajectory | 1.63 → 0.043 → 0.0001 → 0.00009 |

Near-perfect convergence by step 30 (loss < 0.05). The structured JSON tool call patterns are highly learnable. 100% accuracy is expected and desired for deterministic tool calling.

### 6.5 Output Artifacts

```
src/porter_ai_assistant/models/lora_adapters/
├── conversational/
│   └── final/                    (47 MB)
│       ├── adapter_config.json
│       ├── adapter_model.safetensors  (15 MB LoRA weights)
│       ├── tokenizer.json + tokenizer_config.json
│       ├── chat_template.jinja
│       ├── training_metadata.json
│       └── training_args.bin
└── tool_use/
    └── final/                    (47 MB)
        └── (same structure)
```

### 6.6 Next Steps

- **Task 18d:** Merge LoRA adapters with base model → quantize to GGUF (Q4_K_M) for RPi deployment
- **Task 18f:** Integration benchmarks — latency < 2s on RPi 4, accuracy on eval set