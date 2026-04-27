# Zoomer: Powering AI Performance at Meta's Scale

**Source:** [Meta Engineering Blog, Nov 21 2025](https://engineering.fb.com/2025/11/21/data-infrastructure/zoomer-powering-ai-performance-meta-intelligent-debugging-optimization/)

---

## What is Zoomer?

Meta's automated **debugging and performance optimization platform for AI workloads** — covering both training and inference. Generates tens of thousands of profiling reports daily across thousands of hosts simultaneously.

---

## Three-Layer Architecture

### 1. Infrastructure & Platform Layer
- Distributed blob storage via **Manifold**
- Fault-tolerant pipelines handling massive trace files
- Low-latency data collection across thousands of hosts at once

### 2. Analytics & Insights Engine

| Tool | What it profiles |
|---|---|
| Kineto + NVIDIA DCGM | GPU analysis (SM utilization, Tensor Core, memory) |
| StrobeLight | CPU profiling, request-level analysis |
| PyTorch Profiler | Kernel-level GPU operations |
| dyno telemetry | Host-level metrics |
| NCCL analysis | Communication patterns in distributed training |

**Profiling data captured:**
- SM utilization, GPU memory utilization, Tensor Core utilization
- Kernel-level GPU operations
- NCCL collective operations for distributed workloads
- Inference request latency breakdowns
- Training iteration annotations (forward/backward passes)
- Memory allocation and leak detection

### 3. Visualization & UI Layer
- Interactive timelines across thousands of ranks
- Percentile drill-downs and heat maps for outlier detection
- Perfetto trace viewer integration
- Multi-iteration analysis for long-running workloads

---

## Advanced Analysis Capabilities

### Straggler Detection
- Identifies slow ranks in distributed training via comparative timeline analysis
- Critical at 32k–64k GPU scale where one slow node bottlenecks the entire job

### Critical Path Analysis
- Surfaces the longest execution paths to focus optimization effort where it matters most

### Anti-Pattern Detection
- Rule-based system that flags known efficiency issues and surfaces specific recommendations

### Specialized Workload Support
- Purpose-built for GenAI (100k+ GPU configs), ads ranking, and content recommendations

---

## Measured Impact

| Optimization | Result |
|---|---|
| Ads relevance model training time | -75% |
| Associated power consumption | -78% |
| Memory optimization → QPS | +20% |
| Inference power (parameter tuning) | -10% to -45% |
| Inference QPS (bottleneck fixes) | +2% to +50% |
| 32k GPU benchmark speedup | +30% |
| 64k GPU benchmark speedup | +25% |

---

## Hardware Support

NVIDIA GPUs, AMD MI300X, MTIA (Meta's own Training & Inference Accelerator), and CPU-only workloads — consistent analysis interface across all platforms.

---

## Future Direction
- Heterogeneous hardware support
- Proactive optimization analyzers (flag issues before they hurt production)
- Democratizing tooling for all engineers, not just infra specialists

---

## Key Takeaways for DE Practice

- Profiling at scale requires **fault-tolerant, distributed collection pipelines** — trace files from thousands of hosts can't be handled naively
- Straggler detection is a pattern applicable beyond GPU training — any distributed system (Spark, Kafka consumers) has analogous "slow node" problems
- Anti-pattern detection (rule-based + automated) is a practical observability pattern: codify known failure modes, run them continuously
- Critical path analysis is useful for any DAG-based workload (Airflow, dbt) — optimize the bottleneck, not random nodes
