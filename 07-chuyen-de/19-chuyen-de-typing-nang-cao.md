# Chuyên Đề 19 — Type Hints Nâng Cao

## 19.1 Variance — vì sao `list[int]` không phải `list[float]`

```python
def tong(ds: list[float]) -> float:
    return sum(ds)

x: list[int] = [1, 2, 3]
tong(x)   # mypy báo lỗi
```

Về mặt lý thuyết kiểu, `list` là **invariant**: `list[int]` không phải subtype của `list[float]`, vì nếu đúng thì ta có thể `append(1.5)` vào list int.

Ngược lại, `Sequence[float]` chấp nhận `list[int]` — `Sequence` là **covariant**.

```python
from typing import Sequence

def tong(ds: Sequence[float]) -> float:
    return sum(ds)

tong([1, 2, 3])   # OK
```

Bảng variance:

| Container | Variance | Ghi chú |
|-----------|----------|---------|
| `list[T]` | invariant | mutable |
| `Sequence[T]` | covariant | chỉ đọc |
| `Iterable[T]` | covariant | chỉ đọc |
| `Mapping[K, V]` | K invariant, V covariant | chỉ đọc |
| `dict[K, V]` | invariant | mutable |
| `Callable[[A], R]` | A contravariant, R covariant | |

Quy tắc: **tham số nhận vào nên là `Sequence`/`Iterable`/`Mapping`; giá trị trả về nên là `list`/`dict` cụ thể.**

## 19.2 `NewType` — kiểu có ý nghĩa

```python
from typing import NewType

UserId = NewType("UserId", int)
OrderId = NewType("OrderId", int)

def lay_nguoi_dung(uid: UserId) -> dict:
    ...

lay_nguoi_dung(UserId(42))   # OK
lay_nguoi_dung(42)            # mypy báo lỗi — int không phải UserId
```

Runtime `UserId(42)` chỉ là `int`, không tốn gì. mypy coi là kiểu riêng.

Dùng cho ID, đơn vị đo (Meters, Seconds), tránh nhầm tham số.

## 19.3 `Literal` và `Final`

```python
from typing import Final, Literal

Mode = Literal["read", "write", "append"]

def mo_file(path: str, mode: Mode) -> None:
    ...

mo_file("a.txt", "read")    # OK
mo_file("a.txt", "reed")    # mypy báo lỗi

MAX_RETRY: Final = 3
MAX_RETRY = 5               # mypy báo lỗi
```

`Final` cũng dùng được trong class:

```python
class CauHinh:
    VERSION: Final[str] = "1.0"
```

## 19.4 `Protocol` — structural typing

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class Closeable(Protocol):
    def close(self) -> None: ...

class File:
    def close(self) -> None:
        ...

def dong(x: Closeable) -> None:
    x.close()

dong(File())                       # OK, không cần kế thừa
isinstance(File(), Closeable)      # True — nhờ runtime_checkable
```

`@runtime_checkable` chỉ kiểm tra **tên method**, không kiểm tra chữ ký — dùng cẩn thận.

### Protocol với thuộc tính

```python
class CoTen(Protocol):
    ten: str

    def chao(self) -> str: ...
```

## 19.5 Generic nâng cao

```python
from typing import TypeVar, Generic

T = TypeVar("T")
K = TypeVar("K")
V = TypeVar("V")

class Kho(Generic[K, V]):
    def __init__(self) -> None:
        self._data: dict[K, V] = {}

    def them(self, k: K, v: V) -> None:
        self._data[k] = v

    def lay(self, k: K) -> V | None:
        return self._data.get(k)

kho: Kho[str, int] = Kho()
kho.them("a", 1)
kho.lay("a")        # int | None
```

### Bound

```python
from typing import TypeVar
from collections.abc import Hashable

H = TypeVar("H", bound=Hashable)

def dem(ds: list[H]) -> dict[H, int]:
    ...
```

### Constrained

```python
StrOrBytes = TypeVar("StrOrBytes", str, bytes)
```

Chỉ nhận `str` hoặc `bytes`, không nhận subtype khác.

### Variadic generic — `TypeVarTuple` (3.11+)

```python
from typing import TypeVarTuple, Unpack

Ts = TypeVarTuple("Ts")

def lay_dau(*args: Unpack[Ts]) -> tuple[Unpack[Ts]]:
    return args

x = lay_dau(1, "a", 2.0)
# x: tuple[int, str, float]
```

### `ParamSpec` — giữ chữ ký qua decorator

```python
from typing import ParamSpec, TypeVar
from collections.abc import Callable

P = ParamSpec("P")
R = TypeVar("R")

def do_thoi_gian(f: Callable[P, R]) -> Callable[P, R]:
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        return f(*args, **kwargs)
    return wrapper

@do_thoi_gian
def cong(a: int, b: int) -> int:
    return a + b

cong(1, 2)       # mypy biết chữ ký (int, int) -> int
cong(1, "2")     # mypy báo lỗi
```

## 19.6 `TypeGuard` và `TypeIs`

```python
from typing import TypeGuard

def la_list_str(x: object) -> TypeGuard[list[str]]:
    return isinstance(x, list) and all(isinstance(i, str) for i in x)

def xu_ly(x: object) -> None:
    if la_list_str(x):
        print(x[0].upper())   # mypy biết x là list[str]
```

Python 3.13 thêm `TypeIs` — chính xác hơn `TypeGuard` (thu hẹp cả nhánh else).

## 19.7 `overload` — nhiều chữ ký

```python
from typing import overload

@overload
def chuan_hoa(x: int) -> str: ...
@overload
def overload
def chuan_hoa(x: list[int]) -> list[str]: ...
def chuan_hoa(x):
    if isinstance(x, list):
        return [str(i) for i in x]
    return str(x)
```

Chỉ dùng khi quan hệ vào-ra không diễn đạt được bằng generic.

## 19.8 `Annotated` — metadata cho type

```python
from typing import Annotated
from pydantic import BaseModel, Field

class NguoiDung(BaseModel):
    ten: Annotated[str, Field(min_length=1, max_length=50)]
    tuoi: Annotated[int, Field(ge=0, le=150)]
```

`Annotated` gắn metadata mà không đổi kiểu. FastAPI, Pydantic, SQLAlchemy 2.0 đều dùng.

### Tự định nghĩa

```python
from typing import Annotated, get_args

Positive = Annotated[int, "phải dương"]

def kiem_tra(x: Positive) -> None:
    for meta in get_args(x.__metadata__):  # minh họa
        ...
```

## 19.9 Kiểm tra kiểu runtime với `pydantic`

Type hints không kiểm tra runtime. Pydantic biến chúng thành validation.

```python
from pydantic import BaseModel, ValidationError, field_validator

class DonHang(BaseModel):
    ma: str
    so_luong: int
    gia: float

    @field_validator("so_luong")
    @classmethod
    def duong(cls, v: int) -> int:
        if v <= 0:
            raise ValueError("số lượng phải dương")
        return v

try:
    DonHang(ma="A1", so_luong=0, gia=10)
except ValidationError as e:
    print(e.errors())
```

## 19.10 Cấu hình mypy nghiêm ngặt

```toml
[tool.mypy]
python_version = "3.12"
strict = true
warn_unused_ignores = true
warn_redundant_casts = true
warn_unreachable = true
disallow_any_explicit = false
disallow_any_generics = true
no_implicit_reexport = true

[[tool.mypy.overrides]]
module = ["third_party_lib.*"]
ignore_missing_imports = true
```

### Bật dần

Bắt đầu với:

```toml
[tool.mypy]
check_untyped_defs = true
no_implicit_optional = true
warn_return_any = true
```

Sau đó mới `strict = true`.

## 19.11 Stub cho thư viện thiếu type

`mypy.ini` hoặc `pyproject.toml`:

```toml
[[tool.mypy.overrides]]
module = ["some_lib.*"]
ignore_missing_imports = true
```

Hoặc viết stub `some_lib.pyi`:

```python
def ham(a: int, b: str = ...) -> bool: ...
```

Đặt trong `stubs/some_lib/__init__.pyi` và cấu hình `mypy_path = "stubs"`.

## 19.12 Ví dụ — generic Repository có type an toàn

```python
from typing import Protocol, TypeVar
from dataclasses import dataclass

@dataclass
class User:
    id: int
    ten: str

T = TypeVar("T")
ID = TypeVar("ID")

class Repository(Protocol[T, ID]):
    def tim(self, id: ID) -> T | None: ...
    def luu(self, obj: T) -> ID: ...
    def tat_ca(self) -> list[T]: ...

class UserRepo:
    def __init__(self) -> None:
        self._data: dict[int, User] = {}

    def tim(self, id: int) -> User | None:
        return self._data.get(id)

    def luu(self, obj: User) -> int:
        self._data[obj.id] = obj
        return obj.id

    def tat_ca(self) -> list[User]:
        return list(self._data.values())

def dung_repo(repo: Repository[T, ID]) -> list[T]:
    return repo.tat_ca()
```

## 19.13 Bài tập

1. Dùng `NewType` cho `UserId`, `OrderId`, `Email`. Viết hàm chỉ nhận đúng kiểu.
2. Viết `Protocol` `Serializer[T]` với `dumps`/`loads`, hai implementation JSON và pickle.
3. Viết decorator `retry` dùng `ParamSpec` giữ nguyên chữ ký.
4. Dùng `TypeGuard` viết `la_dict_str_int`.
5. Viết generic `Stack[T]` với type hints đầy đủ, chạy mypy strict.
6. Dùng `Annotated` + Pydantic viết model có validation tùy chỉnh.
7. Chạy mypy `strict` trên một module cũ, sửa hết lỗi.

## 19.14 Ghi chú kỹ thuật

- Type hints được đánh giá runtime trừ khi `from __future__ import annotations`.
- `Annotated` metadata không được Python dùng; chỉ thư viện đọc.
- mypy cache trong `.mypy_cache/`; xóa khi kết quả lạ.
- `pyright`/Pylance nhanh hơn mypy, đôi khi khác kết quả.
- Không lạm dụng `Any` — mỗi `Any` là một lỗ hổng trong lưới an toàn kiểu.
- `cast()` chỉ là khẳng định với type checker, không kiểm tra runtime.

Stashed.