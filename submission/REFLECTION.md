# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Nguyễn Phúc Huy Hoàng
**Cohort:** A20-K3
**Ngày submit:** 2026-08-20

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 10 (AMD64)
- **CPU:** 11th Gen Intel Core i3-1115G4 @ 3.00 GHz
- **Cores:** 2 physical / 4 logical
- **CPU extensions:** AVX2
- **RAM:** 7.7 GB
- **Accelerator:** Intel UHD Graphics qua Vulkan; log xác nhận offload 25/25 layers
- **llama.cpp asset đã tải:** `llama-b10488-bin-win-vulkan-x64.zip`
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** `Q4_K_M` (primary) + `UD-Q2_K_XL` (compare)

**Chạy ở đâu:** laptop cá nhân, chạy local

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

Máy chỉ có 7.7 GB RAM nên tôi chọn Qwen3.5 0.8B thay cho Gemma 4 E2B.
Trên Windows PowerShell 5.1, tôi sửa encoding và cách chọn Python trong
`lab.ps1`. Tôi cũng sửa launcher Windows để `Ctrl+C` dừng đúng process con;
trước đó server cũ giữ bộ nhớ Vulkan và làm lần chạy Q4 báo hết bộ nhớ GPU.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 6400 | 919 / 1021 | 75.1 / 80.0 | 5627 / 5844 / 5844 | 13.3 |
| UD-Q2_K_XL | 0.39 | 10924 | 2249 / 2347 | 1011.4 / 1053.4 | 65948 / 68710 / 68710 | 1.0 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

Q2 nhỏ hơn Q4 22% nhưng benchmark chậm hơn: TPOT P50 tăng từ 75.1 lên
1011.4 ms và decode giảm từ 13.3 xuống 1.0 tok/s. Với cùng prompt, Q4
lạc sang manufacturing còn Q2 lạc sang data streaming; cả hai chưa mô tả
đúng LLM serving. Vì không tốt hơn và chậm hơn nhiều, Q2 không đáng dùng.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.18 | 34000 | 54000 | 54000 | 5.4 | 0.0% |
| 50 | 0.25 | 29000 | 52000 | 52000 | 8.4 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.38×
- **P95 tăng:** 0.96× (giảm nhẹ từ 54 s xuống 52 s)
- **Effective concurrency ở 50 users:** 8.4 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.76 / 4 slots (94%)

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

Server đã bão hòa từ khoảng 10 users: effective concurrency 5.4 đã vượt
4 slots. Khi tải tăng 5×, RPS chỉ tăng 1.38×; ở 50 users có 46 request
deferred và busy slots đạt 3.76/4. P95 giảm nhẹ do mẫu nhỏ và chỉ tính
request hoàn tất, không phải còn headroom. Tôi sẽ thử `--parallel 4 → 8`
để giảm queue rồi đo lại RPS/P95, đồng thời theo dõi RAM và context mỗi slot.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Localhost, không có Kubernetes/Compose | stub |
| N17 Data pipeline | Danh sách Python trong bộ nhớ | stub |
| N18 Lakehouse | `TOY_DOCS` dictionary | stub |
| N19 Vector + features | Keyword overlap, không có vector index/embedding server | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.1 ms
- llm: 13690.8 ms
- **stage chiếm nhiều nhất:** LLM (gần 100% của tổng 13690.9 ms)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

LLM là bottleneck đúng như kỳ vọng và chiếm gần 100% tổng latency. Nếu
cần giảm latency 2×, tôi sẽ tối ưu LLM trước bằng cách giảm `max_tokens`
từ 200 xuống khoảng 100 và yêu cầu câu trả lời ngắn gọn. Tối ưu retrieval
0.1 ms gần như không làm thay đổi tổng thời gian.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** tăng thread count từ 2 physical-core threads lên 4 logical-core threads

```
before:  12.5 tok/s tại -t 2
after:   13.5 tok/s tại -t 4
speedup: 1.08×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Đường cong gần như phẳng trong khoảng 12.5–13.5 tok/s và không có knee
rõ rệt. Kết quả tốt nhất nằm ở 4 logical threads thay vì 2 physical cores,
nhưng mức tăng chỉ 8%. Vì 25/25 layers đã offload lên Intel UHD qua Vulkan,
decode chủ yếu phụ thuộc GPU tích hợp và băng thông bộ nhớ dùng chung; `-t`
chỉ tác động đến tokenization, chuẩn bị graph, scheduling và việc cấp lệnh
từ host cho GPU.

Bốn logical threads có thể giữ hàng đợi công việc của GPU đầy hơn một chút.
Tuy nhiên, 1 thread đạt 12.8 tok/s còn 2 threads chỉ đạt 12.5 tok/s, cho thấy
chênh lệch nhỏ này còn chịu ảnh hưởng của run-to-run noise, trạng thái nguồn
và tranh chấp shared memory. Vì vậy đây là speedup thực nhưng khiêm tốn, không
phải bằng chứng rằng throughput sẽ tiếp tục tăng nếu thêm nhiều thread hơn.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** Không thực hiện bonus.

**Numbers:**

```
before:  N/A
after:   N/A
speedup: N/A
```

**Điều này nói lên gì mà deck chưa nói:**

Không áp dụng vì tôi chỉ hoàn thành base track.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Điều làm tôi ngạc nhiên nhất là Q2 nhỏ hơn 22% nhưng trong benchmark lại
chậm hơn Q4 rất nhiều dù log xác nhận đủ 25/25 layers đã offload lên GPU.
Kết quả này cho thấy quantization nhỏ hơn không tự động đồng nghĩa nhanh hơn;
hiệu quả còn phụ thuộc kernel, backend Vulkan và phần cứng cụ thể.

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
- [ ] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [ ] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
