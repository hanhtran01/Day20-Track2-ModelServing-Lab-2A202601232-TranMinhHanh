# 02 - Serve: load test + saturation reading

Host `Linux-x86_64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=4` ·
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 15 | 0.27 | 25000 | 54000 | 54000 | 7.6 | 0.0% |
| 50 | 11 | 0.21 | 24000 | 53000 | 53000 | 5.7 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.77x** (15% of linear) |
| P95 latency | **0.98x** |
| Effective concurrency at 50 users | 5.7 vs `--parallel 4` slots (occupancy/slot ratio 1.43) |

**Saturated.** Throughput delivered only 0.77x for 5x the offered load, and effective concurrency (5.7) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.77x while P95 moved 0.98x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

> **Small sample.** Only 11 requests completed in the
> shorter run, so these percentiles are indicative rather than solid. Note also that
> locust averages only *completed* requests: when the run ends with requests still
> queued, effective concurrency is an **under**-estimate. Trust the throughput-scaling
> row over the concurrency row here, and run longer (`-t 3m`) if you want firmer numbers.

## Đọc kết quả: server bão hoà ở đâu

### Bão hoà dưới 10 user

Con số thuyết phục nhất: tăng tải 5 lần (10 -> 50 user) mà throughput giảm còn 0.77x
(RPS 0.27 -> 0.21). Server còn dư sức thì phải làm được nhiều việc hơn khi có nhiều việc
hơn. Ở đây nó làm được ít hơn, nên điểm bão hoà đã bị vượt qua trước mốc 10 user.

Ba bằng chứng độc lập cùng chỉ một hướng:

1. RPS đi xuống khi tải đi lên: 0.27 -> 0.21 (0.77x) cho 5x tải.
2. Effective concurrency đã là 7.6 ngay ở mức 10 user, trong khi server chỉ có 4 slot. Từ
   mức 10 user đã có ~3.6 request ngồi chờ ngoài slot.
3. `requests_deferred = 46` ngay từ mẫu đo đầu tiên và không giảm suốt 60 s. Hàng đợi không
   lúc nào vơi, tức server đã ở trạng thái quá tải ổn định.

Suy ra công suất thực của máy khoảng 4 request đồng thời, đúng bằng số slot. Vượt quá 4 chỉ
là thêm người vào hàng chờ.

### Latency tăng thêm là queue time, không phải compute

| | 1 user | 10 user | 50 user |
|:--|--:|--:|--:|
| P50 | 6.2 s | 25.0 s (4.0x) | 24.0 s |
| P95 | 7.6 s | 54.0 s (7.1x) | 53.0 s |

Model không chạy chậm đi, thời gian tính mỗi request vẫn thế. Gần 19 giây cộng thêm ở P50
là thời gian nằm trong hàng đợi.

P95 ở 50 user (53 s) nhìn như không xấu hơn 10 user, nhưng đó là ảo: locust chỉ tính request
đã xong, mà ở mức 50 user có 39 trên 50 request chưa xong khi test kết thúc. Những request
chờ lâu nhất bị loại khỏi bảng nên P95 thật phải xấu hơn 53 s.

### Goodput tại SLO

Chọn SLO cho phần này: P95 <= 10 s, mức mà một chatbot còn dùng được.

| Mức tải | P95 | Đạt SLO | Goodput |
|:--|--:|:--|:--|
| 1 user | 7.6 s | có | 100% |
| 10 user | 54.0 s | không | ~0% |
| 50 user | 53.0 s (thật ra tệ hơn) | không | ~0% |

Theo SLO này, máy phục vụ được 1 người. Ở 10 người, throughput trên giấy vẫn 0.27 RPS nhưng
goodput bằng 0: không request nào được trả trong hạn. Đó là lý do throughput một mình là chỉ
số gây nhầm lẫn, server vẫn đang làm việc nhưng không còn ai được phục vụ đúng hạn.

### Nếu phải tune một thứ để tăng goodput

Không chọn `--parallel`. Tăng slot lên 8 chỉ nhận thêm request vào phòng chờ, băng thông RAM
không tăng, nên 12.5 tok/s đó bị chia cho 8 người và P95 xấu đi. Ngược lại giảm `--parallel`
xuống 2 là phép thử đáng làm: ít slot hơn thì mỗi request được nhiều băng thông hơn, xong
sớm hơn, số người đạt SLO có thể tăng dù hàng đợi dài hơn. Đây là suy đoán từ cơ chế, chưa
đo.

Không chọn quantization. Bench ở bước trước đã thử: bản 2-bit không decode nhanh hơn (91.8
vs 93.0 ms/token), TTFT còn xấu hơn 65%, chất lượng kém đi.

Thứ nên đổi đầu tiên là giảm số byte phải đọc cho mỗi token, tức đổi sang model nhỏ hơn (ví
dụ Qwen3.5 0.8B, ~0.5 GB thay vì 2.97 GB). Lý do chọn knob này: mọi bằng chứng trong lab đều
chỉ về băng thông RAM là cổ chai duy nhất. Thread sweep cho thấy 4x số luồng chỉ được 2.17x
tốc độ, batching 4 slot chỉ được 1.15x throughput. Cả hai đều là dấu hiệu thêm sức tính
không giúp gì. Model nhỏ hơn là knob duy nhất giảm thẳng số byte đọc mỗi token.

Phương án thứ hai, rẻ hơn và không đổi model: giới hạn `max_tokens`. Chi phí một request gần
như tỉ lệ thuận với số token sinh ra, nên cắt output từ 64 xuống 32 token gần như giảm nửa
thời gian chiếm slot. Cách này không làm model nhanh hơn, nó làm mỗi request nhẹ hơn, và với
SLO theo P95 thì đó cũng là goodput.

Điều không nên làm là hứa rằng cấu hình sẽ cứu được. Máy này 4 core, không GPU dùng được.
Muốn phục vụ 10 người trong 10 s thì cần đổi phần cứng hoặc đổi sang model nhỏ hơn nhiều.
