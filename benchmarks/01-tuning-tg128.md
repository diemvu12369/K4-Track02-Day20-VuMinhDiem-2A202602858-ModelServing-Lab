# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **8 physical · 12 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 1.1 | 9% |
| 4 | 13.1 | 100% |
| 8 | 11.1 | 84% |
| 12 | 8.6 | 65% |
| 24 | 1.1 | 8% |

**Best**: `-t 4` at 13.1 tok/s
**Slowest tested**: `-t 24` at 1.1 tok/s (12.40x spread)
**Against the physical-core default** (`-t 8`, 11.1 tok/s): 1.18x

Use this in your run:

```bash
LAB_N_THREADS=4 make bench
```

## Your explanation

**Knee ở `-t 4`, không phải ở 8 physical core.** i5-13420H là CPU hybrid: **4 P-core
(có Hyper-Threading, 8 luồng) + 4 E-core**. Vì vậy "8 physical / 12 logical" thực ra là
4 core nhanh + 4 core chậm, và `-t 4` đúng bằng số P-core (13.1 tok/s).

- **4 → 8 thread (−16%)**: llama.cpp chia đều mỗi phép matmul cho mọi thread rồi đồng bộ
  bằng barrier sau từng op. Khi có thread rơi vào E-core (xung thấp hơn, IPC thấp hơn),
  mọi thread khác phải chờ thread chậm nhất ở mỗi barrier. Đồng thời decode đã gần trần
  bandwidth của RAM single-channel DDR5-5200 (~41.6 GB/s lý thuyết; 13 tok/s × ~2–3 GB
  weight đọc mỗi token ≈ 26–39 GB/s), nên thêm thread cũng không có thêm byte/s để dùng.
- **12 thread (−35%)**: thêm cả luồng HT anh em trên P-core. Chúng dùng chung execution
  unit và L1/L2, nên chỉ tăng tranh chấp.
- **24 thread (1.1 tok/s)**: oversubscribe gấp 2 số logical core. OS phải preempt thread
  trong khi các thread khác spin ở barrier, nên throughput sụp.
- **1 thread (1.1 tok/s)** thấp bất thường: thấp hơn 4 thread tới 12×, tức siêu tuyến
  tính. Một thread không tự kéo đủ bandwidth, và dequant + dot trên một core là
  compute-bound; ngoài ra Windows có thể đã đặt thread đó lên E-core. Tôi chưa kiểm chứng
  được nguyên nhân nào trội hơn (cần pin affinity để tách), nên đây vẫn là giả thuyết.

Kết luận: mặc định "số physical core = 8" của lab sai trên CPU hybrid. Đặt `-t 4` cho
decode nhanh hơn **1.18×** so với mặc định. Tôi dùng `LAB_N_THREADS=4` cho `make serve`;
smoke test đo được 13.6 tok/s decode qua HTTP, so với 12.7 tok/s của `make bench` ở `-t 8`.
