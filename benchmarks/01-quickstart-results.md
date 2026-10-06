# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=8` `ngl=0` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 4727 | 600 / 681 | 78.7 / 138.9 | 5310 / 9293 / 9293 | 12.7 |
| UD-Q2_K_XL | 2.24 | 5135 | 823 / 1257 | 79.5 / 161.4 | 5476 / 11423 / 11423 | 12.6 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` and `UD-Q4_K_XL` decode within 2% of each other here, for 0.73 GB difference on disk.

## Your observation

**Trên máy này 2-bit không đáng dùng.** `UD-Q2_K_XL` nhỏ hơn 0.73 GB (2.24 vs 2.97 GB,
-25%) nhưng decode **không nhanh hơn**: 12.6 vs 12.7 tok/s (TPOT P50 79.5 vs 78.7 ms), còn
TTFT P50 lại **chậm hơn 37%** (823 vs 600 ms) và TTFT P95 là 1257 vs 681 ms.

Vì sao: máy chạy CPU-only (`ngl=0`) với **một thanh DDR5-5200 single-channel** (~41.6 GB/s
lý thuyết). Ở 4-bit, decode đã sát trần bandwidth, nên bớt 25% byte lẽ ra cho tới ~1.3×.
Nhưng K-quant 2-bit phải unpack/dequantize phức tạp hơn cho mỗi weight. Khi chỉ có vài
core làm việc, chi phí ALU tăng thêm ăn hết phần bandwidth tiết kiệm được, và decode
chuyển từ bandwidth-bound sang gần compute-bound. Prefill vốn compute-bound nên 2-bit còn
thiệt hơn (TTFT tệ hơn).

Chất lượng: tôi hỏi cùng 3 câu (temperature 0) trên cả hai server. Câu giải thích
memory bandwidth và bài cộng giờ (14:35 + 2h50 = 17:25) thì cả hai đều đúng. Với câu dịch
"Continuous batching lets the server add new requests to a running batch", bản 2-bit dịch
sai thuật ngữ thành *"Hàng đợi liên tục"* ở phương án đầu tiên, còn bản 4-bit dịch đúng
"Batching liên tục ... lô đang chạy". Vậy 2-bit không cho thêm tốc độ mà vẫn mất một ít
chất lượng. Nó chỉ đáng dùng khi RAM là ràng buộc cứng (máy 4–8 GB), không phải để tăng tốc.

*Ghi chú:* đây là lần chạy `make bench` thứ hai. Ở lần đầu, bản Q2 có một request outlier
(TPOT 436 ms, E2E 28 s, nhiều khả năng do tác vụ nền trên laptop) làm P95 vọt lên; chạy
lại thì không còn. Kết luận "2-bit không nhanh hơn" giữ nguyên ở cả hai lần
(lần 1: 11.6 vs 12.9 tok/s).
