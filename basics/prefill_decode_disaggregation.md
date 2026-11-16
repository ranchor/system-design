# Prefill/Decode Disaggregated Serving (LLM Inference)

> **TL;DR** — LLM inference has two phases with opposite bottlenecks: **prefill** processes the whole prompt in one parallel pass (**compute-bound**, determines **TTFT**), and **decode** generates one
> token at a time reusing the **KV cache** (**memory-bandwidth-bound**, determines **TPOT**). Running both on the same GPUs makes them **interfere** (a long prefill stalls everyone's decode steps) and forces one
> parallelism/batch configuration on both. **Prefill/decode (P/D) disaggregation** runs them on **separate GPU pools** and **transfers the KV cache** from prefill to decode over a fast interconnect, so each pool is tuned and
> scaled independently and both SLOs are met with fewer GPUs. It pays off for **long prompts, strict latency SLOs, high load**; it loses for short prompts, light load, or slow interconnects. General concept:
> [Disaggregated serving](disaggregated_serving.md). Related: [Caching](caching.md), [Load balancing](load_balancing.md), [Backpressure](backpressure_load_shedding.md), [Hot keys](hot_keys.md).

![](../resources/basics/llm/prefill_decode_disaggregation.svg)

## Background: what an LLM request does
1. **Prefill**: all prompt tokens (`N` of them) go through the model in parallel, producing the **KV cache** (per layer, per token: keys and values) and the **first output token**. Arithmetic intensity is high → **compute-bound**, scales ~with prompt length (attention cost grows ~quadratically).
2. **Decode**: autoregressive loop; each step feeds one new token, **reads the entire model weights + KV cache** from HBM, writes one more KV entry, and samples the next token. Arithmetic intensity is low → **memory-bandwidth-bound**; larger **batches** amortize the weight reads, so decode wants *many concurrent sequences*.
3. **KV cache memory**: roughly `2 × layers × kv_heads × head_dim × bytes × tokens` per sequence (e.g. tens to hundreds of KB per token for large models; GBs per long sequence). It's the main capacity limiter for decode batch size. **PagedAttention** (vLLM) allocates it in blocks to avoid fragmentation.

| Metric | Meaning | Driven by |
|---|---|---|
| **TTFT** | Time to first token | Queueing + prefill (prompt length, prefill batch) |
| **TPOT / ITL** | Time per output token / inter-token latency | Decode step time (batch size, KV size, memory bandwidth), interference |
| **Throughput** | Tokens/s/GPU | Batch size, utilization |
| **Goodput** | Requests/s that meet *both* TTFT and TPOT SLOs | What disaggregation actually optimizes |

## Prefill operations in detail
What a prefill worker actually does for one request with `N` prompt tokens:

| Step | Operation | Notes |
|---|---|---|
| 1. Admission & scheduling | Queue the request; pick it into a batch under a **token budget** (e.g. a max number of tokens per forward pass), not a request count | Queueing delay here is a large part of TTFT; shortest-job-first helps tail TTFT |
| 2. Prefix-cache lookup | Hash/match the prompt against cached KV blocks (radix tree / block hashes); **reuse matched prefix blocks** and compute only the uncached suffix | Big win for shared system prompts, few-shot examples, multi-turn chat ([Caching](caching.md)) |
| 3. Tokenize & embed | Tokenization (CPU), embedding lookup for each token (+ positional encoding such as RoPE applied in attention) | Multimodal: encoder output embeddings are spliced in here ([Disaggregated serving](disaggregated_serving.md)) |
| 4. Per layer: QKV projection | Large matrix multiplies over all `N` tokens at once (GEMMs) | The compute-dense part; high arithmetic intensity, good tensor-core utilization |
| 5. Per layer: attention | Causal self-attention over the prompt using a fused kernel (**FlashAttention**-style, tiled to avoid materializing the `N×N` matrix) | Cost grows ~`O(N²)`; dominates for very long prompts |
| 6. Write KV cache | Store the layer's **K and V** for all `N` tokens into paged KV blocks | This is the artifact decode needs; its size is what gets transferred in disaggregated mode |
| 7. Per layer: output projection + FFN/MoE | Dense FFN GEMMs, or router + expert GEMMs for MoE; residuals and norms | ~2/3 of FLOPs for typical dense models at moderate `N` |
| 8. Tensor-parallel sync | **All-reduce** after attention and FFN when using tensor parallelism | Needs NVLink-class bandwidth; adds latency that grows with TP degree |
| 9. Final layer & first token | Compute logits **only for the last position**, apply sampling (temperature/top-p), emit token #1 | **TTFT** ends here; in P/D mode the KV blocks + token #1 are then handed to the decode pool |

**Cost model.** Prefill FLOPs ≈ `2 × parameters × N` for the dense parts (plus the quadratic attention term). A 70B model on a 4K-token prompt is on the order of ~5.6×10^14 FLOPs, so a single request can saturate GPUs for a noticeable fraction of a second. Because the work is large and parallel across tokens, **prefill is compute-bound even at batch size 1**, so batching more requests helps little once the token budget fills the GPU; that is the opposite of decode.

**Techniques that shape prefill:**
- **Chunked prefill**: split a long prompt into chunks (e.g. 512-2048 tokens), process one chunk per iteration, optionally mixed with decode tokens ("stall-free batching"). Bounds the time any one prefill blocks others; each chunk attends to the KV already written by earlier chunks.
- **Token-budget batching**: pack several short prompts into one forward pass up to the budget; cap very long prompts via chunking.
- **Prefix caching / KV reuse**: skip recomputation of shared prefixes; routers send requests to workers that already hold the prefix.
- **Context/sequence parallelism** (ring attention, context parallel): split a very long prompt across GPUs so a single prefill fits in memory and finishes faster.
- **Kernel optimizations**: FlashAttention, fused QKV/FFN kernels, FP8/FP4 quantized GEMMs, CUDA graphs less relevant here (variable shapes) than in decode.
- **Pipelined KV transfer**: stream each layer's KV to the decode worker as soon as that layer finishes, overlapping network with the remaining layers' compute.
- **Prefill-only SLOs**: TTFT budget = queue wait + prefill compute + (in P/D mode) first-hop KV transfer setup; monitor each part separately.

**What can go wrong in prefill:** long prompts monopolizing a worker (head-of-line blocking; use chunking and priority queues), out-of-memory from huge activation or KV allocations, cache-miss storms after a deploy that clears the prefix cache, and uneven prompt-length distribution making a fixed batch size inefficient ([Hot keys](hot_keys.md), [Backpressure](backpressure_load_shedding.md)).

## The problem with colocation
- **Prefill-decode interference**: with continuous batching, a long prompt's prefill occupies the GPU for 100s of ms-seconds, delaying every in-flight decode step → **TPOT spikes / jitter**. Conversely, decode-heavy batches slow prefills → higher TTFT.
- **Coupled configuration**: ideal tensor/pipeline parallelism, batch size and even GPU type differ per phase, but a colocated replica has one.
- **Over-provisioning**: to hit strict TTFT *and* TPOT, you add GPUs until the worst interference is tolerable, wasting capacity most of the time.
- **Mitigation without splitting**: **chunked prefill** (Sarathi-Serve) splits prompts into chunks interleaved with decodes, trading some TTFT for steadier TPOT. Often the right first step; disaggregation goes further by removing contention entirely.

## Disaggregated architecture
```
Client ─▶ Router/scheduler ─▶ Prefill worker ──(KV cache transfer)──▶ Decode worker ─▶ stream tokens
                │  pool state, queue depth, KV-cache hit/locality      ▲
                └────────────── control plane: autoscale, rebalance ───┘
```
- **Prefill pool**: optimized for TTFT; small/“token-budget” batches, parallelism tuned for latency (often higher TP), may use compute-dense GPUs. Can reuse **prefix cache** hits to skip recomputation.
- **Decode pool**: optimized for throughput under a TPOT SLO; big batches, memory-bandwidth-rich GPUs, parallelism and KV-memory management tuned for capacity (sometimes lower TP + more replicas, or expert parallelism for MoE).
- **KV transfer**: after prefill, the KV blocks (and first token) move to a chosen decode worker; transfer can be **layer-by-layer pipelined** with prefill compute to hide latency, using **NVLink/NVSwitch (intra-node), RDMA/InfiniBand/RoCE (cross-node), GPUDirect**, or via a host/CPU/remote store (NIXL, Mooncake Transfer Engine, NCCL, LMCache connectors).
- **Router**: picks prefill worker (prefix-cache locality, load) and decode worker (free KV memory, queue); handles **backpressure** and admission ([Backpressure](backpressure_load_shedding.md)). Can be **pull-based** (decode pulls KV when it has capacity) or push-based.
- **Control plane**: autoscales pool sizes by **prompt-length/output-length mix**, queue delay and SLO attainment; rebalances ratio (e.g. 1:2 prefill:decode) as traffic shifts.

## When does it win?
Break-even condition (conceptual): **KV transfer time + extra hop < interference/over-provisioning cost removed.**
| Favors disaggregation | Favors colocation (+ chunked prefill) |
|---|---|
| Long prompts (RAG, agents, code, summarization), long outputs | Short prompts, short outputs |
| Tight TTFT **and** TPOT SLOs simultaneously | Loose or one-sided SLOs |
| High, steady load (big pools to specialize) | Low/bursty load, small cluster |
| Fast interconnect (NVLink/RDMA), big models | Slow network, small models, single node |
| Heterogeneous hardware available | Homogeneous, small fleet |
| Need independent autoscaling per phase | Operational simplicity valued |
Sizing intuition: KV size ∝ prompt tokens; transfer time = KV bytes ÷ bandwidth (e.g. several GB over 200-400 Gb/s RDMA ≈ 100s of ms; over NVLink ≈ ms) → must be small relative to the prefill time it overlaps.

## Design details and trade-offs
- **Sizing the ratio**: prefill capacity ∝ input tokens/s; decode capacity ∝ concurrent sequences × KV memory. Start from measured TTFT/TPOT curves per pool; search for the split that maximizes goodput per GPU (DistServe optimizes placement and parallelism per phase).
- **Prefix caching & KV reuse**: shared system prompts / multi-turn context avoid recomputation; route requests to prefill workers holding the prefix; a **global KV cache pool** (CPU DRAM/SSD/remote, e.g. Mooncake, LMCache) extends capacity beyond HBM ([Caching](caching.md), [Consistent hashing](consistent_hashing.md) for cache-aware routing).
- **Heterogeneous GPUs**: Splitwise showed prefill on compute-rich and decode on cheaper/high-memory-bandwidth GPUs can cut cost/power; KV transfer bandwidth is the constraint.
- **Parallelism per phase**: tensor parallel reduces prefill latency; decode often prefers more replicas/larger batches; MoE models use expert parallelism with different layouts per phase.
- **Failure handling**: if a decode worker dies mid-stream, re-run prefill (or reload KV from a cache tier) and resume, or fail the request; prefill failure is cheap to retry (idempotent); keep KV transfers **time-bounded** with fallback to local decode; health checks and circuit breakers ([Resilience](resilience_patterns.md)).
- **Scheduling policy**: separate queues and admission; prioritize short prompts in prefill (SJF-like) to protect TTFT; decode scheduling to avoid KV memory exhaustion (preemption, swap, recompute).
- **Cost of the split**: extra network traffic and complexity, two pools to scale/monitor, potential decode-side head-of-line blocking if prefill outputs outpace capacity; observability must span both phases (per-request trace with TTFT, transfer time, TPOT).
- **Alternatives/complements**: chunked prefill, speculative decoding, quantization (smaller weights/KV), KV-cache compression/eviction, continuous batching, prefix caching. Disaggregation composes with all of them.

## Where it's used
Research: **DistServe** (OSDI '24), **Splitwise** (ISCA '24), **Mooncake** (Kimi; KV-centric disaggregation, FAST '25), **TetriInfer**. Systems: **vLLM** (disaggregated prefill connectors), **SGLang**, **NVIDIA Dynamo**, **llm-d** (Kubernetes-native, vLLM-based), TensorRT-LLM; large providers run P/D splits internally. Check each project's current docs for maturity and supported transports (this area changes quickly).

## Case study: NVIDIA Dynamo
Dynamo (introduced by NVIDIA in 2025 as the successor to Triton Inference Server for multi-node generative AI; open source on GitHub, `ai-dynamo/dynamo`) is a good concrete example of the architecture above. It supports TensorRT-LLM, vLLM, SGLang and PyTorch backends. Its four components map directly to the design pieces in this note:

| Dynamo component | Role | Maps to |
|---|---|---|
| **Planner** | Watches GPU capacity and latency targets; decides per request whether to disaggregate, and shifts/adds GPUs between the prefill and decode pools | Control plane: autoscaling the P:D ratio |
| **Smart Router** | Tracks where KV blocks live across the cluster (radix-tree hashing of prompts), scores cache overlap, and balances reuse against load, instead of round-robin | Prefix-cache-aware routing ([Consistent hashing](consistent_hashing.md)) |
| **Distributed KV Cache Manager** | Offloads cold KV blocks from HBM to CPU memory, local SSD or object storage at GPU, node and cluster levels | KV-cache tiering ([Disaggregated serving](disaggregated_serving.md)) |
| **NIXL** (Inference Transfer Library) | Async point-to-point transfers across HBM, DRAM, SSD and networked storage with one API over GPUDirect Storage, UCX, S3, etc. | KV transfer layer |

NVIDIA's reported results (vendor numbers, measured on their own configurations; treat as indicative, not as a benchmark you can generalize):
- Up to **30× more requests served** for DeepSeek-R1 671B on GB200 NVL72 (TensorRT-LLM, FP4, 32K input / 8K output). This figure is footnoted as *projected performance, subject to change*, and the baseline was in-flight batching on a different parallelism layout than the disaggregated setup (which used separate context and generation parallelism).
- **More than 2× throughput** for Llama 70B on Hopper GPUs (vLLM, FP8, ~3K input / 50 output): baseline TP8×DP2 vs disaggregated context TP2×DP4 and generation TP8. Note the *different parallelism per phase*, which is exactly the benefit described above.
- Smart Router: improved TTFT and end-to-end latency in their test on 2 HGX-H100 nodes with 8 vLLM replicas of DeepSeek-R1-Distill-Llama-70B and 100K real requests (avg. 4K input / 800 output); exact gains are shown only in charts.

Takeaway for interviews: the headline gains come from combining **per-phase parallelism, cache-aware routing, and KV tiering**, not from the split alone, and they are largest for long-input, reasoning-style workloads.

## Interview angle (how to talk about it)
1. State the two phases and their bottlenecks (compute-bound vs memory-bound) and the two SLOs (TTFT, TPOT).
2. Show the interference problem and mention chunked prefill as the cheaper first fix.
3. Present the split with **KV transfer** as the key cost, name interconnects, and give the break-even reasoning.
4. Discuss routing (prefix locality, load), autoscaling the P:D ratio, failure handling, and observability.
5. Tie to general patterns: right-size resources per workload ([Scaling reads/writes](scaling_reads.md)), independent scaling, backpressure.

## Pitfalls
- Disaggregating small models/short prompts (transfer cost dominates); slow or congested network; fixed P:D ratio that drifts from the workload mix; ignoring KV memory exhaustion on decode; no timeouts/fallback on transfer; measuring tokens/s instead of **goodput**; forgetting multi-turn prefix locality in routing.

## Interview follow-ups
- *"Why is decode memory-bound?"* Each step reads all weights and the KV cache to produce one token per sequence; low FLOPs per byte moved, so batch size and HBM bandwidth set speed.
- *"What does the KV transfer cost and how do you hide it?"* KV bytes ÷ link bandwidth; pipeline per layer with prefill compute, use RDMA/NVLink, compress/quantize KV, prefer co-located nodes.
- *"How do you size the pools?"* From measured prefill tokens/s and decode concurrency under SLO; autoscale on queue delay and SLO attainment.
- *"Chunked prefill vs disaggregation?"* Chunked prefill is simple and reduces interference but still shares GPUs; disaggregation removes it and allows per-phase tuning at the price of transfer and complexity.

# References
* [DistServe: Disaggregating Prefill and Decoding for Goodput-optimized LLM Serving (OSDI '24)](https://arxiv.org/abs/2401.09670)
* [Splitwise: Efficient generative LLM inference using phase splitting (ISCA '24)](https://arxiv.org/abs/2311.18677)
* [Mooncake: A KVCache-centric Disaggregated Architecture for LLM Serving](https://arxiv.org/abs/2407.00079)
* [Sarathi-Serve: chunked prefills (OSDI '24)](https://arxiv.org/abs/2403.02310), [Efficient Memory Management for LLM Serving with PagedAttention](https://arxiv.org/abs/2309.06180)
* [Introducing NVIDIA Dynamo (NVIDIA Technical Blog)](https://developer.nvidia.com/blog/introducing-nvidia-dynamo-a-low-latency-distributed-inference-framework-for-scaling-reasoning-ai-models/)
* [vLLM docs: disaggregated prefilling](https://docs.vllm.ai/en/latest/features/disagg_prefill.html), [NVIDIA Dynamo repo](https://github.com/ai-dynamo/dynamo), [llm-d repo](https://github.com/llm-d/llm-d), [llm-d architecture](https://llm-d.ai/docs/architecture/architecture)
* [Orca (OSDI '22)](https://www.usenix.org/conference/osdi22/presentation/yu), [SGLang / RadixAttention](https://arxiv.org/abs/2312.07104), [FlashAttention](https://arxiv.org/abs/2205.14135)
* Related notes: [LLM inference basics](llm_inference_basics.md), [Continuous batching](continuous_batching_scheduling.md), [KV cache](kv_cache_prefix_caching.md), [Parallelism](llm_parallelism.md), [Serving platform](llm_serving_platform.md)
