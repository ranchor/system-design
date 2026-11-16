# Speculative Decoding & Quantization

> **TL;DR** — Decode is **memory-bandwidth-bound and serial**, so two classes of optimization dominate. **Quantization** shrinks weights (and KV cache) to FP8/INT8/INT4 so fewer bytes move per token and more sequences fit: faster decode, larger batches, lower cost,
> with some quality risk. **Speculative decoding** uses a cheap **draft** (a small model, extra heads, or n-gram lookup) to propose several tokens that the large model **verifies in one parallel pass**, accepting a prefix: **same output distribution**, fewer sequential steps.
> Both help latency (TPOT) most at low-to-moderate batch sizes; gains shrink as batches grow compute-bound. See [LLM inference basics](llm_inference_basics.md), [Batching](continuous_batching_scheduling.md), [KV cache](kv_cache_prefix_caching.md).

## Speculative decoding
**Idea**: because the large model is idle on compute during a memory-bound step, verifying `k` drafted tokens costs about the same as generating one.

```
draft model:  proposes t1 t2 t3 t4         (cheap, sequential)
target model: scores [t1..t4] in ONE forward pass → accepts the longest correct prefix (+1 bonus token)
repeat from the first rejected position
```
- **Correctness**: with the rejection-sampling scheme from the original paper, outputs follow the **same distribution** as the target model alone (lossless). Speed depends on the **acceptance rate** and draft cost.
- **Speedup** ≈ (expected accepted tokens per step) ÷ (1 + draft overhead). Higher for predictable text (code, structured output), lower for creative sampling at high temperature.

| Variant | Draft source | Notes |
|---|---|---|
| **Draft model** | Small model of the same family | Needs a compatible tokenizer; extra memory and a second model to serve |
| **Medusa-style heads** | Extra prediction heads on the target | Training needed; tree verification |
| **EAGLE** | Lightweight autoregression at the feature (second-to-top-layer) level, tree-structured drafts | Reported ~3× faster than vanilla decoding on MT-bench at its submission, preserving output distribution |
| **n-gram / prompt lookup** | Copy spans from the prompt | No extra model; great for summarization, RAG, code edits |
| **Self-speculative / multi-token prediction** | Early exit or models trained to predict several tokens | Model-dependent (e.g. DeepSeek-V3 trains with a multi-token prediction objective) |

**System considerations**
- Scheduler must handle **variable accepted tokens per step** and extra KV for draft tokens (roll back rejected ones) ([Batching](continuous_batching_scheduling.md)).
- Benefit **shrinks at high batch size** (GPU becomes compute-bound), so it suits latency-sensitive, lower-concurrency deployments; adaptive systems enable it only when load is low.
- Draft/target must stay **version-aligned**; monitor acceptance rate per workload.
- Interacts with structured/grammar decoding and sampling params.

## Quantization
**Idea**: represent weights (and optionally activations/KV) with fewer bits. In decode, time ≈ bytes read ÷ bandwidth, so halving bytes nearly doubles per-step speed when weights dominate.

| Target | Common formats | Effect |
|---|---|---|
| **Weights only** | INT4/INT8 (AWQ, GPTQ), FP8 weights | Biggest decode win and memory saving; activations stay higher precision (dequantize on the fly) |
| **Weights + activations** | FP8 (Hopper/Blackwell), INT8 (SmoothQuant-style), FP4 on newest GPUs | Speeds up compute-bound prefill and large batches; needs calibration |
| **KV cache** | FP8 / INT8 / INT4 KV | More concurrent tokens per GPU; small quality risk on long contexts |
| **Adapters** | QLoRA-style low-bit base + LoRA | Cheap multi-tenant fine-tunes served from one base ([Serving platform](llm_serving_platform.md)) |

- **AWQ** (activation-aware weight quantization) observes that a small fraction (~1%) of salient weight channels matters most and protects them by scaling, enabling low-bit weight-only quantization without retraining; it won the MLSys 2024 Best Paper Award.
- **Post-training quantization (PTQ)** needs a small calibration set; **quantization-aware training (QAT)** gives better quality at higher cost.
- **Quality check**: evaluate on *your* tasks (perplexity alone is insufficient); watch long-context, math/code and tool-calling regressions; keep an FP16/FP8 baseline for A/B.
- **Hardware support** determines real speedups (native FP8/FP4 tensor cores vs dequantization overhead).

## Related optimizations (for completeness)
| Technique | What | Where it helps |
|---|---|---|
| **FlashAttention** | Tiled exact attention, less HBM traffic | Prefill and long context |
| **CUDA graphs / kernel fusion** | Remove launch overhead | Small-batch decode latency |
| **Prefix caching, chunked prefill** | See [KV cache](kv_cache_prefix_caching.md), [Batching](continuous_batching_scheduling.md) | TTFT / TPOT |
| **Distillation / smaller models** | Cheaper model for easy requests | Routing by difficulty ([Serving platform](llm_serving_platform.md)) |
| **Sparse attention / sliding window, MLA/GQA** | Smaller KV, less compute | Long context |

## Decision guide
1. Memory-bound decode, latency SLO, moderate load → **weight quantization** first (cheap, broad win), then **speculative decoding** if TPOT is still too high.
2. Throughput/cost at high load → FP8 weights+activations and KV quantization to enlarge batches.
3. Predictable or retrieval-heavy outputs → n-gram/prompt-lookup speculation is nearly free.
4. Always measure acceptance rate, quality metrics and end-to-end goodput, not just tokens/s.

## Pitfalls
- Assuming speculative decoding always helps (it can hurt at high batch size or low acceptance); mismatched draft/target tokenizers; aggressive 3-4-bit quantization without task evals; quantizing KV for long-context workloads without testing; ignoring calibration data drift.

## Interview follow-ups
- *"Why does speculative decoding preserve quality?"* The verification step accepts or resamples tokens so the output distribution equals the target model's.
- *"When does it not help?"* Large batches (already compute-bound), low acceptance rates, very high temperature sampling.
- *"Quantization trade-off?"* Less memory and faster decode vs possible accuracy loss; verify on real workloads.
- *"Which first, quantization or speculation?"* Quantization (broad, simple), then speculation for latency-critical paths.

# References
* [Fast Inference from Transformers via Speculative Decoding (Leviathan et al., ICML 2023)](https://arxiv.org/abs/2211.17192)
* [EAGLE: Speculative Sampling Requires Rethinking Feature Uncertainty](https://arxiv.org/abs/2401.15077)
* [AWQ: Activation-aware Weight Quantization for LLM Compression and Acceleration](https://arxiv.org/abs/2306.00978)
* [DeepSeek-V3 Technical Report (multi-token prediction, FP8 training)](https://arxiv.org/abs/2412.19437)
* [vLLM documentation](https://docs.vllm.ai/)
