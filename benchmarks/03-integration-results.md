# 03 - Integrate: RAG pipeline run

Host `Linux-x86_64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 7401.9 | 7402.0 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 5348.5 | 5348.6 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 5556.2 | 5556.3 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **6102.2** · total **6102.3**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## N16-N19 phần nào thật, phần nào stub

Chỉ N20 là thật, bốn phần còn lại đều là stub.

| Ngày | Thành phần | Trạng thái | Đang dùng cái gì |
|:--|:--|:--|:--|
| N16 | Cloud / IaC | stub | localhost trong WSL2 trên laptop, không k8s, không Compose |
| N17 | Data pipeline | stub | 5 document nằm sẵn trong list Python `TOY_DOCS`, không DAG |
| N18 | Lakehouse | stub | dict trong bộ nhớ, không Delta, không Iceberg, không cả SQLite |
| N19 | Vector + features | stub | không embedding, không vector index; `retrieve()` chạy nhánh dự phòng keyword overlap |
| N20 | Serving | THẬT | `llama-server` b10488, Gemma 4 E2B UD-Q4_K_XL, 4 slot, `--cont-batching`, đã kiểm chứng bằng `/metrics` |

Hệ quả của N19 là stub: `embed = 0.0 ms` không có nghĩa là embedding nhanh vô cùng, nó có
nghĩa là không có embedding nào chạy. Server embedding ở port 8081 không được bật nên
`embed()` trả về rỗng và code rơi xuống nhánh so khớp từ khoá. Tương tự `retrieve = 0.0 ms`:
so khớp từ khoá trên 5 document thì đo bằng `perf_counter` cũng ra 0.0.

### Stage nào chiếm thời gian, có đúng dự đoán không

Đúng dự đoán: `llm` chiếm 100% (trung bình 6102 ms trên tổng 6102 ms). Với máy CPU-only
decode 10.9 tok/s và retrieval stub trên 5 document, không có kịch bản nào khác.

Nhưng "llm 100%" là câu trả lời quá thô vì bên trong stage đó có hai thứ rất khác nhau. Đo
riêng bằng field `timings` của llama-server, cho request có đúng hình dạng RAG (system prompt
+ 5 context + câu hỏi):

| | Token | Thời gian | Tỉ lệ |
|:--|--:|--:|--:|
| Prefill (đọc context) | 151 | 4339 ms | 72% |
| Decode (sinh câu trả lời) | 19 | 1691 ms | 28% |
| Tổng | | 6030 ms | 100% |

Ở bench trước, prompt ngắn thì prefill chỉ chiếm ~11% (TTFT 701 ms trên tổng 6227 ms). Thêm
context vào thì prefill đảo ngược tỉ lệ và thành phần tốn nhất. RAG không làm model chậm đi,
nó làm prompt dài ra, và prefill trả giá cho từng token của phần dài thêm (~28.7 ms/token).

### Nếu phải giảm một nửa latency thì tấn công stage nào

Prefill, bằng prompt caching. Đã đo:

| | Prefill | Tổng |
|:--|--:|--:|
| Lần 1, prompt mới | 151 token / 4339 ms | 6030 ms |
| Lần 2, prompt lặp lại | 147 token lấy từ cache, chỉ 5 token phải tính / 244 ms | 1830 ms |

Prefill giảm 94%, tổng latency giảm 3.3 lần, chỉ nhờ gửi lại đúng prompt cũ.

Lý do chọn knob này:

- Decode không còn gì để cắt. Thread sweep đã tìm đỉnh (10.9 tok/s ở `-t 4`), quantization
  nhỏ hơn không nhanh hơn, batching chỉ được 1.15x. Cả ba đều chỉ về cùng một trần là băng
  thông RAM.
- Prefill thì có thể bỏ hẳn. Cache KV của phần prompt lặp lại nghĩa là công việc đó không
  phải làm lần nữa, rẻ hơn mọi cách làm-nó-nhanh-hơn.
- RAG hưởng lợi nhiều nhất vì system prompt và các document lấy ra thường giống nhau giữa các
  câu hỏi, chỉ câu hỏi cuối là khác. Đúng khuôn mà prefix cache cần.

Việc nên làm tiếp: sắp prompt sao cho phần cố định luôn đứng trước phần thay đổi để cache
dùng được nhiều nhất; chỉ khi đó mới xét đổi model nhỏ hơn để hạ phần decode còn lại.
