# Continuous Batching & LLM Request Scheduling

> **TL;DR** — Naive **static batching** waits for a whole batch to finish, so short requests idle behind long ones and new requests wait. **Continuous (iteration-level) batching**, introduced by Orca and used by vLLM and every modern
> engine, re-forms the batch **every decode step**: finished sequences leave immediately and waiting ones join. Schedulers then juggle **prefill vs decode** work (chunked prefill / stall-free batching), **KV-memory limits** (admit, preempt, swap or recompute),
> and **priorities/SLOs**. The result is far higher GPU utilization at stable latency. See [LLM inference basics](llm_inference_basics.md), [KV cache](kv_cache_prefix_caching.md), [Prefill/decode disaggregation](prefill_decode_disaggregation.md), [Backpressure](backpressure_load_shedding.md).

## Why static batching fails for LLMs
```
Static:      [A████████████][B████][C██████]  ← batch ends when A (longest) ends; B, C slots idle; D waits for next batch
Continuous:  A████████████
             B████ D██████ F███        ← each step, finished seqs leave and queued seqs join
             C██████ E█████████
```
Outputs have **unpredictable lengths**; request-level batching leaves GPU slots empty and inflates tail latency. Orca's **iteration-level scheduling** runs one model iteration at a time and lets the scheduler change the batch between iterations, with **selective batching** (batch only the ops that can be batched when sequences are at different positions).

## The scheduler loop (per iteration)
1. **Collect** running sequences (decode, one token each) and waiting requests (need prefill).
2. **Admit** new requests only if there's KV memory (free blocks) and the **token budget** for this iteration allows.
3. **Build the batch**: decode tokens for all running sequences + (chunked) prefill tokens for admitted ones, up to the max batched tokens / max sequences.
4. **Run** one forward pass; sample next tokens; stream them out.
5. **Retire** finished sequences (EOS / max tokens / stop string / cancelled) and free their KV blocks; **preempt** if memory runs out.

## Prefill vs decode in one batch
- Adding a full long prefill to a decode batch makes that iteration slow → **generation stalls** (TPOT spikes) for everyone already decoding. Prioritizing decodes instead raises TTFT for new requests.
- **Chunked prefill / stall-free batching (Sarathi-Serve)**: split prompts into chunks sized to a **token budget** and mix one chunk with ongoing decodes each iteration. This keeps iteration time uniform (steady TPOT, fewer pipeline bubbles) while still filling the GPU. Reported gains in the paper: up to 2.6× throughput for Mistral-7B on one A100 and 6.9× for Falcon-180B on 8 A100s over Orca/vLLM at the same latency SLOs.
- Trade-off: smaller chunks → smoother decode but more iterations to finish prefill (higher TTFT for long prompts) and re-reading KV for each chunk. Tune the budget to the SLO.
- Going further: **disaggregate** prefill and decode onto separate pools ([Prefill/decode disaggregation](prefill_decode_disaggregation.md)).

## Memory pressure: admission, preemption, swapping
| Mechanism | What it does | Cost |
|---|---|---|
| **Admission control** | Don't start a request unless blocks for prompt (+ expected output) are available | Higher queueing, but no thrash |
| **Preemption by recompute** | Evict a sequence's KV; re-run prefill later | Compute spent twice |
| **Preemption by swap** | Copy KV blocks to CPU memory, bring back later | PCIe transfer, CPU memory use |
| **Max tokens / output caps** | Bound worst-case KV per request | Truncated outputs |
| **Reservation vs lazy allocation** | PagedAttention allocates blocks on demand; reservation is safer but wastes memory | Utilization vs preemption risk |

## Scheduling policies and fairness
- **FCFS** is simple but a long prompt can block short ones; **shortest-job-first / priority queues** protect interactive traffic; **multi-tenant fair queuing** (per-tenant token budgets) prevents noisy neighbors ([Rate limiting](backpressure_load_shedding.md)).
- **SLO-aware scheduling**: prioritize requests closest to their TTFT/TPOT deadline; shed or downgrade when overloaded.
- **Prefix-aware ordering**: group requests sharing a prefix to maximize cache hits ([KV cache](kv_cache_prefix_caching.md)).
- **Queue time is TTFT**: monitor queue depth and oldest-request age; autoscale on those, not GPU utilization.
- **Cancellation**: when a client disconnects, stop generation and free KV immediately (otherwise you pay for tokens nobody reads).

## Interactions with other techniques
- **Speculative decoding** changes the per-iteration work (verify several draft tokens), so the scheduler accounts for variable tokens per sequence ([Speculative decoding](speculative_decoding_quantization.md)).
- **Pipeline parallelism** benefits from uniform iterations (fewer bubbles) ([Parallelism](llm_parallelism.md)).
- **Structured outputs / grammar constraints** add CPU work per step; overlap with GPU compute.
- **CUDA graphs** capture fixed shapes for decode; batch sizes are bucketed to match.

## Pitfalls
- Setting `max_num_batched_tokens` / `max_num_seqs` without measuring; unlimited prompt length; FCFS with mixed workloads; no preemption policy so OOM kills the engine; not cancelling abandoned streams; autoscaling on utilization instead of queue delay.

## Interview follow-ups
- *"What is continuous batching and why does it help?"* Re-forms the batch each iteration so finished sequences free slots immediately; raises utilization and cuts waiting.
- *"Why does a long prompt hurt other users?"* Its prefill occupies an iteration, stalling decodes; chunked prefill or disaggregation fixes it.
- *"What happens when KV memory runs out?"* Admission control, then preemption by recompute or swap.
- *"How do you protect interactive users from batch jobs?"* Priority queues, separate pools, per-tenant token budgets.

# References
* [Orca: A Distributed Serving System for Transformer-Based Generative Models (OSDI '22)](https://www.usenix.org/conference/osdi22/presentation/yu)
* [Taming Throughput-Latency Tradeoff in LLM Inference with Sarathi-Serve (OSDI '24)](https://arxiv.org/abs/2403.02310)
* [Efficient Memory Management for LLM Serving with PagedAttention](https://arxiv.org/abs/2309.06180)
