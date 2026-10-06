# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=6` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 20 | 0.38 | 19000 | 31000 | 31000 | 7.6 | 0.0% |
| 50 | 22 | 0.40 | 29000 | 55000 | 55000 | 12.1 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.05x** (21% of linear) |
| P95 latency | **1.77x** |
| Effective concurrency at 50 users | 12.1 vs `--parallel 4` slots (occupancy/slot ratio 3.02) |

**Saturated.** Throughput delivered only 1.05x for 5x the offered load, and effective concurrency (12.1) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.05x while P95 moved 1.77x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

The server is already saturated by the 10-user point: effective concurrency is
7.6 there, above the four available slots, so the exact knee is at or below 10
users and was not resolved by this two-point test. Raising offered users 5x then
delivered only **1.05x throughput** (0.38 to 0.40 req/s), while P95 rose **1.77x**
(31 to 55 s). At 50 users the combination of effective concurrency **12.1 > 4
slots**, live occupancy **3.89/4**, and **46 deferred** requests shows that compute
capacity stayed nearly flat and the added tail latency is predominantly queue
time.

I define the operating SLO as **P95 <= 35 s**. The 10-user point meets it and
demonstrates 0.38 req/s of SLO-compliant throughput; the 50-user point delivers
only 0.02 req/s more but violates the SLO. My first experiment to raise
goodput@SLO would be increasing `--parallel` from 4 to 8 while increasing total
context enough to preserve context per slot. Full slot occupancy and a deferred
queue directly motivate that knob; CPU threads are a weaker candidate because
the thread sweep was almost flat. I would keep the change only if RPS rises
without TPOT/P95 worsening, since the integrated GPU may already be the shared
bandwidth limit.

