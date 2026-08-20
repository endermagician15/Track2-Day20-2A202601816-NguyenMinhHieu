# 02 - Serve: load test + saturation reading

Host `Linux-x86_64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=6` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 29 | 0.50 | 17000 | 24000 | 26000 | 8.6 | 0.0% |
| 50 | 23 | 0.43 | 23000 | 52000 | 53000 | 11.1 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.87x** (17% of linear) |
| P95 latency | **2.17x** |
| Effective concurrency at 50 users | 11.1 vs `--parallel 4` slots (occupancy/slot ratio 2.77) |

**Saturated.** Throughput delivered only 0.87x for 5x the offered load, and effective concurrency (11.1) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.87x while P95 moved 2.17x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

Server bị bão hòa hoàn toàn ở mốc 50 users. Bằng chứng rõ nhất là: tăng tải gấp 5 lần nhưng throughput thực tế chẳng tăng tí nào (từ 0.50 RPS còn tụt nhẹ xuống 0.43 RPS do overhead), trong khi độ trễ P95 đội lên hơn gấp đôi (từ 24s vọt lên 52s).

Độ trễ tăng này không phải do mô hình tính toán chậm đi, mà hoàn toàn là **thời gian xếp hàng (queue time)**. Bằng chứng là hệ thống chỉ có 4 slots nhưng concurrency lại lên tới 11.1, khiến 46 requests bị đẩy vào hàng đợi chờ slot trống.

Nếu cần nâng Goodput (để giữ P95 trong mức chấp nhận được, ví dụ dưới 25s), knob mình sẽ chỉnh đầu tiên là **tăng `--parallel` từ 4 lên 6 hoặc 8** và áp dụng `LAB_N_THREADS=3`. Tăng slot sẽ giúp server gom batch được nhiều request hơn cùng lúc, giải phóng hàng đợi nhanh hơn mà không tốn thêm nhiều thời gian tính cho mỗi token.
