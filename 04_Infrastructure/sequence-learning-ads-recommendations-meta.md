# Sequence Learning: A Paradigm Shift for Personalized Ads Recommendations

**Source:** [Meta Engineering Blog, Nov 19 2024](https://engineering.fb.com/2024/11/19/data-infrastructure/sequence-learning-personalized-ads-recommendations/)

---

## The Shift: From DLRMs to Sequence Learning

Meta moved away from **Deep Learning Recommendation Models (DLRMs)** with hand-engineered features toward **event-based sequence learning** — letting models learn directly from raw user interaction sequences.

---

## What Was Wrong with Traditional DLRMs

| Problem | Impact |
|---|---|
| Sequential information lost through aggregation | Model can't distinguish order of events |
| Granular attribute context destroyed | "User clicked ad" loses category, timing, context |
| Reliance on human-engineered features | Misses complex patterns humans don't anticipate |
| Redundant, overlapping feature spaces | Higher compute cost, diminishing returns |

---

## Event-Based Features (EBFs)

Standardizes model inputs along 3 dimensions:
1. **Event streams** — sequences of user engagement and conversion events
2. **Sequence length** — how many recent events to incorporate
3. **Event information** — semantic/contextual details (ad category, timestamp, etc.)

---

## Sequence Modeling Architecture

### Event Model
- Synthesizes embeddings from raw event attributes
- Encodes timestamps to preserve ordering signal

### Event Sequence Model
- Person-level summarization using attention mechanisms
- Complexity reduced from **O(N²) → O(M×N)** via multi-headed attention pooling
- M = number of output summary vectors, N = sequence length

---

## Infrastructure & Scaling

### Hardware Co-Design
- Native PyTorch **Jagged tensor** support — handles variable-length sequences efficiently without padding waste
- GPU kernel-level optimization
- **Jagged Flash Attention** module for memory-efficient attention on variable-length sequences

### Scaling Longer Sequences
- Multi-precision quantization to fit more events in memory
- Value-based sampling to prioritize the most informative events

### Richer Semantics
- Multimodal embeddings (beyond just click signals)
- Vector quantization for compact representations

---

## Impact

- **2–4% more conversions** on select segments
- Improved ad prediction accuracy
- Higher advertiser value
- Accelerated research velocity (less feature engineering overhead)

---

## Future Direction
- 100x event sequence scaling
- Linear attention and state space model exploration
- Key-value cache optimization
- Multimodal event enrichment

---

## Key Takeaways for DE Practice

- Sequence order matters — aggregating events into counts/sums destroys signal that models can exploit
- Jagged tensors are a key infrastructure primitive for variable-length sequence data at scale — worth knowing for ML pipelines
- The shift from feature engineering → representation learning is a broader trend: raw events + powerful models beat hand-crafted features over time
- Attention pooling (O(M×N)) is a practical pattern for summarizing long sequences without full O(N²) self-attention cost
