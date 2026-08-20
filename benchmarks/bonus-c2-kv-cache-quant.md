# Bonus B4 / C2 - KV cache quantization (f16 so với q8_0)

Host `Linux-x86_64` · CPU i7-7820HQ (4 core / 8 luồng, AVX2) · llama.cpp `b10488` ·
model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` (2.97 GB) · `threads=4` · `ngl=0` ·
`LAB_N_CTX=8192` chia cho 4 slot (`n_ctx_slot = 2048`) · cùng một lần chạy cho cả ba cấu hình

Cách đo: script tự bật `llama-server` ở port 8090 cho từng cấu hình, đọc `VmRSS` của
process ngay sau khi model load xong, gửi một request RAG dài 544 token rồi lấy
`prefill_ms` / `predicted_ms` từ field `timings` của server, cuối cùng chạy 10 prompt số
học có đáp án cố định (`temperature=0`) để chấm chất lượng tự động.

| Cấu hình | RSS sau load | prefill (544 tok) | decode | eval đúng |
|:--|--:|--:|--:|--:|
| f16 (mặc định) | 4049.4 MB | 31.1 tok/s | 11.78 tok/s | 9/10 |
| `--cache-type-k q8_0 --cache-type-v q8_0` | 4005.6 MB | 32.7 tok/s | 11.16 tok/s | 9/10 |
| q8_0 + `-fa on` | 4005.6 MB | 28.7 tok/s | 10.71 tok/s | 9/10 |

## Kết quả

### RAM tiết kiệm được: 43.8 MB, tức 1.1%

Đây là con số đáng nói nhất, và nó nhỏ đến mức bất ngờ. Chuyển KV cache từ f16 sang q8_0 là
giảm một nửa số byte mỗi entry, nên 43.8 MB tiết kiệm được nghĩa là KV cache f16 chỉ chiếm
khoảng 88 MB, tức ~2.7% tổng footprint. Phần còn lại là weight: 2.97 GB.

Lý do: model 2B-class với 8192 token context tổng, chia 2048 token cho mỗi slot. KV cache tỉ
lệ với số layer, số KV head và độ dài context, và cả ba đại lượng đó đều nhỏ ở model này.
Câu chuyện "FP8 KV cache cứu bộ nhớ" trong deck nói về model 70B với 32k context, nơi KV
cache lớn hơn cả weight. Ở quy mô laptop, tối ưu KV cache là tối ưu đúng vào phần chỉ chiếm
2.7%.

### Latency: prefill nhanh hơn 5%, decode chậm hơn 5%

- prefill 31.1 -> 32.7 tok/s (+5.1%)
- decode 11.78 -> 11.16 tok/s (-5.3%)

Hai chiều ngược nhau, và điều này hợp lý: prefill đọc và ghi KV cache theo khối lớn nên KV
nhỏ hơn thì ít byte phải ghi hơn; còn decode phải đọc lại toàn bộ KV cache của mỗi token đã
sinh, và mỗi lần đọc lại phải giải nén q8_0 về float để tính attention. Chi phí giải nén đó
nằm đúng trong vòng lặp nóng nhất. Đây là suy luận theo cơ chế, chưa xác nhận bằng profiler.

Kết quả tổng: request RAG 544 token nhanh hơn không đáng kể ở prefill và chậm hơn không đáng
kể ở decode, tức gần như không đổi.

### Flash attention trên CPU làm chậm thêm

`-fa on` cộng thêm vào q8_0 làm prefill xuống 28.7 tok/s (-12% so với q8_0) và decode xuống
10.71 tok/s (-4% nữa). Flash attention là kernel được thiết kế để tiết kiệm băng thông bộ nhớ
GPU bằng cách gộp các bước attention lại và tránh ghi ma trận trung gian ra HBM. Trên CPU
không có phân tầng bộ nhớ đó nên cái được tiết kiệm không tồn tại, còn kernel thì phức tạp
hơn. Bật một tối ưu GPU trên CPU ở đây là lỗ.

### Chất lượng: không phát hiện được khác biệt

Cả ba cấu hình đều 9/10 trên bộ 10 prompt số học, và cùng sai đúng một câu: câu về hàng đợi
(4 slot, 6 s mỗi request, 12 request, đáp án 18 s) đều trả lời 36. Lỗi giống nhau ở cả ba
cấu hình nên đó là giới hạn của model, không phải của KV cache.

Cần nói rõ giới hạn: 10 prompt số học ngắn không phải bộ eval đủ mạnh để bắt lỗi do KV
quantization. Lỗi kiểu đó thường xuất hiện ở context dài, khi sai số tích luỹ qua nhiều token
đã cache. Kết luận đúng là "không quan sát được suy giảm ở phép thử này", không phải "không
có suy giảm".

## Kết luận

Trên máy này KV cache quantization không đáng bật: tiết kiệm 1.1% RAM, đổi lấy 5% decode
chậm hơn, và không đo được lợi ích nào khác. Nó chỉ đáng khi KV cache thực sự lớn so với
weight, tức model lớn hơn nhiều hoặc context dài hơn nhiều (`LAB_N_CTX` 32k trở lên với
`--parallel` cao). Đó cũng là kịch bản mà deck đang nói tới.

Điểm đáng mang đi: trước khi tối ưu một thành phần, hãy đo nó chiếm bao nhiêu phần trăm đã.
2.7% footprint thì có nén xuống 0 cũng không cứu được gì.
