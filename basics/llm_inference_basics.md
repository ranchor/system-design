# LLM Inference Basics for System Design

> **TL;DR** — A served LLM is a **stateful, GPU-bound, token-streaming service**. One request = **prefill** (process the prompt in parallel, compute-bound, builds the **KV cache**) then **decode** (one token per step, memory-bandwidth-bound).
> Capacity is set by **GPU memory** (weights + KV cache) and **memory bandwidth**, not request count. Throughput comes from **batching many sequences** (continuous batching) and managing KV memory well (PagedAttention).
> Quality of service is measured by **TTFT** (time to first token), **TPOT/ITL** (time per output token) and **goodput**. Everything else in this section (batching, caching, parallelism, disaggregation, gateways) is a way to trade these off.
> Next: [Continuous batching](continuous_batching_scheduling.md), [KV cache](kv_cache_prefix_caching.md), [Parallelism](llm_parallelism.md), [Prefill/decode disaggregation](prefill_decode_disaggregation.md).

## Request lifecycle
```
prompt ─▶ tokenize ─▶ PREFILL (all prompt tokens in parallel → KV cache + 1st token) ─▶ DECODE loop (1 token/step, reuse KV) ─▶ detokenize ─▶ stream
```
1. **Tokenization**: text → token ids (CPU). Billing and limits are in tokens, not characters.
2. **Prefill**: one forward pass over all `N` input tokens; writes K/V for every layer and token; emits the first output token. Compute-bound ([details](prefill_decode_disaggregation.md#prefill-operations-in-detail)).
3. **Decode**: autoregressive loop. Each step reads **all model weights + the whole KV cache** from HBM to produce one token per sequence → **memory-bandwidth-bound**; batching sequences amortizes the weight reads.
4. **Sampling**: temperature / top-p / top-k, stop sequences, logit processors, structured-output constraints (grammar / JSON schema).
5. **Streaming**: tokens sent as they're produced (SSE/WebSocket) so users see output early ([Real-time updates](realtime_updates.md)).

## What limits capacity
| Resource | Role | Rule of thumb |
|---|---|---|
| **HBM capacity** | Holds weights + KV cache + activations | Weights ≈ params × bytes (70B in FP16 ≈ 140 GB; FP8 ≈ 70 GB). Remaining memory = KV budget → max concurrent tokens |
| **HBM bandwidth** | Bounds decode speed | Per-step time ≳ (weights + KV bytes read) ÷ bandwidth |
| **Compute (FLOPs)** | Bounds prefill speed | Prefill FLOPs ≈ 2 × params × prompt tokens (+ attention term) |
| **Interconnect** | Tensor/expert parallel all-reduce, KV transfer | NVLink intra-node, RDMA across nodes ([Parallelism](llm_parallelism.md)) |
| **Network & CPU** | Tokenization, streaming, gateway | Rarely the bottleneck, but long streams hold connections |

**KV cache size** per token ≈ `2 × layers × kv_heads × head_dim × bytes`. Grouped-query attention (GQA), multi-query attention and latent attention (MLA, as in DeepSeek-V3) shrink `kv_heads`/dimension and therefore the cache. Long contexts and many concurrent users make KV memory, not weights, the constraint.

## Metrics (what to put in an SLO)
| Metric | Meaning | Mostly driven by |
|---|---|---|
| **TTFT** | Request arrival → first token | Queueing + prefill length |
| **TPOT / ITL** | Gap between output tokens | Decode step time, batch size, interference |
| **End-to-end latency** | TTFT + (output tokens × TPOT) | Output length |
| **Throughput** | Tokens/s per GPU or cluster | Batch size, utilization |
| **Goodput** | Requests/s meeting *both* TTFT and TPOT SLOs | What you actually sell |
| **Cost** | $/1M tokens (input and output priced differently) | GPU-hours ÷ tokens, utilization |

Interactive chat needs low TTFT and TPOT of roughly human reading speed; batch/offline jobs optimize throughput and cost; reasoning models produce long outputs so **decode** dominates.

## Key techniques at a glance
| Technique | Solves | Note |
|---|---|---|
| **Continuous batching** (iteration-level scheduling) | Idle GPU between requests of different lengths | [Batching & scheduling](continuous_batching_scheduling.md) |
| **PagedAttention** | KV memory fragmentation/waste | Allocates KV in blocks, like OS virtual memory ([KV cache](kv_cache_prefix_caching.md)) |
| **FlashAttention** | Attention's memory traffic | Tiled, IO-aware exact attention that avoids materializing the N×N matrix |
| **Prefix caching** | Recomputing shared prompts | Reuse KV blocks across requests |
| **Chunked prefill** | Prefill stalling decodes | Interleave chunks with decode tokens |
| **Quantization** (FP8/INT4/AWQ) | Memory + bandwidth | Smaller weights/KV → bigger batches, faster decode ([Speculative decoding & quantization](speculative_decoding_quantization.md)) |
| **Speculative decoding** | Decode is serial | Draft several tokens, verify in one pass |
| **Tensor/pipeline/expert parallelism** | Model doesn't fit / latency | [Parallelism](llm_parallelism.md) |
| **Prefill/decode disaggregation** | Phase interference, coupled config | [Disaggregation](prefill_decode_disaggregation.md) |
| **Semantic caching, routing, fallbacks** | Cost, reliability | [Gateway](llm_serving_platform.md) |

## Serving engines (the building blocks)
vLLM (PagedAttention, continuous batching), SGLang (RadixAttention prefix reuse, structured outputs), TensorRT-LLM (NVIDIA-optimized kernels), Hugging Face TGI, and orchestration layers such as NVIDIA Dynamo and llm-d for distributed deployments. Features and maturity change quickly; check each project's current docs.

## Design checklist (interview)
1. Clarify workload: input/output length distribution, concurrency, latency SLO (TTFT/TPOT), streaming, model size, multi-tenant?
2. Estimate memory: weights + KV budget → tokens in flight → batch size → GPUs.
3. Estimate throughput: decode tokens/s per GPU from bandwidth; prefill tokens/s from FLOPs.
4. Choose parallelism to fit the model and meet latency; replicas for throughput.
5. Add batching, prefix cache, quantization; consider disaggregation for strict dual SLOs with long prompts.
6. Plan the platform: gateway, routing, rate limits (tokens), autoscaling, fallbacks, observability ([Serving platform](llm_serving_platform.md)).

## Pitfalls
- Sizing by requests/s instead of tokens and KV memory; ignoring output-length variance (a few long generations hog KV); autoscaling on CPU; head-of-line blocking by huge prompts; not capping max tokens; charging input and output equally.

## Interview follow-ups
- *"Why is decode slow per token but cheap per request?"* Memory-bound: each step reads all weights; batching many sequences shares that read.
- *"What fills GPU memory besides weights?"* The KV cache; it grows with tokens × concurrent sequences.
- *"How do you reduce TTFT?"* Prefix caching, shorter/limited prompts, more prefill capacity, chunking/priority, tensor parallelism for prefill, disaggregation.
- *"How do you reduce TPOT?"* Less interference (disaggregation/chunked prefill), quantization, speculative decoding, smaller batch or more replicas.

# References
* [Efficient Memory Management for LLM Serving with PagedAttention (SOSP '23)](https://arxiv.org/abs/2309.06180)
* [FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness](https://arxiv.org/abs/2205.14135)
* [Orca: A Distributed Serving System for Transformer-Based Generative Models (OSDI '22)](https://www.usenix.org/conference/osdi22/presentation/yu)
* [DeepSeek-V3 Technical Report (MLA, MoE)](https://arxiv.org/abs/2412.19437)
* [vLLM documentation](https://docs.vllm.ai/)
