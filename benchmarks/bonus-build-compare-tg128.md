# Bonus B1 - Prebuilt vs source build

Host `Linux-x86_64` · CPU `Intel(R) Core(TM) i3-1005G1 CPU @ 1.20GHz`
Vector extensions detected: AVX-512, AVX2
llama.cpp `b10488` both sides · `threads=2` ·
**both pinned to `ngl=0`** so this isolates the compiler ·
metric `tg128`, 3 repetitions

| Binary | Built for | tg128 (tok/s) | Relative |
|:--|--:|--:|--:|
| prebuilt release | runtime CPU dispatch | 5.7 | 1.00x |
| your source build | this CPU (`-DGGML_NATIVE=ON`) | 5.8 | 1.02x |

On this machine, **they are within 3% -- no meaningful difference**.

before: 5.7 tok/s (prebuilt release)
after:  5.8 tok/s (source build, -DGGML_NATIVE=ON)
speedup: 1.02x

Same source revision, same model, same backend, same `-ngl` -- the only difference
is what the compiler was allowed to assume about the CPU.
A gap this small usually means the prebuilt binary already dispatches to the right kernels at runtime (releases ship one libggml-cpu-*.so per microarchitecture and pick via CPUID), or that this workload is bandwidth-bound rather than instruction-bound. Both are real findings -- say which one you think it is.


## Your explanation

Mức chênh lệch chỉ là 1.02x (5.7 tok/s vs 5.8 tok/s), gần như không có sự khác biệt đáng kể. Nguyên nhân chính là do bản prebuilt của llama.cpp đã hỗ trợ cơ chế CPUID dispatch tự động tại runtime, giúp tự động chọn đúng thư viện kernel tối ưu (AVX2 / AVX-512) cho CPU Intel i3-1005G1. Thêm vào đó, quá trình decode của model trên CPU này bị giới hạn bởi băng thông bộ nhớ RAM (memory bandwidth bound) hơn là giới hạn số lượng câu lệnh tính toán (instruction bound), khiến việc biên dịch lại với cờ `-DGGML_NATIVE=ON` không tạo ra sự cải thiện vượt trội nào.
