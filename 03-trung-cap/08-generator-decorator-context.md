# Chương 08 — Iterator, Generator, Decorator, Context Manager

## 8.1 Iterable vs Iterator

- **Iterable**: object có `__iter__` trả iterator. Ví dụ: list, dict, str, file.
- **Iterator**: object có `__iter__` (trả chính nó) và `__next__` (trả phần tử hoặc ném `StopIteration`).

```python
ds = [1, 2, 3]
it = iter(ds)
print(next(it))   # 1
print(next(it))   # 2
print(next(it))   # 3
# next(it)   # StopIteration
```

Vòng lặp `for` thực chất gọi `iter()` rồi `next()` liên tục.

## 8.2 Tự viết iterator

```python
class Dem:
    def __init__(self, n):
        self.n = n
        self.i = 0

    def __iter__(self):
        return self

    def __next__(self):
        if self.i >= self.n:
            raise StopIteration
        self.i += 1
        return self.i - 1

for x in Dem(3):
    print(x)   # 0 1 2
```

## 8.3 Generator — cách gọn hơn

Generator là hàm có `yield`. Tự động cài đặt iterator protocol.

```python
def dem(n):
    i = 0
    while i < n:
        yield i
        i += 1

for x in dem(3):
    print(x)

g = dem(3)
print(next(g))   # 0
print(next(g))   # 1
```

### Trạng thái được giữ giữa các lần gọi

```python
def dem_vo_tan():
    i = 0
    while True:
        yield i
        i += 1

g = dem_vo_tan()
print([next(g) for _ in range(5)])   # [0, 1, 2, 3, 4]
```

### Lợi ích

Generator **lười** — chỉ tính khi cần. Tiết kiệm bộ nhớ.

```python
def doc_file(path):
    with open(path, encoding="utf-8") as f:
        for dong in f:
            yield dong.rstrip()

# Xử lý file 10GB không tốn 10GB RAM
for dong in doc_file("big.txt"):
    if "ERROR" in dong:
        print(dong)
```

### `yield from`

Ủy quyền cho generator/iterable khác.

```python
def gop(*iterables):
    for it in iterables:
        yield from it

print(list(gop([1, 2], [3, 4], "56")))   # [1, 2, 3, 4, '5', '6']
```

Tương đương:

```python
def gop(*iterables):
    for it in iterables:
        for x in it:
            yield x
```

### Gửi giá trị vào generator

```python
def accumulator():
    total = 0
    while True:
        x = yield total
        if x is None:
            break
        total += x

acc = accumulator()
next(acc)             # khởi động
print(acc.send(10))   # 10
print(acc.send(5))    # 15
acc.close()
```

### `throw` và `close`

```python
def gen():
    try:
        yield 1
        yield 2
    except GeneratorExit:
        print("đóng")
    finally:
        print("dọn dẹp")

g = gen()
print(next(g))
g.close()   # in "đóng" rồi "dọn dẹp"
```

### Return trong generator

```python
def gen():
    yield 1
    yield 2
    return "xong"

g = gen()
next(g); next(g)
try:
    next(g)
except StopIteration as e:
    print(e.value)   # "xong"
```

## 8.4 Generator expression

Như list comprehension nhưng dùng `()`.

```python
tong = sum(x*x for x in range(1_000_000))   # không tạo list trung gian

gen = (x*x for x in range(5))
print(next(gen))   # 0
```

Dùng `()` khi chỉ lặp một lần — tiết kiệm bộ nhớ đáng kể.

## 8.5 Closure

Hàm trong hàm giữ được biến của hàm ngoài.

```python
def tao_bo_dem():
    dem = 0
    def tang():
        nonlocal dem
        dem += 1
        return dem
    return tang

f = tao_bo_dem()
print(f(), f(), f())   # 1 2 3
```

### `__closure__`

```python
def ngoai(x):
    def trong():
        return x
    return trong

f = ngoai(10)
print(f.__closure__[0].cell_contents)   # 10
```

## 8.6 Decorator

Decorator là hàm nhận hàm, trả hàm mới. Cú pháp `@`.

```python
import functools

def dem_goi(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        wrapper.so_lan += 1
        return func(*args, **kwargs)
    wrapper.so_lan = 0
    return wrapper

@dem_goi
def chao(ten):
    return f"Chào {ten}"

chao("Nam"); chao("Lan")
print(chao.so_lan)   # 2
```

`functools.wraps` giữ `__name__`, `__doc__` của hàm gốc. Luôn dùng.

### Decorator không có `wraps` — hậu quả

```python
def bad_decorator(func):
    def wrapper():
        return func()
    return wrapper

@bad_decorator
def ham():
    """Docstring gốc"""

print(ham.__name__)   # 'wrapper' — mất tên gốc
print(ham.__doc__)    # None
```

### Decorator có tham số

Cần thêm một lớp hàm.

```python
def lap(lan):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            ket_qua = None
            for _ in range(lan):
                ket_qua = func(*args, **kwargs)
            return ket_qua
        return wrapper
    return decorator

@lap(3)
def hello():
    print("hi")
```

### Decorator đo thời gian

```python
import time
import functools

def do_thoi_gian(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        t0 = time.perf_counter()
        ket_qua = func(*args, **kwargs)
        print(f"{func.__name__}: {time.perf_counter() - t0:.4f}s")
        return ket_qua
    return wrapper

@do_thoi_gian
def tinh_tong(n):
    return sum(range(n))

tinh_tong(10_000_000)
```

### Decorator retry

```python
import time
import functools

def retry(max_lan=3, delay=1.0, ngoai_le=(Exception,)):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for lan in range(max_lan):
                try:
                    return func(*args, **kwargs)
                except ngoai_le as e:
                    if lan == max_lan - 1:
                        raise
                    time.sleep(delay * (2 ** lan))
        return wrapper
    return decorator

@retry(max_lan=3, delay=0.5)
def goi_api():
    ...
```

### Decorator cache

```python
def cache_dict(func):
    cache = {}
    @functools.wraps(func)
    def wrapper(*args):
        if args not in cache:
            cache[args] = func(*args)
        return cache[args]
    return wrapper
```

### Decorator xếp chồng

```python
@decorator_a
@decorator_b
def ham():
    ...
```

Tương đương `decorator_a(decorator_b(ham))`. Thứ tự từ dưới lên.

### Decorator chuẩn hay dùng

```python
from functools import lru_cache, cached_property, wraps

@lru_cache(maxsize=None)
def fib(n):
    return n if n < 2 else fib(n-1) + fib(n-2)

class Data:
    @cached_property
    def tinh_toan_nang(self):
        return sum(range(10**6))
```

`cached_property` (3.8+) chỉ tính một lần, lưu vào `__dict__` instance.

### `@staticmethod`, `@classmethod`, `@property`

Đã học ở chương 7 — đều là decorator.

### `@dataclass` cũng là decorator

```python
from dataclasses import dataclass

@dataclass
class Point:
    x: float
    y: float
```

## 8.7 Context manager

Quản lý tài nguyên: mở/đóng, khóa/mở, vào/ra trạng thái.

```python
class QuanLyFile:
    def __init__(self, path, mode):
        self.path = path
        self.mode = mode

    def __enter__(self):
        self.f = open(self.path, self.mode, encoding="utf-8")
        return self.f

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.f.close()
        return False   # không nuốt ngoại lệ

with QuanLyFile("a.txt", "w") as f:
    f.write("hi")
```

### `__exit__` nhận gì

- `exc_type`: class ngoại lệ (hoặc `None`).
- `exc_val`: instance ngoại lệ.
- `exc_tb`: traceback.

Trả `True` để nuốt ngoại lệ, `False` để lan truyền.

```python
class BoQua(Exception):
    pass

class NuotLoi:
    def __enter__(self): return self
    def __exit__(self, exc_type, exc_val, exc_tb):
        return exc_type is not None   # nuốt mọi ngoại lệ

with NuotLoi():
    raise ValueError("sẽ bị nuốt")
print("tiếp tục")
```

## 8.8 `contextlib`

### `@contextmanager`

Viết context manager bằng generator — gọn hơn.

```python
from contextlib import contextmanager
import time

@contextmanager
def timer(nhan):
    t0 = time.perf_counter()
    try:
        yield
    finally:
        print(f"{nhan}: {time.perf_counter() - t0:.4f}s")

with timer("khối"):
    sum(range(10**6))
```

Phần trước `yield` là `__enter__`, sau `yield` là `__exit__`.

### `suppress`

Nuốt ngoại lệ cụ thể.

```python
from contextlib import suppress

with suppress(FileNotFoundError):
    open("thieu.txt").read()
```

### `redirect_stdout`

```python
from contextlib import redirect_stdout
import io

f = io.StringIO()
with redirect_stdout(f):
    print("bị bắt")
print(f.getvalue())   # "bị bắt\n"
```

### `ExitStack`

Quản lý nhiều context manager động.

```python
from contextlib import ExitStack

with ExitStack() as stack:
    files = [stack.enter_context(open(f"f{i}.txt", "w")) for i in range(3)]
    for f in files:
        f.write("x")
```

## 8.9 Ví dụ tổng hợp

### Đọc file lớn theo pipeline

```python
from itertools import islice

def doc(path):
    with open(path, encoding="utf-8") as f:
        yield from (d.rstrip() for d in f)

def loc_trong(dong):
    return (d for d in dong if d)

def chuan_hoa(dong):
    return (d.lower() for d in dong)

pipeline = chuan_hoa(loc_trong(doc("data.txt")))
for d in islice(pipeline, 5):
    print(d)
```

### Retry + logging decorator

```python
import functools
import logging
import time

logger = logging.getLogger(__name__)

def retry_log(max_lan=3, delay=0.5):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for lan in range(1, max_lan + 1):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    logger.warning(f"Lần {lan} lỗi: {e}")
                    if lan == max_lan:
                        raise
                    time.sleep(delay * lan)
        return wrapper
    return decorator
```

### Context manager đo bộ nhớ

```python
from contextlib import contextmanager
import tracemalloc

@contextmanager
def do_bo_nho():
    tracemalloc.start()
    try:
        yield
    finally:
        current, peak = tracemalloc.get_traced_memory()
        tracemalloc.stop()
        print(f"Đỉnh: {peak / 1024 / 1024:.2f} MB")
```

## 8.10 Bài tập

### Bài 1 — Generator đọc file, lọc dòng trống
```python
def doc_khong_trong(path):
    with open(path, encoding="utf-8") as f:
        for dong in f:
            if dong.strip():
                yield dong.rstrip()
```

### Bài 2 — Fibonacci vô hạn, lấy 20 số
```python
from itertools import islice

def fib():
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b

print(list(islice(fib(), 20)))
```

### Bài 3 — Decorator retry
Đã viết ở trên. Test với hàm ngẫu nhiên lỗi.

### Bài 4 — Decorator cache
Đã viết ở trên.

### Bài 5 — Context manager đổi thư mục
```python
import os
from contextlib import contextmanager

@contextmanager
def trong_thu_muc(path):
    cu = os.getcwd()
    os.chdir(path)
    try:
        yield
    finally:
        os.chdir(cu)
```

## 8.11 Ghi chú kỹ thuật

- Generator có overhead so với vòng lặp thường nhưng tiết kiệm bộ nhớ khi dữ liệu lớn.
- `lru_cache` không thread-safe trước 3.9; từ 3.9 có khóa nội bộ.
- `functools.wraps` copy `__wrapped__` — có thể unwrap bằng `inspect.unwrap`.
- Context manager dùng `with` đảm bảo cleanup kể cả khi `return` hoặc exception.
- `yield` biến hàm thành generator — không dùng `return` giá trị ngoài `StopIteration.value`.

Stashed.