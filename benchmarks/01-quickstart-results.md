# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Linux-x86_64` · llama.cpp `b10488`
Settings: `threads=6` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 11224 | 233 / 277 | 36.5 / 39.7 | 2490 / 2772 / 2772 | 27.4 |
| UD-Q2_K_XL | 0.39 | 9678 | 285 / 333 | 30.7 / 33.1 | 2243 / 2392 / 2392 | 32.5 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.19x faster** than `Q4_K_M` here, for 0.11 GB less on disk.

## Your observation

Bản 2-bit `UD-Q2_K_XL` decode nhanh hơn tầm 19% (32.5 so với 27.4 tok/s) và nhẹ hơn khoảng 110MB vì mỗi bước decode chỉ cần nạp ít dung lượng qua bus bộ nhớ hơn. Đổi lại, TTFT của bản 2-bit bị trễ hơn một chút (285ms so với 233ms) do tốn thêm thao tác giải nén trọng số.

Về độ hữu dụng thực tế: khi mình thử chạy cùng một câu hỏi prompt trên cả hai bản, bản 2-bit của model nhỏ 0.8B này bị giảm chất lượng khá rõ, câu trả lời bắt đầu lặp từ và diễn đạt lủng củng. Vì máy mình có đủ 8GB RAM nên việc đổi chất lượng lấy 19% tốc độ là không đáng; bản 4-bit `Q4_K_M` vẫn là lựa chọn hợp lý hơn nhiều.
