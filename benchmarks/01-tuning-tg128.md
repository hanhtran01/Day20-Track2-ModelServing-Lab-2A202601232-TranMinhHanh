# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Linux-x86_64` · llama.cpp `b10488`
CPU: **4 physical · 8 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 5.0 | 46% |
| 2 | 8.1 | 74% |
| 4 | 10.9 | 100% |
| 8 | 6.1 | 56% |
| 16 | 2.6 | 24% |

**Best**: `-t 4` at 10.9 tok/s
**Slowest tested**: `-t 16` at 2.6 tok/s (4.23x spread)
**Against the physical-core default** (`-t 4`, 10.9 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=4 make bench
```

## Giải thích: knee ở đâu và vì sao

Máy chạy lab: i7-7820HQ, 4 core vật lý / 8 luồng logic (có hyperthreading), 15.5 GB RAM,
WSL2, CPU-only (`ngl=0`).

Knee nằm đúng ở `-t 4`, bằng số core vật lý. Trên 4 thì tốc độ không phẳng lại mà tụt mạnh:
8 luồng còn 6.1 tok/s (-44%), 16 luồng còn 2.6 tok/s (chậm hơn đỉnh 4.2 lần).

### Trước knee: có lợi nhưng lợi ít dần

| Bước | tok/s | Tăng |
|:--|--:|:--|
| 1 -> 2 luồng | 5.0 -> 8.1 | 1.61x (không phải 2x) |
| 2 -> 4 luồng | 8.1 -> 10.9 | 1.35x |
| 1 -> 4 luồng | 5.0 -> 10.9 | 2.17x dù có 4x số luồng |

Gấp 4 số luồng chỉ nhanh hơn 2.17 lần, hiệu suất ~54%. Nếu decode thiếu năng lực tính thì
con số phải gần 4x. Nó không gần 4x vì decode chủ yếu là đọc weight từ RAM: mỗi token phải
quét gần hết 2.97 GB model. Băng thông RAM là tài nguyên dùng chung, thêm core không thêm
băng thông, nên từ 2 luồng trở đi phần lớn thời gian là chờ dữ liệu chứ không phải chờ tính.

### Sau knee: 8 luồng chậm hơn 4 luồng

8 luồng nghe như dùng hết máy, nhưng máy chỉ có 4 core thật. Hai luồng logic của cùng một
core dùng chung khối tính vector và chung L1/L2 cache. Với công việc đã đọc RAM liên tục và
đã dùng hết đơn vị SIMD, hyperthread không thêm năng lực tính nào, chỉ thêm hai luồng tranh
nhau cùng một cache.

Nguyên nhân chính của mức tụt 44% có thể là cách llama.cpp chạy đa luồng: nó chia mỗi phép
tính trong layer cho các thread rồi đồng bộ toàn bộ thread ở cuối mỗi bước, nên luồng chậm
nhất quyết định nhịp cả nhóm, và phần chờ đó lặp lại rất nhiều lần cho mỗi token. Khi 2
luồng chia nhau 1 core thì luôn có luồng bị chậm. Đây là suy đoán từ cơ chế, chưa xác nhận
bằng profiler.

### 16 luồng tệ vì lý do khác

16 luồng là gấp 4 số core thật. OS phải liên tục cắt giờ CPU và đổi qua lại giữa các luồng,
mỗi lần đổi lại làm cache mất dữ liệu đang dùng. Cộng thêm WSL2 chạy trong máy ảo và vẫn
chia CPU với Windows, nên phần lớn thời gian thành chi phí quản lý luồng.

### Before / after cho REFLECTION mục 5

Lab mặc định đã lấy đúng 4 nên so với mặc định thì không nhanh thêm (1.00x). Phép so đáng
nói là so với lựa chọn theo bản năng: thấy máy báo 8 CPU thì đặt luôn `-t 8`, và nhiều công
cụ cũng mặc định lấy số luồng logic.

- `-t 8` (số luồng logic): 6.1 tok/s
- `-t 4` (số core vật lý): 10.9 tok/s
- Nhanh hơn 1.79x, không đổi model, không đổi quantization, không cần GPU

Đây là tối ưu rẻ nhất trong lab, và nó đến từ việc đếm đúng core vật lý thay vì tin con số
CPU mà máy báo.

### Kiểm tra chéo

`-t 4` cho 10.9 tok/s ở `llama-bench`, còn bench qua HTTP ở bước trước cho 10.8 tok/s
(TPOT P50 93 ms/token). Hai cách đo độc lập ra gần như cùng một số nên kết quả đáng tin.
