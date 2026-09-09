---
layout: default
title: "Finding the knee: how much load one L4 takes before it starts queueing"
date: 2026-09-09
tags: [inference, vllm, gpu]
---

# {{ page.title }}

**One L4 running a 1.5B model served ~3,000 tokens/s and stayed responsive up to 256 concurrent requests. Past that, throughput stopped rising and the wait for a first token jumped from under a second to eighteen seconds.** Here's how I found that number, and the mistakes on the way.

## Setup

- One L4 (24 GB) hired by the hour on RunPod, vLLM serving `Qwen/Qwen2.5-1.5B-Instruct`.
- Load generated **on the box** with vLLM's own `vllm bench serve`, against localhost, so no internet latency leaks into the numbers.
- Each request: ~200 tokens in, ~200 tokens out (`--dataset-name random`).
- I swept concurrency 8 → 512, with the request count scaled to 10× concurrency each run, so every point does the same amount of work and the percentiles are stable.

Four numbers per run: throughput (tok/s), TTFT (time to first token), TPOT (time per output token after the first), and ITL (the gap between individual tokens).

## The first attempt was wrong, and that's the point

My first go used a homemade script firing a fixed 24 requests and only timing whole requests. It looked like throughput peaked at concurrency 12 and I invented a story about the KV cache to explain it. Both were wrong. With only 24 requests, the result was dominated by how 24 divides into batches, an artifact of the harness, not the GPU. And timing whole requests hides the thing that actually matters: the split between first-token wait and per-token wait.

Switching to `vllm bench serve` with hundreds of requests and proper TTFT/TPOT fixed both. Lesson banked: use the real tool, and don't explain a number you haven't isolated.

I also predicted the knee would be at concurrency 32. It was ~256. Being wrong by 8× is the useful part: this little model on an L4 is far harder to saturate than I guessed.

## The sweep

| concurrency | tok/s | TTFT median | TTFT p99 | TPOT median | ITL median |
| ----------- | ----- | ----------- | -------- | ----------- | ---------- |
| 8           | 234   | 64ms        | 590ms    | 31ms        | 16ms       |
| 16          | 443   | 151ms       | 959ms    | 34ms        | 16ms       |
| 32          | 873   | 147ms       | 933ms    | 33ms        | 17ms       |
| 64          | 1,312 | 377ms       | 1,185ms  | 47ms        | 20ms       |
| 128         | 2,266 | 395ms       | 1,080ms  | 56ms        | 26ms       |
| 256         | 2,989 | 737ms       | 3,354ms  | 84ms        | 40ms       |
| 512         | 2,842 | 17,986ms    | 21,585ms | 86ms        | 40ms       |

![throughput and P99 TTFT vs concurrency]({{ '/assets/2026-09-09-finding-the-knee.png' | relative_url }})

## What it says

**Throughput scales cleanly to ~256, then stops.** 234 → 2,989 tok/s as concurrency climbs, then 512 actually comes in lower (2,842). At ~256 the GPU is producing tokens as fast as it physically can. It's bandwidth-bound: every step reads the weights plus all the KV, so more requests can't be served faster.

**Past the knee, the extra load just queues.** This is the clearest thing in the data. At 512, throughput is flat but median TTFT is 18 seconds. The requests it can't admit sit in line waiting to start. Meanwhile TPOT and ITL barely move (86ms and 40ms, about the same as 256). So the server didn't "get slow": once a request is admitted, tokens come out at full speed. The 18 seconds is entirely wait-at-the-door. TTFT and TPOT measure two different things, and separating them is the whole game.

**Which concurrency would I run?** If there's a latency budget, 128: it delivers ~2,266 tok/s at a P99 TTFT of ~1.1s, three-quarters of peak throughput at a third of the tail wait of 256. If it's pure offline batch and I only care about tokens per dollar, 256. Above 256, never: no more throughput, exploding latency.

**Cost falls the same way.** The L4 bills the same per hour regardless of load, so cost per token is just the hourly rate divided by tok/s. At 256 (~3,000 tok/s) a token is about **13× cheaper** than at concurrency 8 (~234 tok/s). Same GPU, same money, one knob. That's the unit-economics lever most people skip.

## Honest caveats

- Between two sweeps, throughput at matching concurrency differed by ~2× (e.g. conc 16 was 872 tok/s in one run, 443 in another), probably thermal throttling after sustained load, or a shared host. So trust the **shape** (knee ~256, TTFT blow-up at 512), not the absolute tok/s to two digits.
- I inferred "no cache eviction at 512" from TPOT staying flat and rough KV maths (200/200 requests are small, nowhere near filling 24 GB). I did **not** capture `gpu_cache_usage_perc` or `num_preemptions_total`, so that's reasoning, not proof. Deliberately filling the cache with long context is the next experiment.
- One model, one card, short requests. This is the method and the shape, not universal numbers.
