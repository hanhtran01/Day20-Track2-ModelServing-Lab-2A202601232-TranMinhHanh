# 02 - Continuous batching under load (u50)

Host `Linux-x86_64` · `--parallel 4` · 29 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.77 of 4 slots (94%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 1515 |

Highest sampled value was **3.77 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Nhận xét: batching chạy tốt nhưng không cứu được throughput

### Peak batch width 3.77 / 4 slot (94%)

Batching hoạt động thật. Suốt 29 lần lấy mẫu, `requests_processing` luôn bằng 4 và
`requests_deferred` luôn bằng 46, không đổi từ mẫu đầu tới mẫu cuối. 4 + 46 = 50, đúng bằng
số user locust tạo ra, nghĩa là cả 50 request nằm trong server cùng lúc suốt bài test.

3.77 là trung bình slot có việc trên mỗi bước decode nên không lên đúng 4.00: mỗi lần một
request xong và request mới vào slot, có khoảng ngắn slot đó đang prefill chứ chưa decode.

### Có khớp với effective concurrency trong `02-server-results.md`? Lệch rất xa.

| Nguồn | Số request đang bay |
|:--|--:|
| Little's Law từ locust (RPS x latency trung bình) | 5.7 |
| Gauge của server (`requests_processing + requests_deferred`) | 50 |

Lệch gần 9 lần. Tin số của server.

Lý do: locust chỉ ghi nhận request đã hoàn thành. Ở mức 50 user chỉ có 11 request kịp xong
trong 60 s, còn 39 request vẫn đang chờ khi test kết thúc nên không được ghi vào bảng. Cả
RPS và latency trung bình, hai đầu vào của Little's Law, đều tính chỉ trên nhóm sống sót,
nên 5.7 là con số báo thiếu. Gauge của server đếm trực tiếp hàng đợi ngay lúc đó, không phụ
thuộc request đã xong hay chưa. Đây cũng là lý do phải chạy `make metrics` chồng lên
`make load-50`.

Cùng ý đó nghĩa là P95 = 53 s trong bảng locust cũng báo nhẹ hơn thực tế, vì 39 request bị
bỏ ra ngoài đều đã chờ hơn 60 s.

### Batching gần như không tăng throughput

Trong 57.1 s của cửa sổ đo, `tokens_predicted_total` tăng từ 800 lên 1515:

| | tok/s |
|:--|--:|
| 4 slot song song, đầy tải (715 token / 57.1 s) | 12.5 tổng cả server |
| Một request chạy một mình (`llama-bench`, `-t 4`) | 10.9 |
| Batching lợi được | 1.15x |
| Mỗi người dùng thực nhận | 3.1 |

Batching giúp ở chỗ 4 request dùng chung một lần đọc weight cho mỗi bước decode thay vì đọc
lại 2.97 GB cho từng request. Trên GPU cách này thường cho gần 4x.

Ở đây chỉ được 1.15x, và điều đó khớp với kết quả tune thread: 4 core đã hút hết băng thông
RAM ngay từ khi chạy một request. Batching bỏ được phần đọc trùng lặp, nhưng phần tính toán
của 4 request vẫn chen nhau trên đúng 4 core đó nên tổng công suất gần như không đổi, chỉ
được chia nhỏ hơn cho nhiều người. Đây là suy luận từ số liệu, chưa dùng profiler.

Kết luận: `--cont-batching` đáng bật vì nó giữ 4 slot luôn đầy và không request nào bị bỏ
rơi (0% failure). Nhưng nó không biến máy CPU-only thành máy phục vụ được 50 người: nó chia
đều sự chậm, không xoá được sự chậm.
