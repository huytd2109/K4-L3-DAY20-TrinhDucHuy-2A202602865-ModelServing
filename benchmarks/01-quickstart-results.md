# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=8` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 3527 | 307 / 335 | 7.2 / 7.7 | 755 / 793 / 793 | 139.8 |
| UD-Q2_K_XL | 0.39 | 3866 | 316 / 343 | 7.5 / 8.2 | 782 / 830 / 830 | 134.3 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.04x SLOWER** than `Q4_K_M` here, despite being 0.11 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

### Điều kiện và cách đọc baseline

Đo ngày 06/10/2026 trên Windows 11, Ryzen 7 6800HS (8 core vật lý), RAM 13,7 GB,
RTX 3050 Ti Laptop 4 GB, bằng `.\lab.ps1 bench` (tương đương `make bench`). Đây là
lần benchmark đầu trong phiên này, chạy Q4 trước rồi Q2; không xóa page cache của
OS nên không gọi đây là phép đo cold-cache. Mỗi bản có một request warm-up bị loại
và 10 request tuần tự được tính; temperature=0,7, reasoning=off, 4 slots, tổng
ctx=2048 (512 token/slot). Load là thời gian khởi động đến khi HTTP ready, tách
khỏi latency request. Log kiểm tra sau baseline xác nhận Q4 offload 25/25 layer
lên CUDA; runtime nhận đúng RTX 3050 Ti. JSON lưu cấu hình và trích log này.

TTFT phía client bao gồm tạo kết nối, HTTP, xử lý prompt và chờ content token đầu,
không phải chỉ thời gian prefill của GPU. TPOT được script tính bằng
`(E2E - TTFT) / (n_out - 1)`, có cả overhead stream. `n_out` lấy từ `timings.predicted_n`;
phép thử chất lượng cũng xác nhận server trả block timings (số token khác số chunk).
Percentile dùng nearest-rank: P50 là mẫu thứ 5; với n=10, P95 và P99 đều là mẫu
lớn nhất, nên hai cột E2E đó bằng nhau. Mẫu ít chưa đủ mô tả chắc chắn đuôi latency.

### 2-bit nhanh hơn và nhỏ hơn bao nhiêu?

2-bit **không nhanh hơn** trong lần đo này: 134,3 so với 139,8 tok/s, thấp hơn
**3,9%** (chênh 5,5 tok/s). TPOT P50 tăng từ 7,15 lên 7,45 ms (**4,2%**);
TTFT P50 tăng từ 307,1 lên 316,1 ms. E2E P50 là 782,5 so với 755,2 ms, nhưng
số output token giữa hai bản có thể khác nhau nên không diễn giải E2E như phép
so sánh cùng độ dài output.

File Q4 có 532.517.120 byte, Q2 có 417.718.528 byte: tiết kiệm **114.798.592 byte,
khoảng 109,48 MiB / 0,107 GiB, tức 21,6%** dung lượng file. Cột Size do script
chia cho 1024³ nên thực chất là GiB dù nhãn ghi GB. Tên 2-bit không có nghĩa file
nhỏ đúng một nửa, vì còn tensor dùng độ chính xác khác và metadata.

Máy này có GPU offload, không thuộc ví dụ CPU ít core/no-GPU trong đoạn giải thích
tự sinh phía trên. Kết quả cho thấy giảm byte chưa mang lại lợi ích decode ở cấu
hình hiện tại. Chi phí kernel/dequantization và overhead có thể lấn át phần tiết
kiệm bandwidth với model nhỏ; đây là giả thuyết, chưa có profiling để kết luận
compute-bound hay bandwidth-bound. Chênh lệch nhỏ từ một lần chạy cũng chưa chứng
minh Q2 luôn chậm hơn.

### So chất lượng và quyết định sử dụng

Hỏi cùng một câu tiếng Việt trên Q4 ở port 8080 và Q2 ở port 8090, chạy lần lượt,
cùng temperature=0, seed=42, max_tokens=512, reasoning=off:

> Một request sinh đúng 51 output token. Token đầu tiên đến sau 300 ms, request kết thúc sau 1300 ms tính từ lúc gửi. Hãy trả lời bằng tiếng Việt trong 3 gạch đầu dòng: (1) TTFT là gì và bằng bao nhiêu; (2) TPOT là gì, viết công thức và tính giá trị, không tính token đầu; (3) vì sao không thể thay TTFT và TPOT bằng một số E2E duy nhất?

Đáp án kiểm tra: TTFT=300 ms; TPOT=(1300-300)/(51-1)=20 ms/token; E2E=1300 ms
không cho biết riêng phần chờ token đầu và tốc độ sinh các token sau.

- Q4 viết TTFT là thời gian đến khi trả về "đầu tiên 51 token, tức là khoảng 51 ms";
  gọi TPOT là "Time to Produce One Output", không đưa công thức hay kết quả 20 ms.
  Bản này có nói về thời gian chờ và các bước xử lý nhưng vẫn sai yêu cầu chính.
- Q2 viết "TTFT là một thuật toán học thuật" và tương tự cho TPOT; cả ba ý lặp
  lại nội dung không có giá trị cố định, không tính được số nào. Chất lượng kém
  hơn Q4 trên câu hỏi này.
- Cả hai kết thúc với `finish_reason=stop`, không bị cắt do max_tokens. Toàn bộ
  prompt, câu trả lời nguyên văn và timings nằm trong JSON cùng tên với report.

**Quyết định:** với máy này, giữ Q4 làm baseline cho bước tuning tiếp theo. Q2 chỉ
tiết kiệm khoảng 21,6% dung lượng, decode chậm hơn 3,9% và trả lời kém hơn trong
phép thử này, nên chưa đáng đổi sang Q2. Cả Q4 cũng trả lời sai; một câu hỏi chưa
đủ đánh giá chất lượng tổng quát hay chấp nhận model cho tác vụ cần độ chính xác.

Ghi nhận hỗ trợ AI: Codex chạy phép đo local, đối chiếu câu trả lời và hỗ trợ tổng
hợp nhận xét từ bằng chứng trong repo; không dùng số liệu mô phỏng.
