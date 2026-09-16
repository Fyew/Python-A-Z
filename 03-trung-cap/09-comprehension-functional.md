# Chương 09 — Comprehension và Lập Trình Hàm

## 9.1 List comprehension

Cú pháp: `[biểu_thức for item in iterable if điều_kiện]`

```python
binh_phuong = [x * x for x in range(10)]
# [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]

chan = [x for x in range(20) if x % 2 == 0]
# [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]

cap = [(x, y) for x in range(3) for y in range(3)]
# [(0,0), (0,1), (0,2), (1,0), ...]
```

Tương đương vòng lặp:

```python
binh_phuong = []
for x in range(10):
    binh_phuong.append(x * x)
```

Comprehension nhanh hơn append trong vòng lặp khoảng 30-40% vì CPython tối ưu LIST_APPEND.

### Nhiều điều kiện

```python
ds = [x for x in range(100) if x % 2 == 0 if x % 3 == 0]
# tương đương if x % 2 == 0 and x % 3 == 0
```

### if-else trong biểu thức

```python
loai = ["chẵn" if x % 2 == 0 else "lẻ" for x in range(5)]
# ['chẵn', 'lẻ', 'chẵn', 'lẻ', 'chẵn']
```

Phân biệt: `if` trước `for` = lọc; `if-else` trước `for` = biến đổi.

### Comprehension lồng

```python
matrix = [[1, 2, 3], [4, 5, 6]]
phang = [x for hang in matrix for x in hang]
# [1, 2, 3, 4, 5, 6]

chuyen_vi = [[hang[i] for hang in matrix] for i in range(3)]
# [[1, 4], [2, 5], [3, 6]]
```

Đọc từ trái sang phải theo thứ tự vòng lặp.

## 9.2 Set comprehension

```python
tap = {x % 3 for x in range(10)}
# {0, 1, 2}

tu_dai = {w for w in "hello world".split() if len(w) > 4}
```

Dùng khi cần khử trùng lặp ngay khi tạo.

## 9.3 Dict comprehension

```python
bp = {x: x * x for x in range(5)}
# {0: 0, 1: 1, 2: 4, 3: 9, 4: 16}

dao = {v: k for k, v in {"a": 1, "b": 2}.items()}
# {1: 'a', 2: 'b'}

loc = {k: v for k, v in d.items() if v > 0}
```

### Gộp hai list

```python
ten = ["An", "Bình", "Cường"]
diem = [8, 9, 7]
bang_diem = {t: d for t, d in zip(ten, diem)}
# {'An': 8, 'Bình': 9, 'Cường': 7}
```

## 9.4 Generator expression

Dùng `()` thay `[]`. Không tạo list — sinh giá trị lười.

```python
tong = sum(x * x for x in range(1_000_000))   # không tạo list trung gian

gen = (x * x for x in range(5))
print(next(gen))   # 0
print(next(gen))   # 1
```

### So sánh bộ nhớ

```python
import sys

list_comp = [x for x in range(1_000_000)]
gen_exp = (x for x in range(1_000_000))

print(sys.getsizeof(list_comp))   # ~8MB
print(sys.getsizeof(gen_exp))     # ~200 byte
```

### Khi nào dùng generator

- Dữ liệu lớn, chỉ lặp một lần.
- Pipeline xử lý.
- Truyền vào `sum`, `max`, `min`, `any`, `all`.

```python
any(x > 100 for x in ds)    # dừng ngay khi tìm thấy
max(len(w) for w in words)
```

## 9.5 Lambda

Hàm ẩn danh một biểu thức.

```python
cong = lambda a, b: a + b
print(cong(2, 3))   # 5
```

Tương đương:

```python
def cong(a, b):
    return a + b
```

### Dùng với sort/key

```python
ds = [(1, "b"), (3, "a"), (2, "c")]
ds.sort(key=lambda t: t[1])

sinh_vien = [{"ten": "An", "diem": 8}, {"ten": "Bình", "diem": 9}]
sinh_vien.sort(key=lambda sv: -sv["diem"])
```

### Không lạm dụng lambda

Lambda khó debug (không có tên trong traceback), chỉ nên cho biểu thức ngắn.

```python
# XẤU
f = lambda x: x if x > 0 else -x if x < 0 else 0

# TỐT
def gia_tri_tuyet_doi(x):
    if x > 0:
        return x
    if x < 0:
        return -x
    return 0
```

## 9.6 map, filter, reduce

### map

```python
ds = [1, 2, 3, 4]
print(list(map(lambda x: x * 2, ds)))     # [2, 4, 6, 8]
print(list(map(str, ds)))                  # ['1', '2', '3', '4']
print(list(map(lambda a, b: a + b, [1, 2], [10, 20])))   # [11, 22]
```

### filter

```python
print(list(filter(lambda x: x % 2 == 0, ds)))   # [2, 4]
print(list(filter(None, [0, 1, "", "a", None])))  # [1, 'a']
```

`filter(None, ...)` lọc bỏ giá trị falsy.

### reduce

```python
from functools import reduce

print(reduce(lambda a, b: a + b, ds))          # 10
print(reduce(lambda a, b: a + b, ds, 100))     # 110
print(reduce(lambda a, b: a * b, ds))          # 24
```

### Comprehension thay map/filter

Trong Python hiện đại, comprehension thường được ưu tiên vì dễ đọc hơn.

```python
# map/filter
list(map(lambda x: x*2, filter(lambda x: x%2==0, ds)))

# comprehension — rõ hơn
[x*2 for x in ds if x % 2 == 0]
```

Giữ `map`/`filter` khi dùng hàm có sẵn: `list(map(int, strs))`.

## 9.7 functools

### partial — cố định tham số

```python
from functools import partial

def luy_thua(co_so, so_mu):
    return co_so ** so_mu

binh_phuong = partial(luy_thua, so_mu=2)
print(binh_phuong(5))    # 25

log = partial(print, "LOG:", sep=" | ")
log("khởi động")          # LOG: | khởi động
```

### lru_cache

```python
from functools import lru_cache

@lru_cache(maxsize=128)
def fib(n):
    return n if n < 2 else fib(n-1) + fib(n-2)

fib.cache_info()    # CacheInfo(hits=..., misses=..., maxsize=128, currsize=...)
fib.cache_clear()
```

Yêu cầu tham số hashable. `maxsize=None` là không giới hạn.

### wraps

Đã học ở chương 8. Copy metadata khi viết decorator.

### singledispatch — đa hình theo kiểu

```python
from functools import singledispatch

@singledispatch
def xu_ly(x):
    return f"mặc định: {x}"

@xu_ly.register
def _(x: int):
    return f"số nguyên: {x}"

@xu_ly.register
def _(x: list):
    return f"list {len(x)} phần tử"

print(xu_ly(5))
print(xu_ly([1, 2]))
print(xu_ly("abc"))
```

## 9.8 itertools

### Vô hạn

```python
import itertools as it

list(it.islice(it.count(10, 2), 5))     # [10, 12, 14, 16, 18]
list(it.islice(it.cycle("AB"), 5))      # ['A', 'B', 'A', 'B', 'A']
list(it.repeat("x", 3))                  # ['x', 'x', 'x']
```

Luôn dùng `islice` với iterator vô hạn.

### Kết hợp

```python
list(it.chain([1, 2], [3, 4]))          # [1, 2, 3, 4]
list(it.chain.from_iterable([[1,2],[3,4]]))   # [1, 2, 3, 4]

list(it.combinations([1, 2, 3], 2))     # [(1,2),(1,3),(2,3)]
list(it.combinations_with_replacement([1, 2], 2))   # [(1,1),(1,2),(2,2)]
list(it.permutations([1, 2], 2))        # [(1,2),(2,1)]
list(it.product([1, 2], "ab"))          # [(1,'a'),(1,'b'),(2,'a'),(2,'b')]
```

### Nhóm

```python
data = [("A", 1), ("A", 2), ("B", 3), ("B", 4)]
for key, nhom in it.groupby(data, key=lambda t: t[0]):
    print(key, list(nhom))
# A [(A, 1), (A, 2)]
# B [(B, 3), (B, 4)]
```

`groupby` chỉ gộp phần tử **liền kề** — phải sắp xếp trước nếu cần nhóm toàn bộ.

```python
data.sort(key=lambda t: t[0])
```

### Lọc và chia

```python
ds = [1, 2, 3, 4, 5, 6]
list(it.takewhile(lambda x: x < 4, ds))     # [1, 2, 3]
list(it.dropwhile(lambda x: x < 4, ds))     # [4, 5, 6]

# Chia theo điều kiện
chan, le = it.tee(ds)
chan = filter(lambda x: x % 2 == 0, chan)
le = filter(lambda x: x % 2, le)
```

### Tổ hợp và tích lũy

```python
list(it.accumulate([1, 2, 3, 4]))           # [1, 3, 6, 10]
list(it.accumulate([1, 2, 3, 4], lambda a, b: a * b))   # [1, 2, 6, 24]

list(it.pairwise([1, 2, 3, 4]))             # [(1,2),(2,3),(3,4)] — 3.10+
```

### Nhóm theo batch

```python
def batch(iterable, n):
    it_ = iter(iterable)
    while chunk := list(it.islice(it_, n)):
        yield chunk

list(batch(range(7), 3))
# [[0,1,2], [3,4,5], [6]]
```

## 9.9 operator

```python
import operator as op
from functools import reduce

ds = [1, 2, 3, 4]
print(reduce(op.mul, ds))          # 24
print(sorted(ds, key=op.neg))      # [4, 3, 2, 1]
print(list(map(op.add, [1,2], [3,4])))   # [4, 6]

# itemgetter và attrgetter
from operator import itemgetter, attrgetter

data = [("An", 8), ("Bình", 9)]
data.sort(key=itemgetter(1))
```

Nhanh hơn lambda vì viết bằng C.

## 9.10 Ví dụ — pipeline xử lý log

```python
from itertools import islice
import re

LOG_PATTERN = re.compile(r"\[(\w+)\]\s+(.*)")

def doc(path):
    with open(path, encoding="utf-8") as f:
        yield from (d.rstrip() for d in f)

def chi_error(dong):
    return (d for d in dong if "[ERROR]" in d)

def tach(dong):
    for d in dong:
        m = LOG_PATTERN.match(d)
        if m:
            yield m.group(1), m.group(2)

def dem_theo_loai(cap):
    from collections import Counter
    return Counter(loai for loai, _ in cap)

pipeline = tach(chi_error(doc("app.log")))
print(dem_theo_loai(pipeline))
```

## 9.11 Bài tập

### Bài 1 — Số chia hết cho 3 và 5
```python
ds = [x for x in range(100) if x % 3 == 0 and x % 5 == 0]
```

### Bài 2 — Đảo dict
```python
d = {"a": 1, "b": 2, "c": 3}
dao = {v: k for k, v in d.items()}
```

### Bài 3 — Tổ hợp
```python
from itertools import combinations
list(combinations(range(5), 3))   # 10 tổ hợp
```

### Bài 4 — partial nhân 10
```python
from functools import partial
nhan_10 = partial(lambda k, x: x * k, 10)
print(nhan_10(5))   # 50
```

### Bài 5 — Pipeline lọc log
Đã viết ở trên. Đếm dòng ERROR theo module.

### Bài 6 — Flatten list lồng nhiều cấp
```python
def flatten(ds):
    for x in ds:
        if isinstance(x, list):
            yield from flatten(x)
        else:
            yield x

list(flatten([1, [2, [3, 4]], 5]))   # [1, 2, 3, 4, 5]
```

### Bài 7 — Chunk file lớn
```python
from itertools import islice

def chunk(iterable, size):
    it_ = iter(iterable)
    while chunk := list(islice(it_, size)):
        yield chunk
```

## 9.12 Ghi chú kỹ thuật

- Comprehension có scope riêng; biến bên trong không rò ra ngoài.
- `lambda` chậm hơn `def` một chút vì không có tên trong namespace.
- `map`/`filter` trả iterator — cần `list()` nếu muốn list.
- `lru_cache` giữ tham chiếu tới tham số — cẩn thận với object lớn.
- `itertools` viết bằng C, nhanh hơn vòng lặp Python tương đương.

Stashed.