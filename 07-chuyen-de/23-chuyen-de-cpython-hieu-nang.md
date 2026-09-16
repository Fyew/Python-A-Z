# Chuyên Đề 23 — CPython Hiệu Năng và Free-Threading

## 23.1 Bức tranh hiệu năng

CPython không nhanh bằng C, nhưng "chậm" là mô tả vô nghĩa nếu không có số.

Quy tắc bậc độ lớn (máy desktop hiện đại):

| Thao tác | Thời gian điển hình |
|----------|---------------------|
| Gọi hàm Python | ~50-100 ns |
| Attribute lookup | ~20-40 ns |
| Vòng lặp `for` một phần tử | ~30-50 ns |
| `dict.get` | ~30-60 ns |
| `list.append` | ~30 ns |
| Cộng hai int | ~20-40 ns |
| `sum(range(10**6))` | ~10-20 ms |
| Khởi động interpreter | ~20-50 ms |

Muốn nhanh: giảm số thao tác Python, để C làm việc nặng.

## 23.2 Đo cho đúng

### `timeit` — vi mô

```python
import timeit

print(timeit.timeit("sum(range(1000))", number=10_000))
print(timeit.repeat("sum(range(1000))", number=10_000, repeat=5))
```

`repeat` trả list — lấy `min` để giảm nhiễu.

### `pyperf` — chính xác hơn

```bash
pip install pyperf
pyperf timeit -s "import math" "math.sqrt(2)"
```

Tự động ổn định CPU, chạy nhiều vòng, báo cáo độ tin cậy.

### `cProfile`

```bash
python -m cProfile -s cumtime script.py | head -30
```

Cột quan trọng:

- `tottime`: thời gian trong chính hàm đó.
- `cumtime`: bao gồm hàm con.
- `ncalls`: số lần gọi.

Tối ưu hàm có `tottime` cao, không phải `cumtime` cao.

### `line_profiler`

```bash
pip install line_profiler
kernprof -l -v script.py
```

```python
@profile
def ham():
    ...
```

Chỉ ra từng dòng.

### `py-spy` — profiler không xâm lấn

```bash
pip install py-spy
py-spy top --pid 12345
py-spy record -o profile.svg --pid 12345
```

Không cần sửa code, chạy trên process đang sống.

## 23.3 Specializing adaptive interpreter (3.11+)

Python 3.11 thêm cơ chế bytecode tự chuyên biệt hóa.

```python
import dis

def cong(a, b):
    return a + b

dis.dis(cong, adaptive=True)
```

Sau vài lần chạy, `BINARY_OP` generic được thay bằng `BINARY_OP_ADD_INT` — nhanh hơn vì bỏ kiểm tra kiểu.

Hệ quả thực tế: vòng lặp "nóng" nhanh hơn ~10-60% so với 3.10. Không cần làm gì.

### `sys.monitoring` (3.12+)

API mới thay `sys.settrace`, overhead thấp hơn nhiều.

```python
import sys

mon = sys.monitoring
TOOL_ID = 2

def on_call(code, instruction_offset):
    print(f"gọi {code.co_name}")

mon.use_tool_id(TOOL_ID, "demo")
mon.register_callback(TOOL_ID, mon.events.PY_START, on_call)
mon.set_events(TOOL_ID, mon.events.PY_START)
```

Debugger và profiler hiện đại dùng API này.

## 23.4 GIL và free-threading

CPython 3.13 có bản **free-threaded** (không GIL), gọi là `python3.13t`.

```bash
python3.13t -c "import sys; print(sys._is_gil_enabled())"
```

Trạng thái:

- Chưa production-ready cho mọi workload.
- Nhiều C extension chưa tương thích.
- Hiệu năng đơn luồng có thể chậm hơn 10-40%.
- Đa luồng CPU-bound nhanh hơn thật trên nhiều core.

### Kiểm tra trong code

```python
import sys

def co_gil() -> bool:
    fn = getattr(sys, "_is_gil_enabled", None)
    return fn() if fn else True
```

### Khi nào quan tâm

- Workload CPU-bound nặng, nhiều core, thuần Python.
- Không phụ thuộc C extension chưa hỗ trợ.
- Sẵn sàng chấp nhận rủi ro.

Với phần lớn ứng dụng, chọn `multiprocessing` vẫn an toàn hơn.

## 23.5 Subinterpreters

Mỗi subinterpreter có GIL riêng (trước 3.13) — cách cô lập tốt hơn process về chi phí.

```python
# 3.12+: _xxsubinterpreters, API đang hoàn thiện
import _xxsubinterpreters as sub

interp = sub.create()
sub.run_string(interp, "print('xin chào từ subinterpreter')")
sub.destroy(interp)
```

3.13 mở rộng API. Đây là hướng thay thế multiprocessing cho tác vụ CPU-bound thuần Python.

## 23.6 Tối ưu vi mô có ích

### Giảm attribute lookup

```python
# Chậm
def ham(ds):
    ket_qua = []
    for x in ds:
        ket_qua.append(x.upper())
    return ket_qua

# Nhanh hơn — cache method
def ham(ds):
    ket_qua = []
    append = ket_qua.append
    for x in ds:
        append(x.upper())
    return ket_qua
```

Lợi ích nhỏ nhưng đo được trong vòng lặp hàng triệu lần.

### Tránh tạo object thừa

```python
# Tạo tuple mỗi lần lặp
for i in range(1000):
    if (i, i) in cache:
        ...

# Tốt hơn nếu chỉ cần một giá trị
if i in cache:
    ...
```

### Dùng comprehension thay `append`

```python
ket_qua = [f(x) for x in ds]
```

### Local variable nhanh hơn global

```python
# Global lookup chậm hơn local
def ham():
    for i in range(1_000_000):
        print(i)     # print là global/builtin lookup

def ham_nhanh():
    p = print
    for i in range(1_000_000):
        p(i)
```

### `__slots__` và `frozen=True`

```python
from dataclasses import dataclass

@dataclass(slots=True, frozen=True)
class Diem:
    x: float
    y: float
```

Giảm bộ nhớ ~40-50%, tăng tốc truy cập thuộc tính.

### `array` thay `list` cho số

```python
from array import array

a = array("i", range(1_000_000))   # 4 byte/phần tử, list ~28 byte
```

### `memoryview` cho slice không copy

```python
data = b"\x00" * 1_000_000
view = memoryview(data)
phan = view[100:200]     # không copy
```

### Nối byte/chuỗi

```python
# Nhanh
b"".join(danh_sach_bytes)
"".join(danh_sach_chuoi)

# Chậm O(n^2)
ket_qua = b""
for x in danh_sach_bytes:
    ket_qua += x
```

## 23.7 Khi nào rời khỏi Python

Nếu profiler chỉ ra hot loop thuần số:

1. **NumPy** — nếu dữ liệu là mảng số.
2. **Cython** — viết lại hàm nóng, giữ Python.
3. **C extension** — kiểm soát tối đa.
4. **Rust + PyO3** — an toàn bộ nhớ, hiệu năng C.
5. **Numba** — JIT cho hàm số, chỉ thêm decorator.

```python
from numba import njit

@njit
def tinh(n: int) -> float:
    s = 0.0
    for i in range(n):
        s += i ** 0.5
    return s
```

Lần gọi đầu chậm (biên dịch), các lần sau nhanh gấp hàng chục lần.

### So sánh thực tế

```python
import time
import numpy as np
from numba import njit

N = 10_000_000

def python_thuan(n):
    s = 0.0
    for i in range(n):
        s += i * i
    return s

@njit
def numba_version(n):
    s = 0.0
    for i in range(n):
        s += i * i
    return s

def numpy_version(n):
    a = np.arange(n, dtype=np.float64)
    return (a * a).sum()
```

Thứ tự điển hình: Python thuần > NumPy > Numba (cho vòng lặp có state).

## 23.8 Đo bộ nhớ

```python
import tracemalloc

tracemalloc.start()
# code
snap = tracemalloc.take_snapshot()
for stat in snap.statistics("lineno")[:10]:
    print(stat)
```

### `sys.getsizeof` và đo sâu

```python
import sys

sys.getsizeof([1, 2, 3])          # chỉ list object, không tính phần tử
sys.getsizeof([]) + 3 * sys.getsizeof(1)   # gần đúng
```

### `pympler` cho tổng bộ nhớ

```bash
pip install pympler
```

```python
from pympler import asizeof
asizeof.asizeof([1, 2, [3, 4]])
```

### Rò rỉ bộ nhớ

```python
import gc
import objgraph

objgraph.show_growth(limit=10)
gc.collect()
objgraph.show_most_common_types(limit=10)
```

Rò rỉ phổ biến: cache không giới hạn, closure giữ tham chiếu, callback chưa hủy, exception giữ traceback.

## 23.9 Quy trình tối ưu

1. **Đo** — profiler, không đoán.
2. **Xác định bottleneck** — thường 3% code chiếm 97% thời gian.
3. **Tối ưu thuật toán trước** — O(n²) → O(n log n) thắng mọi micro-opt.
4. **Tối ưu cấu trúc dữ liệu** — set/dict thay list khi cần tra cứu.
5. **Giảm công việc Python** — đẩy sang C/NumPy.
6. **Đo lại** — xác nhận cải thiện thật.
7. **Dừng khi đủ nhanh** — không tối ưu mù quáng.

## 23.10 Bài tập

1. Dùng `timeit` so sánh `+=` chuỗi vs `join` với 100.000 phần tử.
2. Profile một script, tìm hàm tốn `tottime` cao nhất, tối ưu.
3. Dùng `py-spy` record một process, xem flame graph.
4. So sánh `list`, `array`, `numpy` cho 1 triệu số: bộ nhớ và tốc độ.
5. Viết hàm tính tổng bình phương bằng Python thuần, NumPy, Numba — đo cả ba.
6. Dùng `tracemalloc` tìm dòng cấp phát nhiều bộ nhớ nhất.
7. Kiểm tra `sys._is_gil_enabled()` trên Python 3.13, ghi lại kết quả.

## 23.11 Ghi chú kỹ thuật

- `cProfile` thêm overhead 2-3x — chỉ dùng để tìm bottleneck.
- `timeit` cần `number` đủ lớn để nhiễu không lấn kết quả.
- Numba JIT lần đầu chậm; warm-up trước khi đo.
- Free-threading chưa ổn định — kiểm tra C extension trước khi dùng.
- `gc.disable()` có thể tăng tốc nhưng rủi ro rò rỉ chu trình.
- Đừng tối ưu trước khi có số đo. Đừng tối ưu sau khi đủ nhanh.
- Luôn tối ưu trên code dễ đọc; nếu tối ưu làm code khó hiểu, ghi comment giải thích và benchmark.

Stashed.