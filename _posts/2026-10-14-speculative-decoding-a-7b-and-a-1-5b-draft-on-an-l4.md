---
layout: default
title: "Speculative decoding: a 7B and a 1.5B draft on an L4"
date: 2026-10-14
tags: [inference, vllm, gpu]
---

# {{ page.title }}

If you have been reading these, you already know the setup. I keep going back to the same L4. I like it. It is the cheap one, so I can run the experiment a few times without taking out a small loan. While I was looking into that, I came across speculative decoding: a small draft model in front of the big one, supposedly faster, same answer. I wanted to see what that actually does, so I served the 7B alone and then with a 1.5B in front of it.

Quick version of the idea. Both models sit in memory. The small one guesses the next few tokens. The 7B checks that guess in one **pass**, keeps the tokens that match, and stops at the first one that does not. The text is still the 7B's. A pass is one walk through a model's weights. If the guess is right, one walk of the 7B covers several tokens. If it is wrong, you still ran the small model and only kept one. The card does not change, so the hourly price does not change. Tokens per second can, and so can the wait.

## Setup

One L4, vLLM, `Qwen/Qwen2.5-7B-Instruct`. Then the same line plus `Qwen/Qwen2.5-1.5B-Instruct` guessing 5 tokens. `--gpu-memory-utilization 0.92`, `--max-model-len 10240`, prefix caching off. I also set `--enforce-eager`. vLLM can record the GPU work and replay that recording, which is faster. This flag turns the recording off and runs each step as it comes. I leave it on for every one of these runs, so a recording warmup is not mixed into the number.

`vllm bench serve`, 32 requests, 128 tokens out, concurrency 1 and then 16. First benchmark after each boot discarded. The benchmark forces 128 tokens, so the timing is a decode run.

The questions were how many e's are in beekeeper, what is 47 times 86, and a sheep-related riddle: a farmer has 17 sheep, all but 9 die, how many are left. I also sent those three once with temperature at 0. Temperature 0 means the model always picks the single most likely next token. No randomness. Same question, same answer, so if the two servers had disagreed, that would have been the draft and not luck.

## What I saw

Time to first token is the wait before anything comes back. ITL is the gap after that. p50 is typical. p99 is the slow tail.

| | concurrency | duration | tokens/s | TTFT p50 | ITL p50 | ITL p99 | guesses kept | tokens per pass |
|---|---|---|---|---|---|---|---|---|
| 7B alone | 1 | 233s | 17.6 | 75 ms | 57 ms | 58 ms | | 1 |
| 7B alone | 16 | 15.8s | 259 | 330 ms | 60 ms | 62 ms | | 1 |
| 7B + draft | 1 | 166s | 24.7 | 260 ms | 126 ms | 128 ms | 46% | 3.3 |
| 7B + draft | 16 | 28.3s | 145 | 3.6 s | 147 ms | 4.2 s | 40% | 3.0 |

On the 7B alone, one request did 17.6 tokens per second and sixteen did 259. That wave took 15.8 seconds. ITL stayed around 60 ms. Time to first token went from 75 ms to 330 ms, because sixteen prompts all needed prefill and the GPU does one pass at a time. The KV cache was almost empty.

With the draft, one request did 24.7 tokens per second and the wave took 166 seconds instead of 233. It kept 46 percent of the guesses, about 3.3 tokens per pass of the 7B instead of 1, so the rate went up. The wait got worse. Time to first token went from 75 ms to 260 ms, and the gap went from 58 ms to 128 ms, because the draft is also doing prefill and decode.

At sixteen the draft did 145 tokens per second, against 259. Same 4096 tokens, 28.3 seconds instead of 15.8. Time to first token was 3.6 seconds, against 330 ms. The typical gap was 147 ms and the tail was 4.2 seconds. It still kept about 40 percent. The GPU was busy, so those draft passes were not spare.

The 46% and the 40% are all five guessed tokens averaged together. The benchmark also splits them by spot. Position 0 is the first of the five, not an extra token. With one request those five spots were kept 71%, 53%, 42%, 34%, and 27% of the time. With sixteen it was 68%, 49%, 35%, 27%, and 21%. The first guess is usually right. Each one after that is worse. I left the guess at 5 tokens.

**One request:**

![Per-position acceptance with one request]({{ '/assets/2026-10-14-spec-acceptance-conc1.png' | relative_url }})

**Sixteen requests:**

![Per-position acceptance with sixteen requests]({{ '/assets/2026-10-14-spec-acceptance-conc16.png' | relative_url }})

Both servers said 3, 4042, and 9. 4042 and 9 are right. Beekeeper has five e's, so both said 3. The draft did not change the words.

## What I would do

At one request I would turn the draft on if I cared about tokens per second and I could live with the slower gap. At sixteen I would leave it off. Same card, and the busy run spent the time worse.

One more thing worth mentioning. I only measured 1 and 16, so I do not know where it flips. I did not try a shorter guess, and three questions is a check, not an eval.
