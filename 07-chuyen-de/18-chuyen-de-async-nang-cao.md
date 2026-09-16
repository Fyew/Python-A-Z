# Chuyên Đề 18 — Async Nâng Cao và Structured Concurrency

## 18.1 Vấn đề của `asyncio.gather`

```python
results = await asyncio.gather(a(), b(), c())
```

Ba vấn đề:

1. Nếu một coroutine lỗi, các coroutine khác **vẫn chạy** nhưng bạn mất tham chiếu tới chúng.
2. Không có cách hủy nhóm khi một cái lỗi — task mồ côi.
3. Không có scope rõ ràng: task sống lâu hơn block tạo ra nó.

Đây gọi là "unstructured concurrency" — giống `goto` trong lập trình tuần tự.

## 18.2 `asyncio.TaskGroup` (3.11+)

Task group đảm bảo: khi ra khỏi block, **mọi task con đã xong hoặc đã bị hủy**.

```python
import asyncio

async def tai(url: str) -> str:
    await asyncio.sleep(0.5)
    return f"nội dung {url}"

async def main() -> None:
    async with asyncio.TaskGroup() as tg:
        t1 = tg.create_task(tai("a"))
        t2 = tg.create_task(tai("b"))
        t3 = tg.create_task(tai("c"))

    print(t1.result(), t2.result(), t3.result())

asyncio.run(main())
```

Nếu một task ném ngoại lệ:

- Các task anh em bị hủy.
- `TaskGroup` gom tất cả lỗi vào `ExceptionGroup`.
- Block thoát ra với `ExceptionGroup`.

```python
async def loi() -> None:
    raise ValueError("hỏng")

async def main() -> None:
    try:
        async with asyncio.TaskGroup() as tg:
            tg.create_task(tai("a"))
            tg.create_task(loi())
    except* ValueError as eg:
        print(f"Bắt {len(eg.exceptions)} lỗi ValueError")

asyncio.run(main())
```

`except*` (3.11+) xử lý `ExceptionGroup` — mỗi nhánh xử lý một loại lỗi.

## 18.3 Cancellation

Task có thể bị hủy từ bên ngoài. Hủy được triển khai bằng `CancelledError` ném vào điểm `await`.

```python
async def cong_viec() -> None:
    try:
        await asyncio.sleep(10)
    except asyncio.CancelledError:
        print("bị hủy, dọn dẹp tài nguyên")
        raise   # LUÔN re-raise
    finally:
        print("finally luôn chạy")

async def main() -> None:
    task = asyncio.create_task(cong_viec())
    await asyncio.sleep(1)
    task.cancel()
    try:
        await task
    except asyncio.CancelledError:
        print("task đã hủy")
```

Quy tắc vàng:

- **Luôn re-raise `CancelledError`** sau khi dọn dẹp. Nuốt nó là phá vỡ hợp đồng hủy.
- Không dùng `except Exception` để bắt `CancelledError` — từ 3.8 nó kế thừa `BaseException`, không phải `Exception`.
- Dọn dẹp trong `finally`, không trong `except CancelledError`.

## 18.4 Timeout có hủy thật

```python
async def main() -> None:
    async with asyncio.timeout(2):
        await asyncio.sleep(10)
```

`asyncio.timeout` (3.11+) khác `wait_for`: nó không tạo task mới, chỉ đặt deadline cho scope hiện tại. Hiệu quả hơn và hủy đúng chỗ.

### Timeout lồng nhau

```python
async def main() -> None:
    async with asyncio.timeout(10) as outer:
        async with asyncio.timeout(3):
            await asyncio.sleep(5)
    if outer.expired():
        print("outer hết giờ")
```

## 18.5 Backpressure với `asyncio.Queue`

Queue không giới hạn gây tràn bộ nhớ khi producer nhanh hơn consumer.

```python
import asyncio

async def san_xuat(q: asyncio.Queue) -> None:
    for i in range(1000):
        await q.put(i)      # chặn nếu queue đầy

async def tieu_thu(q: asyncio.Queue) -> None:
    while True:
        item = await q.get()
        await asyncio.sleep(0.01)
        q.task_done()

async def main() -> None:
    q: asyncio.Queue[int] = asyncio.Queue(maxsize=10)
    async with asyncio.TaskGroup() as tg:
        tg.create_task(san_xuat(q))
        tg.create_task(tieu_thu(q))
```

`maxsize` tạo áp lực ngược tự nhiên: producer bị chặn khi consumer chậm.

### Hủy consumer khi producer xong

Consumer `while True` không bao giờ kết thúc. Cần sentinel hoặc hủy có kiểm soát.

```python
SENTINEL = object()

async def tieu_thu(q: asyncio.Queue) -> None:
    while (item := await q.get()) is not SENTINEL:
        print(item)

async def main() -> None:
    q: asyncio.Queue = asyncio.Queue()
    consumer = asyncio.create_task(tieu_thu(q))
    for i in range(5):
        await q.put(i)
    await q.put(SENTINEL)
    await consumer
```

## 18.6 Semaphore — giới hạn đồng thời

```python
async def main() -> None:
    sem = asyncio.Semaphore(10)

    async def co_gioi_han(i: int) -> None:
        async with sem:
            await tai(f"url-{i}")

    async with asyncio.TaskGroup() as tg:
        for i in range(1000):
            tg.create_task(co_gioi_han(i))
```

1000 task nhưng tối đa 10 request đồng thời. Task chờ ở `async with sem` tốn rất ít bộ nhớ — chúng là coroutine chưa chạy.

## 18.7 Chạy code đồng bộ trong async

Blocking call trong async sẽ chặn cả event loop.

```python
import asyncio
import time

def tinh_nang(n: int) -> int:
    time.sleep(2)       # chặn event loop
    return sum(range(n))

async def main() -> None:
    ket_qua = await asyncio.to_thread(tinh_nang, 10**7)
    print(ket_qua)
```

`asyncio.to_thread` chạy hàm trong thread pool — event loop không bị chặn.

### CPU-bound thật sự

Thread không giúp với GIL. Dùng process pool:

```python
from concurrent.futures import ProcessPoolExecutor
import asyncio

async def main() -> None:
    loop = asyncio.get_running_loop()
    with ProcessPoolExecutor() as pool:
        ket_qua = await loop.run_in_executor(pool, tinh_nang, 10**8)
```

Lưu ý: đối tượng truyền qua process phải pickle được.

## 18.8 Queue liên kết nhiều worker

```python
import asyncio

async def worker(ten: str, q: asyncio.Queue) -> None:
    while True:
        viec = await q.get()
        try:
            print(f"{ten} xử lý {viec}")
            await asyncio.sleep(0.1)
        finally:
            q.task_done()

async def main() -> None:
    q: asyncio.Queue[int] = asyncio.Queue()
    async with asyncio.TaskGroup() as tg:
        for i in range(4):
            tg.create_task(worker(f"w{i}", q))
        for viec in range(20):
            await q.put(viec)
        await q.join()
        raise asyncio.CancelledError   # dừng worker
```

Trong thực tế dùng sentinel rõ ràng hơn là `CancelledError` thủ công.

## 18.9 Structured concurrency với `anyio`

`anyio` chạy được trên cả `asyncio` và `trio` — hữu ích cho thư viện.

```bash
pip install anyio
```

```python
import anyio

async def main() -> None:
    async with anyio.create_task_group() as tg:
        tg.start_soon(anyio.sleep, 1)
        tg.start_soon(anyio.sleep, 2)

anyio.run(main)
```

### `anyio` chuyển đổi backend

```python
# Chạy trên trio
anyio.run(main, backend="trio")
```

### `anyio.to_thread.run_sync`

```python
async def main() -> None:
    ket_qua = await anyio.to_thread.run_sync(tinh_nang, 10**7)
```

### `anyio` với capacity limiter

```python
limiter = anyio.CapacityLimiter(10)
await anyio.to_thread.run_sync(func, limiter=limiter)
```

## 18.10 Sai lầm thường gặp

### Quên `await`

```python
async def main() -> None:
    asyncio.sleep(1)      # SAI — chỉ tạo coroutine, không chạy
    await asyncio.sleep(1)  # ĐÚNG
```

Python cảnh báo `RuntimeWarning: coroutine was never awaited` khi coroutine bị GC.

### Blocking trong async

```python
async def main() -> None:
    time.sleep(5)              # SAI — chặn event loop
    requests.get(url)          # SAI — sync HTTP
    open("file").read()        # SAI — sync I/O lớn
```

Đúng: `asyncio.sleep`, `aiohttp`/`httpx`, `aiofiles`.

### Tạo task trong vòng lặp không giới hạn

```python
# SAI — 1 triệu task cùng lúc, cạn bộ nhớ
tasks = [asyncio.create_task(tai(u)) for u in urls]

# ĐÚNG — giới hạn bằng semaphore
sem = asyncio.Semaphore(100)
```

### Dùng `asyncio.get_event_loop()` trong hàm async

```python
# Cũ, deprecated
loop = asyncio.get_event_loop()

# Đúng trong coroutine
loop = asyncio.get_running_loop()
```

## 18.11 Đo và debug async

```python
import asyncio

asyncio.run(main(), debug=True)
```

Chế độ debug:

- Cảnh báo coroutine không await.
- Cảnh báo callback chậm > 100ms.
- Log khi task bị GC mà chưa xong.

### Đo task chậm

```python
async def main() -> None:
    loop = asyncio.get_running_loop()
    loop.slow_callback_duration = 0.05
```

### Xem task đang chạy

```python
for task in asyncio.all_tasks():
    print(task.get_name(), task.get_coro())
```

## 18.12 Ví dụ — crawler có giới hạn

```python
import asyncio
import httpx
from urllib.parse import urljoin

async def crawl(
    start: str,
    max_trang: int = 50,
    max_dong_thoi: int = 10,
) -> dict[str, int]:
    ket_qua: dict[str, int] = {}
    sem = asyncio.Semaphore(max_dong_thoi)
    da_tham: set[str] = set()

    async with httpx.AsyncClient(timeout=10, follow_redirects=True) as client:
        async def tham(url: str) -> None:
            async with sem:
                if url in da_tham or len(da_tham) >= max_trang:
                    return
                da_tham.add(url)
                try:
                    r = await client.get(url)
                    ket_qua[url] = r.status_code
                except httpx.HTTPError as e:
                    ket_qua[url] = -1

        async with asyncio.TaskGroup() as tg:
            tg.create_task(tham(start))

    return ket_qua

asyncio.run(crawl("https://example.com"))
```

## 18.13 Bài tập

1. Viết `TaskGroup` tải 100 URL, tối đa 20 đồng thời, timeout tổng 30s.
2. Viết producer-consumer có sentinel, đo throughput khi `maxsize` là 1, 10, 1000.
3. Viết retry với exponential backoff + jitter, hủy được.
4. So sánh `asyncio.to_thread` và `run_in_executor` cho hàm CPU-bound.
5. Viết hàm `gather_an_toan` mô phỏng `TaskGroup` bằng `asyncio.gather` + hủy thủ công.
6. Bắt `ExceptionGroup` từ TaskGroup, phân loại theo loại lỗi.

## 18.14 Ghi chú kỹ thuật

- `TaskGroup` yêu cầu Python 3.11+. Trên 3.10 dùng `anyio.create_task_group`.
- `except*` không thể trộn với `except` thường trong cùng `try`.
- Hủy task là cooperative — code không `await` sẽ không bị hủy.
- `Semaphore` công bằng theo FIFO; không có starvation.
- Mỗi coroutine tốn ~1-2KB, so với ~8MB stack của thread.
- `httpx` hỗ trợ cả sync và async; `aiohttp` chỉ async nhưng nhanh hơn.
- Không gọi `asyncio.run` trong code đã có event loop (Jupyter, FastAPI) — dùng `await`.

Stashed.