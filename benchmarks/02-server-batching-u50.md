# 02 - Continuous batching under load (u50)

Host `Linux-x86_64` · `--parallel 4` · 27 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.84 of 4 slots (96%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | 0.00 |
| `tokens_predicted_total` (final) | 3364 |

Highest sampled value was **3.84 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Lúc chạy tải 50 user, chỉ số `n_busy_slots_per_decode` trung bình đo được đạt đỉnh 3.84/4 slots (khoảng 96% dung lượng). Con số này cho thấy scheduler của llama.cpp đã thực sự gom gần như đầy cả 4 slot vào decode cùng một lượt chứ không chạy tuần tự từng request.

Số 3.84 này nhìn có vẻ lệch so với con số 11.1 tính từ Định luật Little, nhưng thực ra cả hai đều đúng:
- 3.84 là số slot compute thực tế đang chạy trên GPU/CPU cùng lúc (tối đa là 4).
- 11.1 là toàn bộ số request đang có trong hệ thống, bao gồm cả 4 request đang chạy và 46 request đang phải xếp hàng đợi (`requests_deferred = 46`).
Số liệu metrics này chứng minh server đã vắt kiệt 4 slot phần cứng và phần lớn request đến sau bị dồn ứ lại ở hàng đợi.
