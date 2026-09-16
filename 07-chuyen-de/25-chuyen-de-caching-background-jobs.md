# Chuyên Đề 25 — Caching và Background Jobs

## 25.1 Cache là gì

Cache là lớp lưu trữ nhanh, gần nguồn dữ liệu, chấp nhận dữ liệu có thể cũ. Đánh đổi: tốc độ đổi lấy tính nhất quán.

Ba câu hỏi trước khi cache:

1. Dữ liệu thay đổi bao lâu một lần?
2. Chấp nhận cũ bao lâu?
3. Xóa cache khi nào?

Không trả lời được ba câu này — chưa nên cache.

## 25.2 `functools.lru_cache`

```python
from functools import lru_cache

@lru_cache(maxsize=128)
def tinh_nang(n: int) -> int:
    ...
```

- `maxsize`: giới hạn số entry; `None` là không giới hạn.
- Yêu cầu tham số hashable.
- Thread-safe từ 3.9.
- Xóa bằng `cache_clear()`.

### `cache` (3.9+)

```python
from functools import cache

@cache
def fib(n: int) -> int:
    return n if n < 2 else fib(n - 1) + fib(n - 2)
```

Tương đương `lru_cache(maxsize=None)`.

### Xem thống kê

```python
fib.cache_info()
# CacheInfo(hits=..., misses=..., maxsize=None, currsize=...)
```

### Ví dụ — cache kết quả HTTP

```python
from functools import lru_cache
import httpx

@lru_cache(maxsize=256)
def lay_du_lieu(url: str) -> bytes:
    with httpx.Client(timeout=10) as c:
        return c.get(url).content
```

Cache vô thời hạn — chỉ dùng khi URL ổn định.

## 25.3 `cached_property`

```python
from functools import cached_property

class BaoCao:
    def __init__(self, du_lieu: list[int]):
        self.du_lieu = du_lieu

    @cached_property
    def tong(self) -> int:
        print("tính tổng")
        return sum(self.du_lieu)

b = BaoCao([1, 2, 3])
print(b.tong)   # in "tính tổng", trả 6
print(b.tong)   # không in, trả 6
```

Chỉ tính một lần, lưu vào `__dict__`. Yêu cầu class có `__dict__` (không dùng `__slots__`).

## 25.4 TTL cache đơn giản

`lru_cache` không có TTL. Tự viết:

```python
import time
from functools import wraps

def ttl_cache(giay: float):
    def decorator(func):
        cache: dict = {}
        @wraps(func)
        def wrapper(*args):
            now = time.monotonic()
            if args in cache:
                gia_tri, het_han = cache[args]
                if now < het_han:
                    return gia_tri
            gia_tri = func(*args)
            cache[args] = (gia_tri, now + giay)
            return gia_tri
        def clear():
            cache.clear()
        wrapper.cache_clear = clear
        return wrapper
    return decorator

@ttl_cache(60)
def lay_ty_gia(cap: str) -> float:
    ...
```

### `cachetools`

```bash
pip install cachetools
```

```python
from cachetools import TTLCache, LRUCache, cached

cache = TTLCache(maxsize=100, ttl=60)

@cached(cache)
def lay_du_lieu(key: str) -> str:
    ...
```

## 25.5 Redis làm cache

```bash
pip install redis
```

```python
import redis
import json

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def lay_nguoi_dung(uid: int) -> dict:
    key = f"user:{uid}"
    cached = r.get(key)
    if cached:
        return json.loads(cached)

    du_lieu = truy_van_db(uid)
    r.setex(key, 300, json.dumps(du_lieu))   # TTL 5 phút
    return du_lieu
```

### Xóa cache khi ghi

```python
def cap_nhat_nguoi_dung(uid: int, du_lieu: dict) -> None:
    ghi_db(uid, du_lieu)
    r.delete(f"user:{uid}")
```

### Cache-aside pattern

1. Đọc cache.
2. Nếu trúng (hit), trả về.
3. Nếu trượt (miss), đọc nguồn, ghi cache, trả về.
4. Khi ghi, xóa cache liên quan.

Đây là pattern phổ biến nhất — đơn giản, chấp nhận eventual consistency.

### `redis-py` async

```python
import redis.asyncio as aioredis

r = aioredis.from_url("redis://localhost")

async def lay(uid: int) -> str | None:
    return await r.get(f"user:{uid}")
```

### Cache stampede

Nhiều request cùng miss cache cùng lúc → đều truy vấn nguồn.

```python
import time

def lay_co_lock(key: str) -> dict:
    cached = r.get(key)
    if cached:
        return json.loads(cached)

    lock = r.lock(f"lock:{key}", timeout=10)
    if lock.acquire(blocking=True, blocking_timeout=5):
        try:
            cached = r.get(key)   # kiểm tra lại sau khi lấy lock
            if cached:
                return json.loads(cached)
            du_lieu = truy_van_db(key)
            r.setex(key, 300, json.dumps(du_lieu))
            return du_lieu
        finally:
            lock.release()

    time.sleep(0.1)
    return lay_co_lock(key)
```

### Cache warming

Nạp trước dữ liệu hay dùng:

```python
def warm_cache():
    for uid in r.smembers("active_users"):
        lay_nguoi_dung(int(uid))
```

## 25.6 Ghi/xóa cache đúng

Bốn chiến lược:

| Chiến lược | Cách làm | Nhược điểm |
|-----------|----------|------------|
| Cache-aside | app quản lý | phức tạp khi nhiều nguồn ghi |
| Write-through | ghi cache + DB cùng lúc | chậm hơn |
| Write-behind | ghi cache, DB sau | rủi ro mất dữ liệu |
| Read-through | cache tự đọc nguồn | cần cache thông minh |

### Invalidation

- **TTL**: đơn giản, chấp nhận cũ.
- **Explicit delete**: chính xác, dễ quên.
- **Version key**: `user:42:v3` — đổi version là invalidate hàng loạt.
- **Tag-based**: gắn tag, xóa theo tag.

## 25.7 Background jobs với Celery

```bash
pip install celery redis
```

`celery_app.py`:

```python
from celery import Celery

app = Celery(
    "tasks",
    broker="redis://localhost:6379/0",
    backend="redis://localhost:6379/1",
)

app.conf.update(
    task_serializer="json",
    result_serializer="json",
    accept_content=["json"],
    timezone="UTC",
    enable_utc=True,
    task_acks_late=True,
    worker_prefetch_multiplier=1,
)
```

`tasks.py`:

```python
from celery_app import app

@app.task(bind=True, max_retries=3)
def gui_email(self, dia_chi: str, noi_dung: str) -> None:
    try:
        gui(dia_chi, noi_dung)
    except Exception as exc:
        raise self.retry(exc=exc, countdown=2 ** self.request.retries)
```

Chạy worker:

```bash
celery -A celery_app worker --loglevel=info --concurrency=4
```

Gọi task:

```python
gui_email.delay("a@x.com", "xin chào")
ket_qua = gui_email.delay("a@x.com", "hi")
print(ket_qua.get(timeout=10))
```

### Định kỳ với Celery Beat

```python
app.conf.beat_schedule = {
    "dong-bo-moi-5-phut": {
        "task": "tasks.dong_bo",
        "schedule": 300.0,
    },
}
```

```bash
celery -A celery_app beat --loglevel=info
```

### Chaining

```python
from celery import chain

chain(tai_du_lieu.s(url), xu_ly.s(), luu.s())()
```

### Cạm bẫy Celery

- Task phải idempotent — có thể chạy lại khi worker chết.
- Không truyền object lớn qua broker; truyền ID, worker đọc DB.
- `acks_late=True` + idempotent = không mất task khi worker crash.
- Không block worker bằng I/O đồng bộ; dùng `--pool=gevent` hoặc async.

## 25.8 APScheduler — trong process

Không cần broker, chạy ngay trong ứng dụng.

```bash
pip install apscheduler
```

```python
from apscheduler.schedulers.background import BackgroundScheduler

sched = BackgroundScheduler(timezone="UTC")

@sched.scheduled_job("interval", minutes=5)
def dong_bo():
    ...

@sched.scheduled_job("cron", hour=2, minute=0)
def sao_luu():
    ...

sched.start()
```

Phù hợp ứng dụng nhỏ, một instance. Nhiều instance → chạy trùng.

## 25.9 Arq — async job queue

```bash
pip install arq
```

```python
from arq import create_pool
from arq.connections import RedisSettings

async def gui_email(ctx, dia_chi: str) -> None:
    ...

class WorkerSettings:
    functions = [gui_email]
    redis_settings = RedisSettings()
```

```bash
arq WorkerSettings
```

```python
redis = await create_pool(RedisSettings())
await redis.enqueue_job("gui_email", "a@x.com")
```

Arq dùng asyncio, nhẹ hơn Celery, phù hợp dự án async.

## 25.10 Rate limiting

### Token bucket

```python
import time

class TokenBucket:
    def __init__(self, suc_chua: int, toc_do: float):
        self.suc_chua = suc_chua
        self.toc_do = toc_do
        self.token = suc_chua
        self.cap_nhat = time.monotonic()

    def cho_phep(self, n: int = 1) -> bool:
        now = time.monotonic()
        self.token = min(
            self.suc_chua,
            self.token + (now - self.cap_nhat) * self.toc_do,
        )
        self.cap_nhat = now
        if self.token >= n:
            self.token -= n
            return True
        return False
```

### Redis rate limiting

```python
def cho_phep(key: str, gioi_han: int, cua_so: int) -> bool:
    pipe = r.pipeline()
    pipe.incr(key)
    pipe.expire(key, cua_so, nx=True)
    ket_qua, _ = pipe.execute()
    return ket_qua <= gioi_han
```

Sliding window chính xác hơn nhưng tốn bộ nhớ hơn.

### `slowapi` cho FastAPI

```bash
pip install slowapi
```

```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter

@app.get("/")
@limiter.limit("10/minute")
def index(request: Request):
    return {"ok": True}
```

## 25.11 Bài tập

1. Viết `ttl_cache` decorator, test với `time.sleep`.
2. Dùng Redis cache kết quả gọi API, TTL 60s, đo hit/miss.
3. Cài đặt cache stampede protection bằng Redis lock.
4. Viết Celery task gửi email giả, retry 3 lần khi lỗi.
5. Viết task định kỳ dọn dẹp file tạm mỗi giờ.
6. Viết token bucket rate limiter, test với 100 request.
7. Dùng `slowapi` giới hạn 5 request/phút cho một endpoint.
8. So sánh Celery và Arq cho task async — viết cùng một task ở cả hai.

## 25.12 Ghi chú kỹ thuật

- Cache luôn có thể cũ — thiết kế chấp nhận điều đó.
- Không cache dữ liệu nhạy cảm trong Redis không mã hóa.
- `lru_cache` giữ tham chiếu tới tham số — rò rỉ nếu tham số lớn.
- Celery worker nên chạy riêng process, không trong web process.
- Redis single-threaded — tránh `KEYS *` production.
- Rate limiting cần key theo user/IP, không theo instance.
- Job queue cần monitoring (Flower cho Celery) để phát hiện kẹt.
- Luôn có timeout cho mọi job — job treo làm nghẽn queue.

Stashed.