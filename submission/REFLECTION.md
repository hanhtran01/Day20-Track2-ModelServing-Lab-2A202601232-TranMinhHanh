# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Trần Minh Hạnh
**Cohort:** A20-K3B
**Ngày submit:** 2026-08-20

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 10 Pro 22H2 (build 19045) + WSL2 Ubuntu 26.04 LTS
- **CPU:** Intel Core i7-7820HQ @ 2.90 GHz
- **Cores:** 4 physical / 8 logical
- **CPU extensions:** AVX2 (không có AVX-512)
- **RAM:** 15.5 GB
- **Accelerator:** CPU only, `ngl=0` (máy có NVIDIA GeForce 940MX 2 GB nhưng không dùng được)
- **llama.cpp asset đã tải:** `llama-b10488-bin-ubuntu-vulkan-x64.tar.gz`
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=`gemma4-e2b)
- **Quantization:** UD-Q4_K_XL + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi, trong WSL2 Ubuntu, không dùng Colab/Kaggle
_(Nếu dùng cloud fallback: nói rõ vì sao — RAM < 8 GB, setup fail, v.v. Không mất điểm.)_

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

Lab chạy trong WSL2 chứ không phải PowerShell. Một bước fail thật: `llama-server` báo
`libgomp.so.1: cannot open shared object file`, phải `sudo apt install libgomp1` mới chạy
được, repo không nhắc gói này. Ngoài ra không đổi gì, RAM 15.5 GB đủ cho model mặc định.
GPU 940MX không dùng được vì asset là bản Vulkan và WSL thiếu Vulkan ICD nên `ngl=0`. Repo
nằm trên `/mnt/d` nên I/O chậm, load model mất ~29 s.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 28828 | 701 / 815 | 93.0 / 108.4 | 6227 / 7643 / 7643 | 10.8 |
| UD-Q2_K_XL | 2.24 | 21763 | 1156 / 1434 | 91.8 / 101.1 | 6941 / 7448 / 7448 | 10.9 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

2-bit không đáng. Nhỏ hơn 0.73 GB nhưng TTFT P50 chậm hơn 65% (701 -> 1156 ms), TPOT gần như
y nhau (93.0 -> 91.8 ms). Đã hỏi cùng 2 câu trên cả hai: bản 4-bit dựng đúng timeline hàng
đợi và định nghĩa TTFT/TPOT chính xác; bản 2-bit trả lời 4 token rồi dừng, và định nghĩa TPOT
sai thành end-to-end. Chọn 4-bit.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.27 | 25000 | 54000 | 54000 | 7.6 | 0.0% |
| 50 | 0.21 | 24000 | 53000 | 53000 | 5.7 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** _0.77×_
- **P95 tăng:** _0.98×_
- **Effective concurrency ở 50 users:** _5.7_ so với `--parallel` = _4_ slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): _3.77_ / _4_ slots

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

Bão hoà dưới 10 user. Bằng chứng mạnh nhất: tải tăng 5x mà RPS giảm còn 0.77x (0.27 ->
0.21). Thêm hai dấu hiệu: effective concurrency đã 7.6 so với 4 slot ngay ở mức 10 user, và
`requests_deferred = 46` không vơi suốt 60 s. Phần latency thêm là queue time: thời gian
tính mỗi request vẫn như lúc chạy một mình (P50 6.2 s), 19 s cộng thêm là chờ slot. Knob đổi
trước: model nhỏ hơn, vì mọi bằng chứng đều chỉ về băng thông RAM còn tăng `--parallel` chỉ
chia 12.5 tok/s cho nhiều người hơn.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | localhost trong WSL2, không k8s/Compose | stub |
| N17 Data pipeline | 5 document trong list Python, không DAG | stub |
| N18 Lakehouse | dict trong RAM, không Delta/Iceberg/SQLite | stub |
| N19 Vector + features | keyword overlap, không embedding, không vector index | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: _0.0 ms_ (bằng 0 vì bước này không chạy, không phải vì nó nhanh: N19 stub)
- retrieve: _0.0 ms_ (so khớp từ khoá trên 5 document)
- llm: _6102.2 ms_
- **stage chiếm nhiều nhất:** _llm_ (_100%_ của total)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

Bottleneck ở `llm` (100%), đúng kỳ vọng vì máy CPU-only và kho chỉ 5 doc. Nhưng bóc bên
trong `llm`: prefill 4339 ms (72%) so với decode 1691 ms (28%), tức RAG phồng prefill chứ
không làm model chậm. Muốn giảm 2x thì tấn công prefill bằng prompt caching: đo được prefill
giảm 94%, tổng 6030 -> 1830 ms (3.3x).

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** hạ `-t` từ 8 (số luồng logic máy báo) xuống 4 (số core vật lý thật)

```
before:   6.1 tok/s   (-t 8, tg128)
after:   10.9 tok/s   (-t 4, tg128)
speedup: 1.79×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Máy báo 8 CPU nên phản xạ đầu tiên là đặt `-t 8`, và nhiều tool cũng mặc định lấy số luồng
logic. Nhưng 8 đó là 4 core vật lý nhân hyperthreading: hai luồng logic của cùng một core
dùng chung khối tính vector và chung L1/L2 cache. Với decode, công việc đã đọc RAM liên tục
và đã dùng hết đơn vị SIMD, hyperthread không thêm năng lực tính nào, chỉ thêm hai luồng
tranh nhau cùng một cache. Thêm nữa llama.cpp chia mỗi phép tính trong layer cho các thread
rồi đồng bộ toàn bộ thread ở cuối mỗi bước, nên luồng chậm nhất quyết định nhịp cả nhóm và
phần chờ đó lặp lại rất nhiều lần cho mỗi token. Khi 2 luồng chia nhau 1 core thì luôn có
luồng bị chậm, đó là mức tụt 44% quan sát được. Ở `-t 16` còn tệ hơn (2.6 tok/s) vì OS phải
liên tục đổi ngữ cảnh và WSL2 vẫn chia CPU với Windows.

Phần khác với kỳ vọng từ deck, và đáng nói hơn cả con số 1.79x: đường cong scaling trước
knee cũng không tuyến tính. Từ 1 lên 4 luồng, số luồng gấp 4 nhưng tốc độ chỉ 2.17x (5.0 ->
10.9 tok/s, hiệu suất ~54%). Nếu decode thiếu năng lực tính thì phải gần 4x. Nó không gần 4x
vì mỗi token phải quét gần hết 2.97 GB weight từ RAM, và băng thông RAM là tài nguyên dùng
chung nên thêm core không nới rộng đường truyền. Hai kết quả khác xác nhận cùng một trần:
bản 2-bit nhỏ hơn 25% mà decode y nguyên (91.8 so với 93.0 ms/token, vì cả hai model đều đã
nằm trong 15.5 GB RAM), và continuous batching 4 slot chỉ đưa throughput từ 10.9 lên 12.5
tok/s (1.15x, dù batch width đạt 3.77/4). Kết luận: trên máy này mọi knob về song song hoá
đã hết đường, knob duy nhất còn tác dụng là giảm số byte phải đọc cho mỗi token, tức đổi
model nhỏ hơn hoặc cắt số token sinh ra.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** B1 `build-llama` + `compare-builds` · B2 `sweep-ctx` · B4 challenge C2 (KV cache
quantization) · B5 challenge C9 (embedding serving)

**Numbers:**

```
before:  11.4 tok/s  (prebuilt release, runtime CPU dispatch)
after:   12.2 tok/s  (source build, -DGGML_NATIVE=ON)
speedup: 1.06x
```

Cả hai đều Release, cùng revision `b10488`, cùng model, cùng `ngl=0`, cùng `-t 4`, đo trong
một lần chạy nên chỉ khác nhau ở compiler. Chi tiết: `benchmarks/bonus-build-compare-tg128.md`.

**Điều này nói lên gì mà deck chưa nói:**

Bốn thí nghiệm bonus, cộng ba thí nghiệm base, cho cùng một hình mẫu: mọi knob thêm năng lực
tính toán đều chỉ đổi được khoảng 1.1x, còn knob duy nhất bỏ bớt việc phải làm thì đổi được
3.3x.

| Knob | Bản chất | Kết quả trên máy này |
|:--|:--|--:|
| Compile cho đúng CPU (B1) | sinh mã tốt hơn | 1.06x |
| Static batch cho embedding (B5/C9) | gom nhiều việc vào một lần | 1.11x |
| Continuous batching 4 slot (base) | chia sẻ một lần đọc weight | 1.15x |
| Thread 1 -> 4 (base) | thêm 4x năng lực tính | 2.17x |
| KV cache q8_0 (B4/C2) | giảm 1.1% RAM | 0.95x, tức lỗ |
| Prompt caching (base, integrate) | không tính lại phần đã tính | 3.3x |

Deck trình bày các knob này như những cải tiến chung, và trên GPU thì đúng: batching gần
tuyến tính, FP8 KV cache cứu được nhiều GB, kernel khớp silicon tạo khác biệt lớn. Điều deck
không nói là tất cả những con số đó đều giả định phần cứng còn thừa năng lực tính và KV cache
lớn so với weight. Bỏ hai giả định đó đi, như trên laptop 4 core CPU-only, thì cả nhóm knob
song song hoá xẹp về 1.1x và knob KV cache thành lỗ ròng.

Hai kết quả cụ thể đáng chú ý nhất, cả hai đều trái kỳ vọng:

- KV cache quantization tiết kiệm 43.8 MB trên tổng 4049 MB, vì KV cache chỉ chiếm 2.7%
  footprint ở model 2B với ctx 8192. Nén một phần chiếm 2.7% xuống 0 vẫn không cứu được gì.
  Bài học tổng quát: đo tỉ trọng của thành phần trước khi tối ưu nó.
- `-fa on` (flash attention) làm chậm thêm 12% prefill trên CPU, vì nó là kernel thiết kế để
  tiết kiệm băng thông HBM của GPU, mà CPU không có phân tầng bộ nhớ đó. Một tối ưu đúng ở
  nơi khác có thể là lỗ ở đây.

Và B2 cho con số để nhớ khi thiết kế RAG: prefill 8192 token mất 277 giây trên máy này, còn
prefill vượt decode ngay từ khoảng 220 token prompt. Context không hề miễn phí chỉ vì context
window cho phép.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Continuous batching đạt 3.77/4 slot (94%), hoạt động gần như hoàn hảo, nhưng throughput chỉ
tăng 1.15x (10.9 -> 12.5 tok/s) còn mỗi người dùng nhận về 3.1 tok/s. Batching không xoá
được sự chậm, nó chia đều sự chậm cho nhiều người.

Ngạc nhiên thứ hai: bản 2-bit chậm hơn bản 4-bit ở TTFT (1156 so với 701 ms). Trước khi đo
thì tưởng file nhỏ hơn luôn nhanh hơn.

---

## 8. Self-check trước khi push

- [x] `hardware.json` committed
- [x] `models/active.json` committed
- [x] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [x] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [x] `benchmarks/02-server-results.md` committed (`make load-report`)
- [x] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [x] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [x] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [x] 5 screenshots trong `submission/screenshots/`
- [x] `make verify` → **exit 0**
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [x] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
