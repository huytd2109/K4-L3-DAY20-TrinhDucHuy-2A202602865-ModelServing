# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 13 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.92 of 4 slots (98%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 4430 |

Highest sampled value was **3.92 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

### Cấu hình và lượt đo được báo cáo

Đo ngày 06/10/2026 trên server CP3 ở port 8080: Qwen3.5 0.8B Q4_K_M,
llama.cpp b10488, 8 thread, `ngl=99`, `--parallel 4 --cont-batching --metrics`,
reasoning=off, tổng context 2048 (512 token/slot). Giữ nguyên cấu hình giữa hai
mức tải, không áp `ngl=0` của phép sweep CPU CP2.

Locust dùng script gốc `labs/02-serve/load-test.py`, 60 giây/lượt; ramp 10 user
ở 5 user/s và 50 user ở 25 user/s. Trọng số task 80% short / 20% long-rag,
max_tokens=48/96, temperature=0,5, wait=0,2–1,5 giây. Đây là tỷ lệ chọn task
ngẫu nhiên, không buộc số request hoàn tất chia đúng 80/20. Context RAG là tài
liệu mẫu của lab, không phải pipeline retrieval thực tế.

Lần 10 user chạy bằng `.\lab.ps1 load-10`. Lần 50 user đầu có CSV chốt 27 request
nhưng output cuối 29: Locust 2.46.7 ghi CSV định kỳ, không xuất lại snapshot stats
cuối trước khi đóng file. Vì vậy đã chạy lại 50 user cùng metrics. **Bảng này và
CSV u50 báo lần 50 user thứ hai**, không chọn kết quả nhanh nhất. Lần thứ hai dùng
cùng CLI/workload của `.\lab.ps1 load-50`, với wrapper trong bộ nhớ gọi API
`StatsCSV.requests_csv()` tại `close_files()` để xuất các counter thật sau khi
runner dừng. Không sửa tay số liệu, không sửa source repo hay package. Output
`locust-50.log` ghi rõ bước xuất snapshot cuối; count CSV và output đều là 20.

### Bằng chứng batching trong lúc có tải

Metrics chạy đồng thời với lượt u50 cuối. Cả **13/13 mẫu** từ **15:08:58,6 đến
15:09:54,0 (UTC+7)** đều nằm trong cửa sổ tải, có `requests_processing > 0`.
Counter `n_decode_total` tăng thêm 321 giữa mẫu đầu/cuối, chứng minh decode vẫn
đang tiến triển. Sleep cấu hình là 2 giây nhưng khoảng cách timestamp thực tế
**4,5–4,7 giây**, do vòng lặp còn mất thời gian scrape HTTP; không coi đây là
30 mẫu cách nhau đúng 2 giây.

**Peak `n_busy_slots_per_decode` = 3,9156, làm tròn 3,92 trên 4 slot (98%)**, rõ
ràng lớn hơn 1. Đây là trung bình busy slot mỗi bước decode do server báo, với
giá trị cao nhất trong các mẫu, không phải batch width tức thời tối đa. Peak
`requests_processing=4` và `requests_deferred=46` cho thấy scheduler dùng nhiều
slot cùng lúc, nhưng khi đủ slot vẫn còn request phải xếp hàng. Điều này là bằng
chứng continuous batching, không có nghĩa throughput sẽ tiếp tục tăng khi thêm
user. Gauge busy-slots có thể còn giữ giá trị khi server rảnh, nên phải đối chiếu
với processing và counter decode tăng trong cửa sổ tải như ở đây.

`kv_cache_usage_ratio` không được build này xuất: giữ **n/a**, không đổi thành 0.
`tokens_predicted_total` là counter tích lũy của server qua các lượt, không riêng
u50: từ 3288 đến 4430 trong cửa sổ lấy mẫu (delta 1142), không gọi 4430 là số token
chỉ của lần 50 user cuối.

### So với latency và effective concurrency

Đọc trực tiếp dòng Aggregated của hai CSV cuối:

| Users | Completed requests | RPS | Mean (ms) | P50 (ms) | P95 (ms) | P99 (ms) | Failures | RPS × mean latency |
|--:|--:|--:|--:|--:|--:|--:|--:|--:|
| 10 | 23 | 0.392 | 21192.5 | 22000 | 27000 | 34000 | 0 | 8.31 |
| 50 | 20 | 0.336 | 33863.3 | 35000 | 59000 | 59000 | 0 | 11.38 |

Tăng user 5× chỉ đạt **0,86× throughput**, trong khi P95 tăng **2,19×**. Batching
có hoạt động nhưng các slot và hàng đợi đã chịu tải lớn. Với 50 user, effective
concurrency ước lượng **11,38** không bằng busy-slots **3,92** vì RPS × mean
latency tính request trong hệ thống, gồm thời gian chờ; busy-slots chỉ phản ánh
các slot đang làm việc hữu ích ở decode. Để chứng minh batching, tin gauge server
được đọc trong tải; để mô tả trải nghiệm client, dùng latency từ Locust. Không
coi effective concurrency là độ rộng batch hoặc phần trăm sử dụng slot.

Các giá trị trên là đầu vào cho report saturation `02-server-results.md` ở CP5.
Với chỉ 23/20 completion, percentile
Locust là xấp xỉ và mẫu nhỏ. Request còn chờ/đang chạy khi hết 60 giây không được
tính như completion; vì thế 0 failure không đồng nghĩa toàn bộ 50 user đã hoàn
tất request, và RPS × mean chỉ là ước lượng trong một cửa sổ chưa ổn định.

Ảnh `04-locust-10.png` và `05-locust-50.png` chụp trực tiếp đoạn output cuối từ
file log gốc mở trong trình duyệt; nội dung được đối chiếu với file trên đĩa.
Server được giữ chạy để dùng ở checkpoint tiếp theo.
