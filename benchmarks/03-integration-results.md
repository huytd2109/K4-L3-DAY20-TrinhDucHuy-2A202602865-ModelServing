# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.3 | 19030.3 | 19030.8 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 11861.2 | 11861.3 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 11439.9 | 11440.0 |

Mean per stage (ms): embed **0.0** · retrieve **0.2** ·
llm **14110.5** · total **14110.7**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the provided context, **Goodput** is more useful than raw throughput because it focuses on **SLOs (Service Level Objectives)** rather than ignoring them.

Here is the breakdown of the reasoning:

*   **Goodput** counts requests per second (RPS) that met the **TTFT** (Total Time to Failure) and **TPOT** (Total Time to Occupancy) targets.
*   **Raw throughput** ignores SLOs entirely.
*   **

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation in GPU memory** caused by storing the Key-Value (KV) cache in non-contiguous pages.

By organizing the KV cache into separate pages, the model avoids the wasted space that would occur if all KV entries were packed into a single contiguous block of memory. This optimization is particularly beneficial on GPUs where memory bandwidth and la

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps when the **prefill operation is compute-bound** and the **decode operation is memory-bound**.

In this scenario, the system utilizes **disaggregated serving splits** to separate the two phases:
*   **Prefill** (computational work) is split onto separate pools.
*   **Decode** (memory bandwidth work) is split onto separate pools.

This separation allows the engine 


## Which N16-N19 pieces are real

Đo ngày 06/10/2026 bằng `.\lab.ps1 pipeline` (tương đương `make pipeline`), gọi
server CP3 ở `http://localhost:8080`. Model Qwen3.5 0.8B Q4_K_M, llama.cpp b10488,
threads=8, ngl=99, parallel=4, ctx=2048 (512 token/slot), reasoning=off.
Ba query chạy tuần tự, top-k=3, max_tokens=200, temperature=0,3; không truyền
`--embed-url`. Không sửa `pipeline.py` hay nối thêm hệ thống bên ngoài.

| Day | Thành phần thực sự dùng trong lượt chạy | Real hay stub? |
|:--|:--|:--|
| N16 Cloud/IaC | localhost trên laptop; không gọi cluster/cloud/IaC | Stub cho cloud deployment |
| N17 Data pipeline | 6 tài liệu tĩnh trong `TOY_DOCS`, không có ingestion/DAG/CDC | Stub |
| N18 Lakehouse | list Python trong bộ nhớ; không gọi Delta/Iceberg/SQLite | Stub |
| N19 Vector + features | keyword overlap trên `TOY_DOCS`, không embedding/vector index/feature store | Stub |
| N20 Serving | HTTP OpenAI-compatible của `llama-server` local | Real |

Đây là trạng thái **trong phép chạy này**, không phải tuyên bố các bài N16–N19
khác chưa được làm. Retrieval có thực thi thuật toán và trả context thật từ dữ
liệu đồ chơi, nhưng vẫn là stub thay cho vector retrieval của hệ thống đầy đủ.

### Bằng chứng end-to-end

`03-pipeline.log` lưu output gốc: cả 3 query có `contexts`, `timings`, `server`
và `answer`. JSON cùng tên với report lưu toàn bộ câu trả lời, id/score và text
của từng context; các text được đối chiếu với `TOY_DOCS` trong source.

Các nhóm context lần lượt là `goodput, paged, radix`; `paged, radix, disagg`;
`disagg, radix, batching`. Hai query đầu có context score=0 vẫn được top-k chọn,
nên đây không phải bằng chứng vector search hay bộ truy hồi chất lượng cao.

Trước/sau run, `/health` sẵn sàng, `/v1/models` trả model, `/slots` có 4 slot.
`tokens_predicted_total` tăng **4832 → 5247, delta 415**, đúng tổng 190+115+110
output token của 3 query; `prompt_tokens_total` tăng **2824 → 3202, delta 378**.
Hai gauge processing/deferred đều bằng 0 ở các mốc trước/sau, không có load test
đang chạy ở các mốc kiểm tra. Counter token là tích lũy của server, không riêng
pipeline; dùng delta để xác nhận lượt này.

Log server ghi số token cuối trong slot là **340, 260, 254**, cả ba đều
`truncated=0`, dưới 512 token/slot. JSON lưu trích log, cấu hình và hash source.
Giữ system prompt giống nhau trong ba request cho phép dùng chung prefix, nhưng
không có thí nghiệm đối chứng cache on/off để báo speedup. Không suy ra cache
hit chỉ từ việc prompt counter tăng ít hơn output counter: độ dài prompt và
output vốn khác nhau.

### Latency nằm ở đâu và cách giảm 2×

Mean embed **0,0 ms**, retrieve **0,2 ms**, LLM **14.110,5 ms**, total
**14.110,7 ms**. LLM chiếm khoảng **99,9986%**, được script làm tròn thành 100%.
Embed=0,0 do không gọi embedding server và thời gian hàm được làm tròn một chữ
số thập phân; không có nghĩa một embedding model thực chạy mất 0 giây.

Kết quả khớp kỳ vọng với corpus chỉ 6 tài liệu: keyword scoring rất nhỏ so với
sinh văn bản. Server báo prefill trung bình **469,8 ms** và decode **10.926,0 ms**.
Stage LLM đo phía client gồm HTTP, chờ phản hồi và inference; phần chênh khoảng
**2.714,7 ms** so với tổng hai timing server chưa được phân rã. Không gọi toàn bộ
stage này là pure GPU compute hay dùng timing prefill thay cho TTFT client.

Để giảm E2E từ khoảng 14,11 xuống **7,06 giây**, tập trung **stage LLM**, trước
hết thử yêu cầu câu trả lời ngắn và giảm max_tokens từ 200 xuống 64. Run hiện có
190/115/110 output token; ít decode token hơn có thể giảm phần 10,93 giây decode.
Giữ câu hỏi và context cố định, đo lại cả thời gian lẫn độ đầy đủ câu trả lời.
Đây là đề xuất, chưa thực hiện và chưa chứng minh đạt 2×; overhead client và thời
gian cố định có thể làm speedup thấp hơn mức giảm token. Nếu còn chậm, đo riêng
HTTP/client và inference để chọn bước tiếp theo. Tối ưu retrieval 0,2 ms không
thể mang lại giảm tổng latency 2× ở baseline này.

### Giới hạn chất lượng và phép đo

Endpoint và pipeline hoạt động, nhưng **không kết luận câu trả lời đúng**:
query goodput tự mở rộng TTFT thành "Total Time to Failure" và TPOT thành
"Total Time to Occupancy"; query PagedAttention đảo ngược quan hệ nguyên nhân
với non-contiguous pages; query disaggregation trộn lợi ích tách prefill/decode
với prefix caching. Các câu trả lời nguyên văn giữ trong JSON để kiểm tra.
Rút ngắn output cũng cần đánh giá chất lượng, không chỉ latency.

Chỉ có ba query tuần tự trên stub, không có percentile hay kiểm thử RAG dưới tải;
không ngoại suy latency này sang embedding/index/corpus thật. Phần Answers
returned phía trên là trích 400 ký tự mỗi câu, còn JSON lưu câu trả lời đầy đủ.
