# Chương 07 — Lập Trình Hướng Đối Tượng

## 7.1 Class và object

Class là bản thiết kế; object là thực thể tạo từ class.

```python
class Nguoi:
    loai = "Homo sapiens"          # thuộc tính class

    def __init__(self, ten: str, tuoi: int):
        self.ten = ten             # thuộc tính instance
        self.tuoi = tuoi

    def chao(self) -> str:
        return f"Tôi là {self.ten}, {self.tuoi} tuổi"

    def __repr__(self) -> str:
        return f"Nguoi({self.ten!r}, {self.tuoi})"

n = Nguoi("Nam", 25)
print(n.chao())
print(n)
```

### `self`

`self` là tham chiếu tới instance hiện tại. Luôn là tham số đầu tiên của method. Tên `self` là quy ước, không bắt buộc — nhưng đừng đổi.

### `__init__` không phải constructor

`__init__` khởi tạo instance. `__new__` mới tạo instance. Hiếm khi cần override `__new__`.

```python
class Singleton:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
```

### Thuộc tính class vs instance

```python
class Dem:
    tong = 0

    def __init__(self, ten):
        self.ten = ten
        Dem.tong += 1

a = Dem("a")
b = Dem("b")
print(Dem.tong)     # 2
print(a.tong)       # 2 — đọc qua instance vẫn ra class attr
```

Gán qua instance tạo thuộc tính instance che class attr:

```python
a.tong = 99
print(a.tong)       # 99
print(Dem.tong)     # 2
```

## 7.2 Thuộc tính private (quy ước)

Python không có private thật, dùng quy ước:

- `_ten`: protected — "đừng chạm nếu không cần".
- `__ten`: name mangling — bị đổi thành `_ClassName__ten`.

```python
class TaiKhoan:
    def __init__(self, so_du: float):
        self._so_du = so_du
        self.__bi_mat = 12345

    @property
    def so_du(self) -> float:
        return self._so_du

    @so_du.setter
    def so_du(self, value: float) -> None:
        if value < 0:
            raise ValueError("Số dư không được âm")
        self._so_du = value

tk = TaiKhoan(1000)
print(tk.so_du)
tk.so_du = 2000
print(tk.__dict__)
# {'_so_du': 2000, '_TaiKhoan__bi_mat': 12345}
```

Name mangling không phải bảo mật — vẫn truy cập được.

## 7.3 Property — getter/setter có kiểm soát

```python
class NhietDo:
    def __init__(self, celsius: float):
        self._c = celsius

    @property
    def celsius(self) -> float:
        return self._c

    @celsius.setter
    def celsius(self, value: float) -> None:
        if value < -273.15:
            raise ValueError("Dưới 0 tuyệt đối")
        self._c = value

    @property
    def fahrenheit(self) -> float:
        return self._c * 9 / 5 + 32

t = NhietDo(25)
print(t.fahrenheit)   # 77.0
t.celsius = 30
```

### Cache bằng property

```python
class DuLieu:
    def __init__(self):
        self._ket_qua = None

    @property
    def ket_qua(self):
        if self._ket_qua is None:
            self._ket_qua = self._tinh_nang()
        return self._ket_qua

    def _tinh_nang(self):
        return 42
```

## 7.4 Kế thừa

```python
class DongVat:
    def __init__(self, ten: str):
        self.ten = ten

    def keu(self) -> str:
        raise NotImplementedError

    def gioi_thieu(self) -> str:
        return f"Tôi là {self.ten}, kêu {self.keu()}"

class Cho(DongVat):
    def keu(self) -> str:
        return "Gâu gâu"

class Meo(DongVat):
    def keu(self) -> str:
        return "Meo meo"

for dv in [Cho("Milu"), Meo("Mun")]:
    print(dv.gioi_thieu())
```

### `super()`

```python
class Cho(DongVat):
    def __init__(self, ten, giong):
        super().__init__(ten)
        self.giong = giong
```

`super()` tra MRO để tìm method tiếp theo.

### Method resolution order (MRO)

```python
class A:
    def hello(self): return "A"
class B(A):
    def hello(self): return "B"
class C(A):
    def hello(self): return "C"
class D(B, C):
    pass

print(D.__mro__)
# (<class 'D'>, <class 'B'>, <class 'C'>, <class 'A'>, <class 'object'>)
print(D().hello())   # "B"
```

Python dùng C3 linearization để tính MRO. Nếu có xung đột, `class` sẽ lỗi.

### `isinstance` và `issubclass`

```python
c = Cho("Milu")
isinstance(c, Cho)         # True
isinstance(c, DongVat)     # True
issubclass(Cho, DongVat)   # True
```

## 7.5 Đa kế thừa

```python
class Bay:
    def di_chuyen(self): return "bay"

class Boi:
    def di_chuyen(self): return "bơi"

class Vit(Bay, Boi):
    pass

print(Vit().di_chuyen())   # "bay" — Bay đứng trước
```

Đa kế thừa mạnh nhưng dễ rối. Ưu tiên composition hoặc mixin.

### Mixin

Mixin là class nhỏ, thêm hành vi, không đứng độc lập.

```python
class LogMixin:
    def log(self, msg):
        print(f"[{self.__class__.__name__}] {msg}")

class DichVu(LogMixin):
    def chay(self):
        self.log("đang chạy")

DichVu().chay()   # [DichVu] đang chạy
```

## 7.6 Abstract Base Class (ABC)

ABC không thể khởi tạo trực tiếp; buộc subclass cài đặt method trừu tượng.

```python
from abc import ABC, abstractmethod

class Hinh(ABC):
    @abstractmethod
    def dien_tich(self) -> float:
        ...

    @abstractmethod
    def chu_vi(self) -> float:
        ...

class HinhTron(Hinh):
    def __init__(self, r: float):
        self.r = r

    def dien_tich(self) -> float:
        import math
        return math.pi * self.r ** 2

    def chu_vi(self) -> float:
        import math
        return 2 * math.pi * self.r
```

```python
# Hinh()  # TypeError: Can't instantiate abstract class
```

## 7.7 Magic methods (dunder)

### Biểu diễn

```python
class Diem:
    def __init__(self, x, y):
        self.x, self.y = x, y

    def __repr__(self):
        return f"Diem({self.x}, {self.y})"

    def __str__(self):
        return f"({self.x}, {self.y})"
```

- `__repr__`: dành cho dev, dễ debug.
- `__str__`: dành cho người dùng. Nếu không có, Python dùng `__repr__`.

### Toán tử

```python
class Vector:
    def __init__(self, x, y):
        self.x, self.y = x, y

    def __add__(self, other):
        return Vector(self.x + other.x, self.y + other.y)

    def __sub__(self, other):
        return Vector(self.x - other.x, self.y - other.y)

    def __mul__(self, k):
        return Vector(self.x * k, self.y * k)

    def __rmul__(self, k):
        return self * k

    def __neg__(self):
        return Vector(-self.x, -self.y)

    def __eq__(self, other):
        return (self.x, self.y) == (other.x, other.y)

    def __hash__(self):
        return hash((self.x, self.y))

    def __repr__(self):
        return f"Vector({self.x}, {self.y})"

v = Vector(1, 2) + Vector(3, 4)
print(v)             # Vector(4, 6)
print(2 * v)         # Vector(8, 12)
```

### Container

```python
class Hop:
    def __init__(self):
        self._items = []

    def __len__(self):
        return len(self._items)

    def __getitem__(self, i):
        return self._items[i]

    def __setitem__(self, i, v):
        self._items[i] = v

    def __contains__(self, v):
        return v in self._items

    def __iter__(self):
        return iter(self._items)
```

### Context manager

```python
class QuanLyFile:
    def __init__(self, path, mode):
        self.path, self.mode = path, mode

    def __enter__(self):
        self.f = open(self.path, self.mode, encoding="utf-8")
        return self.f

    def __exit__(self, exc_type, exc, tb):
        self.f.close()
        return False
```

### `__call__`

Biến instance thành callable.

```python
class Nhan:
    def __init__(self, k):
        self.k = k

    def __call__(self, x):
        return x * self.k

nhan_ba = Nhan(3)
print(nhan_ba(5))   # 15
```

### `__bool__`, `__hash__`, `__eq__`

```python
class TapRong:
    def __len__(self):
        return 0

bool(TapRong())   # False

class User:
    def __init__(self, id):
        self.id = id

    def __eq__(self, other):
        return isinstance(other, User) and self.id == other.id

    def __hash__(self):
        return hash(self.id)
```

Quy tắc: nếu định nghĩa `__eq__`, định nghĩa luôn `__hash__` nếu muốn dùng dict/set.

## 7.8 `dataclass`

Tự sinh `__init__`, `__repr__`, `__eq__`.

```python
from dataclasses import dataclass, field

@dataclass
class Diem:
    x: float
    y: float

d = Diem(1, 2)
print(d)          # Diem(x=1, y=2)
print(d == Diem(1, 2))   # True
```

### Tùy chọn

```python
@dataclass(frozen=True, slots=True)
class ToaDo:
    x: float
    y: float
```

- `frozen=True`: bất biến, hashable.
- `slots=True` (3.10+): tiết kiệm bộ nhớ.
- `order=True`: sinh `__lt__`, `__gt__`, ...

### Giá trị mặc định

```python
@dataclass
class SinhVien:
    ten: str
    diem: list[float] = field(default_factory=list)
    tuoi: int = 18
```

Không dùng `diem: list = []` — cạm bẫy mutable default.

### `__post_init__`

```python
@dataclass
class Range:
    low: int
    high: int

    def __post_init__(self):
        if self.low > self.high:
            raise ValueError("low > high")
```

### So sánh với NamedTuple

`dataclass` linh hoạt hơn, `NamedTuple` immutable và nhẹ hơn.

## 7.9 `__slots__` — tiết kiệm bộ nhớ

```python
class DiemSlot:
    __slots__ = ("x", "y")

    def __init__(self, x, y):
        self.x, self.y = x, y
```

Không có `__dict__` → không thêm thuộc tính tùy ý. Tiết kiệm ~40-50% bộ nhớ mỗi instance khi có nhiều instance.

## 7.10 Class method và static method

```python
class Ngay:
    def __init__(self, y, m, d):
        self.y, self.m, self.d = y, m, d

    @classmethod
    def tu_chuoi(cls, s: str) -> "Ngay":
        y, m, d = map(int, s.split("-"))
        return cls(y, m, d)

    @staticmethod
    def la_nam_nhuan(y: int) -> bool:
        return y % 4 == 0 and (y % 100 != 0 or y % 400 == 0)

n = Ngay.tu_chuoi("2024-03-15")
print(Ngay.la_nam_nhuan(2024))
```

- `@classmethod`: nhận `cls`, dùng cho factory.
- `@staticmethod`: không nhận `self`/`cls`, chỉ là hàm trong namespace của class.

## 7.11 Composition thay kế thừa

"Has-a" thay vì "is-a".

```python
class DongCo:
    def chay(self) -> str:
        return "động cơ chạy"

class Xe:
    def __init__(self):
        self.dong_co = DongCo()

    def khoi_dong(self) -> str:
        return self.dong_co.chay()
```

Kế thừa sâu tạo lớp cứng nhắc; composition linh hoạt hơn.

## 7.12 Ví dụ tổng hợp — hệ thống ngân hàng

```python
from dataclasses import dataclass, field
from datetime import datetime

class LoiNganHang(Exception):
    pass

class LoiSoDuKhongDu(LoiNganHang):
    pass

@dataclass
class GiaoDich:
    loai: str
    so_tien: float
    thoi_diem: datetime = field(default_factory=datetime.now)

    def __str__(self):
        return f"{self.thoi_diem:%Y-%m-%d %H:%M} {self.loai} {self.so_tien:,.0f}"

class TaiKhoan:
    def __init__(self, chu: str, so_du: float = 0):
        self.chu = chu
        self._so_du = so_du
        self.lich_su: list[GiaoDich] = []

    @property
    def so_du(self) -> float:
        return self._so_du

    def gui(self, so_tien: float) -> None:
        if so_tien <= 0:
            raise ValueError("Số tiền phải dương")
        self._so_du += so_tien
        self.lich_su.append(GiaoDich("gửi", so_tien))

    def rut(self, so_tien: float) -> None:
        if so_tien <= 0:
            raise ValueError("Số tiền phải dương")
        if so_tien > self._so_du:
            raise LoiSoDuKhongDu(f"Số dư {self._so_du:,.0f}")
        self._so_du -= so_tien
        self.lich_su.append(GiaoDich("rút", so_tien))

    def __repr__(self):
        return f"TaiKhoan({self.chu!r}, {self._so_du:,.0f})"
```

## 7.13 Bài tập

### Bài 1 — Phân số

```python
import math
from dataclasses import dataclass

@dataclass(frozen=True)
class PhanSo:
    tu: int
    mau: int = 1

    def __post_init__(self):
        if self.mau == 0:
            raise ValueError("Mẫu bằng 0")
        g = math.gcd(self.tu, self.mau)
        object.__setattr__(self, "tu", self.tu // g)
        object.__setattr__(self, "mau", self.mau // g)

    def __add__(self, other):
        return PhanSo(self.tu * other.mau + other.tu * self.mau, self.mau * other.mau)

    def __str__(self):
        return f"{self.tu}/{self.mau}"
```

### Bài 2 — Hình học trừu tượng

Xây ABC `Hinh` với `HinhTron`, `ChuNhat`, `TamGiac`.

### Bài 3 — Stack và Queue

```python
class Stack:
    def __init__(self):
        self._data = []
    def push(self, x): self._data.append(x)
    def pop(self): return self._data.pop()
    def __len__(self): return len(self._data)

from collections import deque
class Queue:
    def __init__(self):
        self._data = deque()
    def enqueue(self, x): self._data.append(x)
    def dequeue(self): return self._data.popleft()
```

### Bài 4 — Sinh viên dùng dataclass
Định nghĩa `SinhVien`, sắp xếp theo điểm trung bình giảm dần.

### Bài 5 — Hệ thống ngân hàng
Mở rộng ví dụ trên với `KhachHang` quản lý nhiều `TaiKhoan`.

## 7.14 Ghi chú kỹ thuật

- Mỗi instance thường có `__dict__` — tốn bộ nhớ. `__slots__` hoặc dataclass `slots=True` giúp giảm.
- Gọi method tốn ~50ns. Attribute lookup có cache.
- `super()` trong đa kế thừa phức tạp — đọc MRO khi bối rối.
- Tránh kế thừa sâu > 3 cấp. Ưu tiên composition.
- `frozen=True` + `slots=True` + `eq=True` cho value object.

Stashed.