---
layout: default
title: "A small harness for my vLLM runs"
date: 2026-10-30
tags: [inference, vllm, gpu]
---

# {{ page.title }}

Every post on here started the same way. Rent the L4, start vLLM, run the bench, scrape `/metrics` before and after, then try to get those files back onto my Mac before the pod eats my evening. The last bit was the worst. scp did not work on that image, so one night I ended up base64-ing a tarball in the terminal and pasting it back, at half nine, while paying by the hour. That was the night I decided I was not doing this by hand again.

So I wrote [a small harness](https://github.com/boncheff/vllm-harness). It runs the bench, saves the metrics from that run, and shows them on a page.

## What it does

There are three scripts. `capture.sh` runs on the pod. It checks vLLM is answering, runs one bench I throw away, then the real one. While the real one runs it scrapes `/metrics` every second, and it takes a full scrape just before and just after.

`pull_run.py` runs on my Mac. It logs into the pod over SSH, sends `capture.sh`, waits, and brings the tarball back. Then it works out the run: counters are after minus before, so earlier runs are not in the number, and gauges like `num_requests_running` and `kv_cache_usage_perc` keep the highest value from the polls. They are usually back at zero by the time the bench finishes, which is why the polling is there at all. If I give it the hourly price it works out dollars per million output tokens too.

`room.py` serves a page on localhost. One block per run: TTFT, ITL, tokens per second, cost, prefix cache hit rate, preemptions, then a couple of charts. It only reads the saved files. I do not type any numbers in.

## The first run

The first real run was the 1.5B on an L4 at $0.60 an hour. 128 requests, concurrency 8. It also caught me out. Prefix caching is on by default and the throwaway bench used the same prompts, so the real bench hit the cache 95% of the time. TTFT dropped from 142 ms to 45 ms. ITL stayed around 14 ms.

It is not clever. It just means the next run takes one command instead of an evening.
