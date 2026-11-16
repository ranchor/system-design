# Distributed Training of LLMs (Overview)

> **TL;DR** — Training is **throughput-oriented, synchronous and failure-prone** at thousands of GPUs. Memory per GPU holds weights, gradients, **optimizer state** (Adam ≈ 12+ bytes per parameter in mixed precision) and activations, so we combine **data parallelism** with **sharding** (ZeRO / FSDP), **tensor** and **pipeline parallelism** (3D parallelism), **expert parallelism** for MoE, and **activation checkpointing**.
> Collective communication (**all-reduce, all-gather, reduce-scatter, all-to-all**) over NVLink/InfiniBand dominates scaling efficiency, and **checkpointing + fault tolerance** determines goodput. Most interview-relevant ideas parallel the serving side: [Parallelism](llm_parallelism.md), [LLM inference basics](llm_inference_basics.md), [Scalability](scalability.md).

## Memory and what to shard
| State | Per-parameter cost (mixed precision, Adam) | Sharding |
|---|---|---|
| Weights (bf16/fp16) | 2 B | ZeRO-3 / FSDP shards across data-parallel ranks |
| Gradients | 2 B | ZeRO-2+ |
| Optimizer state (fp32 master weights, momentum, variance) | ~12 B | ZeRO-1+ |
| Activations | grows with batch × sequence × layers | Activation checkpointing (recompute), sequence/context parallelism |

**ZeRO** (DeepSpeed) removes DP redundancy in stages: stage 1 shards optimizer state, stage 2 adds gradients, stage 3 adds parameters, at the cost of extra all-gathers. **FSDP** (PyTorch) implements the same fully-sharded idea.

## Parallelism combination
- **DP / FSDP**: replicate or shard across batches; gradient **all-reduce** (or reduce-scatter + all-gather) each step.
- **TP**: shard matrices inside a layer, within an NVLink node.
- **PP**: split layers into stages with micro-batches (GPipe, 1F1B schedules, interleaving) to reduce bubbles.
- **EP**: MoE experts across GPUs with all-to-all; watch load balance (auxiliary losses/bias terms).
- **CP / sequence parallelism**: split long sequences.
- Typical layout: TP within node × PP across nodes × DP/FSDP across replicas; Megatron-LM-style composition reported scaling to trillion-parameter models on thousands of GPUs.

## Communication and efficiency
- Metrics: **MFU** (model FLOPs utilization; tens of percent is typical at scale), tokens/s/GPU, scaling efficiency.
- Overlap communication with compute (bucketed gradient all-reduce, prefetching shards), topology-aware placement (rails, fat-tree), mixed precision (bf16/FP8), fused kernels, FlashAttention.
- Stragglers slow synchronous steps: monitor per-rank step time, detect slow nodes/links.

## Fault tolerance and checkpointing
- At thousands of GPUs, failures are routine (GPU, NIC, host, link). Strategy: **frequent checkpoints** of model + optimizer + data-loader state + RNG; **asynchronous** snapshot to CPU memory then to distributed storage; sharded checkpoints so each rank writes its part ([Large blobs](large_blobs.md), [Object storage](../problems/object_storage_s3.md)).
- Automatic **detect → evict bad node → restart from last checkpoint** with spares; elastic training where supported. Cost of lost work = time since last checkpoint, so checkpoint interval balances overhead against recomputation.
- **Data pipeline**: deterministic, resumable shuffling and sharded streaming datasets; avoid duplicates after restart.
- Numerical issues: loss spikes, NaNs; keep rollback checkpoints and skip-batch tooling.

## Post-training and RL at scale
Fine-tuning (LoRA/QLoRA reduces optimizer memory), preference optimization, and RL with verifiable rewards add a **serving loop inside training**: rollout workers (inference engines, often vLLM/SGLang) generate samples, trainers update weights, and updated weights must be **synced back to inference** efficiently. This makes batching, KV cache and parallelism choices from the serving notes directly relevant ([Continuous batching](continuous_batching_scheduling.md)).

## Training vs serving cheat sheet
| | Training | Serving |
|---|---|---|
| Goal | Max throughput (MFU), convergence | Latency SLO (TTFT/TPOT), goodput, cost |
| Memory hog | Optimizer state + activations | KV cache |
| Comms | Gradient/param collectives | TP all-reduce, EP all-to-all, KV transfer |
| Failure impact | Stalls whole job; restart from checkpoint | Replica loss; reroute |
| Scheduling | Gang scheduled, long-running | Autoscaled, request-driven |

## Pitfalls
- TP across slow links; checkpoints too infrequent or synchronous (stall GPUs); non-resumable data loaders; ignoring stragglers; mixing precision without loss scaling/stability checks; over-sharding small models (communication > compute); no spare capacity for hot-swap.

## Interview follow-ups
- *"Why does Adam make memory so large?"* Master weights plus two moment tensors per parameter; ZeRO/FSDP shards them.
- *"DP vs FSDP?"* DP replicates everything and all-reduces gradients; FSDP shards states and all-gathers parameters on demand.
- *"How do you survive node failures during a months-long run?"* Sharded async checkpoints, health monitoring, automated eviction/restart with spares.
- *"What limits scaling efficiency?"* Communication, pipeline bubbles, stragglers, data pipeline.

# References
* [ZeRO: Memory Optimizations Toward Training Trillion Parameter Models](https://arxiv.org/abs/1910.02054)
* [Megatron-LM: Training Multi-Billion Parameter Language Models Using Model Parallelism](https://arxiv.org/abs/1909.08053)
* [Efficient Large-Scale Language Model Training on GPU Clusters Using Megatron-LM](https://arxiv.org/abs/2104.04473)
* [PyTorch FSDP: Experiences on Scaling Fully Sharded Data Parallel](https://arxiv.org/abs/2304.11277)
* [DeepSeek-V3 Technical Report](https://arxiv.org/abs/2412.19437)
