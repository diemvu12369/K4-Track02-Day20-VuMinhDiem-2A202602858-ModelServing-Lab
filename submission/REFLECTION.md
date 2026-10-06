# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Vũ Minh Điềm
**MSSV:** 2A202602858
**Cohort:** 4 - K4 (Track 02)
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 11 Home Single Language (AMD64)
- **CPU:** 13th Gen Intel Core i5-13420H (hybrid: 4 P-core + 4 E-core)
- **Cores:** 8 physical / 12 logical
- **CPU extensions:** AVX2 (Raptor Lake, không có AVX-512)
- **RAM:** 15.7 GB — một thanh DDR5-5200 16 GB, **single-channel** (~41.6 GB/s lý thuyết)
- **Accelerator:** NVIDIA RTX 3050 6 GB Laptop GPU có trong máy, nhưng runtime chạy **CPU only** (`ngl=0`)
- **llama.cpp asset đã tải:** `llama-b10488-bin-win-vulkan-x64.zip`
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi (không dùng Colab/Kaggle).

**Setup story** (≤ 80 chữ):

`.\lab.ps1` không parse được trên Windows PowerShell 5.1 (file UTF-8 không BOM chứa ký tự
"—"), và Python in ra console cp1252 bị `UnicodeEncodeError`. Workaround: đặt
`PYTHONUTF8=1` và gọi thẳng các script `.venv\Scripts\python labs\...` tương ứng với từng
target. Runtime Vulkan prebuilt không liệt kê được GPU nên lab tự để `ngl=0`; toàn bộ base
track chạy trên CPU.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 5916 | 631 / 837 | 80.9 / 127.0 | 5729 / 6787 / 6787 | 12.4 |
| UD-Q2_K_XL | 2.24 | 9223 | 780 / 868 | 74.7 / 85.6 | 5209 / 6164 / 6164 | 13.4 |

(`threads=8`, `ngl=0`, `ctx=2048`, `max_tokens=64`, 10 request mỗi quant. Đây là lần chạy
thứ 3, trùng với screenshot `02b-bench-table.png`.)

**Quan sát** (≤ 60 chữ):

Lần này 2-bit decode nhanh hơn 1.08×, nhưng qua 3 lần chạy tỉ lệ dao động 0.90×–1.08×,
tức nằm trong nhiễu; TTFT thì cả 3 lần 2-bit đều chậm hơn (lần này +24%). Hỏi cùng 3 câu
trên cả hai: toán đều đúng, nhưng 2-bit dịch sai "continuous batching" thành "hàng đợi
liên tục". Không đáng, trừ khi thiếu RAM.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.23 | 34000 | 35000 | 35000 | 5.8 | 0.0% |
| 50 | 0.17 | 40000 | 57000 | 57000 | 6.2 | 0.0% |

(Server: `--parallel 4`, `-t 4`, `ctx=2048`, `ngl=0`. Chỉ 8 và 10 request hoàn thành
trong 60 s, nên percentile chỉ mang tính tham khảo.)

- **Offered load tăng 5×, throughput thực tăng:** 0.76×
- **P95 tăng:** 1.63×
- **Effective concurrency ở 50 users:** 6.2 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.76 / 4 slots (`requests_deferred` lên tới 46)

**Saturation reading** (≤ 80 chữ):

Bão hoà ngay từ 10 users: effective concurrency 5.8 > 4 slot, P50 34 s so với ~5.3–5.7 s của
một request đơn lẻ. Lên 50 users throughput không tăng (0.76×) nhưng P95 tăng 1.63×. Phần
thêm là queue time: 4 slot luôn bận, 46 request deferred, tốc độ decode không đổi. Với
SLO P95 ≤ 10 s thì goodput = 0. Knob đổi trước: GPU offload, vì trần là bandwidth RAM
single-channel, thêm `--parallel` chỉ chia nhỏ cùng bandwidth đó.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | không dùng, chạy local | stub |
| N17 Data pipeline | corpus mẫu có sẵn trong `pipeline.py` | stub |
| N18 Lakehouse | không dùng | stub |
| N19 Vector + features | không có embedding server, retrieve bằng keyword overlap | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.1 ms
- llm: 5781.0 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ):

Bottleneck là LLM, đúng kỳ vọng vì retrieve chỉ là keyword overlap. Server tốn ~1.4–1.8 s
prefill + ~1.5–3 s decode mỗi query; còn ~2 s chênh phía client tôi chưa giải thích được.
Muốn giảm 2×: prompt caching và cắt context để giảm prefill, sau đó GPU offload để tăng
tốc cả prefill lẫn decode.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** hạ số thread decode từ `-t 8` (mặc định = số physical core) xuống `-t 4` (= số P-core)

```
before:  11.1 tok/s  (tg128, -t 8)
after:   13.1 tok/s  (tg128, -t 4)
speedup: 1.18×
```

Kiểm chứng qua HTTP: `make bench` ở `-t 8` cho decode 12.4–12.9 tok/s qua 3 lần chạy; smoke test trên
`make serve` với `LAB_N_THREADS=4` đo được 13.6 tok/s.

**Tại sao nó work:**

Deck kỳ vọng throughput tăng đến số *physical core* rồi mới chững. Máy tôi peak ở **một
nửa** con số đó, vì i5-13420H là CPU hybrid: "8 physical core" gồm 4 P-core nhanh và 4
E-core chậm hơn (xung thấp hơn, IPC thấp hơn). llama.cpp chia đều mỗi phép matmul cho mọi
thread rồi chờ ở barrier sau mỗi op. Với `-t 8`, một số thread chạy trên E-core, và ở mỗi
barrier cả nhóm phải đợi thread chậm nhất, nên càng thêm thread thì mỗi bước decode càng
lâu. `-t 12` còn tệ hơn (8.6 tok/s) vì thêm luồng Hyper-Threading dùng chung execution unit
và cache của P-core; `-t 24` sụp còn 1.1 tok/s vì oversubscribe làm OS phải preempt thread
trong khi các thread khác spin ở barrier.

Lý do thứ hai là memory bandwidth. Decode phải đọc gần như toàn bộ weight cho mỗi token, và
máy tôi chỉ có **một thanh DDR5-5200 chạy single-channel**, tức ~41.6 GB/s lý thuyết.
Ở 13 tok/s với ~2–3 GB weight mỗi token, mức dùng đã khoảng 26–39 GB/s, sát trần. Khi
bandwidth đã gần bão hoà với 4 P-core thì thread thứ 5 trở đi không có thêm byte/s để dùng,
chỉ thêm chi phí đồng bộ. Điều này cũng giải thích kết quả 2-bit ở §2: không còn bandwidth
dư, nhưng 2-bit lại tốn thêm compute để dequantize. Điểm tôi chưa giải thích chắc chắn là
`-t 1` chỉ đạt 1.1 tok/s, thấp hơn 4 thread tới 12× (siêu tuyến tính); giả thuyết là một
core đơn lẻ bị compute-bound khi dequantize và có thể bị Windows xếp lên E-core, cần pin
CPU affinity để kiểm tra.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** không làm bonus.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Laptop có RTX 3050 nhưng cổ chai thực sự lại là RAM chỉ cắm một thanh (single-channel), và
mặc định "số thread = số physical core" sai trên CPU hybrid: dùng ít thread hơn lại nhanh hơn.

---

## 8. Self-check trước khi push

- [x] `hardware.json` committed
- [x] `models/active.json` committed
- [x] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [x] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [x] `benchmarks/02-server-results.md` committed (`make load-report`)
- [x] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [x] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [x] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [x] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Claude Code (Claude Opus 5.5): chạy các lệnh của lab trên máy tôi (probe, setup, bench,
tune, serve, smoke, load test, metrics, pipeline), xử lý lỗi encoding/PowerShell trên
Windows, và soạn nháp các phần nhận xét trong `benchmarks/*.md` và REFLECTION dựa trên số
liệu đo được. Tôi đã đọc lại và chịu trách nhiệm về các lập luận này. Không có số liệu nào
được sửa tay.
