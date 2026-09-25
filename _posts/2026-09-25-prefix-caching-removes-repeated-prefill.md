---
layout: default
title: "Prefix caching removes repeated prefill. It does not make decode cheaper."
date: 2026-09-25
tags: [inference, vllm, gpu]
---

# {{ page.title }}

A chat call is stateless, so you send the whole transcript every time. If the old turns are still in the **KV cache**, the server does not prefill them again. It reuses those blocks, prefills only the new tokens, and those tokens **attend to** the cached prefix so attention actually sees the history. **Decode** is only the new assistant reply, cache or not. Evict the blocks and you prefill the history again.

That reuse is **prefix caching**. vLLM calls it [automatic prefix caching](https://docs.vllm.ai/en/stable/design/prefix_caching/). A new request has to start with the same token IDs, and only a **full block** hits. Block size here is **16 tokens**. **Prefill** is building K and V for the prompt tokens you do not already have. **Decode** is still one new output token per decode step (one forward pass). The cache does not make that step cheaper.

Two things have to be true at once. The cache has to be on, and the next request has to share a prefix. Drop either one and you prefill the whole prompt again. I ran that: same prompt length, prefix shared or not, and I looked at **TTFT** and **ITL**.

## Setup

One L4 on RunPod, vLLM serving `Qwen/Qwen2.5-7B-Instruct`. `--gpu-memory-utilization 0.92`, `--max-model-len 10240`, `--enforce-eager`. Load on the same machine with `vllm bench serve`, 16 requests, concurrency 1, 64 output tokens.

On this vLLM build the cache is **on** by default. Off is `--no-enable-prefix-caching`. I discarded the first benchmark after each boot.

Three runs, same prompt length (~2176 tokens plus a few tokens from the chat template):

- Cache off, every request starts with the same 2048 tokens, then 128 unique.
- Cache on, same shared 2048.
- Cache on, no shared prefix: 2176 unique tokens.

## What I observed


|                              | TTFT p50  | TTFT p99 | ITL p99 | hit rate       |
| ---------------------------- | --------- | -------- | ------- | -------------- |
| cache off, shared 2048       | 643ms     | 655ms    | 58ms    | (not scraped)  |
| cache on, shared 2048        | **98ms**  | 519ms    | 58ms    | **88%**        |
| cache on, unique 2176        | 648ms     | 662ms    | 58ms    | **4%**         |


Hit rate is tokens, not requests: `prefix_cache_hits_total` over `prefix_cache_queries_total`. Snapshot `/metrics` before the run and after, then subtract, so the discard bench is not in the rate. Shared + on: 30976 hits / 35280 queries. Unique + on: 1280 / 35280.

**TTFT** median went from **643ms** to **98ms** when the cache was on and the prefix matched. p99 stayed ~**500ms** because request 1 still prefills. The other fifteen reused those blocks. **ITL** p99 was **58ms** on every run. Decode did not move.

The unique run is the control. Cache still on, nothing in common, **TTFT** back to **648ms**. So the **98ms** was the shared tokens.

A second boot with the cache on repeated the same shape: shared **TTFT** p50 **102ms**, hit rate **88%**; unique **TTFT** p50 **590ms**, hit rate **4%**. **ITL** still **58ms**.

One model, one card, concurrency 1, this load. I have p50 and p99, not the single highest **ITL**. Off may have had a few slower decode gaps that never reached p99. I did not scrape hit rate on the off run. With the cache off there is nothing to hit.
