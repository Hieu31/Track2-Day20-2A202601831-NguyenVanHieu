# 02 - Serve: load test + saturation reading

Host `Linux-x86_64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=2` ·
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 4 | 0.13 | 27000 | 30000 | 30000 | 3.7 | 0.0% |
| 50 | 8 | 0.19 | 43000 | 43000 | 43000 | 6.0 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.41x** (28% of linear) |
| P95 latency | **1.43x** |
| Effective concurrency at 50 users | 6.0 vs `--parallel 4` slots (occupancy/slot ratio 1.51) |

**Saturated.** Throughput delivered only 1.41x for 5x the offered load, and effective concurrency (6.0) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.41x while P95 moved 1.43x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

> **Small sample.** Only 4 requests completed in the
> shorter run, so these percentiles are indicative rather than solid. Note also that
> locust averages only *completed* requests: when the run ends with requests still
> queued, effective concurrency is an **under**-estimate. Trust the throughput-scaling
> row over the concurrency row here, and run longer (`-t 3m`) if you want firmer numbers.

## Your reading

The server is clearly saturated when moving from 10 to 50 users. The key evidence is that a **5x increase in offered load only yielded a 1.41x increase in throughput** (from 0.13 to 0.19 RPS). The number that convinced me is the **effective concurrency of 6.0**, which exceeds the 4 available decode slots (`--parallel 4`). This means the extra load is piling up in the queue, pushing the P95 latency from 30s to 43s (a 1.43x increase).

If my SLO for P95 latency is 35 seconds, the 50-user load fails this target. To raise goodput at this SLO, the first knob I would change is the **thread count (`LAB_N_THREADS`)** to hit the optimal decode speed, or **use a faster quantization (like Q2)**. Because this is a CPU setup bound by memory bandwidth, blindly increasing `--parallel` (more slots) would likely degrade the decode speed of all active slots, causing even more requests to violate the SLO. Optimizing the single-stream speed first ensures slots free up faster, reducing queue time and raising goodput.
