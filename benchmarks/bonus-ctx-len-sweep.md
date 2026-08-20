# Bonus - Context-length sweep (prefill cost)

Host `Linux-x86_64` · llama.cpp `b10488` ·
`threads=6` `ngl=99` · RAM 3.7 GB

| Prompt tokens | Prefill (tok/s) | TTFT contribution (ms) | vs linear scaling |
|:--|--:|--:|--:|
| 256 | 140.1 | 1826.7 | 1.00x |
| 1024 | 190.6 | 5372.8 | 0.74x |
| 2048 | 182.0 | 11254.0 | 0.77x |
| 4096 | 156.5 | 26172.5 | 0.90x |

At 4096 tokens, prefill costs **26172 ms**, which is
**0.90x** linear scaling -- so on this hardware, over this range, prefill is
still growing **roughly linearly**, not quadratically.

That is the correct finding, not a failed experiment. Attention is O(N^2), but it is only
one term: the per-layer linear projections and MLP are O(N), and on a 2B-class model at
short prompts they dominate. The quadratic term only overtakes them once N gets large
enough. Your prefill cost is currently bounded by throughput, not by sequence length.

To find where it *does* bend, extend the grid:

```bash
python bonus/sweeps/ctx-len-sweep.py --grid 1024,4096,8192,16384,32768
```

Watch the "vs linear" column: the first row that climbs meaningfully above 1.0 is where
attention starts to matter on your machine. Report that crossover point.

Either way, this is the number to remember when someone proposes stuffing more retrieved
context into a RAG prompt "because the context window allows it". Prefill is paid in full,
on every request, before the first token appears.

## Your finding

Khi quét context từ 256 lên 4096 tokens, thời gian prefill tăng từ 1.8s lên tới 26.2s. Tỷ lệ tăng này vẫn bám sát đường tuyến tính (khoảng 0.90x), tức là trong dải dưới 4k tokens với model 0.8B, chi phí tính toán của các lớp FFN/Linear $O(N)$ vẫn chiếm chủ đạo chứ chưa thấy rõ đoạn bẻ cong bậc hai của Attention $O(N^2)$.

Nhưng điều đáng chú ý là con số tuyệt đối: prefill 4096 tokens mất tận 26 giây trước khi nhả ra token đầu tiên (TTFT). Bài học thực tế cho RAG là không thể cứ nhồi bừa context vào prompt chỉ vì cửa sổ context cho phép. Nếu muốn app phản hồi nhanh dưới 5s, mình chỉ nên nhét 2-3 chunk tài liệu ngắn (khoảng 500-1000 tokens), hoặc bắt buộc phải có Prompt Caching để tái sử dụng KV cache của phần tài liệu tĩnh.
