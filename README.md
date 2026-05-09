# SRE Copilot

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/bhagyeshrathi07/SRE-Copilot/blob/main/sre_copilot.ipynb)
[![Python 3.10+](https://img.shields.io/badge/python-3.10+-blue.svg)](https://www.python.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

> An AI-powered incident triage agent for on-call SREs. Built as a multi-tool ReAct agent over dual quantized LLMs, with RAG, text-to-SQL, and live API integrations — plus a systematic study of four prompting techniques.

---

## The Problem

When an on-call engineer gets paged at 3 AM, they need to quickly:

- Find the right runbook for the alert they're seeing
- Check if there have been recent deploys that could have caused the issue
- Look up who owns the affected service and the escalation path
- Determine whether the problem is upstream (cloud provider outage)
- Cross-reference with past incidents for known root causes

**SRE Copilot automates that triage workflow.** The engineer asks a natural language question, and the agent dynamically chooses between five data sources to answer it.

---

## Architecture

```
                    User Query
                        │
                        ▼
    ┌─────────────────────────────────────┐
    │       AGENT LOOP (ReAct)            │
    │   System Prompt + Tool Specs        │
    │        ┌────────────┐               │
    │        │  LLM Gen   │ ◄── Qwen2.5-32B (high quality)
    │        │            │     Qwen2.5-3B  (fast)
    │        └─────┬──────┘               │
    │              │ Parse tool call JSON │
    │              ▼                      │
    │        ┌──────────────┐             │
    │        │ Tool Router  │             │
    │        └──┬──┬──┬──┬──┴──┐          │
    └───────────┼──┼──┼──┼─────┼──────────┘
                │  │  │  │     │
                ▼  ▼  ▼  ▼     ▼
            ┌─────┐┌────┐┌─────┐┌─────┐┌──────┐
            │ RAG ││ DB ││ GCP ││ OSV ││ Web  │
            │Vec- ││SQL ││Stat-││Vuln ││Search│
            │tor  ││ite ││us   ││API  ││(DDG) │
            └─────┘└────┘└─────┘└─────┘└──────┘
              │
              └─► ChromaDB (5,084 chunks from 768 GitLab runbooks)
```

Both 32B and 3B models run **simultaneously on a single A100-80GB** via 4-bit NF4 quantization (~60GB VRAM total).

---

## Key Features

### Five integrated tools
- **`search_runbooks`** — RAG over 768 GitLab production runbooks (5,084 chunks, Qwen3-Embedding-0.6B, ChromaDB)
- **`query_database`** — Text-to-SQL over an SRE schema (incidents, deploys, service_ownership, alerts)
- **`check_cloud_status`** — Live GCP/AWS status API integration
- **`search_vulnerabilities`** — OSV API for CVE lookups
- **`web_search`** — DuckDuckGo for real-time external info

### Four prompting techniques (head-to-head)
- **Baseline** — Standard ReAct loop
- **Prompt Chaining** — Plan → Execute → Synthesize (3 LLM calls)
- **Self-Reflection** — Generate → Critique → Refine (2 LLM calls)
- **Meta-Prompting** — Rewrite the prompt before answering

### Performance & security
- **KV cache reuse** for system prompt prefill acceleration
- **Defense-in-depth security**: SQL injection blocking (3-layer regex + SELECT-only), prompt hardening, comment stripping
- **Systematic red-team testing** against 5 prompt injection attack vectors

---

## Results

### Evaluation: 2 models × 4 techniques × 10 queries = 80 runs

| Model | Technique       | Tool Correct | Keyword | Grounded | Actionable | Avg Time |
|-------|-----------------|:------------:|:-------:|:--------:|:----------:|:--------:|
| 32B   | Baseline        | 0.80         | 0.93    | 0.40     | 0.50       | 31.4s    |
| 32B   | Chaining        | **1.00**     | **1.00**| **1.00** | 0.90       | 47.6s    |
| 32B   | Self-Reflection | 0.90         | 1.00    | 0.90     | **1.00**   | 42.7s    |
| 32B   | Meta Prompting  | 0.90         | 0.90    | 0.70     | 0.70       | 65.5s    |
| 3B    | Baseline        | 0.90         | 0.93    | 0.80     | 0.70       | 31.1s    |
| 3B    | Chaining        | 0.70         | 1.00    | 0.70     | 0.60       | 33.3s    |
| 3B    | Self-Reflection | 0.90         | 0.97    | 0.90     | **1.00**   | **23.6s**|
| 3B    | Meta Prompting  | 0.50         | 0.83    | 0.90     | 0.80       | 29.7s    |

### Three key findings

**1. Self-reflection is the most reliable quality lift across both model sizes.** On the 32B model, grounded scores went from 0.40 → 0.90 and actionable from 0.50 → 1.00. On the 3B model, the lift is smaller but still meaningful — and the 3B+reflection configuration (23.6s) is *faster* than the 32B baseline (31.4s) while producing better quality. This is consistent with the broader trend that inference-time compute can substitute for parameter scale.

**2. Advanced techniques have minimum capability thresholds.** Prompt chaining was the best-on-paper technique for 32B (perfect 1.00 on tool correctness and grounding), but it broke on the 3B model — the smaller model couldn't reliably generate valid JSON plans, causing silent fallbacks to baseline behavior. Meta-prompting was even worse on 3B, dropping tool correctness to 0.50 because the reformulated queries confused the model. The lesson: techniques requiring structured intermediate outputs need a model capable of producing them.

**3. Programmatic security beats prompt-level security.** Both models were vulnerable to translation-based prompt extraction attacks (3/5 red-team attacks blocked per model). But the SQL safety layer caught **every** SQL injection attempt, even when the LLM was tricked into generating malicious queries. Programmatic guardrails (regex filters, allowlists, sandboxed execution) should always be the ground truth — the LLM is best-effort.

---

## Tech Stack

| Layer            | Tools                                                              |
|------------------|--------------------------------------------------------------------|
| Models           | Qwen2.5-32B-Instruct, Qwen2.5-3B-Instruct, Qwen3-Embedding-0.6B    |
| Quantization     | bitsandbytes (4-bit NF4, double quant, bf16 compute)               |
| Inference        | transformers, accelerate, SDPA attention                           |
| RAG              | LangChain, ChromaDB, Qwen3-Embedding-0.6B                          |
| Structured data  | SQLite + custom safety layer                                       |
| External APIs    | httpx (GCP status, OSV), DuckDuckGo Search                         |
| UI               | Gradio (with shareable public link)                                |
| Hardware         | NVIDIA A100-SXM4-80GB (Google Colab Pro+)                          |

---

## Quick Start

### Run in Colab (recommended)

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/bhagyeshrathi07/SRE-Copilot/blob/main/sre_copilot.ipynb)

1. Click the badge above
2. Select **Runtime → Change runtime type → A100 GPU** (Colab Pro+ required)
3. Add your Hugging Face token to Colab Secrets as `HF_TOKEN`
4. Run cells top-to-bottom

The full pipeline (model loading, RAG ingestion, eval, demo) takes ~25 minutes end-to-end. The agent and Gradio demo launch in Section 7.

### Run locally

Local execution requires an 80GB GPU (A100, H100). Smaller setups would need to either run a single model or use higher-precision quantization tradeoffs.

```bash
git clone https://github.com/bhagyeshrathi07/SRE-Copilot.git
cd SRE-Copilot
pip install -r requirements.txt   # see notebook for the full dep list
export HF_TOKEN="your_hf_token"
jupyter notebook SRE_copilot.ipynb
```

---

## Repository Structure

```
SRE-Copilot/
├── SRE_copilot.ipynb       # Main notebook (model loading → eval → demo)
├── README.md               # You are here
└── results/                # Evaluation artifacts
    ├── evaluation_results.csv
    ├── full_eval_matrix.csv
    ├── security_32b.csv
    ├── security_3b.csv
    └── cache_benchmark.csv
```

The notebook is organized as:

| Section | Contents                                                |
|---------|---------------------------------------------------------|
| 0       | Environment setup, model loading, checkpointing         |
| 1       | Data pipeline — runbook ingestion + SQLite seeding      |
| 2       | ReAct agent: tool specs, system prompt, agent loop      |
| 3       | Advanced prompting: chaining, reflection, meta-prompt   |
| 4       | KV cache reuse + prefill benchmark                      |
| 5       | Security testing — 5 prompt injection attacks           |
| 6       | Evaluation framework — 22 gold queries, 4 metrics       |
| 7       | Gradio chat UI                                          |

---

## Limitations & Future Work

This is a research / demonstration project. Honest assessment of the gaps:

- **Synthetic data.** The incident/deploy/alert tables are seeded synthetically. Production deployment would require real PagerDuty/OpsGenie/Jira integrations.
- **Heuristic evaluation metrics.** Keyword matching and grounding-signal detection are crude proxies. LLM-as-judge would give more robust quality scores.
- **Latency.** 24–65 seconds per query is too slow for a live on-call workflow. Production would require an inference server (vLLM, TGI) for ~5–10× throughput improvement, plus token-by-token streaming in the UI.
- **No conversation memory.** Each query is independent. Multi-turn investigation ("what about the auth-service?") isn't supported.
- **Translation-based prompt extraction defeats both models.** A well-known attack class that requires structured output enforcement (Outlines, Instructor) to fully mitigate.
- **Retrieval is dense-only.** Hybrid retrieval (BM25 + dense) would improve recall on exact terms like service names and error codes.

---

## Citation / Acknowledgements

- **Models**: [Qwen2.5](https://github.com/QwenLM/Qwen2.5) (Alibaba Cloud), [Qwen3-Embedding](https://huggingface.co/Qwen/Qwen3-Embedding-0.6B)
- **Quantization**: [bitsandbytes](https://github.com/TimDettmers/bitsandbytes) — Dettmers et al., *QLoRA* (2023)
- **ReAct framework**: Yao et al., *ReAct: Synergizing Reasoning and Acting in Language Models* (2023)
- **Runbooks**: GitLab's public [SRE runbooks repository](https://gitlab.com/gitlab-com/runbooks)
- **Postmortems**: [awesome-incident-postmortem](https://github.com/ggalihpp/awesome-incident-postmortem)

---

## Contact

Built by **Bhagyesh Rathi** — MS AI candidate at San Jose State University.

- Portfolio: [bhagyesh.dev](https://bhagyesh.dev)
- LinkedIn: [linkedin.com/in/bhagyeshrathi07](https://linkedin.com/in/bhagyeshrathi07)
- GitHub: [@bhagyeshrathi07](https://github.com/bhagyeshrathi07)

If you're hiring for AI/ML internships and this work is relevant, I'd love to hear from you.
