# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 15 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.76 of 4 slots (94%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 1149 |

Highest sampled value was **3.76 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Peak `n_busy_slots_per_decode` = **3.76 / 4 slot (94%)**, `requests_processing` = 4 suốt
lần đo và `requests_deferred` lên tới **46**. Scheduler thật sự gom 4 request vào chung
mỗi bước decode (continuous batching đang hoạt động), còn khoảng 46 request khác phải
xếp hàng chờ slot.

So với effective concurrency trong `02-server-results.md` (6.2 ở 50 users), hai số
**không mâu thuẫn** vì chúng đo hai thứ khác nhau. Batch width (tối đa 4) là số request
đang được *tính toán*; Little's Law (6.2) đếm cả request đang *chờ*, nên phần 6.2 − 4
chính là queue. Để trả lời "batching có chạy không", tôi tin gauge của server hơn: Little's
Law ở đây chỉ dựa trên 10 request hoàn thành và bị **ước lượng thấp**, vì locust bỏ qua
các request còn treo khi hết 60 s (deferred = 46 cho thấy thực tế có khoảng 50 request
trong hệ thống).
