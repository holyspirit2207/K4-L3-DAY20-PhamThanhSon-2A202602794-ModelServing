# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 15 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.89 of 4 slots (97%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 2316 |

Highest sampled value was **3.89 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

The highest sampled `n_busy_slots_per_decode` was **3.89 of 4 slots (97%)**,
while `requests_processing` reached 4. This is direct evidence that continuous
batching was active: llama.cpp was advancing almost four live requests together
in its decode steps rather than serving one request at a time. During the same
overlapping 60-second window, `requests_deferred` peaked at 46 and
`tokens_predicted_total` rose from 1192 in the first sample to 2316 in the last,
so the samples captured real work under sustained overload rather than an idle
server.

The 50-user load report gives an effective concurrency of **12.1** from Little's
Law (`0.40 req/s x 30.38 s average latency`). It should not equal 3.89: effective
concurrency counts every request in the system, including requests waiting for a
slot, whereas `n_busy_slots_per_decode` measures active decode-slot occupancy.
The peak deferred queue of 46 explains the difference. For the batching claim I
trust the server gauge, because it is emitted by the scheduler during decode;
Locust concurrency and Little's Law remain useful for quantifying the queue and
the end-user latency caused by saturation.

