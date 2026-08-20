# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 14303.6 | 14303.9 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 7658.8 | 7658.9 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 19109.9 | 19110.0 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **13690.8** · total **13690.9**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the context provided, **goodput** is more useful than raw throughput because it specifically accounts for the **SLOs** (Service Level Objectives).

The text states that:
1.  **Goodput** counts only requests per second that met the TTFT and TPOT targets.
2.  **Throughput at saturation** ignores SLOs.

Therefore, goodput ensures that the system's performance remains aligned with the service

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation in GPU memory** by storing the KV cache in non-contiguous pages. This design removes the wasted space that would otherwise exist if the cache were stored contiguously within the GPU's memory hierarchy.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps when the model's **compute-bound prefill** requires significant processing power, while the **memory-bound decode** requires significant bandwidth.

By separating these steps, the model can:
1.  **Maximize compute efficiency**: Prefill is computationally expensive and can be bound by the GPU's memory bandwidth, whereas decoding is often bound by the CPU's memory 


## Thành phần nào là real và thành phần nào là stub

| Day | Thành phần | Trạng thái trong lần chạy này |
|:--|:--|:--|
| N16 Cloud/IaC | Hạ tầng triển khai | **Stub** — pipeline và server chạy trên localhost, không dùng Kubernetes hay Compose stack |
| N17 Data pipeline | Nguồn và luồng dữ liệu | **Stub** — dữ liệu là danh sách Python nằm trong bộ nhớ, không có DAG hoặc batch job |
| N18 Lakehouse | Kho dữ liệu | **Stub** — `TOY_DOCS` dạng dictionary thay cho Delta/Iceberg/SQLite |
| N19 Vector + features | Embedding và retrieval | **Stub** — không có embedding server hoặc vector index; retrieval dùng keyword overlap |
| N20 Serving | Model serving | **Real** — pipeline gọi `llama-server` thật qua `/v1/chat/completions` |

LLM là bottleneck đúng như kỳ vọng: trung bình mất 13,690.8 ms trên tổng
13,690.9 ms, tương đương gần 100% latency; embedding là 0.0 ms và retrieval
chỉ 0.1 ms. Nếu cần giảm một nửa latency, tôi sẽ tối ưu giai đoạn LLM trước,
đặc biệt giảm `max_tokens` từ 200 xuống khoảng 100 và yêu cầu câu trả lời ngắn
gọn. Tối ưu retrieval gần như không ảnh hưởng đến tổng latency hiện tại.
