# Bonus B1 - Prebuilt vs source build

Host `Linux-x86_64` · CPU `Intel(R) Core(TM) i7-7820HQ CPU @ 2.90GHz`
Vector extensions detected: AVX2
llama.cpp `b10488` both sides · `threads=4` ·
**both pinned to `ngl=0`** so this isolates the compiler ·
metric `tg128`, 3 repetitions

| Binary | Built for | tg128 (tok/s) | Relative |
|:--|--:|--:|--:|
| prebuilt release | runtime CPU dispatch | 11.4 | 1.00x |
| your source build | this CPU (`-DGGML_NATIVE=ON`) | 12.2 | 1.06x |

On this machine, the source build is **1.06x faster**.

before: 11.4 tok/s (prebuilt release)
after:  12.2 tok/s (source build, -DGGML_NATIVE=ON)
speedup: 1.06x

Same source revision, same model, same backend, same `-ngl` -- the only difference
is what the compiler was allowed to assume about the CPU.



## Giải thích

### Vì sao chỉ được 1.06x

CPU này là i7-7820HQ (Kaby Lake), có AVX2 và FMA nhưng không có AVX-512, không có AMX. Bản
prebuilt không phải một binary chung: nó ship kèm nhiều thư viện CPU rồi chọn lúc chạy. Trong
`runtime/b10488/` có `libggml-cpu-haswell.so`, `-skylakex`, `-icelake`, `-sandybridge`,
`-sse42`, `-x64` và vài bản khác. Kaby Lake dùng cùng tập lệnh với Haswell ở mức AVX2/FMA,
nên prebuilt đã nạp đúng kernel AVX2 ngay từ đầu.

Vậy `-DGGML_NATIVE=ON` thêm được gì? Không thêm tập lệnh mới, vì không còn tập lệnh nào để
bật. Nó chỉ cho compiler biết chính xác model CPU để tinh chỉnh: chọn thứ tự lệnh, unroll,
inline, chi phí cache theo đúng Kaby Lake thay vì theo một baseline Haswell chung. Đó là 6%.

Kết luận trực tiếp: 1.06x là mức đúng như kỳ vọng khi bản prebuilt đã dispatch tới kernel
khớp tập lệnh. Nếu CPU có AVX-512 mà prebuilt lại rơi xuống kernel AVX2, khoảng cách sẽ lớn
hơn nhiều.

### Vì sao compiler không thể thắng đậm ở workload này

`tg128` là decode, và decode trên máy này bị chặn bởi băng thông RAM chứ không bởi tập lệnh.
Ba phép đo độc lập trong lab đều nói cùng một chuyện:

- Thread sweep: 4x số luồng chỉ được 2.17x tốc độ (5.0 -> 10.9 tok/s)
- Quantization: file nhỏ hơn 25% mà decode không nhanh hơn (91.8 so với 93.0 ms/token)
- Continuous batching: 4 slot chỉ được 1.15x throughput

Khi CPU phần lớn thời gian đang chờ weight từ RAM về, sinh mã tốt hơn chỉ làm phần "chờ ít
hơn một chút" nhanh hơn, không rút ngắn được phần chờ. 6% có lẽ chính là phần compute thực
sự bị cắt bớt trong tổng thời gian. Đây là suy luận từ các số liệu trên, chưa dùng profiler
đếm cache miss để xác nhận.

### Lưu ý về cách đọc con số này

Prebuilt ở phép so này cho 11.4 tok/s, trong khi thread sweep trước đó cho 10.9 tok/s cùng
`-t 4`. Chênh lệch đến từ số lần lặp và trạng thái máy lúc đo, không phải từ binary. Vì vậy
1.06x chỉ đáng tin khi hai bên được đo trong cùng một lần chạy như script này làm, và không
nên so số của bảng này với số của bảng khác.

Cả hai bên đều là Release và đều `ngl=0`, nên khoảng cách này thuần là compiler, không lẫn
Debug với Release và không lẫn CPU với GPU.
