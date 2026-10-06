# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **8 physical · 16 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 18.7 | 48% |
| 4 | 31.7 | 81% |
| 8 | 39.0 | 100% |
| 16 | 29.6 | 76% |
| 32 | 20.3 | 52% |

**Best**: `-t 8` at 39.0 tok/s
**Slowest tested**: `-t 1` at 18.7 tok/s (2.08x spread)
**Against the physical-core default** (`-t 8`, 39.0 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=8 make bench
```

## Your explanation

Đo ngày 06/10/2026 trên laptop Windows 11, Ryzen 7 6800HS (8 physical / 16 logical
core), Qwen3.5 0.8B Q4_K_M, llama.cpp b10488. Chạy entrypoint `tune.py` của
`.\lab.ps1 tune`, với `LAB_N_GPU_LAYERS=0` để đo ảnh hưởng thread khi không offload
layer lên GPU. Mỗi mức đo `tg128` hai lần (`-p 0 -n 128 -r 2`). Script được gọi
qua wrapper trong bộ nhớ để lưu thêm output nguyên bản vào JSON; không sửa code
benchmark. Runtime có CUDA backend nhưng cột `ngl=0` ở tất cả các điểm. Đây là
phép đo decode của llama-bench, không phải latency HTTP/TTFT của server ở CP1.

**Knee quan sát nằm ở 8 thread**, đúng số core vật lý: từ 1 → 4 → 8 thread,
throughput tăng 18,71 → 31,74 → 39,01 tok/s. Lên 16 thread giảm còn 29,65 tok/s
(76,0% mức tốt nhất); lên 32 còn 20,26 tok/s (51,9%). Trong lưới đã thử, không
có lợi ích khi thêm thread vượt 8. Chưa đo các mức trung gian như 6 hoặc 10 nên
không khẳng định vị trí knee chính xác hơn lưới này.

Cơ chế phù hợp với đường cong: decode phải đọc lại weights cho mỗi token;
thêm core lúc đầu giúp chia việc, nhưng các core cùng chia sẻ băng thông RAM.
Thread SMT ở mức 16 chia sẻ tài nguyên thực thi và cache của 8 core vật lý,
không tạo thêm memory channel. Ở 32 thread, số worker còn vượt 16 logical core,
làm tăng chi phí lập lịch/chờ và đồng bộ. Những tài nguyên dùng chung đó có thể
khiến throughput giảm dù số thread tăng. Đây là diễn giải phù hợp với kết quả,
không phải bằng chứng profiling xác nhận bandwidth là bottleneck duy nhất;
chưa đo memory bandwidth, cache miss, context switch hay thermal throttling.

**Before/after cho REFLECTION §5:** hạ `-t 16` → `-t 8`, cùng model và `ngl=0`,
29,65 → 39,01 tok/s, speedup **1,32×** (tăng **31,6%**). Điểm before là cấu hình
16 thread chủ động thử trong sweep, không phải mặc định trước đó của server.
Mặc định theo core vật lý đã là 8, nên **so với mặc định chỉ là 1,00×**; không có
cải thiện mới để báo ở cấu hình mặc định. Slowest tested là 1 thread, không phải
32 thread, dù mức 32 minh họa rõ tác hại của oversubscription.

Output gốc ghi mean ± độ lệch chuẩn: 1 thread 18,71 ± 2,30; 4 thread 31,74 ± 1,67;
8 thread 39,01 ± 0,43; 16 thread 29,65 ± 1,40; 32 thread 20,26 ± 0,37 tok/s.
Hai repetition mỗi điểm và một sweep tuần tự là bằng chứng giới hạn; không xem
speedup này là bảo đảm cho mọi workload hay cho GPU serving. Baseline CP1 vẫn
giữ nguyên vì đã dùng 8 thread, nhưng với CUDA offload nên không so trực tiếp
139,8 tok/s của CP1 với 39,01 tok/s của sweep CPU này.

Lệnh tái lập trên Windows:

```powershell
$env:LAB_N_GPU_LAYERS = '0'
.\lab.ps1 tune
Remove-Item Env:LAB_N_GPU_LAYERS
```

Giữ `-t 8` cho chế độ CPU trên máy này. Biến `LAB_N_GPU_LAYERS=0` chỉ được đặt
trong tiến trình đo; không thay cấu hình mặc định của repo cho checkpoint serving.
