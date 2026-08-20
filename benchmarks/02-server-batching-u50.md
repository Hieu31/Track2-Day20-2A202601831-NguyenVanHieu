# 02 - Continuous batching under load (u50)

Host `Linux-x86_64` · `--parallel 4` · 30 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.78 of 4 slots (95%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 1328 |

Highest sampled value was **3.78 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Peak batch width ghi nhận là 3.78, gần đạt mức giới hạn 4 slots của hệ thống. Kết quả này hoàn toàn thống nhất với `effective concurrency = 6.0` quan sát được trong `02-server-results.md`. Mức concurrency 6.0 cho thấy số yêu cầu gửi tới liên tục vượt quá 4 slot xử lý song song hiện có, do đó server gần như luôn phải giữ các slot bận rộn (busy_slots ~ 3.78) để giải quyết, số còn dư (khoảng 2 request) sẽ phải chờ trong hàng đợi làm tăng queue time. Tôi tin tưởng cả 2 chỉ số này vì chúng củng cố luận điểm tính năng continuous batching đang hoạt động chính xác và đạt mức bão hòa.
