# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Linux-x86_64` · llama.cpp `b10488`
Settings: `threads=2` `ngl=0` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 17716 | 1634 / 1774 | 183.9 / 214.6 | 12931 / 15159 / 15159 | 5.4 |
| UD-Q2_K_XL | 2.24 | 14377 | 2711 / 3901 | 209.2 / 220.0 | 15858 / 17758 / 17758 | 4.8 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.12x SLOWER** than `UD-Q4_K_XL` here, despite being 0.73 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

Trên máy của tôi (Intel i3), bản 2-bit xử lý chậm hơn (4.8 tok/s so với 5.4 tok/s của 4-bit) dù tiết kiệm được 0.73GB RAM. Điều này là do CPU giới hạn sức mạnh tính toán (compute-bound) nên việc giải nén (dequantize) format 2-bit tốn chi phí lớn hơn so với băng thông RAM tiết kiệm được. Do đó, bản 4-bit đáng dùng hơn vì tốc độ cao hơn và không bị suy giảm chất lượng.
