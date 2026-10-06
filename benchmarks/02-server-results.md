# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=4` ·
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 8 | 0.23 | 34000 | 35000 | 35000 | 5.8 | 0.0% |
| 50 | 10 | 0.17 | 40000 | 57000 | 57000 | 6.2 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.76x** (15% of linear) |
| P95 latency | **1.63x** |
| Effective concurrency at 50 users | 6.2 vs `--parallel 4` slots (occupancy/slot ratio 1.54) |

**Saturated.** Throughput delivered only 0.76x for 5x the offered load, and effective concurrency (6.2) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.76x while P95 moved 1.63x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

> **Small sample.** Only 8 requests completed in the
> shorter run, so these percentiles are indicative rather than solid. Note also that
> locust averages only *completed* requests: when the run ends with requests still
> queued, effective concurrency is an **under**-estimate. Trust the throughput-scaling
> row over the concurrency row here, and run longer (`-t 3m`) if you want firmer numbers.

## Your reading

**Server đã bão hoà ngay từ 10 users.** Bằng chứng thuyết phục nhất: ở 10 users,
effective concurrency đã là **5.8 > 4 slot**, và P50 = 34 s trong khi một request đơn lẻ
(64 token) chỉ mất ~5.3 s E2E trong `make bench`, tức phần lớn thời gian là chờ. Lên 50
users (offered load gấp 5) throughput **không tăng** (0.23 → 0.17 RPS, 0.76×; mức giảm nằm
trong nhiễu vì chỉ có 8–10 request hoàn thành), trong khi P95 phồng **1.63×** (35 s → 57 s).
`make metrics` xác nhận: 4/4 slot bận, 46 request bị deferred.

Phần latency tăng thêm là **queue time, không phải compute time**. Số slot tính toán không
đổi (4), tốc độ decode của CPU không đổi, nhưng số request đứng chờ slot tăng lên tới 46.
Little's Law (6.2 > 4) và `requests_deferred > 0` đều chỉ ra điều đó.

**goodput@SLO**: tôi chọn SLO là P95 E2E ≤ 10 s cho câu trả lời 64 token. Một user đơn lẻ
đạt SLO (~5.3 s), nhưng ở cả 10 lẫn 50 users **không request nào** đạt (min 16.5 s và
20.6 s), nên goodput = 0 req/s dù throughput > 0. Đó chính là khoảng cách giữa peak
throughput và goodput.

**Knob tôi đổi trước tiên là GPU offload (`-ngl`), không phải `--parallel`.** Trên máy này
trần là decode trên CPU, bị chặn bởi RAM single-channel ~41.6 GB/s; thêm slot chỉ chia
cùng một lượng bandwidth cho nhiều stream hơn, làm TPOT mỗi stream tăng. RTX 3050 6 GB có
~190 GB/s VRAM và model 2.97 GB vừa VRAM, nên offload tăng trực tiếp tok/s của mỗi bước
decode. Bản runtime prebuilt (Vulkan) không liệt kê được device nên lab tự chạy `ngl=0`;
cần sửa runtime/driver hoặc tự build CUDA. Song song, tôi sẽ thêm giới hạn hàng đợi
(admission control) để những request được nhận vẫn nằm trong SLO, thay vì để tất cả cùng
trễ.
