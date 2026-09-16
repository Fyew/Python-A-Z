# Phụ Lục B — Debugging và Xử Lý Lỗi

## B.1 Đọc traceback

```text
Traceback (most recent call last):
  File "main.py", line 10, in <module>
    ket_qua = chia(a, b)
  File "utils.py", line 4, in chia
    return a / b
ZeroDivisionError: division by zero
```

Đọc từ **dưới lên**: dòng cuối là loại lỗi và thông điệp; các dòng trên là đường đi từ điểm gọi tới điểm lỗi.

## B.2 breakpoint() — debugger tích hợp

Python 3.7+ có `breakpoint()`.

```python
def xu_ly(ds):
    ket_qua = []
    for x in ds:
        breakpoint()          # dừng ở đây
        ket_qua.append(x * 2)
    return ket_qua
```

Trong pdb:

| Lệnh | Ý nghĩa |
|------|---------|
| `n` / `next` | dòng tiếp theo |
| `s` / `step` | bước vào hàm |
| `c` / `continue` | chạy tiếp |
| `q` / `quit` | thoát |
| `p expr` | in biểu thức |
| `pp expr` | pretty print |
| `l` / `list` | hiện code xung quanh |
| `w` / `where` | stack trace |
| `b N` | đặt breakpoint dòng N |
| `cl N` | xóa breakpoint |
| `u` / `d` | lên/xuống frame |

## B.3 pdb từ dòng lệnh

```bash
python -m pdb script.py
```

Chạy script, dừng ngay dòng đầu.

### Post-mortem

```python
import pdb

try:
    ham_loi()
except Exception:
    pdb.post_mortem()
```

Hoặc chạy lại script lỗi với:

```bash
python -m pdb script.py
# rồi c
```

Dừng tại điểm lỗi.

## B.4 Print debugging có kỷ luật

```python
def ham(a, b):
    print(f"[DEBUG] ham({a=}, {b=})")   # 3.8+
    ...
```

`f"{a=}"` tự in tên biến — gọn hơn `f"a={a}"`.

### Dùng logging thay print

```python
import logging

logging.basicConfig(level=logging.DEBUG, format="%(levelname)s %(message)s")
log = logging.getLogger(__name__)

log.debug("giá trị: %s", value)
```

Tắt bằng `level=logging.INFO` khi production.

### icecream

```bash
pip install icecream
```

```python
from icecream import ic

def ham(a, b):
    ic(a, b)
    return a + b

ic(ham(2, 3))
# ic| a: 2, b: 3
# ic| ham(2, 3): 5
```

## B.5 Traceback module

```python
import traceback

try:
    ham()
except Exception:
    traceback.print_exc()

# Lấy chuỗi
s = traceback.format_exc()

# In stack hiện tại
traceback.print_stack()
```

### In traceback gọn

```python
traceback.print_exc(limit=2)   # chỉ 2 frame gần nhất
```

## B.6 Logging traceback

```python
import logging

logger = logging.getLogger(__name__)

try:
    ham()
except Exception:
    logger.exception("Lỗi khi chạy ham")
```

`logger.exception` tự thêm traceback — chỉ dùng trong `except`.

## B.7 Các loại lỗi thường gặp

| Lỗi | Nguyên nhân | Cách sửa |
|-----|-------------|----------|
| `SyntaxError` | Sai cú pháp | Đọc dòng báo |
| `IndentationError` | Thụt lề sai | Dùng 4 space |
| `NameError` | Biến chưa khai báo | Kiểm tra scope |
| `TypeError` | Sai kiểu | Ép kiểu |
| `ValueError` | Đúng kiểu, sai giá trị | Validate |
| `KeyError` | Dict thiếu key | Dùng `.get` |
| `IndexError` | List vượt biên | Kiểm tra len |
| `AttributeError` | Object không có attr | Kiểm tra class |
| `ImportError` | Module lỗi | Cài lại |
| `ModuleNotFoundError` | Chưa cài | `pip install` |
| `FileNotFoundError` | File thiếu | Kiểm tra path |
| `PermissionError` | Thiếu quyền | `chmod` / sudo |
| `ZeroDivisionError` | Chia 0 | Kiểm tra mẫu |
| `RecursionError` | Đệ quy quá sâu | Vòng lặp |
| `MemoryError` | Hết RAM | Tối ưu |
| `UnicodeDecodeError` | Encoding sai | Chỉ định `utf-8` |

## B.8 Assertions

```python
assert x > 0, f"x phải dương, nhận {x}"
```

`assert` bị tắt khi chạy `python -O`. Không dùng cho validation production.

## B.9 Kiểm tra giả định

```python
def chia(a, b):
    assert b != 0, "mẫu không được bằng 0"
    return a / b
```

Chỉ dùng trong development/test.

## B.10 Reproduce bug

Quy trình:

1. **Reproduce** — tạo ví dụ nhỏ nhất lặp lại lỗi.
2. **Isolate** — thu hẹp phạm vi, xóa code không liên quan.
3. **Understand** — hiểu vì sao lỗi.
4. **Fix** — sửa nguyên nhân, không sửa triệu chứng.
5. **Test** — viết test ngăn lỗi tái xuất.

### Minimal reproduction

```python
# Thay vì chạy cả app, tạo script nhỏ
import minimal_lib

# chỉ gọi hàm lỗi
minimal_lib.ham_loi(args)
```

## B.11 Rubber duck debugging

Giải thích vấn đề thành lời — cho một người, một con vịt cao su, hoặc chính mình. Quá trình diễn đạt thường làm lộ giả định sai.

## B.12 Debug trong IDE

VS Code:

- `F5` — chạy debugger.
- Đặt breakpoint bằng click lề trái.
- Xem biến trong panel Variables.
- Watch expression.
- Call stack panel.

PyCharm: tương tự với UI mạnh hơn.

## B.13 Log context

```python
import logging
import uuid
from contextvars import ContextVar

request_id: ContextVar[str] = ContextVar("request_id", default="")

class ContextFilter(logging.Filter):
    def filter(self, record):
        record.request_id = request_id.get()
        return True

logging.basicConfig(format="%(request_id)s %(levelname)s %(message)s")
logging.getLogger().addFilter(ContextFilter())

request_id.set(str(uuid.uuid4()))
logging.info("bắt đầu request")
```

Truy vết request qua log.

## B.14 Debug async

```python
import asyncio

async def main():
    await asyncio.sleep(1)

asyncio.run(main(), debug=True)
```

`debug=True` bật cảnh báo coroutine không await, task chậm.

## B.15 Debug memory

```bash
pip install memory-profiler tracemalloc
```

```python
import tracemalloc

tracemalloc.start()
# code
snapshot = tracemalloc.take_snapshot()
for stat in snapshot.statistics("lineno")[:10]:
    print(stat)
```

### objgraph

```python
import objgraph
objgraph.show_most_common_types(limit=10)
objgraph.show_growth()
```

## B.16 Debug deadlock

```python
import threading
import faulthandler

faulthandler.dump_traceback_later(10, exit=True)
```

In stack của tất cả thread sau 10s. Hoặc gửi `SIGQUIT`.

## B.17 Kiểm tra hiệu năng

```python
import cProfile, pstats

profiler = cProfile.Profile()
profiler.enable()
# code
profiler.disable()
pstats.Stats(profiler).sort_stats("cumtime").print_stats(20)
```

## B.18 Bài tập

1. Cố tình gây `ZeroDivisionError`, đọc traceback, dùng `breakpoint()` để debug.
2. Viết script lỗi, dùng `pdb.post_mortem()` để điều tra.
3. Dùng `cProfile` tìm hàm chậm nhất trong một script.
4. Dùng `tracemalloc` tìm dòng cấp phát nhiều bộ nhớ nhất.
5. Viết ContextFilter gắn request_id cho log.
6. Dùng `faulthandler` debug chương trình treo.

Stashed.