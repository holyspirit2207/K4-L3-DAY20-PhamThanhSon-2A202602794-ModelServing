# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Phạm Thanh Sơn
**MSSV:** 2A202602794
**Cohort:** Cohort 4
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 11
- **CPU:** AMD Ryzen 5 5600U with Radeon Graphics
- **Cores:** 6 physical / 12 logical
- **CPU extensions:** AVX2
- **RAM:** 13.9 GB
- **Accelerator:** AMD Radeon integrated GPU via Vulkan
- **llama.cpp asset đã tải:** `llama-b10488-bin-win-vulkan-x64.zip`
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL + UD-Q2_K_XL

**Chạy ở đâu:** laptop của tôi
_(Nếu dùng cloud fallback: nói rõ vì sao — RAM < 8 GB, setup fail, v.v. Không mất điểm.)_

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

OpenAI Codex assisted with setup and terminal operations on this laptop. Windows
required a process-scoped ExecutionPolicy bypass, and PowerShell 5.1 misread
UTF-8 characters in `lab.ps1`, so three decorative dashes were changed to ASCII
and `PYTHONIOENCODING=utf-8` was used. Because `serve.py` lost quoting for the
repository path containing spaces, `llama-server.exe` was launched directly with
identical flags. A leftover Roblox process was stopped and the baseline was rerun
to remove Vulkan interference.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 5725 | 434 / 446 | 59.2 / 61.2 | 4171 / 4287 / 4287 | 16.9 |
| UD-Q2_K_XL | 2.24 | 4630 | 477 / 492 | 60.7 / 60.9 | 4290 / 4328 / 4328 | 16.5 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

Q2 was 24.6% smaller and loaded 1.09 s faster, but decode was 1.02× slower
(16.5 versus 16.9 tok/s) and median TTFT was 10% higher. Both quantizations
answered the Little's Law check correctly and both misunderstood the harder
Vietnamese TTFT/TPOT prompt, so this test found no Q2-specific regression. With
sufficient RAM, Q4 is the better default.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.38 | 19000 | 31000 | 31000 | 7.6 | 0.0% |
| 50 | 0.40 | 29000 | 55000 | 55000 | 12.1 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.05×
- **P95 tăng:** 1.77×
- **Effective concurrency ở 50 users:** 12.1 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.89 / 4 slots

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

Server is saturated by 10 users: its effective concurrency is already 7.6 versus
four slots, so the exact knee is ≤10. Increasing offered load 5× raises throughput
only 1.05× (0.38→0.40 RPS), while P95 rises 1.77× (31→55 s); 3.89/4 busy slots and
46 deferred prove the increase is queue time. With a P95≤35 s SLO, demonstrated
goodput is 0.38 RPS. I would first test `--parallel 8`, because full slots and the
deferred queue—not CPU threads—are the immediate constraint.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Not connected to this local run | stub |
| N17 Data pipeline | Hard-coded in-memory `TOY_DOCS` | stub |
| N18 Lakehouse | No persistent/lakehouse lookup | stub |
| N19 Vector + features | Keyword overlap; no embeddings/vector index | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.1 ms
- llm: 4889.9 ms
- **stage chiếm nhiều nhất:** llm (approximately 100% of total)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

LLM is 4889.9/4890.2 ms (~100%), as expected because embedding is disabled and
retrieval scans tiny in-memory `TOY_DOCS`. Amdahl's law makes optimizing 0.1 ms
retrieval irrelevant. For a 2× target I would reduce retrieved context/top-k and
output tokens first, then test a faster backend while checking quality; Q2 was
not faster on this machine.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Use the 6 physical cores (`-t 6`) instead of the naive all-logical-core setting (`-t 12`). The lab's physical-core default was already the measured optimum.

```
before:  17.63 tok/s at -t 12
after:   18.22 tok/s at -t 6
speedup: 1.03×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

_Giải thích như đang nói với bạn ngồi cạnh. Bám vào **cơ chế**, không phải "vibes":
memory bandwidth? vector width? cache residency? scheduling? queueing? Nếu kết quả
**khác** với kỳ vọng từ deck — nói rõ, và giải thích vì sao. Grader thưởng điểm cho
lập luận đúng về một kết quả bất ngờ, hơn là một con số đẹp không được giải thích._

The sweep did not show a dramatic textbook knee. One thread already delivered
17.41 tok/s (96% of the best), three delivered 17.74 tok/s, and the nominal peak
was 18.22 tok/s at the six physical cores. Using all 12 logical cores reduced
throughput to 17.63 tok/s; even 24 oversubscribed threads reached 17.92 tok/s.
The total 1--24 thread spread was only 1.05×, so I treat the non-monotonic changes
above six threads as near the run-to-run noise floor rather than claiming a large
oversubscription collapse.

The mechanism is specific to this run: `ngl=99` offloaded almost all layers to
the Vulkan integrated GPU. CPU thread count therefore was not the primary decode
bottleneck, and extra logical threads could not create more GPU execution units
or shared-memory bandwidth. Above the six physical cores they instead shared
core/cache resources and added host scheduling or dequantization overhead. I kept
`-t 6` because it was the measured best and matches the hardware topology, but
the honest result is a modest 1.03× improvement over `-t 12`, not a large CPU-only
scaling win. The full evidence is in `benchmarks/01-tuning-tg128.md`.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** _<B1 build-compare / B2 sweep nào / B4 challenge nào / B5 lựa chọn nào>_

**Numbers:**

```
before:  <số>
after:   <số>
speedup: <X.Y>×
```

**Điều này nói lên gì mà deck chưa nói:**

_(để trống nếu bạn không làm phần này)_

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

_(1–2 câu. Không bắt buộc, nhưng grader đọc hết.)_

_(để trống nếu bạn không làm phần này)_

---

## 8. Self-check trước khi push

- [ ] `hardware.json` committed
- [ ] `models/active.json` committed
- [ ] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [ ] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [ ] `benchmarks/02-server-results.md` committed (`make load-report`)
- [ ] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [ ] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [ ] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [ ] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [ ] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Tôi sử dụng OpenAI Codex để hỗ trợ xử lý một số lỗi Windows PowerShell,
encoding UTF-8 và quoting đường dẫn; hỗ trợ chạy, kiểm tra các script của lab;
và chỉnh sửa cách trình bày báo cáo. Toàn bộ số liệu được sinh từ các script
trong repo trên laptop cá nhân của tôi. Tôi đã xem lại số liệu và chịu trách
nhiệm về các kết luận trong báo cáo.
