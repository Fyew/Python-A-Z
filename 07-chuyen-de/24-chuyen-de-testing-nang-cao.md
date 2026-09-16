# Chuyên Đề 24 — Testing Nâng Cao: Property-Based, Mutation, Fuzzing

## 24.1 Giới hạn của test ví dụ

Test truyền thống kiểm tra vài trường hợp cụ thể. Nó không chứng minh gì về các trường hợp bạn chưa nghĩ tới.

```python
def test_dao_nguoc():
    assert dao_nguoc("abc") == "cba"
    assert dao_nguoc("") == ""
```

Hai ví dụ này không bắt được lỗi với Unicode, chuỗi rỗng, hay ký tự kết hợp.

## 24.2 Property-based testing với Hypothesis

```bash
pip install hypothesis
```

Bạn mô tả **tính chất** đúng với mọi đầu vào hợp lệ; Hypothesis sinh hàng trăm đầu vào thử.

```python
from hypothesis import given, strategies as st

def dao_nguoc(s: str) -> str:
    return s[::-1]

@given(st.text())
def test_dao_nguoc_hai_lan(s: str) -> None:
    assert dao_nguoc(dao_nguoc(s)) == s

@given(st.text())
def test_do_dai_giu_nguyen(s: str) -> None:
    assert len(dao_nguoc(s)) == len(s)
```

Hypothesis tự tìm đầu vào làm test fail, rồi **shrink** về ví dụ nhỏ nhất.

```text
Falsifying example: test_do_dai_giu_nguyen(s='\x00')
```

### Strategies phổ biến

```python
from hypothesis import strategies as st

st.integers()                          # mọi int
st.integers(min_value=0, max_value=100)
st.floats(allow_nan=False, allow_infinity=False)
st.text(min_size=0, max_size=100)
st.text(alphabet="abc")
st.lists(st.integers(), min_size=0, max_size=10)
st.dictionaries(st.text(), st.integers())
st.tuples(st.integers(), st.text())
st.sampled_from(["a", "b", "c"])
st.booleans()
st.none()
st.one_of(st.integers(), st.text())
st.just(42)
st.builds(Diem, x=st.integers(), y=st.integers())
```

### Ví dụ — property của sắp xếp

```python
@given(st.lists(st.integers()))
def test_sap_xep(ds: list[int]) -> None:
    ket_qua = sorted(ds)
    assert len(ket_qua) == len(ds)
    assert all(ket_qua[i] <= ket_qua[i + 1] for i in range(len(ket_qua) - 1))
    assert sorted(ket_qua) == ket_qua
    assert set(ket_qua) == set(ds)
```

### Ví dụ — encode/decode roundtrip

```python
@given(st.text())
def test_json_roundtrip(s: str) -> None:
    import json
    assert json.loads(json.dumps(s)) == s
```

### Giả định (assume)

```python
from hypothesis import assume

@given(st.integers(), st.integers())
def test_chia(a: int, b: int) -> None:
    assume(b != 0)
    assert chia(a, b) * b == pytest.approx(a)
```

`assume` loại bỏ đầu vào không hợp lệ.

### Stateful testing

```python
from hypothesis.stateful import RuleBasedStateMachine, rule, invariant

class MayTinh(RuleBasedStateMachine):
    def __init__(self) -> None:
        super().__init__()
        self.gia_tri = 0

    @rule(x=st.integers())
    def cong(self, x: int) -> None:
        self.gia_tri += x

    @rule(x=st.integers())
    def tru(self, x: int) -> None:
        self.gia_tri -= x

    @invariant()
    def khong_bao_gio_qua_100(self) -> None:
        assert abs(self.gia_tri) < 10**6
```

## 24.3 Fixture nâng cao

### Factory fixture

```python
import pytest

@pytest.fixture
def tao_nguoi_dung():
    created: list = []
    def _tao(ten: str = "An", tuoi: int = 20):
        u = {"ten": ten, "tuoi": tuoi}
        created.append(u)
        return u
    yield _tao
    # dọn dẹp toàn bộ
    created.clear()

def test_tao(tao_nguoi_dung):
    u = tao_nguoi_dung("Bình", 25)
    assert u["ten"] == "Bình"
```

### `autouse` fixture

```python
@pytest.fixture(autouse=True)
def reset_state():
    yield
    _global_state.clear()
```

### Fixture với tham số

```python
@pytest.fixture(params=["sqlite", "postgres"], ids=["sqlite", "pg"])
def db(request):
    if request.param == "sqlite":
        return sqlite_conn()
    return postgres_conn()
```

### `request.addfinalizer`

```python
@pytest.fixture
def tai_nguyen(request):
    r = mo_tai_nguyen()
    request.addfinalizer(r.close)
    return r
```

## 24.4 Mutation testing

Mutation testing đo **chất lượng test**, không chỉ độ phủ.

```bash
pip install mutmut
```

```bash
mutmut run --paths-to-mutate src/
mutmut results
```

Công cụ sửa code (đổi `>` thành `>=`, `+` thành `-`, xóa dòng) và xem test có bắt được không. Test bắt được mutation = test tốt. Test bỏ sót = test yếu.

Ví dụ mutation:

```python
# Gốc
def la_nguoi_lon(tuoi: int) -> bool:
    return tuoi >= 18

# Mutant 1
    return tuoi > 18     # test phải bắt được nếu có test tuoi=18

# Mutant 2
    return True          # test phải bắt được với tuoi=10
```

### Gợi ý

- Mutation score 70-80% là tốt.
- Chạy mutation test theo đợt, không phải mỗi commit (chậm).
- Tập trung vào module nghiệp vụ, không vào code glue.

## 24.5 Fuzzing

Fuzzing ném dữ liệu ngẫu nhiên vào code để tìm crash.

```bash
pip install atheris    # cần clang
```

```python
import atheris
import sys

def test_one_input(data: bytes) -> None:
    try:
        xu_ly(data)
    except (ValueError, KeyError):
        pass   # ngoại lệ hợp lệ

atheris.Setup(sys.argv, test_one_input)
atheris.Fuzz()
```

### `hypothesis` cũng làm fuzzing

```python
@given(st.binary(max_size=1024))
def test_khong_crash(data: bytes) -> None:
    try:
        phan_tich(data)
    except (ValueError, TypeError):
        pass
```

Chạy `hypothesis` với `--hypothesis-seed=random` để tìm trường hợp mới.

## 24.6 Test async

```bash
pip install pytest-asyncio
```

```python
import pytest

@pytest.mark.asyncio
async def test_tai_du_lieu():
    ket_qua = await tai("http://x")
    assert ket_qua is not None
```

`pyproject.toml`:

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
```

### Test TaskGroup

```python
@pytest.mark.asyncio
async def test_task_group_loi():
    with pytest.raises(ExceptionGroup):
        async with asyncio.TaskGroup() as tg:
            tg.create_task(async_loi())
```

## 24.7 Test database

```python
@pytest.fixture
def db():
    conn = sqlite3.connect(":memory:")
    conn.row_factory = sqlite3.Row
    conn.executescript(SCHEMA)
    yield conn
    conn.close()

def test_tao_nguoi(db):
    db.execute("INSERT INTO users(name) VALUES(?)", ("An",))
    db.commit()
    row = db.execute("SELECT * FROM users").fetchone()
    assert row["name"] == "An"
```

SQLite in-memory nhanh, cô lập, dùng cho phần lớn test.

### Testcontainers cho Postgres

```bash
pip install testcontainers
```

```python
from testcontainers.postgres import PostgresContainer

@pytest.fixture(scope="session")
def pg():
    with PostgresContainer("postgres:16") as c:
        yield c.get_connection_url()
```

## 24.8 Test API với dependency override

```python
from fastapi.testclient import TestClient
from main import app, lay_db

def db_gia():
    conn = sqlite3.connect(":memory:")
    conn.executescript(SCHEMA)
    yield conn

app.dependency_overrides[lay_db] = db_gia
client = TestClient(app)

def test_tao_item():
    r = client.post("/items", json={"ten": "A", "gia": 10})
    assert r.status_code == 201
```

## 24.9 Golden file testing

```python
from pathlib import Path

def test_render(golden_dir: Path):
    ket_qua = render_template(dulieu)
    golden = golden_dir / "expected.html"

    if not golden.exists():
        golden.write_text(ket_qua, encoding="utf-8")

    assert ket_qua == golden.read_text(encoding="utf-8")
```

Chạy lần đầu tạo golden, lần sau so sánh. Cập nhật có kiểm soát khi output thay đổi có chủ ý.

## 24.10 Benchmark test

```python
import pytest

@pytest.mark.benchmark
def test_hieu_nang(benchmark):
    ket_qua = benchmark(tinh_toan, 1000)
    assert ket_qua == 500500
```

```bash
pip install pytest-benchmark
pytest --benchmark-only
```

Lưu baseline để phát hiện regression.

## 24.11 Coverage nâng cao

```bash
pytest --cov=src --cov-branch --cov-report=html
```

`--cov-branch` bắt được nhánh chưa chạy (if/else), không chỉ dòng.

```toml
[tool.coverage.run]
branch = true
source = ["src"]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "if TYPE_CHECKING:",
    "raise NotImplementedError",
]
fail_under = 85
```

## 24.12 Bài tập

1. Viết property test cho `sorted` (đã có), thêm property cho `sum`.
2. Dùng Hypothesis test hàm parse ngày tháng với nhiều định dạng.
3. Viết stateful test cho `Stack` (push/pop phải khôi phục trạng thái).
4. Chạy `mutmut` trên một module nhỏ, xem mutation score.
5. Viết fuzz test cho hàm parse JSON tùy chỉnh.
6. Dùng `testcontainers` test một query Postgres thật.
7. Viết golden file test cho một template.
8. Đo branch coverage và tìm nhánh chưa test.

## 24.13 Ghi chú kỹ thuật

- Hypothesis cache vào `.hypothesis/`; commit hoặc gitignore đều được.
- `@given` chạy mặc định 100 ví dụ; tăng bằng `settings(max_examples=1000)`.
- Mutation testing rất chậm; chỉ chạy nightly hoặc trước release.
- Fuzzing cần build native; Hypothesis đủ cho hầu hết trường hợp Python.
- Test database in-memory không bắt được lỗi đặc thù của Postgres/MySQL.
- Golden file cần review kỹ khi cập nhật — tránh "cập nhật để test xanh".
- Benchmark test nên chạy riêng, không trộn với CI nhanh.

Stashed.