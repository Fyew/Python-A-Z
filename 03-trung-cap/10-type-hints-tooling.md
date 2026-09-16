# Chương 10 — Type Hints và Công Cụ Phát Triển

## 10.1 Vì sao dùng type hints

Type hints không bắt buộc ở runtime nhưng:

- IDE gợi ý chính xác hơn.
- mypy/pyright phát hiện lỗi trước khi chạy.
- Code tự tài liệu hóa.
- Refactor an toàn hơn.

```python
def cong(a: int, b: int) -> int:
    return a + b

cong(1, "2")   # vẫn chạy nếu ép kiểu, mypy báo lỗi
```

## 10.2 Cú pháp cơ bản

```python
ten: str = "Nam"
tuoi: int = 25
diem: float = 8.5
la_sv: bool = True

def chao(ten: str, tuoi: int) -> str:
    return f"{ten} {tuoi}"
```

### Kiểu trong collection (3.9+)

```python
ds: list[int] = [1, 2, 3]
d: dict[str, int] = {"a": 1}
s: set[str] = {"x"}
t: tuple[int, str] = (1, "a")
```

Python 3.8 phải dùng `List`, `Dict` từ `typing`.

## 10.3 Union và Optional

```python
from typing import Optional

def tim(id: int) -> Optional[str]:   # str | None
    ...

def xu_ly(x: int | str) -> None:     # 3.10+
    ...
```

`Optional[X]` tương đương `X | None`.

## 10.4 Các kiểu phổ biến

```python
from typing import Any, Callable, Iterable, Sequence, Mapping

def ap(f: Callable[[int], int], x: int) -> int:
    return f(x)

def tong(ds: Iterable[int]) -> int:
    return sum(ds)

def dau_tien(seq: Sequence[str]) -> str:
    return seq[0]

def tra_cuu(m: Mapping[str, int], k: str) -> int:
    return m.get(k, 0)

def bat_ky(x: Any) -> Any:
    return x
```

- `Iterable` — có thể lặp.
- `Sequence` — có index và len.
- `Mapping` — dict-like.
- `Any` — tắt kiểm tra.

## 10.5 Literal và TypedDict

```python
from typing import Literal, TypedDict

Che = Literal["admin", "user", "guest"]

def kiem_tra(che: Che) -> bool:
    return che == "admin"

class CauHinh(TypedDict):
    host: str
    port: int
    debug: bool = False   # 3.11+ có NotRequired

def ket_noi(cfg: CauHinh) -> None:
    print(cfg["host"])
```

`TypedDict` mô tả dict có cấu trúc cố định.

## 10.6 Protocol — duck typing có kiểu

```python
from typing import Protocol

class CoKeu(Protocol):
    def keu(self) -> str: ...

class Cho:
    def keu(self) -> str:
        return "gâu"

def cho_keu(x: CoKeu) -> str:
    return x.keu()

cho_keu(Cho())   # OK — không cần kế thừa
```

`Protocol` cho phép "structural subtyping" — chỉ cần có method đúng, không cần kế thừa.

## 10.7 Generic

```python
from typing import TypeVar

T = TypeVar("T")

def dau_tien(ds: list[T]) -> T:
    return ds[0]

def doi_cho(a: T, b: T) -> tuple[T, T]:
    return b, a
```

### Giới hạn

```python
from typing import TypeVar
import numbers

Num = TypeVar("Num", bound=numbers.Number)

def nhan_doi(x: Num) -> Num:
    return x * 2
```

## 10.8 overload

Báo cho type checker biết hàm có nhiều chữ ký.

```python
from typing import overload

@overload
def chuan_hoa(x: int) -> str: ...
@overload
def chuan_hoa(x: str) -> str: ...
def chuan_hoa(x):
    return str(x).strip()
```

## 10.9 Self và final

```python
from typing import Self, final

class Builder:
    def __init__(self):
        self.gia_tri = 0

    def set(self, x: int) -> Self:
        self.gia_tri = x
        return self

@final
class KhongKeThua:
    ...
```

`Self` (3.11+) chỉ instance của chính class (kể cả subclass).

## 10.10 Type alias

```python
ToaDo = tuple[float, float]
BangDiem = dict[str, list[int]]

def khoang_cach(a: ToaDo, b: ToaDo) -> float:
    return ((a[0]-b[0])**2 + (a[1]-b[1])**2) ** 0.5
```

3.12+ có cú pháp `type`:

```python
type ToaDo = tuple[float, float]
```

## 10.11 Môi trường ảo

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
deactivate
```

### pip-tools

```bash
pip install pip-tools
pip-compile requirements.in      # sinh requirements.txt có phiên bản khóa
pip-sync requirements.txt         # cài đúng phiên bản
```

### uv — quản lý nhanh

```bash
pip install uv
uv venv
uv pip install requests
uv pip compile requirements.in -o requirements.txt
```

`uv` nhanh hơn pip nhiều lần.

## 10.12 Công cụ chất lượng code

### ruff — linter + formatter

```bash
pip install ruff
ruff check .
ruff check --fix .
ruff format .
```

`pyproject.toml`:

```toml
[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "N", "UP", "B", "SIM", "RUF"]
ignore = ["E501"]

[tool.ruff.lint.isort]
known-first-party = ["myapp"]
```

### black — formatter

```bash
pip install black
black .
```

### mypy — type checker

```bash
pip install mypy
mypy src/
```

`pyproject.toml`:

```toml
[tool.mypy]
python_version = "3.12"
strict = true
warn_unused_ignores = true
warn_redundant_casts = true
disallow_untyped_defs = true
```

`strict = true` bật nhiều kiểm tra nghiêm ngặt.

### pyright (dùng trong VS Code)

Nhanh hơn mypy, tích hợp sẵn trong Pylance.

### pre-commit

```bash
pip install pre-commit
pre-commit install
```

`.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.6.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.11.0
    hooks:
      - id: mypy
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
```

## 10.13 Cấu trúc project chuẩn

```text
myproject/
├── pyproject.toml
├── README.md
├── .gitignore
├── .pre-commit-config.yaml
├── src/
│   └── myproject/
│       ├── __init__.py
│       ├── core.py
│       └── cli.py
└── tests/
    ├── test_core.py
    └── conftest.py
```

Dùng `src/` layout để tránh import nhầm.

## 10.14 pyproject.toml đầy đủ

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "myproject"
version = "0.1.0"
description = "Dự án demo"
requires-python = ">=3.10"
dependencies = [
    "requests>=2.31",
    "pydantic>=2.0",
]

[project.optional-dependencies]
dev = ["pytest", "ruff", "mypy", "pre-commit"]

[project.scripts]
myproject = "myproject.cli:main"

[tool.ruff]
line-length = 100

[tool.mypy]
strict = true

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-v --cov=myproject"
```

## 10.15 Bài tập

### Bài 1 — Thêm type hints
Thêm hints đầy đủ cho một module cũ, chạy mypy strict, sửa hết lỗi.

### Bài 2 — Cấu hình ruff + mypy
Tạo `pyproject.toml`, chạy `ruff check`, `mypy`.

### Bài 3 — venv và requirements
Tạo venv, cài ba thư viện, đóng băng requirements.

### Bài 4 — TypedDict
Viết `TypedDict` cho cấu hình ứng dụng.

### Bài 5 — Generic
Viết hàm tìm phần tử lớn nhất trong list bất kỳ, có type hints.

```python
from typing import TypeVar
T = TypeVar("T")

def lon_nhat(ds: list[T], key=None) -> T:
    return max(ds, key=key)
```

### Bài 6 — Protocol
Định nghĩa `Protocol` cho interface `Repository` và viết hai implementation.

## 10.16 Ghi chú kỹ thuật

- Type hints được đánh giá tại thời điểm định nghĩa hàm (trừ khi dùng `from __future__ import annotations`).
- `from __future__ import annotations` biến tất cả annotation thành chuỗi — tránh lỗi forward reference.
- mypy `strict` bắt đầu nghiêm ngặt; bật dần để không quá tải.
- `ruff` thay thế flake8, isort, pyupgrade trong một công cụ.
- `pyproject.toml` là chuẩn hiện đại thay `setup.py` + `setup.cfg`.

Stashed.