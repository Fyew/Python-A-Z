# Chương 05 — Hàm, Scope, Module, Package

## 5.1 Hàm là gì

Hàm là khối code có tên, nhận tham số, trả giá trị. Hàm giúp tái sử dụng, che giấu chi tiết, đặt tên cho ý định.

```python
def chao(ten: str) -> str:
    """Trả về lời chào cho tên."""
    return f"Xin chào {ten}"

ket_qua = chao("Nam")
print(ket_qua)   # Xin chào Nam
```

### Cấu trúc một định nghĩa hàm

```text
def tên_hàm(tham_số) -> kiểu_trả_về:
    """docstring"""
    thân hàm
    return giá_trị
```

### Hàm không trả về gì

```python
def in_thong_tin(ten: str) -> None:
    print(f"Tên: {ten}")

ket_qua = in_thong_tin("Nam")
print(ket_qua)   # None
```

Hàm không có `return` ngầm trả về `None`. Hàm chỉ có `return` trống cũng trả `None`.

### Gọi hàm trước khi định nghĩa?

Trong Python, hàm phải được định nghĩa trước khi gọi **tại thời điểm chạy**, không phải tại thời điểm biên dịch. Nhưng nếu định nghĩa nằm trong cùng file và đứng sau lời gọi ở mức module, sẽ lỗi `NameError`.

```python
# LỖI
print(cong(1, 2))   # NameError

def cong(a, b):
    return a + b
```

Trong thân hàm khác thì không sao:

```python
def goi():
    return cong(1, 2)   # OK — cong chưa tồn tại lúc định nghĩa goi

def cong(a, b):
    return a + b

print(goi())   # 3 — lúc gọi, cong đã tồn tại
```

## 5.2 Docstring

Docstring là chuỗi đầu tiên trong hàm/class/module, dùng để tài liệu hóa.

```python
def chia(a: float, b: float) -> float:
    """Chia a cho b.

    Tham số:
        a: tử số
        b: mẫu số, phải khác 0

    Trả về:
        Thương a/b

    Ném:
        ValueError: nếu b bằng 0
    """
    if b == 0:
        raise ValueError("Mẫu số không được bằng 0")
    return a / b
```

Xem docstring:

```python
print(chia.__doc__)
help(chia)
```

## 5.3 Tham số hàm

### Tham số vị trí (positional)

```python
def cong(a, b, c):
    return a + b + c

cong(1, 2, 3)     # theo vị trí
cong(a=1, b=2, c=3)   # theo từ khóa
cong(1, b=2, c=3)     # kết hợp
```

### Tham số mặc định

```python
def chao(ten, loi_chao="Xin chào"):
    return f"{loi_chao} {ten}"

chao("Nam")                  # "Xin chào Nam"
chao("Nam", "Chào buổi sáng") # "Chào buổi sáng Nam"
chao("Nam", loi_chao="Hi")   # "Hi Nam"
```

### Cạm bẫy mutable default

Đây là lỗi kinh điển của người mới.

```python
def them(x, ds=[]):     # SAI
    ds.append(x)
    return ds

print(them(1))   # [1]
print(them(2))   # [1, 2]  ← không phải [2]!
print(them(3))   # [1, 2, 3]
```

Lý do: giá trị mặc định được tạo **một lần** khi hàm được định nghĩa, không phải mỗi lần gọi.

```python
def them_dung(x, ds=None):   # ĐÚNG
    if ds is None:
        ds = []
    ds.append(x)
    return ds

print(them_dung(1))   # [1]
print(them_dung(2))   # [2]
```

Cách sửa khác: dùng tuple hoặc giá trị bất biến làm mặc định.

```python
def them(x, ds=()):
    return (*ds, x)
```

### `*args` — tham số vị trí tùy ý

```python
def tong(*args):
    return sum(args)

tong(1, 2, 3)          # 6
tong()                 # 0
tong(*[1, 2, 3, 4])    # 10
```

Bên trong hàm, `args` là tuple.

### `**kwargs` — tham số từ khóa tùy ý

```python
def in_cau_hinh(**kwargs):
    for k, v in kwargs.items():
        print(f"{k} = {v}")

in_cau_hinh(host="localhost", port=8080)
in_cau_hinh(**{"a": 1, "b": 2})
```

Bên trong hàm, `kwargs` là dict.

### Kết hợp

```python
def ham(a, b=2, *args, **kwargs):
    print(a, b, args, kwargs)

ham(1)                    # 1 2 () {}
ham(1, 3)                 # 1 3 () {}
ham(1, 3, 4, 5)           # 1 3 (4, 5) {}
ham(1, 3, 4, x=9, y=10)   # 1 3 (4,) {'x': 9, 'y': 10}
```

### Positional-only và keyword-only

```python
def ham(a, b, /, c, d, *, e, f):
    print(a, b, c, d, e, f)

ham(1, 2, 3, 4, e=5, f=6)       # hợp lệ
ham(1, 2, c=3, d=4, e=5, f=6)   # hợp lệ
# ham(a=1, b=2, c=3, d=4, e=5, f=6)  # lỗi — a, b positional-only
# ham(1, 2, 3, 4, 5, 6)              # lỗi — e, f keyword-only
```

- `/` — tham số trước nó chỉ truyền theo vị trí.
- `*` — tham số sau nó chỉ truyền theo từ khóa.
- Không cần cả hai, nhưng hữu ích để khóa API.

### Truyền tham số qua tuple/dict

```python
def cong(a, b, c):
    return a + b + c

args = (1, 2, 3)
print(cong(*args))       # 6

kwargs = {"a": 1, "b": 2, "c": 3}
print(cong(**kwargs))    # 6

params = [1, 2, 3]
config = {"c": 10}
print(cong(*params[:2], **config))
```

## 5.4 Hàm là first-class object

Hàm trong Python là object: gán vào biến, truyền làm tham số, trả về từ hàm, đặt trong collection.

```python
def binh_phuong(x):
    return x * x

f = binh_phuong
print(f(5))                # 25

def ap_dung(func, ds):
    return [func(x) for x in ds]

print(ap_dung(binh_phuong, [1, 2, 3]))   # [1, 4, 9]
```

### Hàm bậc cao

```python
def tao_nhan(k):
    def nhan(x):
        return x * k
    return nhan

nhan_ba = tao_nhan(3)
print(nhan_ba(5))     # 15
```

### Sắp xếp với key

```python
ds = ["banana", "apple", "cherry"]
print(sorted(ds, key=len))
print(sorted(ds, key=str.lower))
```

## 5.5 Scope — phạm vi biến

Quy tắc **LEGB**:

- **L**ocal — trong hàm hiện tại.
- **E**nclosing — trong hàm bao ngoài (closure).
- **G**lobal — cấp module.
- **B**uilt-in — hàm có sẵn của Python.

```python
x = "global"

def ngoai():
    x = "enclosing"
    def trong():
        print(x)      # tìm enclosing
    trong()

ngoai()   # "enclosing"
```

### `global`

```python
dem = 0

def tang():
    global dem
    dem += 1

tang()
print(dem)   # 1
```

Không dùng `global` nếu chỉ đọc — Python tự tìm được.

```python
def doc():
    print(dem)   # OK, đọc được

def ghi():
    dem = 5      # tạo biến local mới, KHÔNG sửa global
```

### `nonlocal`

Sửa biến ở scope bao ngoài gần nhất (không phải global).

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

### Bẫy biến trong closure

```python
funcs = []
for i in range(3):
    funcs.append(lambda: i)

print([f() for f in funcs])   # [2, 2, 2] — không phải [0, 1, 2]

# Sửa: bind giá trị vào default arg
funcs = []
for i in range(3):
    funcs.append(lambda i=i: i)

print([f() for f in funcs])   # [0, 1, 2]
```

### Liệt kê scope

```python
def ham():
    a = 1
    print(locals())

ham()

print(globals()["__name__"])
```

## 5.6 Đệ quy

```python
def giai_thua(n: int) -> int:
    if n <= 1:
        return 1
    return n * giai_thua(n - 1)

print(giai_thua(5))   # 120
```

### Giới hạn đệ quy

```python
import sys
print(sys.getrecursionlimit())   # thường 1000
sys.setrecursionlimit(10000)
```

Tăng giới hạn nguy hiểm — có thể làm tràn stack thật (segfault). Hạn chế đệ quy sâu.

### Đệ quy đuôi

Python **không** tối ưu đệ quy đuôi. Viết vòng lặp nếu có thể.

```python
def fib_de_quy(n):
    if n < 2:
        return n
    return fib_de_quy(n-1) + fib_de_quy(n-2)   # O(2^n) — thảm họa

def fib_vong_lap(n):
    a, b = 0, 1
    for _ in range(n):
        a, b = b, a + b
    return a
```

### Memoization

```python
from functools import lru_cache

@lru_cache(maxsize=None)
def fib(n):
    return n if n < 2 else fib(n-1) + fib(n-2)
```

## 5.7 Type hints trong hàm

```python
def cong(a: int, b: int) -> int:
    return a + b

def xu_ly(ds: list[int]) -> dict[str, int]:
    return {"tong": sum(ds), "so_luong": len(ds)}

def tim(id: int) -> str | None:
    ...
```

Type hints không được kiểm tra runtime, nhưng IDE và mypy dùng.

```python
print(cong.__annotations__)
# {'a': <class 'int'>, 'b': <class 'int'>, 'return': <class 'int'>}
```

## 5.8 Module

Module là file `.py`. Tên file là tên module.

`toan.py`:

```python
PI = 3.14159

def cong(a, b):
    return a + b

def nhan(a, b):
    return a * b

class HinhTron:
    def __init__(self, r):
        self.r = r
```

`main.py`:

```python
import toan

print(toan.PI)
print(toan.cong(1, 2))
print(toan.HinhTron(5).r)
```

### Các kiểu import

```python
import toan                     # toàn bộ
import toan as t                # alias
from toan import cong           # một tên
from toan import cong, nhan     # nhiều tên
from toan import *              # tất cả (không khuyến khích)
```

`from toan import *` ô nhiễm namespace. Chỉ dùng khi module có `__all__`.

```python
# toan.py
__all__ = ["cong", "nhan"]
```

### Module được tìm ở đâu

`sys.path` — danh sách đường dẫn tìm module:

1. Thư mục chứa script đang chạy.
2. Biến môi trường `PYTHONPATH`.
3. Thư mục chuẩn của Python (stdlib).
4. `site-packages` — nơi `pip` cài gói.

```python
import sys
print(sys.path)
```

### `__name__` và `__main__`

```python
# utils.py
def cong(a, b):
    return a + b

print(f"Chạy với __name__ = {__name__}")

if __name__ == "__main__":
    print(cong(1, 2))
```

```bash
$ python utils.py
Chạy với __name__ = __main__
3

$ python -c "import utils"
Chạy với __name__ = utils
```

## 5.9 Package

Package là thư mục chứa module, có file `__init__.py` (từ Python 3.3 có thể không cần, nhưng nên có để rõ ràng).

```text
myapp/
├── __init__.py
├── utils/
│   ├── __init__.py
│   ├── string_utils.py
│   └── math_utils.py
└── core.py
```

`myapp/utils/math_utils.py`:

```python
def cong(a, b):
    return a + b
```

`myapp/core.py`:

```python
from myapp.utils.math_utils import cong
# hoặc
from .utils.math_utils import cong   # relative import
```

### `__init__.py`

Chạy khi package được import. Dùng để khởi tạo hoặc xuất API.

```python
# myapp/__init__.py
from .core import cong

__version__ = "0.1.0"
```

Bây giờ `from myapp import cong` hoạt động.

### Relative import

```python
from . import sibling        # cùng package
from .. import parent_thing  # package cha
from .sub import thing       # package con
```

Relative import chỉ hoạt động trong package, không hoạt động trong script chạy trực tiếp.

## 5.10 `if __name__ == "__main__"`

Đảm bảo code chỉ chạy khi file được gọi trực tiếp.

```python
# script.py
def main():
    print("Chương trình chính")

if __name__ == "__main__":
    main()
```

Khi `import script`, `main()` không chạy. Khi `python script.py`, `main()` chạy.

## 5.11 `argparse` — CLI

```python
import argparse

def main():
    parser = argparse.ArgumentParser(description="Công cụ demo")
    parser.add_argument("ten", help="Tên người dùng")
    parser.add_argument("-t", "--tuoi", type=int, default=0, help="Tuổi")
    parser.add_argument("-v", "--verbose", action="store_true", help="In chi tiết")
    parser.add_argument("--level", choices=["low", "medium", "high"], default="medium")

    args = parser.parse_args()

    if args.verbose:
        print(f"Đang xử lý {args.ten}...")
    print(f"{args.ten}, {args.tuoi} tuổi, mức {args.level}")

if __name__ == "__main__":
    main()
```

```bash
python script.py Nam --tuoi 25 -v
python script.py --help
```

### Subcommand

```python
parser = argparse.ArgumentParser()
sub = parser.add_subparsers(dest="lenh", required=True)

p_add = sub.add_parser("add", help="Thêm")
p_add.add_argument("item")

p_del = sub.add_parser("del", help="Xóa")
p_del.add_argument("item")

args = parser.parse_args()
```

## 5.12 `click` — CLI hiện đại (thư viện ngoài)

```bash
pip install click
```

```python
import click

@click.command()
@click.option("--ten", prompt="Tên bạn", help="Tên người dùng")
@click.option("--tuoi", default=0, type=int)
def chao(ten, tuoi):
    click.echo(f"Chào {ten}, {tuoi} tuổi")

if __name__ == "__main__":
    chao()
```

`click` gọn hơn `argparse` cho CLI phức tạp.

## 5.13 Bài tập

### Bài 1 — Hàm kiểm tra số hoàn hảo
```python
def la_hoan_hao(n: int) -> bool:
    if n < 2:
        return False
    return sum(i for i in range(1, n) if n % i == 0) == n

print(la_hoan_hao(6))    # True
print(la_hoan_hao(28))   # True
```

### Bài 2 — Sắp xếp không dùng `sorted`
```python
def sap_xep(ds: list[int]) -> list[int]:
    ds = ds.copy()
    for i in range(len(ds)):
        for j in range(i + 1, len(ds)):
            if ds[i] > ds[j]:
                ds[i], ds[j] = ds[j], ds[i]
    return ds
```

### Bài 3 — Package hình học

```text
hinhhoc/
├── __init__.py
├── tron.py
├── chu_nhat.py
└── tam_giac.py
```

```python
# hinhhoc/tron.py
import math

def chu_vi(r: float) -> float:
    return 2 * math.pi * r

def dien_tich(r: float) -> float:
    return math.pi * r ** 2
```

### Bài 4 — Fibonacci đệ quy vs vòng lặp
```python
from functools import lru_cache
import time

@lru_cache(maxsize=None)
def fib_de_quy(n: int) -> int:
    return n if n < 2 else fib_de_quy(n-1) + fib_de_quy(n-2)

def fib_vong_lap(n: int) -> int:
    a, b = 0, 1
    for _ in range(n):
        a, b = b, a + b
    return a
```

### Bài 5 — Decorator đo thời gian

```python
import time
from functools import wraps

def do_thoi_gian(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        t0 = time.perf_counter()
        kq = func(*args, **kwargs)
        print(f"{func.__name__}: {time.perf_counter() - t0:.4f}s")
        return kq
    return wrapper

@do_thoi_gian
def tinh_tong(n):
    return sum(range(n))

tinh_tong(1_000_000)
```

### Bài 6 — CLI đơn giản
Viết script nhận `--ten`, in lời chào. Dùng `argparse`.

## 5.14 Ghi chú kỹ thuật

- Hàm trong Python có overhead gọi ~50-100ns. Với vòng lặp cực nóng, cân nhắc inline.
- `*args`/`**kwargs` làm hàm chậm hơn đôi chút — dùng khi cần.
- Global state gây khó test. Ưu tiên truyền tham số.
- Package nên có `pyproject.toml` khi phân phối.
- Không đặt tên module trùng module chuẩn (`json.py`, `types.py`).

Stashed.