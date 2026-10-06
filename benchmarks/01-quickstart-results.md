# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=8` `ngl=0` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 5916 | 631 / 837 | 80.9 / 127.0 | 5729 / 6787 / 6787 | 12.4 |
| UD-Q2_K_XL | 2.24 | 9223 | 780 / 868 | 74.7 / 85.6 | 5209 / 6164 / 6164 | 13.4 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.08x faster** than `UD-Q4_K_XL` here, for 0.73 GB less on disk.

## Your observation

**Trên máy này 2-bit không đáng dùng.** Ở lần chạy trên (cũng là lần trong screenshot
`02b-bench-table.png`), `UD-Q2_K_XL` nhỏ hơn 0.73 GB (2.24 vs 2.97 GB, -25%) và decode
nhanh hơn **1.08×** (13.4 vs 12.4 tok/s, TPOT P50 74.7 vs 80.9 ms). Đổi lại, TTFT P50
**chậm hơn 24%** (780 vs 631 ms), và load chậm hơn (9.2 s vs 5.9 s).

Nhưng 1.08× không phải là speedup ổn định. Tôi đã chạy `make bench` 3 lần trên cùng máy,
cùng cài đặt (`-t 8`, `ngl=0`):

| Lần | Q4 decode (tok/s) | Q2 decode (tok/s) | Q2 / Q4 |
|:--|--:|--:|--:|
| 1 | 12.9 | 11.6 | 0.90× |
| 2 | 12.7 | 12.6 | 0.99× |
| 3 (bảng trên) | 12.4 | 13.4 | 1.08× |

Tỉ lệ dao động từ 0.90× đến 1.08×, nghĩa là chênh lệch nằm trong nhiễu đo của laptop (tác
vụ nền, nhiệt độ, boost clock), không phải hiệu ứng thật. Riêng TTFT thì cả 3 lần bản 2-bit
đều chậm hơn.

Vì sao không có speedup như lý thuyết: máy chạy CPU-only (`ngl=0`) với **một thanh DDR5-5200
single-channel** (~41.6 GB/s lý thuyết). Nếu decode thuần bandwidth-bound thì bớt 25% byte
phải cho khoảng 1.3×. Nhưng K-quant 2-bit phải unpack/dequantize phức tạp hơn cho mỗi weight;
với vài core làm việc, chi phí ALU tăng thêm ăn gần hết phần bandwidth tiết kiệm được. Prefill
vốn compute-bound nên 2-bit còn thiệt hơn (TTFT chậm hơn ở cả 3 lần).

Chất lượng: tôi hỏi cùng 3 câu (temperature 0) trên cả hai server. Câu giải thích memory
bandwidth và bài cộng giờ (14:35 + 2h50 = 17:25) thì cả hai đều đúng. Với câu dịch
"Continuous batching lets the server add new requests to a running batch", bản 2-bit dịch
sai thuật ngữ thành *"Hàng đợi liên tục"* ở phương án đầu tiên, còn bản 4-bit dịch đúng
"Batching liên tục ... lô đang chạy". Tốc độ decode lợi tối đa ~8% và không ổn định, TTFT
chậm hơn, chất lượng giảm: 2-bit chỉ đáng dùng khi RAM là ràng buộc cứng (máy 4–8 GB).
