---
layout: default
title: "One flag, half the weights: what fp8 did to my 7B on an L4"
date: 2026-10-05
tags: [inference, vllm, gpu]
---

# {{ page.title }}

Most people run a model on whatever the defaults give them and leave it there. I wanted to see what one precision flag actually buys me, so I served the same 7B two ways on the same card and watched the numbers move.

Quick version of the idea before we dive in. A weight is just a number. **bf16** stores each one in 16 bits. **fp8** stores it in 8. Half the bytes. On every **decode** step the GPU still walks the whole pile of weights to produce one output token, so if the pile is smaller, that walk is shorter. **Prefill** (the first pass over your prompt) also reads those weights, but it only happens once per request, while decode happens for every token you generate. So I expected decode to move the most. The trade is quality. Fewer bits means each weight is stored less precisely, like rounding every number to fewer decimal places, so the model becomes a slightly rougher copy of itself and the answers can get a bit worse.

## Setup

One L4 on RunPod, vLLM serving `Qwen/Qwen2.5-7B-Instruct`. `--gpu-memory-utilization 0.92`, `--max-model-len 10240`, `--enforce-eager`. Load on the same machine with `vllm bench serve`: 16 requests at once, 2000 tokens in, 200 out. I discarded the first benchmark after each boot and captured the second one.

Two runs, one thing changed. First `--dtype bfloat16` with no quantization. Then the same line plus `--quantization fp8`. While each run was going I scraped `/metrics` in another window to catch `kv_cache_usage_perc` during the 16, because after the run it just drops back to zero.

## What I saw

|      | duration | tok/s | TTFT p50 | ITL p99 | KV peak during the run |
| ---- | -------- | ----- | -------- | ------- | ---------------------- |
| bf16 | 13.7s    | 233   | 260ms    | 69ms    | ~35%                   |
| fp8  | 9.0s     | 356   | 220ms    | 45ms    | ~17%                   |

Decode is where it showed up. **ITL** p99 dropped from 69ms to 45ms and output went from 233 to 356 tokens per second, so the whole run finished in 9s instead of 13.7s. **TTFT** improved too, 260ms to 220ms, but not nearly as much. That lines up with my initial guess: decode walks the weights for every token, prefill walks them once.

The KV cache part is the bit I like. Same 16 chats, same tokens, but on bf16 they filled about 35% of the cache and on fp8 only about 17%. The K and V are not smaller. The cache is just bigger, because the weights took less room and left more of the card for it. Now imagine this if there were millions of users using your model. Here is the scrape during each run:

**bf16, climbing past 0.35:**

![bf16 KV cache usage during the run]({{ '/assets/2026-10-05-fp8-kv-bf16.png' | relative_url }})

**fp8, topping out around 0.17:**

![fp8 KV cache usage during the run]({{ '/assets/2026-10-05-fp8-kv-fp8.png' | relative_url }})

No preemption on either side. This load was never big enough to fill the cache, so that second win is "more headroom," not "it stopped evicting." You would feel that headroom when you push concurrency or context, which I did not do here.

## The honest part

I did not measure quality properly. I sent one question to each server and both answered in plain English, the fp8 one a touch more generic, neither broken. That is not an eval. If you care about output quality, run a real one on your own prompts before you trust fp8 in anything that matters. I also used the on-the-fly `--quantization fp8`, not a calibrated checkpoint, and this is one model on one card at one load.

So I am not telling you to flip fp8 on in production. I am telling you the defaults are a starting point, not a law. One flag here cut the decode wait by a third and freed up half the cache, and it cost me a few minutes to check. If your use case can take the quality hit, that is a big win sitting behind a flag most people never touch. Worth getting your hands dirty and measuring it on your own model.
