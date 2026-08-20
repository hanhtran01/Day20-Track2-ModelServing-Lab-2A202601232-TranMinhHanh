# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Linux-x86_64` · llama.cpp `b10488`
Settings: `threads=4` `ngl=0` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 28828 | 701 / 815 | 93.0 / 108.4 | 6227 / 7643 / 7643 | 10.8 |
| UD-Q2_K_XL | 2.24 | 21763 | 1156 / 1434 | 91.8 / 101.1 | 6941 / 7448 / 7448 | 10.9 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` and `UD-Q4_K_XL` decode within 2% of each other here, for 0.73 GB difference on disk.

## Nhận xét: 2-bit có đáng không? Không.

Máy chạy lab: i7-7820HQ (4 core / 8 luồng, AVX2, không AVX-512), 15.5 GB RAM, WSL2,
CPU-only (`ngl=0`), `threads=4`, `ctx=2048`.

### Số liệu

Bản 2-bit nhỏ hơn 0.73 GB (-25%), nhưng:

- TTFT P50 chậm hơn 65% (701 -> 1156 ms), P95 chậm hơn 76% (815 -> 1434 ms)
- TPOT P50 gần như y nhau: 93.0 -> 91.8 ms (nhanh hơn 1.3%, coi như nhiễu)
- Decode 10.8 -> 10.9 tok/s, không đổi
- E2E P50 tệ hơn: 6227 -> 6941 ms

Kết quả bất ngờ, vì cứ tưởng file nhỏ hơn thì nhanh hơn. 2-bit chỉ thắng ở dung lượng đĩa.

### Giải thích

Decode không nhanh lên vì nó bị chặn bởi băng thông RAM, mà máy có 15.5 GB nên cả hai model
đều nằm gọn trong RAM. Bớt 0.73 GB không mua thêm được băng thông nào, và phần tiết kiệm
có lẽ bị bù lại bởi chi phí CPU giải nén block 2-bit. Đây là suy đoán, công cụ của lab
không tách riêng được hai phần này.

Prefill thì chậm đi vì nó xử lý cả prompt một lượt nên nặng về tính toán chứ không nặng về
đọc RAM. Log server cho thấy prompt eval ~59 ms/token, rất chậm với 4 core không AVX-512.
Có thể block 2-bit cần nhiều bước unpack/scale hơn block 4-bit, nên khi cổ chai là CPU thì
file nhỏ hơn lại chạy chậm hơn.

Điều rút ra: quantization tác động lên prefill và decode theo hai chiều ngược nhau. Báo một
số end-to-end duy nhất sẽ che mất chuyện này, nên lab bắt tách TTFT và TPOT là có lý.

(Load model 28.8 s so với 21.8 s đúng theo tỉ lệ file, nhưng đó là đọc đĩa lúc khởi động,
không liên quan tới tốc độ inference.)

### Thử chất lượng

Hỏi cùng 2 câu trên cả hai server (4-bit ở `:8080`, 2-bit ở `:8090`), `temperature=0`.

Câu 1: "4 slot, mỗi request 6 s, 10 request đến cùng lúc, request cuối xong sau bao lâu?"
(đáp án đúng 18 s)

- 4-bit: mở đầu ghi sai "42 seconds" nhưng liệt kê đúng từng đợt t=0 / 6s / 12s. Sai con số,
  suy luận đúng.
- 2-bit: trả lời "60 seconds" rồi dừng, được 4 token, không giải thích gì.

Câu 2: "Phân biệt TTFT và TPOT, đúng 3 bullet."

- 4-bit: đúng 3 bullet, định nghĩa đúng cả hai.
- 2-bit: đúng format nhưng định nghĩa TPOT sai, gọi là "Token Processing Time" và tả thành
  end-to-end.

Mỗi câu chỉ 1 sample nên chưa đủ kết luận chắc chắn. Nhưng cả hai lần 2-bit đều sai theo
cùng kiểu: bỏ phần suy luận, hoặc sai kiến thức mà câu văn vẫn mượt.

### Kết luận

Chọn UD-Q4_K_XL cho cả lab. 2-bit tiết kiệm 0.73 GB đĩa nhưng TTFT xấu hơn 65%, decode
không nhanh hơn, chất lượng kém hơn: thua cả ba mặt.

2-bit chỉ đáng dùng khi RAM hoặc đĩa là giới hạn cứng, ví dụ máy 4-8 GB không load nổi bản
4-bit. Máy này 15.5 GB nên không gặp giới hạn đó.
