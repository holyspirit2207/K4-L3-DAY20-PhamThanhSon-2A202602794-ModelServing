# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **6 physical · 12 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 17.4 | 96% |
| 3 | 17.7 | 97% |
| 6 | 18.2 | 100% |
| 12 | 17.6 | 97% |
| 24 | 17.9 | 98% |

**Best**: `-t 6` at 18.2 tok/s
**Slowest tested**: `-t 1` at 17.4 tok/s (1.05x spread)
**Against the physical-core default** (`-t 6`, 18.2 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=6 make bench
```

## Your explanation

There is no sharp knee in this sweep: one thread already reaches 96% of the
best throughput, three threads reach 97%, and the nominal peak is at the six
physical cores (18.22 tok/s). Moving from all 12 logical threads back to six
physical threads improves throughput from 17.63 to 18.22 tok/s, only **1.03x**.
The 24-thread oversubscribed point recovers to 17.92 tok/s, so the small changes
above six threads are not a clean monotonic collapse and should be treated as
run-to-run variation rather than overclaimed.

The flat curve is consistent with this run's `ngl=99`: almost all model layers
are offloaded to the Vulkan integrated GPU, so CPU worker count is not the main
decode bottleneck. Extra logical threads cannot add GPU execution units or more
shared-memory bandwidth; they share physical-core execution resources and add
host scheduling/dequantization overhead instead. The practical saturation point
is therefore about 3--6 threads, with six retained as the best measured and
physical-core-aligned setting. This result differs from a CPU-only sweep, where
a clearer bandwidth knee and a larger oversubscription penalty would be expected.

