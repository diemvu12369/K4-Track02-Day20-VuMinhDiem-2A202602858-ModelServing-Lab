# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 6976.7 | 6976.9 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 5096.7 | 5096.8 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 5269.7 | 5269.7 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **5781.0** · total **5781.1**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | không dùng, chạy local trên laptop | stub |
| N17 Data pipeline | không dùng, corpus mẫu có sẵn trong `pipeline.py` | stub |
| N18 Lakehouse | không dùng | stub |
| N19 Vector + features | STUB 1 và 2: không có embedding server, retrieve bằng keyword overlap | stub |
| N20 Serving | `llama-server` (Gemma 4 E2B UD-Q4_K_XL, `-t 4`, CPU) | real |

**Stage chiếm nhiều nhất là llm (~100% của 5781 ms trung bình).** Điều này khớp kỳ vọng:
retrieve bằng keyword overlap trên vài tài liệu chỉ tốn 0.1 ms, embed = 0 vì không gọi
model embedding, còn LLM chạy trên CPU ~13 tok/s. Theo timing của server, mỗi query tốn
~1.4–1.8 s prefill (113–149 token prompt, ~80 tok/s) và ~1.5–3.0 s decode (23–30 token).
Tổng prefill + decode trung bình ~3.6 s, thấp hơn ~2 s so với llm stage đo phía client.
Tôi chưa tách được phần chênh này (tokenize/chat template, HTTP non-streaming, hoặc server
còn xử lý dở hàng đợi từ load test chạy ngay trước đó), nên ghi nhận là chưa giải thích.

**Muốn giảm latency 2×**, tôi sẽ tấn công LLM stage, cụ thể là **prefill**: với RAG nó
chiếm khoảng một nửa thời gian server và sẽ phình theo độ dài context. Cách rẻ nhất là
prompt caching cho phần system prompt/context lặp lại, và giảm top-k hoặc độ dài chunk.
Cách mạnh nhất là GPU offload, vì cả prefill (compute-bound) lẫn decode (bandwidth-bound)
đều nhanh hơn nhiều lần trên RTX 3050 so với CPU.
