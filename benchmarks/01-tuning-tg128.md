# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Linux-x86_64` · llama.cpp `b10488`
CPU: **2 physical · 4 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 3.7 | 58% |
| 2 | 6.4 | 98% |
| 4 | 6.5 | 100% |

**Best**: `-t 4` at 6.5 tok/s
**Slowest tested**: `-t 1` at 3.7 tok/s (1.74x spread)
**Against the physical-core default** (`-t 2`, 6.4 tok/s): 1.02x

Use this in your run:

```bash
LAB_N_THREADS=4 make bench
```

## Your explanation

Điểm uốn (knee) nằm ở `-t 4` (đạt 6.5 tok/s). Máy có 2 physical cores và 4 logical cores. Tốc độ tăng gần gấp đôi (3.7 -> 6.4) khi nâng từ 1 lên 2 physical cores (chứng tỏ CPU lúc này đang thiếu sức mạnh tính toán). Khi dùng thêm core logic lên mức 4 threads, mức tăng cực kỳ khiêm tốn (6.4 -> 6.5) vì phần cứng đã tiệm cận kịch trần giới hạn băng thông bộ nhớ (memory bandwidth) của hệ thống. Do đó, `-t 4` là tốt nhất nhưng `-t 2` cũng mang lại hiệu năng gần như tương đương.
