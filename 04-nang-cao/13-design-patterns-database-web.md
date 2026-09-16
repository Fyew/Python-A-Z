# Chương 13 — Design Patterns, Database, Web

## 13.1 Design pattern là gì

Design pattern là giải pháp tái sử dụng cho vấn đề thiết kế lặp lại. Không phải code cụ thể — là khuôn mẫu tư duy.

Ba nhóm chính (theo GoF):

- **Creational**: khởi tạo object — Singleton, Factory, Builder.
- **Structural**: tổ chức object — Adapter, Decorator, Facade.
- **Behavioral**: tương tác — Strategy, Observer, Command.

Python có đặc trưng riêng: first-class function, dynamic typing, protocol — nhiều pattern kinh điển trở nên đơn giản hơn hoặc không cần thiết.

## 13.2 Singleton

Đảm bảo một class chỉ có một instance.

```python
class CauHinh:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance.data = {}
        return cls._instance

a = CauHinh()
b = CauHinh()
assert a is b
```

Trong Python, cách đơn giản hơn là module-level object:

```python
# config.py
data = {}

def get(key, default=None):
    return data.get(key, default)

def set_(key, value):
    data[key] = value
```

Module chỉ được import một lần — tự nhiên là singleton.

### Khi nào dùng

- Cấu hình toàn cục.
- Connection pool.
- Logger.

### Khi nào tránh

Singleton là global state trá hình — khó test, khó mock. Ưu tiên dependency injection.

## 13.3 Factory

Tách logic khởi tạo khỏi code dùng.

```python
from abc import ABC, abstractmethod

class Hinh(ABC):
    @abstractmethod
    def dien_tich(self) -> float: ...

class Tron(Hinh):
    def __init__(self, r): self.r = r
    def dien_tich(self): return 3.14159 * self.r ** 2

class Vuong(Hinh):
    def __init__(self, c): self.c = c
    def dien_tich(self): return self.c ** 2

def tao_hinh(loai: str, **kw) -> Hinh:
    match loai:
        case "tron": return Tron(**kw)
        case "vuong": return Vuong(**kw)
        case _: raise ValueError(f"Không biết loại: {loai}")
```

### Factory với registry

```python
_registry: dict[str, type[Hinh]] = {}

def dang_ky(ten: str):
    def decorator(cls):
        _registry[ten] = cls
        return cls
    return decorator

@dang_ky("tron")
class Tron(Hinh): ...

def tao(ten: str, **kw):
    return _registry[ten](**kw)
```

Registry cho phép thêm loại mới mà không sửa factory.

## 13.4 Builder

Xây object phức tạp theo bước.

```python
from dataclasses import dataclass, field

@dataclass
class HttpRequest:
    method: str = "GET"
    url: str = ""
    headers: dict = field(default_factory=dict)
    body: bytes | None = None
    timeout: float = 30.0

class RequestBuilder:
    def __init__(self, url: str):
        self._req = HttpRequest(url=url)

    def method(self, m: str) -> "RequestBuilder":
        self._req.method = m
        return self

    def header(self, k: str, v: str) -> "RequestBuilder":
        self._req.headers[k] = v
        return self

    def body(self, b: bytes) -> "RequestBuilder":
        self._req.body = b
        return self

    def build(self) -> HttpRequest:
        return self._req

req = RequestBuilder("https://api.x.com") \
    .method("POST") \
    .header("Content-Type", "application/json") \
    .body(b'{"a":1}') \
    .build()
```

Fluent interface — method trả `self`.

## 13.5 Strategy

Chọn thuật toán tại runtime.

```python
from typing import Protocol

class ChienLuocGia(Protocol):
    def tinh(self, gia: float) -> float: ...

def gia_thuong(gia: float) -> float:
    return gia

def gia_vip(gia: float) -> float:
    return gia * 0.8

def gia_khuyen_mai(gia: float) -> float:
    return gia * 0.5

class DonHang:
    def __init__(self, gia: float, chien_luoc=gia_thuong):
        self.gia = gia
        self.chien_luoc = chien_luoc

    def thanh_tien(self) -> float:
        return self.chien_luoc(self.gia)

print(DonHang(1000, gia_vip).thanh_tien())   # 800
```

Trong Python, function first-class làm Strategy tự nhiên — không cần class.

## 13.6 Observer

Đối tượng thông báo cho nhiều observer khi trạng thái đổi.

```python
class ChuDe:
    def __init__(self):
        self._observers: list = []

    def dang_ky(self, f):
        self._observers.append(f)

    def huy(self, f):
        self._observers.remove(f)

    def thong_bao(self, msg: str):
        for f in self._observers:
            f(msg)

cd = ChuDe()
cd.dang_ky(lambda m: print("A nhận:", m))
cd.dang_ky(lambda m: print("B nhận:", m))
cd.thong_bao("hello")
```

Ứng dụng: event system, pub/sub, GUI.

## 13.7 Adapter

Chuyển interface này thành interface khác.

```python
class Cu:
    def gui_email(self, s: str) -> None:
        print(f"Email: {s}")

class Moi:
    def send(self, message: str) -> None:
        ...

class Adapter(Moi):
    def __init__(self, cu: Cu):
        self.cu = cu

    def send(self, message: str) -> None:
        self.cu.gui_email(message)
```

## 13.8 Facade

Interface đơn giản che hệ thống phức tạp.

```python
class HeThongThanhToan:
    def xac_thuc(self): ...
    def kiem_tra_so_du(self): ...
    def tru_tien(self): ...
    def ghi_log(self): ...

class ThanhToanFacade:
    def __init__(self):
        self._ht = HeThongThanhToan()

    def thanh_toan(self, so_tien: float) -> bool:
        self._ht.xac_thuc()
        self._ht.kiem_tra_so_du()
        self._ht.tru_tien()
        self._ht.ghi_log()
        return True
```

## 13.9 Repository

Tách logic truy cập dữ liệu khỏi nghiệp vụ.

```python
from abc import ABC, abstractmethod

class KhoNguoiDung(ABC):
    @abstractmethod
    def tim(self, id: int): ...

    @abstractmethod
    def luu(self, nguoi: dict) -> int: ...

    @abstractmethod
    def tat_ca(self) -> list[dict]: ...

class KhoSQLite(KhoNguoiDung):
    def __init__(self, conn):
        self.conn = conn

    def tim(self, id):
        row = self.conn.execute("SELECT * FROM users WHERE id=?", (id,)).fetchone()
        return dict(row) if row else None

    def luu(self, nguoi):
        cur = self.conn.execute("INSERT INTO users(name) VALUES(?)", (nguoi["name"],))
        self.conn.commit()
        return cur.lastrowid

    def tat_ca(self):
        return [dict(r) for r in self.conn.execute("SELECT * FROM users")]
```

Test dễ hơn với `KhoGia` (in-memory).

## 13.10 Dependency Injection

Truyền dependency vào thay vì tự tạo bên trong.

```python
class DichVu:
    def __init__(self, kho: KhoNguoiDung, logger):
        self.kho = kho
        self.logger = logger

    def tao_nguoi(self, ten: str) -> int:
        self.logger.info(f"Tạo {ten}")
        return self.kho.luu({"name": ten})
```

Dễ test — truyền mock.

## 13.11 SQLite

```python
import sqlite3

conn = sqlite3.connect("app.db")
conn.row_factory = sqlite3.Row
cur = conn.cursor()

cur.execute("""
CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    email TEXT UNIQUE,
    created_at TEXT DEFAULT CURRENT_TIMESTAMP
)
""")

cur.execute("INSERT INTO users(name, email) VALUES(?, ?)", ("Nam", "nam@x.com"))
conn.commit()

cur.execute("SELECT * FROM users WHERE name=?", ("Nam",))
for row in cur.fetchall():
    print(dict(row))

conn.close()
```

### Tham số hóa — chống SQL injection

```python
# SAI
cur.execute(f"SELECT * FROM users WHERE name='{ten}'")

# ĐÚNG
cur.execute("SELECT * FROM users WHERE name=?", (ten,))

# Với nhiều giá trị
cur.execute("SELECT * FROM users WHERE id IN (?, ?, ?)", (1, 2, 3))

# Named
cur.execute("SELECT * FROM users WHERE name=:name", {"name": ten})
```

### Transaction

```python
with conn:
    conn.execute("UPDATE users SET name=? WHERE id=?", ("Mới", 1))
    conn.execute("DELETE FROM logs WHERE user_id=?", (1,))
# tự commit nếu không lỗi, rollback nếu lỗi
```

### executemany

```python
data = [("An", "an@x.com"), ("Bình", "binh@x.com")]
cur.executemany("INSERT INTO users(name, email) VALUES(?, ?)", data)
conn.commit()
```

## 13.12 SQLAlchemy

ORM phổ biến nhất.

```bash
pip install sqlalchemy
```

```python
from sqlalchemy import create_engine, String, select, ForeignKey
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, Session, relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50))
    email: Mapped[str | None] = mapped_column(String(100), unique=True)
    posts: Mapped[list["Post"]] = relationship(back_populates="author")

class Post(Base):
    __tablename__ = "posts"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    author_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    author: Mapped[User] = relationship(back_populates="posts")

engine = create_engine("sqlite:///app.db", echo=True)
Base.metadata.create_all(engine)

with Session(engine) as s:
    u = User(name="Nam", email="nam@x.com")
    u.posts.append(Post(title="Bài đầu tiên"))
    s.add(u)
    s.commit()

    stmt = select(User).where(User.name == "Nam")
    for user in s.scalars(stmt):
        print(user.name, len(user.posts))
```

### Async SQLAlchemy

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession

engine = create_async_engine("sqlite+aiosqlite:///app.db")

async with AsyncSession(engine) as s:
    result = await s.execute(select(User))
    for user in result.scalars():
        print(user.name)
```

## 13.13 FastAPI

```bash
pip install fastapi uvicorn[standard] pydantic
```

`main.py`:

```python
from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel, Field

app = FastAPI(title="Demo API", version="1.0.0")

class ItemIn(BaseModel):
    ten: str = Field(min_length=1, max_length=100)
    gia: float = Field(gt=0)

class ItemOut(ItemIn):
    id: int

kho: dict[int, ItemOut] = {}
dem = 0

@app.get("/items", response_model=list[ItemOut])
def danh_sach():
    return list(kho.values())

@app.get("/items/{item_id}", response_model=ItemOut)
def lay(item_id: int):
    if item_id not in kho:
        raise HTTPException(404, "Không tìm thấy")
    return kho[item_id]

@app.post("/items", response_model=ItemOut, status_code=201)
def tao(item: ItemIn):
    global dem
    dem += 1
    out = ItemOut(id=dem, **item.model_dump())
    kho[dem] = out
    return out

@app.delete("/items/{item_id}", status_code=204)
def xoa(item_id: int):
    kho.pop(item_id, None)
```

Chạy:

```bash
uvicorn main:app --reload --port 8000
# http://127.0.0.1:8000/docs — Swagger UI tự sinh
```

### Pydantic validation

```python
from pydantic import BaseModel, EmailStr, Field, field_validator

class NguoiDung(BaseModel):
    ten: str = Field(min_length=1, max_length=50)
    email: EmailStr
    tuoi: int = Field(ge=0, le=150)

    @field_validator("ten")
    @classmethod
    def ten_sach(cls, v: str) -> str:
        return v.strip()
```

FastAPI tự trả lỗi 422 khi validation thất bại.

### Dependency injection

```python
from fastapi import Depends

def lay_db():
    conn = sqlite3.connect("app.db")
    conn.row_factory = sqlite3.Row
    try:
        yield conn
    finally:
        conn.close()

@app.get("/users")
def users(db = Depends(lay_db)):
    return [dict(r) for r in db.execute("SELECT * FROM users")]
```

### Middleware

```python
import time
from fastapi import Request

@app.middleware("http")
async def them_thoi_gian(request: Request, call_next):
    t0 = time.perf_counter()
    response = await call_next(request)
    response.headers["X-Process-Time"] = f"{time.perf_counter() - t0:.4f}"
    return response
```

### Background task

```python
from fastapi import BackgroundTasks

def ghi_log(msg: str):
    with open("log.txt", "a") as f:
        f.write(msg + "\n")

@app.post("/gui")
def gui(bt: BackgroundTasks):
    bt.add_task(ghi_log, "đã gửi")
    return {"ok": True}
```

### Test FastAPI

```python
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def test_tao_item():
    r = client.post("/items", json={"ten": "A", "gia": 10})
    assert r.status_code == 201
    assert r.json()["id"] == 1
```

## 13.14 Bài tập

### Bài 1 — API quản lý sách
CRUD đầy đủ với FastAPI và Pydantic.

### Bài 2 — Lưu SQLite
Thay dict bằng SQLite, dùng Repository pattern.

### Bài 3 — Middleware log
Log mọi request với thời gian và status.

### Bài 4 — Test API
Viết test cho toàn bộ endpoint bằng TestClient.

### Bài 5 — Strategy phí vận chuyển
Nhiều chiến lược tính phí, chọn theo vùng.

### Bài 6 — Observer event bus
Xây event bus đơn giản, các module đăng ký lắng nghe.

## 13.15 Ghi chú kỹ thuật

- SQLite ghi đồng bộ, một writer tại một thời điểm. Dùng WAL mode cho đọc đồng thời.
- SQLAlchemy ORM tiện nhưng có overhead; query thô nhanh hơn cho báo cáo.
- FastAPI dùng ASGI — hỗ trợ async; Flask/Django truyền thống là WSGI.
- Pydantic v2 viết bằng Rust — nhanh hơn v1 nhiều lần.
- Đừng over-engineer: pattern là công cụ, không phải mục tiêu.

Stashed.