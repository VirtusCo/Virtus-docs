# Development Logs

Session-by-session engineering logs documenting Porter's development. Each log captures decisions made, problems encountered, and solutions implemented during a development session.

## Log Index

| Date | Log | Topics |
|------|-----|--------|
| **25 Feb 2026** | `25_Feb_Logs.md` | Hardware investigation, LIDAR identification (S2PRO/YD-47 via `tri_test`), architecture decisions, repo setup |
| **07 Mar 2026** | `07_Mar_Logs.md` | Tasks 1--6: LIDAR driver, lidar processor, orchestrator FSM. Hardware validation, bug fixes |
| **07 Mar 2026** | `07_Mar_CICD_Logs.md` | CI/CD pipeline: 6 pipeline fixes, GitHub Actions verify.yml (9 jobs) |
| **07 Mar 2026** | `07_Mar_ESP32_Logs.md` | ESP32 firmware development, Zephyr RTOS bring-up |
| **07 Mar 2026** | `07_Mar_AI_Logs.md` | AI assistant initial development, model selection |
| **08 Mar 2026** | `08_Mar_AI_Logs.md` | QLoRA fine-tuning, Qwen 2.5 1.5B training, benchmarks |
| **08 Mar 2026** | `08_Mar_Dataset_Cleanup_Logs.md` | Training dataset curation, 12K example cleanup |
| **08 Mar 2026** | `08_Mar_Model_Switch_Logs.md` | Gemma 3 270M to Qwen 2.5 1.5B migration, quality comparison |
| **08 Mar 2026** | `08_Mar_GUI_CICD_Logs.md` | Flutter GUI development, CI/CD integration for Flutter builds |
| **09 Mar 2026** | `09_Mar_Qwen_Training_Logs.md` | Qwen training refinements, compact prompt engineering |
| **09 Mar 2026** | `09_Mar_DPO_Training_Logs.md` | DPO reinforcement learning, synthetic preference data, 4 GGUF variants |
| **09 Mar 2026** | `09_Mar_Tool_Use_Fix_Logs.md` | Tool-use compliance fix, retrain with compact prompt, GUI humanization |
| **09 Mar 2026** | `09_Mar_SSE_RAG_Logs.md` | SSE streaming implementation, RAG knowledge base (41 docs, TF-IDF) |
| **09 Mar 2026** | `09_Mar_CPU_SLAM_Optimization_Logs.md` | CPU/SLAM coexistence tuning: n_threads=2, n_ctx=1024, Docker resource limits |

## Reading Guide

Logs are named by date and topic. To understand a specific subsystem's development history, read the logs in chronological order filtered by topic:

- **LIDAR + Navigation:** 25 Feb, 07 Mar (main)
- **ESP32 Firmware:** 07 Mar ESP32
- **AI Assistant:** 07 Mar AI, 08 Mar AI, 08 Mar Model Switch, 09 Mar Qwen, 09 Mar DPO, 09 Mar Tool Use Fix
- **Infrastructure:** 07 Mar CI/CD, 08 Mar GUI CI/CD
- **Performance:** 09 Mar CPU/SLAM, 09 Mar SSE/RAG
