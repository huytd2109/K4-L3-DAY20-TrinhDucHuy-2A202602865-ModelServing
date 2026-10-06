# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=8` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 23 | 0.39 | 22000 | 27000 | 34000 | 8.3 | 0.0% |
| 50 | 20 | 0.34 | 35000 | 59000 | 59000 | 11.4 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.86x** (17% of linear) |
| P95 latency | **2.19x** |
| Effective concurrency at 50 users | 11.4 vs `--parallel 4` slots (occupancy/slot ratio 2.85) |

**Saturated.** Throughput delivered only 0.86x for 5x the offered load, and effective concurrency (11.4) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.86x while P95 moved 2.19x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

### Điểm bão hoà và bằng chứng queue

Kết luận **Saturated** được xác nhận ở mức **50 user**: RPS từ 0,391996 xuống
0,336080, tức **0,857×** (giảm 14,3%), chỉ **17,1%** mức tăng tuyến tính kỳ vọng
theo số user. P95 từ 27.000 lên 59.000 ms, tức **2,185×**, tăng thêm 32 giây.
Trong bài này tải tăng 5× là số user ảo; Locust là tải vòng kín có wait time,
không phải thí nghiệm cố định arrival rate tăng đúng 5×.

Ở 50 user, `L = 0,336080 × 33,863322 = 11,38`, so với 4 slot, tỉ lệ occupancy
**2,85**. `02-server-metrics-u50.csv` ghi peak busy-slots **3,9156/4**, processing
**4**, deferred **46** trong lúc tải chạy. Deferred là bằng chứng trực tiếp có
request chờ slot; busy-slots gần 4 chứng minh scheduler đã gộp nhiều request vào
các bước decode. Cấu hình vẫn là Qwen3.5 0.8B Q4_K_M, threads=8, ngl=99,
parallel=4, ctx=2048, reasoning=off, giống giữa hai lượt.

Vì vậy latency có **queue time** dưới tải 50 user; không thể kết luận chỉ có
compute time. Tuy nhiên hai CSV không có timestamp riêng lúc vào/ra hàng đợi,
nên **không gán toàn bộ 32 giây tăng P95 cho queue**, cũng không định lượng được
queue/compute split. Compute có thể thay đổi khi chia sẻ GPU, prefill xen decode
và hỗn hợp request thay đổi. Hai lượt không có cùng tỷ lệ completion long-rag:
4/23 ở u10 và 5/20 ở u50. Đây là nguồn biến thiên phải giữ trong diễn giải.

Ở 10 user, L=8,31 cũng lớn hơn 4 và gợi ý đã có hàng đợi. Chưa đo dưới 10 user
hay sweep thêm điểm, nên chưa xác định chính xác user count bắt đầu bão hoà.
Little's Law dùng chắc nhất trong trạng thái ổn định; ở cửa sổ 60 giây, RPS và
mean chỉ lấy completion nên L ở đây là ước lượng. L dưới số slot cũng không tự
chứng minh rằng không request nào từng phải chờ. Tin deferred trong lúc tải để
xác nhận queue, không dùng L như phép đo slot utilization.

### Goodput với SLO tự chọn

Chọn **SLO P95 E2E ≤ 30 giây** cho hỗn hợp chat/RAG của lab này, tương đương mục
tiêu ít nhất 95% request hoàn tất trong 30 giây. Đây là ngưỡng demo của bài, chưa
là SLO production. Goodput đếm completion có E2E ≤ 30 giây trên cùng cửa sổ thời
gian với RPS, không tự động bằng 0 khi P95 vượt ngưỡng.

| Users | Completion trong 30 giây | Tỷ lệ đạt ngưỡng | Goodput@30s (request/s) | P95 SLO |
|--:|--:|--:|--:|:--|
| 10 | 22/23 | 95,7% | 0,375 | Đạt: 27 giây |
| 50 | 4–10/20 | 20–50% | 0,067–0,168 | Không đạt: 59 giây |

Các số này **suy ra từ percentile**, không phải đếm histogram request được lưu:
CSV chỉ có summary. Theo hàm percentile của Locust đang cài, P95 của 23 mẫu là
mẫu thứ 22 tăng dần; nó khoảng 27 giây, còn max thực 33,803 giây, nên đúng 22 mẫu
u10 đạt 30 giây. Với u50, nhóm long-rag có mẫu thứ 3 khoảng 26 giây, thứ 4 khoảng
35 giây, suy ra 3/5 đạt; nhóm short có ít nhất 1 và tối đa 7/15 đạt. Kết hợp với
P50 aggregate khoảng 35 giây cho giới hạn 4–10/20. Đã tính biên làm tròn 1 giây
của histogram Locust; các bucket 26/27 và 35 giây nằm rõ hai phía ngưỡng 30 giây.
Không ghi một tỷ lệ chính xác chưa biết cho u50. JSON cùng tên lưu cách tính,
giới hạn và hash hai CSV đầu vào.

Goodput@30s ở u50 do đó chỉ giữ khoảng **18–45%** mức u10 trong các completion đã
đo, dù số user tăng. Không ngoại suy sang toàn bộ request đến: nhiều request vẫn
chờ/chạy khi hết 60 giây. Cả hai CSV có 0 lỗi được ghi nhận nhưng không chứng
minh mọi request đã hoàn tất. Script không in Small sample vì count là 23/20,
cả hai không dưới 20; đây vẫn là mẫu nhỏ, tail percentile còn yếu.

### Knob thử trước

Thử **`LAB_PARALLEL=8` trước**, vì 4 slot đang đầy và có tới 46 request deferred.
Đồng thời đặt `LAB_N_CTX=4096` để giữ **512 token/slot**, tránh context mỗi slot
bị giảm từ 512 xuống 256 khi tăng slot, làm thay đổi khả năng chứa prompt RAG.
Giữ model, quantization, threads=8, ngl=99 và workload cố định; đo lại cùng SLO.
Lý do thử slot trước thread: CP2 đã cho 8 thread là tốt nhất ở CPU, còn phép
serving này có GPU và bằng chứng trực tiếp của hàng đợi tại 4 slot.

Đây là **thí nghiệm đề xuất, chưa chạy và chưa chứng minh speedup**. Thêm slot
có thể giúp batch hiệu quả hơn nhưng cũng có thể làm GPU/KV chịu áp lực hơn,
kéo dài compute. Chỉ chấp nhận nếu goodput@30s tăng, P95 giảm và không tăng lỗi;
nếu không, giữ 4 slot. Không thay cấu hình server đang chạy trong checkpoint này.
