# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Linux-x86_64` · llama.cpp `b10488`
CPU: **6 physical · 12 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 11.1 | 50% |
| 3 | 22.4 | 100% |
| 6 | 13.7 | 61% |
| 12 | 6.2 | 28% |
| 24 | 3.4 | 15% |

**Best**: `-t 3` at 22.4 tok/s
**Slowest tested**: `-t 24` at 3.4 tok/s (6.54x spread)
**Against the physical-core default** (`-t 6`, 13.7 tok/s): 1.63x

Use this in your run:

```bash
LAB_N_THREADS=3 make bench
```

## Your explanation

Điểm gãy (knee) trên máy mình rơi vào đúng `-t 3` với 22.4 tok/s. Ban đầu mình nghĩ 6 core vật lý (`-t 6`) phải nhanh nhất, nhưng thực tế `-t 6` chỉ đạt 13.7 tok/s (thua 1.63x), còn khi đẩy lên 12-24 luồng thì tốc độ tụt thảm hại xuống 3.4 tok/s.

Lý do ở đây là:
- Khi chạy matrix multiplication trên CPU qua WSL2, 3 luồng là vừa đủ để tận dụng băng thông bộ nhớ và cache L3 của chip i5-11400H mà không bị nghẽn bus.
- Khi bật đủ 6 core hoặc bật cả luồng ảo (Hyper-Threading 12/24), các thread phải liên tục chờ nhau ở các barrier đồng bộ sau mỗi phép tính ma trận. Lúc này CPU tốn thời gian context-switch và bị tranh chấp cache L2/L3 nhiều hơn là thực sự tính toán. Vì vậy set `LAB_N_THREADS=3` cho tốc độ tốt nhất trên máy mình.
