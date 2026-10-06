# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Trịnh Đức Huy
**MSSV:** 2A202602865
**Cohort:** 4
**Ngày submit:** 2026-10-06 (UTC+7).

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 11 (AMD64), Python 3.13.15.
- **CPU:** AMD Ryzen 7 6800HS Creator Edition.
- **Cores:** 8 physical / 16 logical.
- **CPU extensions:** — (probe Windows không xuất flags; chưa đo riêng).
- **RAM:** 13,7 GB theo `hardware.json`.
- **Accelerator:** NVIDIA GeForce RTX 3050 Ti Laptop GPU, 4096 MiB; dùng CUDA, probe cũng phát hiện Vulkan.
- **llama.cpp asset đã tải:** `llama-b10488-bin-win-cuda-12.4-x64.zip` (b10488).
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`).
- **Quantization:** Q4_K_M (primary) + UD-Q2_K_XL (compare), từ `models/active.json`.

**Chạy ở đâu:** laptop local; không Colab/Kaggle.
_(Nếu dùng cloud fallback: nói rõ vì sao — RAM < 8 GB, setup fail, v.v. Không mất điểm.)_

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

Windows không có make, dùng lab.ps1 tương đương và runtime CUDA prebuilt. Python virtualenv cần quyền truy cập executable ngoài workspace khi chạy qua sandbox. Sweep thread đặt ngl=0 để đo CPU; baseline/serving giữ ngl=99, threads=8, ctx=2048, parallel=4, reasoning=off. Probe chạy lại khi hoàn thiện bài, khớp phần cứng ban đầu.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 3527 | 307 / 335 | 7.2 / 7.7 | 755 / 793 / 793 | 139.8 |
| UD-Q2_K_XL | 0.39 | 3866 | 316 / 343 | 7.5 / 8.2 | 782 / 830 / 830 | 134.3 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

2-bit chậm 3,9%, nhỏ 21,6% (~109,48 MiB). Cùng bài TTFT/TPOT, cả hai sai; Q2 lạc đề hơn. Giữ Q4. Baseline đầu phiên: 10 mẫu/bản, bỏ warm-up, không xóa page cache; chưa phải cold-cache. Máy dùng CUDA, chưa profiling cơ chế Q2 chậm. Size thực chất GiB.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.39 | 22000 | 27000 | 34000 | 8.3 | 0.0% |
| 50 | 0.34 | 35000 | 59000 | 59000 | 11.4 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** **0,86×** (giảm 14,3%; 17,1% mức tuyến tính).
- **P95 tăng:** **2,19×** (27 → 59 giây).
- **Effective concurrency ở 50 users:** **11,38** so với `--parallel` = **4** slots.

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): **3,92** / **4** slots (peak processing=4, deferred=46 trong lúc có tải).

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

50 user bão hoà: RPS 0,86×, P95 2,19×; L=11,38>4, deferred=46 chứng minh queue, chưa tách queue/compute. Thử parallel=8, ctx=4096 giữ 512 token/slot vì slot kín; chưa thực nghiệm. SLO P95≤30s: goodput u10≈0,375; u50=0,067–0,168 RPS, suy từ percentile. Dùng lượt u50 chạy lại (20 completion); hai CSV 23/20 mẫu còn ít. 5× là user, không phải arrival rate.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | localhost trên laptop; không cloud/cluster/IaC trong lượt này | stub |
| N17 Data pipeline | list tĩnh `TOY_DOCS`; không ingestion/DAG/CDC | stub |
| N18 Lakehouse | list Python trong bộ nhớ; không Delta/Iceberg/SQLite | stub |
| N19 Vector + features | keyword overlap; không embedding/vector index/feature store | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: **0,0 ms** (không gọi embedding server; làm tròn).
- retrieve: **0,2 ms**.
- llm: **14.110,5 ms**.
- **stage chiếm nhiều nhất:** **llm** (**100%** của total khi làm tròn; 99,9986%).

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

LLM chiếm gần 100%, đúng kỳ vọng với 6 toy docs. Muốn giảm 2×, thử câu trả lời ngắn/max_tokens=64 rồi đo lại E2E và chất lượng; chưa chứng minh speedup. Retrieval 0,2 ms không giúp đủ. Ba query có context/answer nhưng còn lỗi nội dung. Mean total=14.110,7 ms.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** hạ `-t 16` xuống `-t 8` trong phép thử CPU (`ngl=0`), giữ nguyên
Qwen3.5 0.8B Q4_K_M và llama.cpp b10488 trên Ryzen 7 6800HS (8 physical / 16 logical
core). Hai điểm đều là số đo thật từ sweep `tg128`, mỗi điểm 2 repetition,
ngày 06/10/2026. Before là điểm 16 thread chủ động thử, không phải mặc định server.

```
before:  -t 16, ngl=0: 29,65 ± 1,40 tok/s
after:   -t 8,  ngl=0: 39,01 ± 0,43 tok/s
speedup: 39,01 / 29,65 = 1,32× (+31,6%)
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

_Giải thích như đang nói với bạn ngồi cạnh. Bám vào **cơ chế**, không phải "vibes":
memory bandwidth? vector width? cache residency? scheduling? queueing? Nếu kết quả
**khác** với kỳ vọng từ deck — nói rõ, và giải thích vì sao. Grader thưởng điểm cho
lập luận đúng về một kết quả bất ngờ, hơn là một con số đẹp không được giải thích._

Knee trong lưới thử nằm ở 8 thread: throughput tăng từ 18,71 (1 thread) lên 31,74
(4) và 39,01 tok/s (8), rồi giảm còn 29,65 (16) và 20,26 tok/s (32). Decode đọc
lại weights mỗi token; thêm thread ban đầu chia việc giữa các core, nhưng chúng
cùng chia sẻ băng thông RAM. SMT ở 16 thread chia sẻ tài nguyên thực thi/cache
của 8 core vật lý; mức 32 còn oversubscribe 16 logical core, tăng chi phí chờ,
lập lịch và đồng bộ. Hạ về 8 phù hợp với việc giảm tranh chấp tài nguyên đó.
Đường cong ủng hộ cách giải thích này nhưng chưa có profiling memory bandwidth,
cache miss hay context switch để xác nhận bottleneck duy nhất.

Điểm tốt nhất trùng mặc định 8 core vật lý: speedup so với mặc định là **1,00×**,
không phải 1,32×. Kết quả cho thấy nên tránh dùng đủ 16 logical thread cho decode
CPU này. Chưa đo các mức giữa 4/8/16 và chỉ có hai repetition nên knee là kết
luận trong lưới đã thử. Đây là throughput llama-bench với `ngl=0`; không suy ra
TTFT/TPOT hay speedup cho CP1 chạy CUDA. Bảng và output gốc lưu tại
`benchmarks/01-tuning-tg128.md` và JSON cùng tên; giữ baseline CP1 để so về sau.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** Không làm bonus; sweep thread §5 thuộc core path.

**Numbers:**

```
before:  Không áp dụng
after:   Không áp dụng
speedup: Không áp dụng
```

**Điều này nói lên gì mà deck chưa nói:**

Không áp dụng.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

_(1–2 câu. Không bắt buộc, nhưng grader đọc hết.)_

File Q2 nhỏ hơn 21,6% nhưng decode không nhanh hơn trên GPU này.
Giảm precision và tăng logical thread đều cần đo thực tế, không thể mặc định nhanh hơn.

---

## 8. Self-check trước khi push

- [ ] `hardware.json` committed
- [ ] `models/active.json` committed
- [ ] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [ ] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [ ] `benchmarks/02-server-results.md` committed (`make load-report`)
- [ ] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [ ] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [ ] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [x] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [x] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [x] Repo GitHub ở chế độ **public**
- [ ] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [x] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

Trạng thái local: `.\lab.ps1 verify` exit 1 với 10 mục bằng chứng chưa được
Git track; nội dung REFLECTION đã pass. Chưa stage/commit/push theo yêu cầu.
Cohort và LMS tạm để; chưa xác nhận nộp bài. Tên/public của origin đã kiểm tra
qua API GitHub ngày 06/10/2026.

Ảnh đối chiếu: `01-hardware-probe.png` (output probe), `02-bench.png` (bảng
baseline), `03a-serve.png` + `03b-smoke.png` (server listening, completion và
metric khác 0; split được README cho phép), `04-locust-10.png` và
`05-locust-50.png` (summary cuối, lần lượt 23/20 request). Chụp trực tiếp file
log/report thật bằng Playwright; ảnh bench trình bày lại ban đầu đã được thay.


---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Codex hỗ trợ đọc, kiểm tra output/CSV/JSON/metrics, đối chiếu số
đo và soạn bản nháp REFLECTION, gồm lập luận cơ chế. Học viên tự đọc và
giải thích được lập luận theo `docs/RULES.md` §3. Goodput u50 là giới hạn suy từ percentile; thí nghiệm đề xuất chưa được thực hiện. Không dùng số liệu mô phỏng.
