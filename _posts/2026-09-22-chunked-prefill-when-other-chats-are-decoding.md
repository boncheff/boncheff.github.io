---
layout: default
title: "Chunked prefill when other chats are already decoding"
date: 2026-09-22
tags: [inference, vllm, gpu]
---

# {{ page.title }}

`--enable-chunked-prefill` is a flag in vLLM. When a large prompt arrives while other chats are already **decoding** (one output token per pass), the flag decides how that prompt gets into the GPU.

A **pass** here means one trip through the model for whatever chats vLLM batched together for that step. **Prefill** is processing input tokens; **decode** is emitting output tokens. This post is about prefill for a long prompt landing while decode is already running.

**On**, the large prompt is cut into chunks and fed across many passes. The eight chats that are already decoding can still be in the same batch as one chunk of that prompt, so they still get one output token per pass. The chat with the large prompt does not get its first output token until prefill on that prompt is done. This does not give anyone an earlier first token. It only changes how the input is fed.

**Off**, the whole large prompt has to go in as one block, in as few passes as the server will allow. After that, each pass is mostly the eight chats decoding.

The usual claim is that the flag **on** keeps **ITL** (milliseconds between output tokens during decode) down for the chats already decoding, and the large prompt waits a bit longer for its first token (**TTFT**). That is what I went in expecting. It is not what I observed.

**With the flag off, ITL p99 on those eight was 70ms. With it on, 572ms.** TTFT p50 on the large prompts was 4.5s off and 4.7s on, about the same, not the trade-off people describe. ITL p50 did not move. Only ITL p99 did.

Three things to take from this before the numbers:

- With the flag on, ITL p99 on the eight went from 70ms to 572ms, not “smoother decode.”
- The chats with the large prompts did not wait longer for the first token.
- I have ITL p99, not the single highest ITL. Those are not the same number.

## Setup

- One L4 (24 GB) hired by the hour on RunPod, vLLM serving `Qwen/Qwen2.5-7B-Instruct`, bf16.
- `--gpu-memory-utilization 0.92`, `--max-model-len 10240`, `--enforce-eager`. The flag I set was `--enable-chunked-prefill` on or off. I did not log `--max-num-batched-tokens`. With the flag off, the scheduler often needs a pass at least as large as `--max-model-len`, so the batch token limit may have moved with the flag. I cannot claim only one thing changed.
- Load generated on the same machine with `vllm bench serve` against localhost.
- Two windows. First: 8 chats, 256 tokens in, 800 out, already decoding. Then, while they were still decoding, 4 chats with 8000 tokens in and 200 out.
- `/metrics` showed 12 requests running in both runs.

## What I predicted

With the flag off, each 8000-token prompt has to go into one pass, so that pass slows the eight and ITL goes up. With the flag on, the prompt is chunked so the eight keep sharing passes with prefill on the large prompts, and those four chats wait a bit longer for their first token instead.

## What happened

|          | ITL p50 | ITL p99   | large prompts' TTFT p50 |
| -------- | ------- | --------- | ----------------------- |
| flag off | 60ms    | **70ms**  | 4.5s                    |
| flag on  | 60ms    | **572ms** | 4.7s                    |

The eight decoding chats: 113 tok/s with the flag off, 114 with it on. Duration 56.4s off, 56.2s on. TTFT on the large prompts did not move (4.5s vs 4.7s).

## Why (a guess, as I did not measure this)

ITL is the wait between two output tokens. ITL p99 is that wait, 99% of the way along if you sort them. I did not measure the single highest ITL. p99 is not that number.

ITL p50 is 60ms both ways. Only ITL p99 moved (70ms vs 572ms).

One way to read 70 vs 572 without claiming I counted passes: the eight produced roughly 6400 ITL samples (8 × 800 out). With the flag off, each 8000-token prompt might land in one big pass while the eight are still decoding: four such passes, maybe ~32 slow waits if all eight are in the batch each time, under 1% of 6400, so ITL p99 stays near a normal pass (~70ms). With the flag on, the same 8000 tokens are fed in chunks, and each chunk is batched with the eight, so more slow passes, enough that ITL p99 hits one (~572ms). I did not log `--max-num-batched-tokens` and I did not count passes.

## The number I nearly published

My first run with the flag on was the first benchmark after a boot. On this GPU that first run is slow. 16 chats at 2000 in took 21.6s with ITL p99 of 531ms. The next identical run took 13.7s with ITL p99 of 69ms. I almost compared that slow first run after boot (flag on) to a warm run (flag off) and blamed the flag for the difference. It was not the flag.

The numbers in the table are from a later protocol: boot, one warm-up benchmark and discard the timing, a second warm-up (13.7s, ITL p99 69ms), then the eight decoding chats plus the four large prompts. Same 70 vs 572.

After a restart, discard the first benchmark before you compare flags.

## Caveats

One model, one card, this load. I did not keep `/metrics` on the way up to 12 running for the flag-off run, so I cannot say how the four large prompts entered the scheduler, only that 12 were running once they were in.

I asked for p50 and p99 only, so I do not have the single highest ITL. Flag off may have had a few slower waits that never reached p99. If the usual claim about this flag is about that worst wait rather than p99, both can be true. That is the next thing I would measure. I did not measure it here.
