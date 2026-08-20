# Bonus B5 / C9 - Embedding serving: một regime khác

Host `Linux-x86_64` · CPU i7-7820HQ (4 core / 8 luồng, AVX2) · llama.cpp `b10488` ·
`llama-server --embedding` ở port 8081 · `threads=4` · `ngl=0` ·
embedding dim 1536 · corpus 8 doc

Lệnh chạy:

```bash
make serve-embed        # terminal 1
make embed-demo         # terminal 2
```

Lưu ý bắt buộc nói rõ: lab không có embedding model chuyên dụng, nên server này chạy chính
GGUF của chat model ở pooling mode. Vector lấy ra là mean-pooled decoder state, tức một
sentence encoder yếu. Retrieval thật cần model riêng như Qwen3-Embedding, BGE-M3 hoặc
EmbeddingGemma. Mọi kết luận dưới đây là về regime phục vụ, không phải về chất lượng
retrieval.

## Retrieval hoạt động

Query: "Does embedding serving use a KV cache and a decode loop like chat serving?"

| Rank | Cosine | Document |
|--:|--:|:--|
| 1 | 0.843 | Embedding serving is prefill-bound: one forward pass, no KV cache, no decode loop |
| 2 | 0.780 | RadixAttention reuses a shared prompt prefix across requests via a radix tree |
| 3 | 0.747 | Speculative decoding drafts several tokens and verifies them in one forward pass |

Doc đúng lên hạng 1, nhưng khoảng cách tới hạng 2 chỉ 0.063 và hạng 3 vẫn 0.747. Phổ điểm bị
nén sát nhau, đúng như dự đoán với một chat model dùng làm encoder: nó cho điểm cao cho mọi
câu cùng chủ đề chứ không phân biệt được câu nào trả lời đúng câu hỏi. Với ngưỡng cắt 0.75
thì doc số 3 cũng lọt vào context.

## Throughput theo batch size: phẳng

| Batch | Tổng thời gian | Mỗi text | texts/s |
|--:|--:|--:|--:|
| 1 | 687.8 ms | 687.8 ms | 1.5 |
| 2 | 1342.0 ms | 671.0 ms | 1.5 |
| 4 | 2439.9 ms | 610.0 ms | 1.6 |
| 8 | 4914.9 ms | 614.4 ms | 1.6 |
| 16 | 9914.7 ms | 619.7 ms | 1.6 |

Batch 16 chỉ nhanh hơn batch 1 khoảng 1.11x tính trên mỗi text. Đây là kết quả trái với kỳ
vọng: embedding serving được cho là lấy throughput từ static batch lớn, vì mỗi text chỉ cần
một forward pass và có thể xếp chung thành một ma trận lớn.

Lý do phẳng: static batch giúp khi phần cứng còn đơn vị tính rảnh, tức khi tăng batch làm
matmul chuyển từ nhỏ-và-kém-hiệu-quả sang lớn-và-hiệu-quả. Trên GPU điều đó đúng vì GPU có
hàng nghìn core chờ việc. Trên 4 core CPU đã bị chặn bởi băng thông RAM thì batch 1 đã dùng
hết tài nguyên rồi, batch 16 chỉ xếp hàng cùng lượng việc đó lâu hơn. Đây cùng một cơ chế đã
thấy ở continuous batching trong base track: 4 slot chỉ cho 1.15x throughput.

## Hai regime cần chiến lược batching trái ngược nhau

| | Chat / decode (base track) | Embedding (bonus này) |
|:--|:--|:--|
| Công việc mỗi request | 1 prefill + N bước decode | 1 forward pass duy nhất |
| KV cache | có, phải giữ suốt request | không |
| Nguồn throughput | continuous batching, nhét request mới vào slot trống mỗi bước decode | static batch lớn, xếp theo độ dài token |
| Đơn vị latency quan tâm | TTFT và TPOT | thời gian mỗi text, không có TTFT |
| Trên máy này | 12.5 tok/s tổng, 3.1 tok/s mỗi user | 1.6 texts/s, gần như không đổi theo batch |

Chat serving cần scheduler động vì mỗi request kéo dài nhiều bước và kết thúc ở thời điểm
khác nhau: ai xong thì nhường slot. Embedding serving không cần scheduler động vì mỗi request
chỉ một bước; cái nó cần là gom đủ text để tạo batch lớn rồi đẩy một lần, và gom text có độ
dài gần nhau để không phải pad thừa.

Hệ quả khi phục vụ cả hai sau cùng một autoscaler: hai regime scale theo hai tín hiệu khác
nhau. Chat cần thêm replica khi số slot bận và queue depth tăng; embedding cần thêm replica
khi hàng đợi batch không kịp gom trong thời gian chờ cho phép. Dùng chung một metric, ví dụ
chỉ nhìn CPU hoặc chỉ nhìn RPS, sẽ scale sai một trong hai: embedding có RPS cao nhưng mỗi
request rẻ, chat có RPS thấp nhưng mỗi request giữ tài nguyên rất lâu. Ngoài ra chúng cạnh
tranh trực tiếp trên cùng băng thông RAM nếu chạy trên cùng máy, nên đặt hai regime cạnh nhau
trên một node là cách nhanh nhất để làm TTFT của chat xấu đi mà dashboard của embedding vẫn
xanh.

## Giới hạn của phép đo này

- Chat GGUF dùng làm encoder nên phổ similarity bị nén, không kết luận được gì về recall.
- Corpus 8 doc, một query. Đây là smoke test của regime, không phải benchmark retrieval.
- Chỉ đo trên CPU. Trên GPU đường cong batch gần như chắc chắn sẽ dốc lên, và đó mới là lý
  do người ta xếp static batch cho embedding service. Không kiểm chứng được với phần cứng
  hiện tại.
