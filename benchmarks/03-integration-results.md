# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 5519.8 | 5520.4 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 4562.5 | 4562.6 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 4587.4 | 4587.5 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **4889.9** · total **4890.2**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

- **N16 Cloud/IaC: stub/not connected.** This run is local on the laptop; no
  cloud cluster or IaC resource participates in the request path.
- **N17 Data pipeline: stub.** Documents are the hard-coded in-memory `TOY_DOCS`;
  there is no ingestion or transformation job.
- **N18 Lakehouse: stub.** No lakehouse table or persistent document store is
  queried by this pipeline.
- **N19 Vector + features: stub.** `retrieve()` uses keyword overlap over
  `TOY_DOCS`; no embedding model, feature store, or vector index is active.
- **N20 Serving: real.** All three generated answers came from the local
  OpenAI-compatible `llama-server` endpoint on port 8080.

The LLM stage dominates at 4889.9 of 4890.2 ms (approximately 100%), which is
expected because embedding is disabled and retrieval scans only a tiny in-memory
list. By Amdahl's law, optimizing the 0.1 ms retrieval stage cannot materially
change total latency. To target a 2x reduction I would attack the LLM stage:
first reduce retrieved context/top-k to lower prefill and cap output tokens to
lower decode, then test a faster model/backend or accelerator while checking
answer quality. Q2 is not the automatic answer here because the clean baseline
showed it was slightly slower than Q4 on this machine.

