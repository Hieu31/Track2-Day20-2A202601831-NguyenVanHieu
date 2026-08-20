# 03 - Integrate: RAG pipeline run

Host `Linux-x86_64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 13247.5 | 13247.7 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 10242.3 | 10242.5 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 10627.7 | 10627.9 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **11372.5** · total **11372.7**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

- **N16 (Cloud/IaC):** stub
- **N17 (Data pipeline):** stub
- **N18 (Lakehouse):** stub
- **N19 (Vector + features):** stub
- **N20 (Serving):** real

Đúng như kỳ vọng, khâu LLM áp đảo hoàn toàn khi chiếm 100% thời gian (11372.5 ms) vì các tác vụ khác đã được stub. Nếu cần giảm 2x độ trễ tổng thể, tôi chắc chắn sẽ tập trung tối ưu phần LLM. Cách đơn giản nhất là áp dụng Prompt Caching (vì context RAG có tính tái sử dụng cao) nhằm triệt tiêu phần lớn thời gian prefill, hoặc hạ thấp model xuống mức 0.8B để cải thiện tốc độ giải mã.
