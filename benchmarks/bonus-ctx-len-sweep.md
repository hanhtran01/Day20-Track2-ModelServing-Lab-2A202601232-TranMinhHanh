# Bonus - Context-length sweep (prefill cost)

Host `Linux-x86_64` · llama.cpp `b10488` ·
`threads=4` `ngl=0` · RAM 15.5 GB

| Prompt tokens | Prefill (tok/s) | TTFT contribution (ms) | vs linear scaling |
|:--|--:|--:|--:|
| 256 | 37.7 | 6785.1 | 1.00x |
| 1024 | 36.2 | 28302.9 | 1.04x |
| 2048 | 35.0 | 58464.2 | 1.08x |
| 4096 | 32.5 | 126069.6 | 1.16x |
| 8192 | 29.5 | 277318.9 | 1.28x |

At 8192 tokens, prefill costs **277319 ms** --
1.28x what linear scaling from the smallest point would predict. That excess
is attention's O(N^2) term becoming visible, and every millisecond of it lands in TTFT
before the user sees a single token.

Either way, this is the number to remember when someone proposes stuffing more retrieved
context into a RAG prompt "because the context window allows it". Prefill is paid in full,
on every request, before the first token appears.

## Kết quả tìm được

### Prefill vượt decode ngay từ khoảng 220 token prompt

Trên máy này decode chạy 10.9 tok/s, tức 93 ms mỗi token sinh ra. Một câu trả lời 64 token
mất khoảng 5950 ms decode. Prefill ở dải ngắn chạy ~37 tok/s, tức ~27 ms mỗi token đọc vào.
Hai phần bằng nhau ở khoảng:

```
5950 ms / 27 ms mỗi token  ~  220 token prompt
```

Nghĩa là chỉ cần prompt dài hơn ~220 token thì prefill đã chiếm quá nửa latency. Con số này
khớp với đo ở bước integrate: request RAG có 151 token prompt và 19 token trả lời cho prefill
72% tổng thời gian.

220 token là rất ít: một system prompt tử tế cộng 1-2 đoạn context đã hết. Không phải "RAG
làm model chậm", mà là mỗi token context phải trả tiền ở prefill, trước khi người dùng thấy
chữ đầu tiên.

### Có thấy chỗ cong bậc hai, nhưng nó chưa phải nguyên nhân chính

Cột "vs linear" đi từ 1.00x lên 1.28x trên dải rộng 32 lần (256 -> 8192 token). Chỗ cong có
thật và tăng đơn điệu, nên đây là số hạng O(N^2) của attention hiện dần ra. Nhưng 1.28x nghĩa
là phần vượt chỉ 28%, trong khi bản thân phần tuyến tính đã tăng 32 lần. Ở kích cỡ model 2B
và dải này, chi phí chính vẫn là các phép chiếu tuyến tính và MLP theo O(N), attention chỉ
mới bắt đầu góp phần.

Cách đọc đúng: đắt hay không phần lớn không do attention bậc hai, mà do đơn giản là phải đọc
nhiều token hơn. Chỗ cong bậc hai là thứ sẽ giết hệ thống ở 32k hoặc 128k context, chưa phải
ở 8k.

### RAG pipeline chịu được bao nhiêu chunk

Lấy SLO P95 <= 10 s như ở phần load test, và giả sử câu trả lời 64 token (5950 ms decode),
thì ngân sách còn lại cho prefill khoảng 4000 ms, tức ~150 token prompt. Với chunk 100 token
thì đó là 1 chunk, cộng system prompt.

Các mốc còn lại cho thấy vì sao không thể nhét thêm:

| Prompt | Prefill | So với SLO 10 s |
|:--|--:|:--|
| 256 token (~2 chunk) | 6.8 s | đã ăn hết 68% ngân sách |
| 2048 token (~20 chunk) | 58 s | quá 6 lần |
| 8192 token (~80 chunk) | 277 s | quá 28 lần, gần 5 phút chỉ để đọc prompt |

Trên phần cứng này, "context window 8192 nên cứ nhét 8000 token vào" tương đương chờ gần 5
phút trước khi có chữ đầu tiên.

### Việc nên làm với kết quả này

Prompt caching là cách duy nhất thoát khỏi bảng trên mà không phải cắt context: đo ở bước
integrate cho thấy prompt lặp lại chỉ phải prefill 5 trên 152 token, giảm 94% thời gian
prefill. Nên xếp system prompt và các chunk cố định lên đầu, chỉ để câu hỏi thay đổi ở cuối.

Một điểm nữa từ chính bảng này: prefill 29-37 tok/s so với decode 10.9 tok/s, tức mỗi token
đọc vào rẻ hơn khoảng 3 lần một token sinh ra. Đó là lý do rút ngắn `max_tokens` có tác dụng
mạnh hơn rút ngắn context, nếu buộc phải chọn một trong hai.
