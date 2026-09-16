# Chương 12 — Testing, Đóng Gói, Hiệu Năng

## 12.1 Vì sao test

- Phát hiện lỗi sớm — rẻ hơn sửa.
- Refactor tự tin.
- Tài liệu sống về hành vi.
- Bắt regression khi nâng cấp.

## 12.2 pytest cơ bản

```bash
pip install pytest pytest-cov
```

`src/calc.py`:

```python
def cong(a: int, b: int) -> int:
    return a + b

def chia(a: float, b: float) -> float:
    if b == 0:
        raise ValueError("chia cho 0")
    return a / b
```

`tests/test_calc.py`:

```python
import pytest
from calc import cong, chia

def test_cong():
    assert cong(2, 3) == 5

def test_chia():
    assert chia(10, 2) == 5

def test_chia_zero():
    with pytest.raises(ValueError, match="chia cho 0"):
        chia(1, 0)
```

Chạy:

```bash
pytest -v
pytest --cov=src --cov-report=term-missing
```

pytest tự tìm file `test_*.py` hoặc `*_test.py` và hàm `test_*`.

## 12.3 Fixture

```python
import pytest

@pytest.fixture
def db():
    conn = {"ket_noi": True}
    yield conn
    conn["ket_noi"] = False

def test_db(db):
    assert db["ket_noi"]
```

### Scope

```python
@pytest.fixture(scope="session")
def client():
    ...

@pytest.fixture(scope="module")
def db_conn():
    ...

@pytest.fixture(scope="function")   # mặc định
def du_lieu():
    ...
```

### conftest.py

Fixture dùng chung đặt trong `conftest.py` — pytest tự tìm.

```python
# tests/conftest.py
import pytest

@pytest.fixture
def api_client():
    return TestClient(app)
```

## 12.4 Parametrize

```python
@pytest.mark.parametrize("a,b,kq", [
    (1, 2, 3),
    (0, 0, 0),
    (-1, 1, 0),
])
def test_cong_nhieu(a, b, kq):
    assert cong(a, b) == kq
```

### Parametrize fixture

```python
@pytest.fixture(params=["sqlite", "postgres"])
def db(request):
    return request.param
```

## 12.5 Mock và patch

```python
from unittest.mock import patch, MagicMock

def test_goi_api():
    with patch("requests.get") as mock_get:
        mock_get.return_value.status_code = 200
        mock_get.return_value.json.return_value = {"ok": True}
        assert mock_get("http://x").status_code == 200
        mock_get.assert_called_once_with("http://x")
```

### monkeypatch

```python
def test_env(monkeypatch):
    monkeypatch.setenv("API_KEY", "test")
    import os
    assert os.environ["API_KEY"] == "test"

def test_thu_muc(monkeypatch, tmp_path):
    monkeypatch.chdir(tmp_path)
    assert str(tmp_path) in str(Path.cwd())
```

### tmp_path

```python
def test_ghi_file(tmp_path):
    p = tmp_path / "test.txt"
    p.write_text("hello")
    assert p.read_text() == "hello"
```

`tmp_path` là fixture có sẵn, tự dọn dẹp.

## 12.6 Coverage

```bash
pytest --cov=myproject --cov-report=html
# mở htmlcov/index.html
```

Mục tiêu thực tế:

- 80-90% cho code nghiệp vụ.
- 100% không cần thiết và tốn thời gian.
- Coverage cao không đảm bảo test tốt.

## 12.7 Đóng gói

### pyproject.toml

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "mytool"
version = "0.1.0"
description = "Công cụ demo"
requires-python = ">=3.10"
dependencies = ["requests>=2.31"]

[project.optional-dependencies]
dev = ["pytest", "ruff", "mypy"]

[project.scripts]
mytool = "mytool.cli:main"
```

### Cấu trúc

```text
mytool/
├── pyproject.toml
├── README.md
└── src/
    └── mytool/
        ├── __init__.py
        └── cli.py
```

### Build và publish

```bash
pip install build twine
python -m build
# tạo dist/mytool-0.1.0.tar.gz và .whl

twine check dist/*
twine upload dist/*   # cần tài khoản PyPI
```

### Cài local để dev

```bash
pip install -e .
```

`-e` (editable) cho phép sửa code không cần cài lại.

## 12.8 Profiling

### cProfile

```bash
python -m cProfile -s cumtime script.py
```

```python
import cProfile
cProfile.run("ham_nang()", "profile.out")

import pstats
p = pstats.Stats("profile.out")
p.sort_stats("cumulative").print_stats(10)
```

### timeit

```python
import timeit
print(timeit.timeit("sum(range(100))", number=10000))

# Đo đoạn code
t = timeit.timeit(
    "''.join(str(i) for i in range(100))",
    number=1000,
)
print(t)
```

### line_profiler

```bash
pip install line_profiler
kernprof -l -v script.py
```

```python
@profile
def ham():
    ...
```

### memory_profiler

```bash
pip install memory-profiler
python -m memory_profiler script.py
```

## 12.9 Tối ưu hiệu năng

### Cấu trúc dữ liệu đúng

```python
# Kiểm tra tồn tại: set O(1) vs list O(n)
ds_set = set(range(1_000_000))
print(999999 in ds_set)   # nhanh

# Queue: deque O(1) hai đầu, list.pop(0) O(n)
from collections import deque
```

### Tránh vòng lặp thừa

```python
# Chậm
kq = []
for x in range(1_000_000):
    kq.append(x * x)

# Nhanh hơn ~30%
kq = [x * x for x in range(1_000_000)]
```

### Dùng hàm built-in (viết bằng C)

```python
# Chậm
tong = 0
for x in ds:
    tong += x

# Nhanh
tong = sum(ds)
```

### Nối chuỗi

```python
# Chậm O(n^2)
s = ""
for w in ds:
    s += w

# Nhanh O(n)
s = "".join(ds)
```

### Local variable nhanh hơn global

```python
# Chậm — tra global mỗi lần
def ham():
    for i in range(1000000):
        global_val + i

# Nhanh — local
def ham():
    v = global_val
    for i in range(1000000):
        v + i
```

### __slots__

```python
class Diem:
    __slots__ = ("x", "y")
    def __init__(self, x, y):
        self.x, self.y = x, y
```

Giảm ~40-50% bộ nhớ mỗi instance.

### Cache

```python
from functools import lru_cache

@lru_cache(maxsize=None)
def fib(n):
    return n if n < 2 else fib(n-1) + fib(n-2)
```

### numpy cho số học

```python
import numpy as np
a = np.arange(1_000_000)
print((a * a).sum())   # nhanh hơn list hàng chục lần
```

### Generator thay list khi chỉ lặp một lần

```python
# Tốn bộ nhớ
tong = sum([x * x for x in range(10**7)])

# Tiết kiệm
tong = sum(x * x for x in range(10**7))
```

### Cython / C extension / Rust

Với hot path thực sự, viết bằng C, Cython, hoặc Rust (PyO3).

```cython
# fast.pyx
def tong(int n):
    cdef long s = 0
    cdef int i
    for i in range(n):
        s += i
    return s
```

### Benchmark thực tế

```python
import timeit

print(timeit.timeit("sum(range(1000000))", number=100))
print(timeit.timeit("sum([i for i in range(1000000)])", number=100))
print(timeit.timeit("sum(i for i in range(1000000))", number=100))
```

## 12.10 Quy tắc tối ưu

1. **Đo trước, tối ưu sau.** Đừng đoán.
2. **Tối ưu thuật toán trước vi mô.** O(n²) → O(n log n) thắng mọi micro-opt.
3. **Tối ưu 3% code chiếm 97% thời gian.**
4. **Đừng hy sinh tính dễ đọc** nếu hiệu năng đủ dùng.
5. **Cache và lazy** thường thắng tối ưu vòng lặp.

## 12.11 CI/CD

`.github/workflows/test.yml`:

```yaml
name: Test
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.10", "3.11", "3.12"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - run: pip install -e ".[dev]"
      - run: ruff check .
      - run: mypy src/
      - run: pytest --cov=src
```

## 12.12 Bài tập

### Bài 1 — Test PhanSo
Viết test cho class `PhanSo` chương 7, coverage > 90%.

### Bài 2 — Fixture dữ liệu
Fixture tạo list ngẫu nhiên cho hàm sắp xếp.

### Bài 3 — Đóng gói CLI
Đóng gói CLI nhỏ, chạy `pip install -e .`.

### Bài 4 — Profile và tối ưu
Profile đoạn code chậm, tối ưu ít nhất 5 lần.

### Bài 5 — So sánh bằng timeit
So sánh list comprehension vs for loop vs numpy.

### Bài 6 — Mock API
Test hàm gọi API bằng `patch`, không gọi mạng thật.

## 12.13 Ghi chú kỹ thuật

- pytest chạy hàm test theo thứ tự alphabet trong file.
- `pytest -x` dừng ở lỗi đầu tiên; `-k` lọc theo tên.
- `pytest --lf` chạy lại test lỗi lần trước.
- Coverage đo dòng chạy, không đo nhánh logic.
- `python -O` tắt assert và docstring — không dùng cho production nếu cần assert.
- Profiler thêm overhead ~2-3x; chỉ dùng để tìm bottleneck.

Stashed.