# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Nguyễn Minh Hiếu
**Cohort:** A20-K4
**Ngày submit:** 2026-08-21

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Linux (Ubuntu WSL2 trên Windows 11)
- **CPU:** 11th Gen Intel(R) Core(TM) i5-11400H @ 2.70GHz
- **Cores:** 6 physical / 12 logical
- **CPU extensions:** AVX2, AVX-512
- **RAM:** 7.7 GB
- **Accelerator:** NVIDIA GeForce RTX 3050 Laptop GPU (4096 MiB)
- **llama.cpp asset đã tải:** llama-b10488-bin-ubuntu-vulkan-x64.tar.gz
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của mình

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

Mình dùng model Qwen3.5 0.8B để vừa vặn với 8GB RAM của laptop. Do chạy qua WSL2 (Ubuntu), lúc đầu `make setup` bị lỗi không tìm thấy binary Python vì venv cũ được tạo từ Windows nên cần xóa đi tạo lại `.venv` chuẩn Linux. Bên cạnh đó cần thêm exclusion cho thư mục lab trong Windows Defender để tránh bị quét file GGUF liên tục gây giật lag I/O (sẽ xóa thư mục local sau này).

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 11224 | 233 / 277 | 36.5 / 39.7 | 2490 / 2772 / 2772 | 27.4 |
| UD-Q2_K_XL | 0.39 | 9678 | 285 / 333 | 30.7 / 33.1 | 2243 / 2392 / 2392 | 32.5 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

Bản UD-Q2_K_XL decode nhanh hơn tầm 19% (32.5 so với 27.4 tok/s) và nhẹ hơn 110MB. Nhưng khi test thử câu hỏi dài, bản 2-bit của model 0.8B trả lời lủng củng và lặp từ thấy rõ. Máy có 8GB RAM nên em có thể chạy bản 4-bit Q4_K_M.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.50 | 17000 | 24000 | 26000 | 8.6 | 0.0% |
| 50 | 0.43 | 23000 | 52000 | 53000 | 11.1 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 0.87×
- **P95 tăng:** 2.17×
- **Effective concurrency ở 50 users:** 11.1 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.84 / 4 slots

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

Server nghẽn cứng ở 50 users: tăng tải 5x nhưng throughput thực tế đi ngang (~0.43-0.50 RPS), còn P95 vọt từ 24s lên 52s. Độ trễ tăng này là do Queue Time (46 request bị deferred vì concurrency 11.1 vượt trần 4 slots). Để cứu Goodput@SLO, em sẽ tăng `--parallel` lên 6 hoặc 8 để mở thêm slot cho Continuous Batching nuốt bớt hàng đợi.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Cloud IaC provisioning | stub |
| N17 Data pipeline | Ingestion & ETL | stub |
| N18 Lakehouse | Storage & Lakehouse | stub |
| N19 Vector + features | Vector search / Embed | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.1 ms
- retrieve: 0.2 ms
- llm: 10100.1 ms
- **stage chiếm nhiều nhất:** llm (100.0% của total)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

Nghẽn gần như 100% ở stage LLM (hơn 10s) vì bước decode bị giới hạn bởi tốc độ đọc RAM. Muốn cắt giảm độ trễ 2x thì phải đánh thẳng vào LLM: bật Prompt Caching cho mớ context tài liệu tĩnh và áp dụng Speculative Decoding hoặc siết bớt `max_tokens`.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Tối ưu hóa số luồng tính toán từ mặc định `-t 6` (6 physical cores) xuống `-t 3` (`LAB_N_THREADS=3`)

```
before:  13.7 tok/s (tại -t 6 mặc định)
after:   22.4 tok/s (tại -t 3 tối ưu)
speedup: 1.63×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Ban đầu cứ tưởng bật đủ 6 core vật lý (`-t 6`) trên con chip i5-11400H sẽ cho max speed, ai ngờ sweep ra `-t 3` mới là điểm ngon nhất (22.4 tok/s so với 13.7 tok/s của `-t 6`, vọt lên 1.63 lần). Ép lên 12 hay 24 luồng thì tụt thảm hại chỉ còn 3.4 tok/s.

Cơ chế ở đây là bài toán cân bằng giữa băng thông bộ nhớ và chi phí đồng bộ luồng trong WSL2:
- Để 3 luồng, CPU tận dụng vừa vặn băng thông RAM và dung lượng cache L3, dữ liệu luân chuyển mượt mà không bị nghẽn bus.
- Khi ép chạy 6 hoặc 12 luồng, các thread liên tục phải đứng chờ nhau ở barrier đồng bộ sau mỗi phép nhân ma trận (matrix tiles). Lúc này CPU tốn rất nhiều chu kỳ context-switch và bị tranh chấp cache (cache thrashing) thay vì tính toán thật. Thành ra hạ về 3 luồng vừa nhẹ máy vừa giúp throughput tăng vọt 63% mà chả cần đụng vào compiler.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** B2 (Context-length sweep - `make sweep-ctx`)

**Numbers:**

```
before:  1826.7 ms TTFT (tại 256 tokens context)
after:   26172.5 ms TTFT (tại 4096 tokens context)
speedup: 14.33× latency growth (0.90× so với linear scaling)
```

**Điều này nói lên gì mà deck chưa nói:**

Lý thuyết thường nhấn mạnh vào chi phí Attention bậc hai $O(N^2)$ khi xử lý context dài. Tuy nhiên kết quả đo thực tế từ 256 đến 4096 tokens lại cho thấy TTFT vẫn tăng gần như tuyến tính (khoảng 0.90x), vì ở tầm context này với model 0.8B thì các phép chiếu Linear và MLP $O(N)$ vẫn chiếm phần lớn khối lượng tính toán.

Cái thấy rõ nhất là chi phí thời gian thực tế: prefill 4k tokens mất tận hơn 26 giây mới nhả token đầu. Thành ra đi làm RAG thực tế, giới hạn prompt không phải là context window nhét vừa bao nhiêu tokens, mà là user có kiên nhẫn ngồi đợi 26s hay không. Bắt buộc phải có Prompt Caching với Chunked Prefill nếu muốn mở rộng context.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Bất ngờ nhất là việc tăng số thread từ 3 lên 6 hoặc 12 (bật hết core vật lý và siêu phân luồng) lại làm tốc độ decode sụt dốc không phanh từ 22.4 tok/s xuống còn 3.4 tok/s (chậm đi hơn 6.5 lần), ngược hẳn với suy nghĩ ban đầu là cứ nhiều core thì sẽ chạy nhanh hơn.


---

## 8. Self-check trước khi push

- [ ] `hardware.json` committed
- [ ] `models/active.json` committed
- [ ] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [ ] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [ ] `benchmarks/02-server-results.md` committed (`make load-report`)
- [ ] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [ ] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [ ] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
