# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=6` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 5725 | 434 / 446 | 59.2 / 61.2 | 4171 / 4287 / 4287 | 16.9 |
| UD-Q2_K_XL | 2.24 | 4630 | 477 / 492 | 60.7 / 60.9 | 4290 / 4328 / 4328 | 16.5 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.02x SLOWER** than `UD-Q4_K_XL` here, despite being 0.73 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

This clean baseline was recorded after closing Roblox and confirming that no
`llama-server` process was left running; the scripted warm-up was discarded.
On this Ryzen 5 5600U with Vulkan offload, `UD-Q2_K_XL` reduced model size from
2.97 GB to 2.24 GB: 0.73 GB, or **24.6%**, smaller. It also loaded 1.09 s faster
(4.63 vs 5.72 s). However, its median TPOT was 60.7 ms versus 59.2 ms for Q4,
so decode throughput fell from 16.9 to 16.5 tok/s: Q2 was **1.02x slower**
(about 2.5%). Median TTFT was also 10.0% higher (477 vs 434 ms).

The near-tie means that this workload/backend is not receiving a useful decode
speedup from moving fewer weight bytes. On this integrated-GPU Vulkan path, the
extra work needed to dequantize the 2-bit format cancels or slightly exceeds its
memory-traffic saving; the small 2.5% gap should also be treated as near the
run-to-run noise floor, not as evidence that Q4 is universally faster.

For a controlled quality check, both servers received the same deterministic
Little's Law prompt (`temperature=0`): `L = 8 req/s x 2.5 s`. Both returned the
correct answer, `20`. Both also misunderstood a harder Vietnamese prompt about
the TTFT/TPOT acronyms, so this small test found no Q2-specific regression but
is not a broad quality guarantee. Because RAM is sufficient and Q2 brings no
latency benefit here, **Q4 is the better default on this machine**; Q2 is worth
using only when saving 0.73 GB of disk/RAM matters more than preserving the
additional precision.

