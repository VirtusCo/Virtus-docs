# 09 Mar 2026 — DPO Reinforcement Learning Training

## Scope

- Fix critical bug in `dpo_train.py` (lesson #35: fresh LoRA B=0 → zero gradients)
- Train DPO on both tool_use and conversational adapters using synthetic preference data
- Convert DPO-trained adapters to merged GGUF Q4_K_M
- Benchmark SFT vs DPO head-to-head on latency, quality, and tool compliance

## Hardware

- **GPU:** NVIDIA GeForce RTX 5070 Laptop GPU, 8151 MiB VRAM
- **Training:** bf16 (no quantisation — DPO needs full-precision ref logprobs), batch 1, grad_accum 8, max_length 512, 3 epochs, lr 5e-5, beta 0.1

---

## 1. Bug Fix: Fresh LoRA B=0 Produces Zero Gradients (Lesson #35)

**Problem:** `dpo_train.py` loaded the base model with a fresh `LoraConfig` and `ref_model=None`. TRL's `DPOTrainer` uses the initial model weights as the reference. Since `lora_B` is initialized to zero, policy == reference → logprob difference is zero → DPO loss = log(sigmoid(0)) = 0.6931 → zero gradients. The model never learns.

**Fix applied:**
1. Load base model in bf16 (no quantisation)
2. Load pre-trained SFT adapter via `PeftModel.from_pretrained(model, sft_path, is_trainable=True)`
3. Remove `LoraConfig` creation entirely — reuse the SFT adapter's config
4. Remove `peft_config` from `DPOTrainer` constructor
5. Increase `max_length` from 256 → 512

This ensures policy ≠ reference from step 0 (SFT adapter has non-zero B weights), giving DPO meaningful gradients immediately.

---

## 2. Synthetic Preference Data Generation

DPO requires `(prompt, chosen, rejected)` triplets. Used synthetic corruption strategy:

| Corruption Type | Description | Applied To |
|----------------|-------------|------------|
| `strip_tags` | Remove `<tool_call>`/`</tool_call>` tags from tool responses | tool_use |
| `ramble` | Append irrelevant verbose text to good responses | Both |
| `wrong_json` | Corrupt JSON keys in tool calls | tool_use |
| `echo` | Parrot back the user's question | Both |

**Datasets generated:**
- tool_use: 1500 preference pairs → `data/tool_use/dpo_preferences_synthetic.jsonl`
- conversational: 2000 preference pairs → `data/conversational/dpo_preferences_synthetic.jsonl`

---

## 3. DPO Tool-Use Training

```
Dataset: 1500 preference pairs (synthetic)
Duration: 45.3 minutes
Steps: 50 logging steps
GPU memory: ~6.3 GB (peak ~7.3 GB during ref logprob precompute)
```

**DPO Config:**
- `beta=0.1` (KL penalty coefficient)
- `loss_type="sigmoid"` (standard DPO)
- `precompute_ref_log_probs=True` (avoids keeping ref model in VRAM)
- `precompute_ref_batch_size=1`
- `bf16=True`, cosine scheduler, warmup_ratio 0.1

**Training metrics:**
| Metric | Start | End |
|--------|-------|-----|
| Loss | 0.4221 | 1.5e-6 |
| rewards/chosen | 0.861 | 20+ |
| rewards/rejected | -0.167 | -20+ |
| rewards/accuracies | 0.875 | 1.0 |
| rewards/margins | 1.028 | 40+ |
| eval_loss | — | 1.34e-6 |

Loss converged from 0.42 → near-zero rapidly. Reward margins grew from ~1.0 → 40+, indicating the model strongly prefers chosen (correct tool calls) over rejected (corrupted) responses.

---

## 4. DPO Conversational Training

```
Dataset: 2000 preference pairs (synthetic)
Duration: 22.4 minutes
eval_loss: 2.25e-10
```

Same DPO config. Faster due to shorter average response length in conversational data. Near-zero eval loss indicates strong preference alignment.

---

## 5. GGUF Conversion

Updated `convert_to_gguf.py` with `--variant dpo` argument. Conversion pipeline:
1. Load base model (Qwen 2.5 1.5B Instruct, bf16)
2. Load DPO adapter from `models/lora_adapters/{adapter}/dpo/final/`
3. Merge adapter weights into base model
4. Convert to F16 GGUF via `llama.cpp/convert_hf_to_gguf.py`
5. Quantise F16 → Q4_K_M via `llama-quantize`
6. Delete F16 intermediate

**Output files:**
| File | Size |
|------|------|
| `porter-conversational-dpo-Qwen2.5-1.5B-Instruct-Q4_K_M.gguf` | 941 MB |
| `porter-tool_use-dpo-Qwen2.5-1.5B-Instruct-Q4_K_M.gguf` | 941 MB |

---

## 6. SFT vs DPO Benchmark Results

Ran `benchmark_dpo_vs_sft.py` — 10 test prompts per model, 4 models total.

### Latency

| Model | Avg ms | P50 ms | P95 ms | <2s% | tok/s |
|-------|--------|--------|--------|------|-------|
| conv_sft | 1223 | 1150 | 1939 | 100% | 41.7 |
| conv_dpo | 1172 | 1063 | 1949 | 100% | 42.7 |
| tool_sft | 680 | 728 | 925 | 100% | 41.5 |
| tool_dpo | 724 | 790 | 1093 | 100% | 41.4 |

### Tool Compliance

| Model | Tool Call Rate |
|-------|---------------|
| tool_sft | 100% (10/10) |
| tool_dpo | 100% (10/10) |

### Quality Observations (Sample Responses)

**Conversational — "Where is Gate B7?":**
- SFT: "Gate B7 is a 10-minute walk from here in Terminal 2. Head beside the ATM, then continue straight..."
- DPO: "Gate B7 is located in Terminal 5, Concourse C. It's about 3 minutes on foot. The path goes beside the ATM. I'll guide you there."

DPO responses tend to be more concise and confident — less hedging, more direct assistant-like tone.

**Tool-Use — "I need directions to Gate C12":**
- SFT: `<tool_call>{"name": "get_directions", "arguments": {"destination": "Gate C12", "from_location": "Current Location"}}</tool_call>`
- DPO: `<tool_call>{"name": "get_directions", "arguments": {"destination": "Gate C12", "from_location": "Current location"}}</tool_call>`

Both produce valid tool calls. DPO tends to include more optional parameters (e.g., adding `"airport": "Dubai"` to flight queries).

### Summary

- **Conversational DPO**: ~4% faster avg latency (1223→1172ms), marginally higher throughput (41.7→42.7 tok/s), more concise responses
- **Tool-Use DPO**: Same 100% tool compliance, marginal latency increase (680→724ms), richer parameter filling
- **Both 100% under 2s** across all test prompts
- DPO provides measurable but modest improvements over SFT — primarily in response style (conciseness, confidence) rather than raw accuracy

---

## 7. Files Modified/Created

| File | Action |
|------|--------|
| `scripts/dpo_train.py` | Fixed lesson #35 bug — load SFT adapter, remove fresh LoraConfig |
| `scripts/convert_to_gguf.py` | Added `--variant dpo` support |
| `scripts/benchmark_dpo_vs_sft.py` | New — 4-model benchmark comparison |
| `models/lora_adapters/tool_use/dpo/final/` | DPO adapter + metadata |
| `models/lora_adapters/conversational/dpo/final/` | DPO adapter + metadata |
| `models/gguf/porter-conversational-dpo-*.gguf` | Merged DPO GGUF |
| `models/gguf/porter-tool_use-dpo-*.gguf` | Merged DPO GGUF |
| `models/dpo_vs_sft_benchmark.json` | Benchmark results |
| `data/tool_use/dpo_preferences_synthetic.jsonl` | 1500 preference pairs |
| `data/conversational/dpo_preferences_synthetic.jsonl` | 2000 preference pairs |

---

## 8. Lessons Learned

1. **Never use DPO with fresh LoRA + ref_model=None** — B=0 means policy == reference, zero gradients, loss stuck at 0.6931. Always load a pre-trained SFT adapter.
2. **Synthetic preference data works for DPO** — 4 corruption types (strip_tags, ramble, wrong_json, echo) produce meaningful preference signals. The model learns to prefer structured, concise responses.
3. **DPO on synthetic data near-zero eval_loss is expected** — with corruptions that are obviously wrong (missing tags, broken JSON), the model easily separates chosen/rejected. This doesn't mean the model is perfect — it means the preference task was easy. Real human preference data would be harder.
4. **precompute_ref_log_probs=True saves VRAM** — precomputes reference logprobs once, then discards the second model copy. Peak VRAM: ~7.3 GB during precompute, ~6.3 GB during training. Essential for 8 GB GPU.
5. **nohup + PYTHONUNBUFFERED=1 for long training** — terminal tools truncate output at 60KB. Use `PYTHONUNBUFFERED=1 nohup python3 script.py > /tmp/logfile 2>&1 &` and monitor with `grep` on the log.
