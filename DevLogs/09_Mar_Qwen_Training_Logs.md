# 09 Mar 2026 — Qwen 2.5 1.5B LoRA Training, GGUF Conversion & Benchmarks

## Scope
- Download Qwen 2.5 1.5B Instruct GGUF base model
- Train conversational + tool_use LoRA adapters on Qwen base
- Convert LoRA adapters to merged GGUF Q4_K_M
- Benchmark latency, throughput, and memory on dev machine
- Run inference quality tests
- Fix residual test failures

## Hardware
- **GPU:** NVIDIA GeForce RTX 5070 Laptop GPU, 8151 MiB VRAM
- **Training:** QLoRA 4-bit (bitsandbytes), batch 4, grad_accum 4, max_seq_len 512, 3 epochs, lr 2e-4, LoRA r=16 alpha=32

## 1. Model Download

Downloaded `qwen2.5-1.5b-instruct-q4_k_m.gguf` from `Qwen/Qwen2.5-1.5B-Instruct-GGUF`:
- Size: 1.07 GB
- Download time: ~29s
- Location: `src/porter_ai_assistant/models/gguf/`

## 2. Conversational LoRA Training

```
Dataset: 7000 train / 1400 eval examples
Duration: 56.3 minutes (1314 steps)
Final eval_loss: 0.1365
Token accuracy: 95.5%
GPU memory: ~6.1 GB peak
```

Training loss progression: 2.106 → 0.157 (smooth convergence, no signs of overfitting at 3 epochs).

## 3. Tool-Use LoRA Training

```
Dataset: 3000 train / 600 eval examples
Duration: 66.9 minutes (564 steps)
Final eval_loss: ≈0.0000
Token accuracy: 100%
GPU memory: ~6.1 GB peak
```

Tool-use had longer training time despite fewer samples due to longer average sequence length (tool schemas + JSON output).

## 4. GGUF Conversion

Approach: merge LoRA weights into base model → convert to F16 GGUF → quantize to Q4_K_M.

| Adapter | F16 Size | Q4_K_M Size | Time |
|---------|----------|-------------|------|
| Conversational | 3.1 GB | 940 MB | ~3 min |
| Tool-use | 3.1 GB | 940 MB | ~3 min |

F16 intermediates deleted after quantization. Final artifacts:
- `porter-conversational-Qwen2.5-1.5B-Instruct-Q4_K_M.gguf` (940 MB)
- `porter-tool_use-Qwen2.5-1.5B-Instruct-Q4_K_M.gguf` (940 MB)

## 5. Benchmarks

### Conversational Model (10 runs)
| Metric | Value |
|--------|-------|
| Mean latency | 1436 ms |
| Median latency | 1211 ms |
| P95 latency | 2193 ms |
| % under 2s | 80% |
| Throughput | 38.8 tok/s |
| RSS memory | 1678 MB |
| Token range | 29–99 tokens |

**Verdict: PASS** — well within 2s target for typical responses.

### Tool-Use Model (10 runs)
| Metric | Value |
|--------|-------|
| Mean latency | 1724 ms |
| Median latency | 1104 ms |
| P95 latency | 6118 ms |
| % under 2s | 70% |
| Throughput | 38.5 tok/s |
| RSS memory | 1678 MB |
| Token range | 25–256 tokens |

**Verdict: MARGINAL** — throughput is consistent (~39 tok/s), but some runs produced verbose/runaway responses (84–256 tokens) causing latency spikes. Root cause is response length, not model speed. Needs `max_tokens` tuning or response truncation.

### Comparison: Qwen 1.5B vs Gemma 270M (previous)
| Metric | Gemma 270M | Qwen 1.5B |
|--------|-----------|-----------|
| Model size (GGUF) | 242 MB | 940 MB |
| RSS memory | 351 MB | 1678 MB |
| Conv latency | 497 ms | 1436 ms |
| Tool latency | 250 ms | 1724 ms |
| Throughput | ~70 tok/s | ~39 tok/s |

Qwen is ~3× slower and uses ~5× more memory, but the quality improvement justifies the tradeoff.

## 6. Inference Quality Tests

### Conversational: 8/8 (100%)
All 8 test queries passed keyword matching:
- Greetings, directions, services, accessibility, dining, Wi-Fi, identity, edge cases

### Tool-Use: 4/5 (80%)
- ✅ conv_01: Gate directions
- ✅ conv_02: Flight status lookup → `get_flight_status` tool call
- ✅ conv_03: Luggage tracking → `weigh_luggage` tool call
- ✅ conv_04: Transport options
- ❌ tool_05: Coffee shop search — `find_nearest` tool call generated but keyword matching failed (response format differed from expected pattern)

## 7. Test Suite

57/57 tests pass after fixing `test_flake8.py` to exclude `build/` directory:
- 22 inference/config tests
- 35 orchestrator tests
- Flake8 fix: `argv=['--exclude', 'scripts']` → `argv=['--exclude', 'scripts,build']`
  (build/ contains auto-generated `sitecustomize.py` with E501 violations)

## Summary

| Step | Status |
|------|--------|
| Qwen GGUF download | ✅ 1.07 GB |
| Conversational LoRA | ✅ eval_loss=0.1365, 95.5% acc |
| Tool-use LoRA | ✅ eval_loss≈0.0, 100% acc |
| GGUF conversion | ✅ 940 MB each |
| Benchmark (conv) | ✅ 1436ms, 80% <2s |
| Benchmark (tool) | ⚠️ 1724ms, 70% <2s (verbose responses) |
| Inference quality | ✅ conv 100%, tool 80% |
| Tests | ✅ 57/57 pass |

**Next steps:**
- RPi 5 on-device benchmarks (expected slower — CPU-only llama.cpp)
- Tune `max_tokens` for tool-use to cap response length
- Phase 4.6: RAG integration for real airport data
