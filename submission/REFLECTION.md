# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Nguyen Van Hieu
**Cohort:** AICB-P2T2
**Ngày submit:** 2026-08-20

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Linux
- **CPU:** Intel(R) Core(TM) i3-1005G1 CPU @ 1.20GHz
- **Cores:** 2 physical / 4 logical
- **CPU extensions:** AVX2, AVX-512
- **RAM:** 10.9 GB
- **Accelerator:** CPU only
- **llama.cpp asset đã tải:** b10488
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi
_(Nếu dùng cloud fallback: nói rõ vì sao — RAM < 8 GB, setup fail, v.v. Không mất điểm.)_

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

Mọi cài đặt ban đầu diễn ra suôn sẻ nhờ script `make setup`. Dù máy tính không có GPU và dùng vi xử lý Intel Core i3, lượng RAM 10.9GB vẫn đủ rộng rãi để chứa model Gemma 4 E2B phiên bản lượng tử hóa, hệ thống đã cấu hình chạy CPU-only.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 17716 | 1634 / 1774 | 183.9 / 214.6 | 12931 / 15159 / 15159 | 5.4 |
| UD-Q2_K_XL | 2.24 | 14377 | 2711 / 3901 | 209.2 / 220.0 | 15858 / 17758 / 17758 | 4.8 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

Bản 2-bit xử lý chậm hơn (4.8 vs 5.4 tok/s) vì đánh đổi băng thông RAM lấy giới hạn tính toán (compute) của CPU i3. Khi chạy thử, câu trả lời của Q4_K_XL trơn tru và logic hơn. Do đó, việc hạ xuống bản 2-bit là không đáng vì vừa chậm hơn, vừa giảm chất lượng sinh văn bản.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.13 | 27000 | 30000 | 30000 | 3.7 | 0.0% |
| 50 | 0.19 | 43000 | 43000 | 43000 | 6.0 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.41×
- **P95 tăng:** 1.43×
- **Effective concurrency ở 50 users:** 6.0 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.78 / 4 slots

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

Server bão hòa quanh mức 4 requests. Bằng chứng là lượng user x5 nhưng RPS chỉ tăng x1.41, và effective concurrency đạt 6.0 (vượt quá 4 slots xử lý tối đa). Do đó, P95 bị phình to do thời gian xếp hàng (queue time). Để tăng goodput, tôi sẽ tối ưu số luồng (`LAB_N_THREADS`) trước tiên nhằm giải phóng slot nhanh hơn thay vì mù quáng tăng `--parallel` làm quá tải bộ nhớ.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | - | stub |
| N17 Data pipeline | - | stub |
| N18 Lakehouse | - | stub |
| N19 Vector + features | - | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.1 ms
- llm: 11372.5 ms
- **stage chiếm nhiều nhất:** llm (100% of total)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

LLM là bottleneck tuyệt đối (chiếm 100% thời gian) đúng như dự đoán vì pipeline dùng stub cho phần trước. Để giảm 2x latency, tôi sẽ tối ưu hóa TTFT bằng kỹ thuật Prompt Caching (do context RAG lặp lại cao), hoặc chuyển xuống model Qwen 0.8B nhẹ hơn để giải mã nhanh hơn.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Tăng số luồng tính toán từ `-t 1` lên `-t 4`

```
before:  3.7 tok/s
after:   6.5 tok/s
speedup: 1.76×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

_Giải thích như đang nói với bạn ngồi cạnh. Bám vào **cơ chế**, không phải "vibes":
memory bandwidth? vector width? cache residency? scheduling? queueing? Nếu kết quả
**khác** với kỳ vọng từ deck — nói rõ, và giải thích vì sao. Grader thưởng điểm cho
lập luận đúng về một kết quả bất ngờ, hơn là một con số đẹp không được giải thích._

Việc tối ưu hóa số luồng (`-t`) trực tiếp can thiệp vào khả năng khai thác tính toán của CPU. Máy tính của tôi có cấu trúc 2 physical cores và 4 logical cores. Khi chạy với 1 luồng (`-t 1`), CPU bị "nghẽn nút cổ chai" hoàn toàn bởi giới hạn tính toán (compute-bound) của core đó, khiến tốc độ giải mã chậm. Nhưng khi sử dụng cấu hình `-t 4`, ứng dụng lợi dụng công nghệ Hyper-threading đẩy hiệu năng xử lý song song, tốc độ giải mã tăng từ 3.7 tok/s lên 6.5 tok/s (tăng 1.76 lần).

Đáng chú ý là tốc độ chỉ thực sự bứt phá khi tăng từ 1 lên 2 luồng vật lý (3.7 -> 6.4 tok/s). Khoảng cách từ luồng thứ 2 lên thứ 4 (core logic) chỉ mang lại mức tăng rất nhỏ (6.4 -> 6.5 tok/s). Điều này phản ánh rõ ràng sự chuyển đổi trạng thái của hệ thống: tại mức 2 luồng, quá trình tính toán đủ nhanh để đẩy toàn bộ nút cổ chai sang nghẽn băng thông bộ nhớ (memory bandwidth bound). Khi băng thông RAM đã cạn kiệt ở giới hạn của nó, việc tăng thêm luồng không còn ý nghĩa. Do đó, `-t 4` hay `-t 2` đều là "điểm uốn" (knee) mang lại hiệu quả vượt trội.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** 

**Numbers:**

```
before:  
after:   
speedup: 
```

**Điều này nói lên gì mà deck chưa nói:**

_(để trống nếu bạn không làm phần này)_

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

_(1–2 câu. Không bắt buộc, nhưng grader đọc hết.)_

_(để trống nếu bạn không làm phần này)_

---

## 8. Self-check trước khi push

- [x] `hardware.json` committed
- [x] `models/active.json` committed
- [x] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [x] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [x] `benchmarks/02-server-results.md` committed (`make load-report`)
- [x] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [x] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [x] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [x] 5 screenshots trong `submission/screenshots/`
- [x] `make verify` → **exit 0**
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [x] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
