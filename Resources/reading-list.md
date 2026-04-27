# Reading List

Articles saved for future reading, organized by topic.

---

## Data Infrastructure (for Data Engineers)

Platforms, pipelines, tooling, and infrastructure at scale.

| Title | Date | Summary |
|---|---|---|
| [Trust But Canary: Configuration Safety at Scale](https://engineering.fb.com/2026/04/08/security/trust-but-canary-configuration-safety-at-scale-meta-tech-podcast/) | Apr 2026 | Canary deployments + progressive rollouts for safe config changes at scale; AI-assisted bisection for root cause; blameless incident culture |
| [Investing in Infrastructure: Meta's Renewed Commitment to jemalloc](https://engineering.fb.com/2026/03/02/data-infrastructure/investing-in-infrastructure-metas-renewed-commitment-to-jemalloc/) | Mar 2026 | Renewed open-source stewardship of jemalloc; huge-page allocator improvements, memory packing/purging, ARM64 support; org lesson on infra debt vs. feature velocity |
| [DrP: Meta's Root Cause Analysis Platform at Scale](https://engineering.fb.com/2025/12/19/data-infrastructure/drp-metas-root-cause-analysis-platform-at-scale/) | Dec 2025 | Automated RCA platform with 2,000+ analyzers, 50k daily analyses, 20–80% MTTR reduction; SDK-based investigation codification + AI4Ops roadmap |
| [Zoomer: Powering AI Performance at Meta's Scale Through Intelligent Debugging and Optimization](https://engineering.fb.com/2025/11/21/data-infrastructure/zoomer-powering-ai-performance-meta-intelligent-debugging-optimization/) | Nov 2025 | Automated AI profiling platform; GPU/CPU/distributed training analysis; straggler detection, critical path analysis; 20–75% training time reductions |
| [Creating AI Agent Solutions for Warehouse Data Access and Security](https://engineering.fb.com/2025/08/13/data-infrastructure/agentic-solution-for-warehouse-data-access/) | Aug 2025 | Multi-agent RBAC replacement; data-user + data-owner agents; query-level access budgets, implicit intention inference, rule-based safety guardrails |
| [Accelerating GPU Indexes in Faiss with NVIDIA cuVS](https://engineering.fb.com/2025/05/08/data-infrastructure/accelerating-gpu-indexes-in-faiss-with-nvidia-cuvs/) | May 2025 | cuVS integration into Faiss v1.10; CAGRA vs HNSW (12x build speedup); IVF Flat/PQ benchmarks on H100 GPU across 5M–100M vectors |
| [Sequence Learning: A Paradigm Shift for Personalized Ads Recommendations](https://engineering.fb.com/2024/11/19/data-infrastructure/sequence-learning-personalized-ads-recommendations/) | Nov 2024 | Event-based sequence learning replacing DLRMs; jagged tensors, attention pooling O(N²)→O(M×N); 2–4% conversion lift |

---

## ML Applications (for Data Scientists)

ML models, AI agents, and real-world ML applications.

| Title | Date | Summary |
|---|---|---|
| [How Meta Used AI to Map Tribal Knowledge in Large-Scale Data Pipelines](https://engineering.fb.com) | Apr 2026 | Using AI to capture and transfer undocumented knowledge across data pipelines |
| [KernelEvolve: How Meta's Ranking Engineer Agent Optimizes AI Infrastructure](https://engineering.fb.com) | Apr 2026 | An autonomous agent that iteratively optimizes AI ranking kernels |
| [Meta Adaptive Ranking Model: Bending the Inference Scaling Curve to Serve LLM-Scale Models for Ads](https://engineering.fb.com) | Mar 2026 | How Meta adapted LLM-scale models for efficient ad ranking inference |
| [Ranking Engineer Agent (REA): The Autonomous AI Agent Accelerating Meta's Ads Ranking Innovation](https://engineering.fb.com) | Mar 2026 | An AI agent that autonomously improves ads ranking systems |
| [The Death of Traditional Testing: Agentic Development Broke a 50-Year-Old Field, JITTesting Can Revive It](https://engineering.fb.com) | Feb 2026 | How agentic AI development is disrupting software testing paradigms |
| [Adapting the Facebook Reels RecSys AI Model Based on User Feedback](https://engineering.fb.com) | Jan 2026 | Techniques for continuously adapting recommendation systems using real user signals |
| [DrP: Meta's Root Cause Analysis Platform at Scale](https://engineering.fb.com) | Dec 2025 | Appears in both categories — great for data scientists working on model reliability |
