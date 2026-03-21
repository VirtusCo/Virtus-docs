# Virtue AI Assistant

Virtue is Porter's on-device AI assistant. It runs entirely on the Raspberry Pi with no cloud dependency, providing airport passengers with wayfinding, flight information, and check-in assistance through natural language.

## Model Stack

| Component | Detail |
|-----------|--------|
| **Base model** | Qwen 2.5 1.5B Instruct |
| **Quantization** | Q4_K_M GGUF (~940 MB) |
| **Runtime** | llama-cpp-python (CPU-only) |
| **Fine-tuning** | QLoRA (conv + tool-use adapters) |
| **Reinforcement** | DPO on synthetic preference data |
| **Training hardware** | NVIDIA RTX 5070 (8 GB VRAM) |

## Architecture

```
Passenger query (voice or text)
    |
    v
Intent Classifier (keyword/regex)
    |
    ├── High confidence ──> Tool Executor (14 tools) ──> Formatted response
    |
    └── Low confidence  ──> LLM inference (Qwen 2.5) ──> Generated response
                                 |
                                 └── RAG context injected (TF-IDF, 41 docs)
```

## LoRA Adapters

Two separate LoRA adapters are loaded at runtime (never merged at 4-bit):

| Adapter | Training Loss | Token Accuracy | Purpose |
|---------|--------------|----------------|---------|
| **Conversational** | 0.1365 (SFT), 2.25e-10 (DPO) | 95.5% | Natural airport Q&A |
| **Tool-use** | 0.0513 (SFT), 1.34e-6 (DPO) | 98.15% | Structured tool calling |

!!! warning "System prompt parity"
    System prompts at inference must be character-for-character identical to the training data. Any extra whitespace or text breaks tool-use compliance.

## 14 Tools

Virtue can invoke these tools in response to passenger queries:

| Tool | Function |
|------|----------|
| `get_directions` | Navigate to gates, lounges, facilities |
| `get_flight_status` | Real-time flight information |
| `find_nearest` | Locate restrooms, ATMs, restaurants |
| `escort_passenger` | Initiate follow-me mode |
| `show_map` | Display terminal map on screen |
| `request_assistance` | Call airport staff |
| `check_baggage_weight` | Trigger weighing scale |
| `get_boarding_info` | Gate, boarding time, seat |
| `get_weather` | Destination weather |
| `translate_phrase` | Basic phrase translation |
| `find_transport` | Taxi, bus, metro options |
| `report_issue` | Log passenger complaint |
| `get_amenities` | WiFi, charging, lounges |
| `emergency_help` | Contact security/medical |

## RAG Knowledge Base

- **Documents:** 41 across 5 JSON files (facilities, terminals, transport, dining, services)
- **Retriever:** TF-IDF + keyword boosting
- **Latency:** < 1 ms retrieval, zero external dependencies

## Performance (RPi 5)

| Metric | Value |
|--------|-------|
| **P50 latency** | ~1.4 s |
| **P95 latency** | ~1.8 s |
| **Throughput** | ~42 tokens/s |
| **Memory (RSS)** | ~1.7 GB |
| **Threads** | 2 (reserves 2 cores for SLAM/Nav2) |
| **Context window** | 1024 tokens (saves ~28 MB vs 2048) |

## Resource Constraints

!!! info "CPU/SLAM coexistence"
    The AI assistant is configured with `n_threads=2` and runs at nice priority 10 inside a Docker container limited to 2 CPUs and 2 GB RAM. This ensures SLAM and Nav2 retain two dedicated cores for real-time navigation.
