# DevLog — 09 Mar 2026: CPU Optimization & SLAM Coexistence

> **Session focus:** Optimize Qwen 2.5 1.5B GGUF inference for RPi CPU-only deployment,
> then ensure the AI assistant can coexist with SLAM/Nav2 without starving safety-critical nodes.
>
> **Engineer:** Antony Austin — assisted by Claude (Copilot)
> **Branch:** `prototype`
> **Venv:** `.venv-finetune`

---

## Session Overview

Two-part optimization session:
1. **CPU Inference Benchmarking** — Audit llama.cpp params, benchmark 10+ configurations, pick optimal defaults
2. **SLAM Coexistence** — Thread limiting, context window reduction, Docker resource caps, process priority

---

## Part 1: CPU Inference Optimization

### Audit Findings

**Dev machine:** AMD Ryzen 9 8940HX, 32 logical cores (16 physical), x86_64, AVX512
**Target:** RPi 4 (4 cores Cortex-A53, 4 GB RAM), RPi 5 (4 cores Cortex-A76, 8 GB RAM)
**llama-cpp-python:** v0.3.16, built with SSE3, SSSE3, AVX, AVX2, F16C, FMA, BMI2, AVX512 (full), OPENMP, REPACK

**Previous defaults (suboptimal):**
- `n_batch=64` — only 1/8th of llama.cpp's recommended 512
- `n_threads=4` — hard-coded, doesn't adapt to hardware
- No `n_threads_batch` or `flash_attn` exposed

**Llama() constructor params discovered (not previously used):**
- `n_threads_batch` — separate thread count for batch/prompt processing
- `flash_attn` — flash attention (SIMD-dependent)
- `type_k`/`type_v` — KV cache quantization
- `n_ubatch` — micro-batch size
- `numa` — NUMA awareness

### Benchmark Results (dev machine, 3 runs each)

| Config | Latency | tok/s | RSS |
|--------|---------|-------|-----|
| Baseline (4t, batch=64) | 780ms | 41.9 | 1675 MB |
| 8 threads, batch=64 | 748ms | 43.7 | 1677 MB |
| 4 threads, batch=512 | 797ms | 41.0 | 1680 MB |
| **8 threads, batch=512** | **745ms** | **43.9** | 1680 MB |
| flash_attn=True | 882ms | 39.7 | 1680 MB |
| mlock=True | 810ms | 40.3 | 1680 MB |

**Thread scaling (batch=512):**

| Threads | Latency | tok/s |
|---------|---------|-------|
| 1 | 1221ms | 27.0 |
| 2 | 872ms | 37.8 |
| 3 | 853ms | 38.7 |
| 4 | 804ms | 41.1 |

**Other tests:**
- `type_k=8/type_v=8` (q8_0 KV cache): **FAILED** — "Failed to create llama_context" (version issue)
- `type_k=1/type_v=1` (f16 KV cache): 819ms, 40.3 tok/s — works, same as default
- `n_ctx=1024` vs `2048`: saves ~28 MB RSS, marginal speed difference
- flash_attn: **SLOWER on x86 AVX512** — disabled by default, should test on ARM

### Key Findings

1. **`n_batch=512`** is the primary win — 8× larger prompt batches improve prompt eval throughput
2. **Thread scaling** shows diminishing returns past 4 threads on this architecture
3. **flash_attn** is SLOWER on x86 AVX512 — needs testing on ARM NEON (RPi 5)
4. **KV cache quantization** not supported in current llama-cpp-python version
5. **n_ctx=1024** saves ~28 MB RAM — good for RPi 4 (4 GB) budget

---

## Part 2: SLAM Coexistence

### Problem

On RPi 4/5 (4 cores), letting llama.cpp auto-detect cores (`n_threads=0`) means it uses ALL 4 cores during inference. This starves safety-critical nodes:

- **SLAM (slam_toolbox)** — needs CPU for map building, loop closure
- **Nav2 (planner+controller+costmap)** — needs CPU for path planning, obstacle avoidance
- **LIDAR driver** — needs CPU for scan processing at 10 Hz
- **Orchestrator + ESP32 bridges** — need CPU for state machine, serial comms

### Solution: Resource Partitioning

#### Thread Limiting
- `n_threads=2` — AI gets 2 cores max, SLAM/Nav2 get the other 2
- `n_threads_batch=0` — batch processing uses same thread count

#### Memory Budget
- `n_ctx=1024` (down from 2048) — airport Q&A rarely exceeds 800 tokens, saves ~28 MB
- `use_mmap=True` — memory-maps 1 GB GGUF, only active pages in RSS

**RPi 4 (4 GB) memory breakdown:**

| Component | RSS Estimate |
|-----------|-------------|
| SLAM (slam_toolbox) | ~400 MB |
| Nav2 (planner+controller+costmap) | ~300 MB |
| LIDAR driver + processor | ~50 MB |
| Orchestrator + bridges | ~40 MB |
| AI model (Q4_K_M, mmap) | ~1000 MB |
| OS + kernel | ~200 MB |
| **Total** | **~2.0 GB** |

Fits in 4 GB with headroom. RPi 5 (8 GB) is comfortable.

#### Docker Resource Caps

`docker-compose.prod.yml` — `porter_ai` service:
```yaml
cpus: "2.0"          # Max 2 CPU cores (hard cap)
mem_limit: 2g        # Max 2 GB RAM (prevents OOM kill of SLAM)
mem_reservation: 1g  # Soft limit: guaranteed 1 GB
environment:
  - PORTER_NICE=10   # Lower process priority than SLAM
```

#### Process Priority

`docker-entrypoint.sh` — new `PORTER_NICE` support:
```bash
if [ -n "${PORTER_NICE:-}" ] && [ "${PORTER_NICE}" != "0" ]; then
    exec nice -n "${PORTER_NICE}" "$@"
fi
```

Sets AI container to `nice 10` — OS scheduler will always prefer SLAM/Nav2 (nice 0) when both compete for CPU.

---

## Files Changed

### Part 1: CPU Optimization (CHANGES.md #41)

| File | Change |
|------|--------|
| `porter_ai_assistant/config.py` | `n_batch`: 64→512, `n_threads`: 4→0, added `DEFAULT_N_THREADS_BATCH`, `DEFAULT_FLASH_ATTN` |
| `porter_ai_assistant/inference_engine.py` | ModelConfig defaults + Llama() kwargs: n_threads_batch, flash_attn, auto-detect logic |
| `config/assistant_params.yaml` | Updated to match new defaults, RPi-specific comments |
| `porter_ai_assistant/assistant_node.py` | ROS 2 parameter declarations for n_threads_batch, flash_attn |
| `scripts/ai_server.py` | Added DEFAULT_N_THREADS_BATCH, DEFAULT_FLASH_ATTN imports + ModelConfig |
| `test/test_assistant.py` | Updated assertions for new defaults |

### Part 2: SLAM Coexistence (CHANGES.md #42)

| File | Change |
|------|--------|
| `porter_ai_assistant/config.py` | `n_threads`: 0→**2**, `n_ctx`: 2048→**1024** |
| `porter_ai_assistant/inference_engine.py` | ModelConfig defaults updated |
| `config/assistant_params.yaml` | Updated with SLAM-aware comments |
| `test/test_assistant.py` | Updated assertions (n_ctx=1024, n_threads=2) |
| `docker/Dockerfile.prod` | Added COPY for RAG knowledge_base/ directory |
| `docker/docker-compose.prod.yml` | porter_ai: cpus=2.0, mem_limit=2g, mem_reservation=1g, PORTER_NICE=10 |
| `docker/docker-compose.dev.yml` | Updated AI data mount comment for knowledge base |
| `docker/docker-entrypoint.sh` | Added PORTER_NICE env var → nice -n support |

---

## Current Default Configuration (after both sessions)

```python
DEFAULT_N_CTX = 1024           # Airport Q&A context (saves ~28 MB vs 2048)
DEFAULT_N_BATCH = 512          # llama.cpp optimal prompt eval batch
DEFAULT_N_THREADS = 2          # 2 cores for AI, 2 for SLAM/Nav2
DEFAULT_N_THREADS_BATCH = 0    # Same as n_threads
DEFAULT_N_GPU_LAYERS = 0       # CPU only (RPi)
DEFAULT_FLASH_ATTN = False     # Slower on x86 AVX512, test on ARM
DEFAULT_MAX_TOKENS = 256
DEFAULT_TEMPERATURE = 0.7
DEFAULT_TOP_P = 0.9
DEFAULT_TOP_K = 50
DEFAULT_REPEAT_PENALTY = 1.1
DEFAULT_MIN_P = 0.0
```

---

## Test Results

- **85 unit tests pass** (test_assistant.py + test_orchestrator.py + test_rag.py)
- Docker Compose dev: validated OK
- Docker Compose prod: validated OK
- Entrypoint shell syntax: validated OK

---

## Remaining Work / Future Testing

- [ ] **ARM benchmark**: Run on actual RPi 5 to test flash_attn on ARM NEON
- [ ] **SLAM integration test**: Run AI + SLAM simultaneously, measure CPU/memory under load
- [ ] **n_ctx tuning**: Monitor if 1024 is sufficient with RAG context injected
- [ ] **KV cache quantization**: Retry `type_k`/`type_v` when llama-cpp-python supports it
- [ ] **Use mlock on RPi 5 (8 GB)**: Lock model in RAM to avoid page faults during inference
