# Chương 11 — Concurrency, Song Song, Asyncio

## 11.1 Phân biệt các khái niệm

- **Concurrency** (đồng thời): nhiều tác vụ cùng tiến triển, xen kẽ nhau.
- **Parallelism** (song song): nhiều tác vụ chạy **thực sự cùng lúc** trên nhiều core.
- **Đồng bộ** (sync): chờ tác vụ trước xong mới làm tiếp.
- **Bất đồng bộ** (async): không chờ, chuyển sang việc khác.

## 11.2 GIL — Global Interpreter Lock

CPython dùng GIL: tại một thời điểm chỉ một thread chạy bytecode Python.

Hệ quả:

- **I/O-bound**: threading hiệu quả — GIL nhả khi chờ I/O.
- **CPU-bound**: threading gần như vô ích — dùng multiprocessing.
- Python 3.13 có bản free-threaded (không GIL) thử nghiệm, chưa production-ready.

```python
import sys
print(sys._is_gil_enabled())   # 3.13+, kiểm tra GIL
```

## 11.3 Threading

```python
import threading
import time

def viec(n):
    time.sleep(1)
    print(f"xong {n}")

threads = [threading.Thread(target=viec, args=(i,)) for i in range(5)]
for t in threads:
    t.start()
for t in threads:
    t.join()

print("tất cả xong")
```

5 thread chạy song song, tổng thời gian ~1s thay vì 5s.

### ThreadPoolExecutor

```python
from concurrent.futures import ThreadPoolExecutor
import urllib.request

def tai(url):
    with urllib.request.urlopen(url, timeout=5) as r:
        return len(r.read())

urls = [f"https://example.com/{i}" for i in range(10)]
with ThreadPoolExecutor(max_workers=8) as ex:
    ket_qua = list(ex.map(tai, urls))
```

### submit và future

```python
with ThreadPoolExecutor() as ex:
    futures = [ex.submit(tai, u) for u in urls]
    for f in futures:
        print(f.result())
```

### Đồng bộ hóa — Lock

```python
import threading

lock = threading.Lock()
counter = 0

def tang():
    global counter
    for _ in range(100_000):
        with lock:
            counter += 1

threads = [threading.Thread(target=tang) for _ in range(4)]
for t in threads: t.start()
for t in threads: t.join()
print(counter)   # 400000
```

Không có lock, `counter` sẽ sai do race condition.

### RLock — lock tái nhập

```python
lock = threading.RLock()
with lock:
    with lock:      # OK với RLock, deadlock với Lock
        pass
```

### Event, Semaphore, Condition

```python
import threading

event = threading.Event()

def cho():
    event.wait()
    print("đã được báo")

t = threading.Thread(target=cho)
t.start()
event.set()

sem = threading.Semaphore(3)   # tối đa 3 truy cập đồng thời
with sem:
    ...
```

### Queue — an toàn giữa thread

```python
from queue import Queue

q = Queue()
for i in range(10):
    q.put(i)

while not q.empty():
    print(q.get())
```

`queue.Queue` thread-safe. `PriorityQueue` và `LifoQueue` cũng có.

## 11.4 Multiprocessing

Mỗi process có bộ nhớ riêng, chạy song song thật, tránh GIL.

```python
from concurrent.futures import ProcessPoolExecutor

def tinh(n):
    return sum(i * i for i in range(n))

if __name__ == "__main__":
    with ProcessPoolExecutor() as ex:
        print(list(ex.map(tinh, [10**6, 10**6, 10**6])))
```

**Bắt buộc** `if __name__ == "__main__"` trên Windows và macOS (spawn method).

### Chia sẻ dữ liệu

```python
from multiprocessing import Process, Queue, Value, Array

def worker(q):
    q.put("kết quả")

if __name__ == "__main__":
    q = Queue()
    p = Process(target=worker, args=(q,))
    p.start()
    print(q.get())
    p.join()
```

### Pool

```python
from multiprocessing import Pool

with Pool(4) as p:
    print(p.map(abs, [-1, -2, -3]))
```

### Cạm bẫy

- Mỗi process có bộ nhớ riêng — truyền dữ liệu lớn tốn thời gian serialize.
- Khởi tạo process tốn ~10-100ms.
- Object truyền giữa process phải pickle được.

## 11.5 Asyncio

Lập trình bất đồng bộ một luồng với event loop.

```python
import asyncio

async def chao(ten):
    await asyncio.sleep(1)
    print(f"Chào {ten}")

async def main():
    await asyncio.gather(
        chao("A"), chao("B"), chao("C"),
    )

asyncio.run(main())
```

Ba coroutine chạy đồng thời, tổng ~1s.

### Coroutine, Task, Future

- **Coroutine**: hàm `async def`, phải `await` hoặc schedule.
- **Task**: coroutine đã được schedule lên event loop.
- **Future**: kết quả sẽ có trong tương lai.

```python
async def main():
    task = asyncio.create_task(chao("X"))
    print("đang chờ...")
    await task
```

### gather vs wait

```python
async def main():
    ket_qua = await asyncio.gather(coro1(), coro2())
    # trả list kết quả, ném ngoại lệ nếu có

    done, pending = await asyncio.wait([coro1(), coro2()])
```

`gather` giữ thứ tự kết quả; `wait` trả tập hợp.

### Timeout

```python
async def main():
    try:
        await asyncio.wait_for(asyncio.sleep(10), timeout=1)
    except asyncio.TimeoutError:
        print("quá thời gian")
```

3.11+ có `asyncio.timeout()`:

```python
async def main():
    async with asyncio.timeout(1):
        await asyncio.sleep(10)
```

### Async HTTP với aiohttp

```python
import asyncio
import aiohttp

async def tai(session, url):
    async with session.get(url) as r:
        return await r.text()

async def main():
    async with aiohttp.ClientSession() as session:
        tasks = [tai(session, f"https://example.com/{i}") for i in range(20)]
        ket_qua = await asyncio.gather(*tasks)
        print(len(ket_qua))

asyncio.run(main())
```

### Async context manager và iterator

```python
class TaiNguyen:
    async def __aenter__(self):
        return self
    async def __aexit__(self, *args):
        pass

async def dem():
    for i in range(3):
        await asyncio.sleep(0.1)
        yield i

async def main():
    async for x in dem():
        print(x)
```

### Hàng đợi async

```python
async def san_xuat(q):
    for i in range(5):
        await q.put(i)
        await asyncio.sleep(0.1)

async def tieu_thu(q):
    while True:
        item = await q.get()
        print("nhận", item)
        q.task_done()

async def main():
    q = asyncio.Queue()
    await asyncio.gather(san_xuat(q), tieu_thu(q))
```

### Semaphore async — giới hạn đồng thời

```python
async def main():
    sem = asyncio.Semaphore(10)

    async def co_gioi_han(url):
        async with sem:
            return await tai(url)

    await asyncio.gather(*[co_gioi_han(u) for u in urls])
```

### Chạy hàm đồng bộ trong async

```python
import asyncio

def tinh_nang():
    return sum(range(10**7))

async def main():
    ket_qua = await asyncio.to_thread(tinh_nang)
    # hoặc chạy trong process pool
    loop = asyncio.get_running_loop()
    ket_qua = await loop.run_in_executor(None, tinh_nang)
```

`asyncio.to_thread` (3.9+) đơn giản hơn.

## 11.6 Chọn mô hình nào

| Loại tác vụ | Công cụ |
|-------------|---------|
| I/O-bound, ít kết nối | threading |
| I/O-bound, nhiều kết nối | asyncio |
| CPU-bound | multiprocessing |
| Tính toán số | numpy / C extension / Rust |
| Vừa I/O vừa CPU | asyncio + process pool |

## 11.7 Ví dụ — so sánh hiệu năng

```python
import time
import requests
from concurrent.futures import ThreadPoolExecutor
import asyncio
import aiohttp

URLS = [f"https://httpbin.org/delay/1" for _ in range(10)]

def sync():
    t0 = time.perf_counter()
    for u in URLS:
        requests.get(u, timeout=10)
    return time.perf_counter() - t0

def threaded():
    t0 = time.perf_counter()
    with ThreadPoolExecutor(max_workers=10) as ex:
        list(ex.map(lambda u: requests.get(u, timeout=10), URLS))
    return time.perf_counter() - t0

async def async_():
    async with aiohttp.ClientSession() as s:
        async def get(u):
            async with s.get(u) as r:
                return await r.read()
        t0 = time.perf_counter()
        await asyncio.gather(*[get(u) for u in URLS])
        return time.perf_counter() - t0
```

Kết quả điển hình: sync ~10s, threaded ~1s, async ~1s.

## 11.8 Bài tập

### Bài 1 — Tải URL bằng threading
Đo thời gian, so với tuần tự.

### Bài 2 — asyncio + aiohttp
Làm lại bài 1, so sánh.

### Bài 3 — CPU-bound với multiprocessing
Tính tổng bình phương 10 triệu số, so với tuần tự.

### Bài 4 — Producer-consumer
Dùng `queue.Queue`, một thread sản xuất, hai thread tiêu thụ.

### Bài 5 — Async retry với backoff
```python
import asyncio

async def retry(coro_func, max_lan=3, delay=1.0):
    for lan in range(max_lan):
        try:
            return await coro_func()
        except Exception as e:
            if lan == max_lan - 1:
                raise
            await asyncio.sleep(delay * 2 ** lan)
```

### Bài 6 — Semaphore giới hạn
Giới hạn 10 request đồng thời trong 100 request.

## 11.9 Ghi chú kỹ thuật

- `asyncio.gather` mặc định propagate exception đầu tiên; dùng `return_exceptions=True` để lấy tất cả.
- Thread trong Python có overhead ~8MB stack mỗi thread; process ~20MB+.
- `asyncio` không nhanh hơn threading cho CPU-bound — chỉ tốt cho I/O.
- `concurrent.futures` thống nhất API cho thread và process.
- `queue.Queue` cho thread; `asyncio.Queue` cho async; `multiprocessing.Queue` cho process.

Stashed.