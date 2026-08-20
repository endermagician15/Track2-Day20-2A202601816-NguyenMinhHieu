# 03 - Integrate: RAG pipeline run

Host `Linux-x86_64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 16287.6 | 16288.7 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.2 | 0.4 | 4133.4 | 4134.4 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 9879.4 | 9879.5 |

Mean per stage (ms): embed **0.1** · retrieve **0.2** ·
llm **10100.1** · total **10100.9**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the provided context, **Goodput** is more useful than raw throughput because it filters out requests that fail to meet their Service Level Objectives (SLOs).

While raw throughput measures the total requests per second (TPOT) that pass through the system, Goodput specifically counts only the requests that successfully met the targets (TTFT and TPOT). This ensures that the system's perform

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation in GPU memory** caused by storing key-value pairs in non-contiguous pages.

By utilizing non-contiguous pages, it removes the wasted internal fragmentation that would otherwise consume most of the GPU's memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps when **prefill is compute-bound and decode is memory-bound**, allowing the system to utilize different resources for each phase.

Specifically:
*   **Prefill** is compute-bound, meaning it requires significant processing power (CPU/GPU) to generate the model parameters.
*   **Decode** is memory-bound, meaning it requires significant bandwidth to load and process 


## Which N16-N19 pieces are real

- **N16 Cloud/IaC:** Stub
- **N17 Data pipeline:** Stub
- **N18 Lakehouse:** Stub
- **N19 Vector + features:** Stub (trong lab dùng keyword overlap fallback in-memory)
- **N20 Serving:** Real (`llama-server` chạy endpoint `/v1/chat/completions`)

**Phân tích:**
Đúng như dự đoán, khâu ngốn thời gian nhất là **LLM generation** (mất tầm 10.1s, chiếm gần như 100% độ trễ cả pipeline), trong khi embed và retrieve dạng stub chỉ tốn chưa tới 1ms. Điều này dễ hiểu vì bước decode của LLM phải đọc ghi trọng số qua bộ nhớ tuần tự từng token một.

Nếu cần cắt giảm một nửa độ trễ của pipeline này, mình chắc chắn sẽ đánh thẳng vào khâu **LLM**:
1. Bật **Prompt Caching** (hoặc Radix Attention) để không phải prefill lại các đoạn context cố định.
2. Dùng **Speculative Decoding** hoặc giới hạn `max_tokens` ngắn lại, vì sinh ít token thì thời gian decode giảm tuyến tính ngay lập tức.
