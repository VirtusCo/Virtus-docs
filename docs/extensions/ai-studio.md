# Virtus AI Studio

A complete MLOps workbench for training, benchmarking, exporting, and deploying AI models to the Porter robot. Supports YOLO (object detection), LLM fine-tuning, and reinforcement learning workflows.

## Features

- **Train** — QLoRA fine-tuning for LLMs (Qwen, Llama, Gemma), YOLO training for object detection
- **Benchmark** — Latency, throughput, accuracy, and memory profiling across quantization levels
- **Export** — Convert models to deployment formats: GGUF, ONNX, HEF (Hailo), TFLite
- **Deploy** — Push models to Raspberry Pi over SSH, update running inference services

## Supported Model Formats

| Format | Use Case | Target Hardware |
|--------|----------|-----------------|
| **GGUF** | LLM inference via llama.cpp | RPi 5 (CPU) |
| **HEF** | Hailo-8/8L accelerated inference | RPi 5 + Hailo AI HAT |
| **ONNX** | Cross-platform inference | Any |
| **TFLite** | Edge inference | RPi, mobile |

## Training Workflow

```
Dataset (JSON/CSV)
    |
    v
Training Config (webview form)
    ├── Base model selection
    ├── QLoRA parameters (rank, alpha, dropout)
    ├── Training hyperparameters (lr, epochs, batch size)
    └── Hardware config (GPU selection, mixed precision)
    |
    v
Training Runner (extension host)
    ├── Real-time loss/accuracy charts
    ├── GPU utilization monitoring
    └── Checkpoint management
    |
    v
Export Pipeline
    ├── LoRA merge
    ├── Quantization (Q4_K_M, Q5_K_M, Q8_0)
    └── Format conversion (GGUF, ONNX, HEF)
```

## Benchmarking

The benchmark panel runs standardized tests and displays results in sortable tables:

| Metric | Description |
|--------|-------------|
| **TTFT** | Time to first token (LLM) |
| **Throughput** | Tokens/second (LLM) or FPS (YOLO) |
| **Memory** | Peak RSS during inference |
| **Accuracy** | Token accuracy (LLM) or mAP (YOLO) |
| **Latency P50/P95** | Response time distribution |

## Deployment

!!! info "One-click deploy"
    Select a model artifact and target device. AI Studio handles the SSH transfer, service restart, and post-deploy health check.

Models are deployed to `/opt/virtus/models/` on the Raspberry Pi and picked up by the `porter_ai_assistant` node on next restart.
