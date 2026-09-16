# Chương 06 — File I/O, Ngoại Lệ, Logging

## 6.1 Mở file — cách đúng

```python
with open("data.txt", "r", encoding="utf-8") as f:
    noi_dung = f.read()
```

`with` đảm bảo file được đóng kể cả khi có ngoại lệ. Không dùng `open()` trần rồi quên `close()`.

```python
# SAI — rò rỉ file handle
f = open("data.txt")
data = f.read()
f.close()   # quên dòng này là rò rỉ
```

## 6.2 Các mode mở file

| Mode | Ý nghĩa | Ghi đè? | Tạo mới? |
|------|---------|---------|----------|
| `r` | đọc | - | lỗi nếu không có |
| `w` | ghi | có | có |
| `a` | ghi thêm | không | có |
| `x` | tạo mới | - | lỗi nếu tồn tại |
| `r+` | đọc + ghi | - | lỗi nếu không có |
| `w+` | đọc + ghi | có | có |
| `rb` | đọc nhị phân | - | - |
| `wb` | ghi nhị phân | có | có |

Luôn chỉ định `encoding="utf-8"` khi đọc/ghi text. Mặc định phụ thuộc hệ điều hành (Windows thường là cp1252) — nguồn lỗi Unicode phổ biến.

## 6.3 Đọc file

### Đọc toàn bộ

```python
with open("data.txt", encoding="utf-8") as f:
    noi_dung = f.read()      # chuỗi
```

Tốn bộ nhớ với file lớn.

### Đọc theo dòng (lười)

```python
with open("data.txt", encoding="utf-8") as f:
    for dong in f:
        print(dong.rstrip())
```

Đây là cách hiệu quả — không nạp cả file vào bộ nhớ.

### Đọc tất cả dòng vào list

```python
with open("data.txt", encoding="utf-8") as f:
    ds_dong = f.readlines()   # giữ ký tự \n
```

### Đọc từng dòng

```python
with open("data.txt", encoding="utf-8") as f:
    while (dong := f.readline()):
        print(dong.rstrip())
```

### Đọc N byte

```python
with open("data.bin", "rb") as f:
    chunk = f.read(4096)   # đọc tối đa 4096 byte
```

## 6.4 Ghi file

### Ghi đè

```python
with open("out.txt", "w", encoding="utf-8") as f:
    f.write("Dòng 1\n")
    f.write("Dòng 2\n")
```

### Ghi thêm

```python
with open("out.txt", "a", encoding="utf-8") as f:
    f.write("Thêm vào cuối\n")
```

### Ghi nhiều dòng

```python
ds = ["dòng 1\n", "dòng 2\n", "dòng 3\n"]
with open("out.txt", "w", encoding="utf-8") as f:
    f.writelines(ds)
```

`writelines` không thêm `\n` — phải tự thêm.

## 6.5 File nhị phân

```python
with open("anh.png", "rb") as f:
    header = f.read(8)
    print(header)

with open("copy.png", "wb") as f_out, open("anh.png", "rb") as f_in:
    while chunk := f_in.read(8192):
        f_out.write(chunk)
```

### Copy file an toàn

```python
import shutil

shutil.copy("nguon.txt", "dich.txt")       # copy nội dung
shutil.copy2("nguon.txt", "dich.txt")      # copy cả metadata
shutil.copytree("src_dir", "dst_dir")      # copy thư mục
```

## 6.6 `pathlib` — cách hiện đại

`pathlib` thay thế `os.path` với API hướng đối tượng.

```python
from pathlib import Path

p = Path("data") / "input.txt"
print(p.exists())
print(p.is_file())
print(p.is_dir())
print(p.suffix)      # '.txt'
print(p.stem)        # 'input'
print(p.name)        # 'input.txt'
print(p.parent)      # 'data'
print(p.absolute())
```

### Đọc/ghi nhanh

```python
p = Path("data.txt")
p.write_text("nội dung", encoding="utf-8")
noi_dung = p.read_text(encoding="utf-8")

p.write_bytes(b"\x00\x01")
raw = p.read_bytes()
```

### Duyệt thư mục

```python
for f in Path(".").glob("*.py"):
    print(f)

for f in Path(".").rglob("**/*.py"):
    print(f)

for f in Path(".").iterdir():
    print(f)
```

`rglob` đệ quy.

### Tạo/xóa

```python
Path("newdir").mkdir(exist_ok=True)
Path("a/b/c").mkdir(parents=True, exist_ok=True)
Path("file.txt").unlink(missing_ok=True)
Path("dir").rmdir()   # chỉ xóa thư mục rỗng
```

### Metadata

```python
st = Path("file.txt").stat()
print(st.st_size, st.st_mtime)

import os
print(Path("file.txt").owner())
```

## 6.7 JSON

```python
import json

data = {
    "ten": "Nam",
    "tuoi": 25,
    "so_thich": ["code", "game"],
    "dia_chi": {"thanh_pho": "Hà Nội"},
}
```

### Ghi

```python
with open("data.json", "w", encoding="utf-8") as f:
    json.dump(data, f, ensure_ascii=False, indent=2)
```

`ensure_ascii=False` giữ tiếng Việt không bị escape.

### Đọc

```python
with open("data.json", encoding="utf-8") as f:
    loaded = json.load(f)
```

### Chuỗi

```python
s = json.dumps(data, ensure_ascii=False)
d = json.loads(s)
```

### Kiểu tương ứng

| Python | JSON |
|--------|------|
| `dict` | object |
| `list`, `tuple` | array |
| `str` | string |
| `int`, `float` | number |
| `True/False` | true/false |
| `None` | null |

`datetime`, `Decimal`, `set` không serialize được mặc định — cần custom encoder.

```python
from datetime import datetime

class MyEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, datetime):
            return obj.isoformat()
        if isinstance(obj, set):
            return list(obj)
        return super().default(obj)
```

## 6.8 CSV

```python
import csv

# Ghi
with open("sv.csv", "w", newline="", encoding="utf-8") as f:
    w = csv.writer(f)
    w.writerow(["ten", "diem"])
    w.writerow(["Nam", 8.5])
    w.writerows([["Lan", 9.0], ["Minh", 7.5]])

# Đọc
with open("sv.csv", encoding="utf-8") as f:
    for row in csv.reader(f):
        print(row)

# Đọc dạng dict
with open("sv.csv", encoding="utf-8") as f:
    for row in csv.DictReader(f):
        print(row["ten"], row["diem"])
```

`newline=""` bắt buộc trên Windows để tránh dòng trống.

### DictWriter

```python
with open("out.csv", "w", newline="", encoding="utf-8") as f:
    fieldnames = ["ten", "diem"]
    w = csv.DictWriter(f, fieldnames=fieldnames)
    w.writeheader()
    w.writerow({"ten": "Nam", "diem": 8.5})
```

## 6.9 Ngoại lệ

### Cấu trúc try/except

```python
try:
    x = int("abc")
except ValueError as e:
    print(f"Lỗi giá trị: {e}")
except (TypeError, KeyError) as e:
    print(f"Lỗi khác: {e}")
except Exception as e:
    print(f"Lỗi tổng quát: {e}")
else:
    print("Không có lỗi")
finally:
    print("Luôn chạy")
```

- `try` — code có thể lỗi.
- `except` — bắt ngoại lệ.
- `else` — chạy khi `try` không lỗi.
- `finally` — luôn chạy, kể cả khi có `return` hoặc ngoại lệ.

### Thứ tự bắt ngoại lệ

Bắt cụ thể trước, tổng quát sau:

```python
try:
    ...
except FileNotFoundError:
    ...
except OSError:
    ...
except Exception:
    ...
```

`except Exception` trước `except ValueError` sẽ nuốt hết — sai.

### Cây ngoại lệ

```text
BaseException
├── SystemExit
├── KeyboardInterrupt
├── GeneratorExit
└── Exception
    ├── ArithmeticError
    │   └── ZeroDivisionError
    ├── LookupError
    │   ├── IndexError
    │   └── KeyError
    ├── ValueError
    ├── TypeError
    ├── AttributeError
    ├── NameError
    ├── RuntimeError
    ├── OSError
    │   ├── FileNotFoundError
    │   ├── PermissionError
    │   ├── FileExistsError
    │   └── ConnectionError
    └── ...
```

Không bắt `BaseException` — sẽ chặn `Ctrl+C`.

### Ném ngoại lệ

```python
def chia(a, b):
    if b == 0:
        raise ValueError("Mẫu số không được bằng 0")
    return a / b
```

### Ngoại lệ tùy chỉnh

```python
class LoiNghiepVu(Exception):
    """Lỗi nghiệp vụ chung."""
    pass

class LoiSoDuKhongDu(LoiNghiepVu):
    def __init__(self, so_du, so_tien):
        self.so_du = so_du
        self.so_tien = so_tien
        super().__init__(f"Số dư {so_du} không đủ để rút {so_tien}")

def rut_tien(so_du, so_tien):
    if so_tien > so_du:
        raise LoiSoDuKhongDu(so_du, so_tien)
    return so_du - so_tien
```

### Re-raise và chain

```python
try:
    with open("thieu.txt") as f:
        data = f.read()
except FileNotFoundError as e:
    raise RuntimeError("Không đọc được cấu hình") from e
```

`from e` giữ nguyên nhân gốc trong traceback.

### `raise` trần

```python
try:
    ...
except ValueError:
    log_error()
    raise   # ném lại ngoại lệ hiện tại
```

### `assert`

```python
assert x > 0, "x phải dương"
```

`assert` bị tắt khi chạy với `python -O`. Không dùng cho validation production.

## 6.10 Logging

`print` để debug nhanh, `logging` cho code thật.

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s %(levelname)s %(name)s: %(message)s",
    filename="app.log",
    encoding="utf-8",
)

logger = logging.getLogger(__name__)

logger.debug("Chi tiết debug")
logger.info("Thông tin")
logger.warning("Cảnh báo")
logger.error("Lỗi")
logger.critical("Nghiêm trọng")
```

### Mức log

`DEBUG` < `INFO` < `WARNING` < `ERROR` < `CRITICAL`

`basicConfig(level=logging.INFO)` chỉ hiện từ INFO trở lên.

### Ghi cả ra console và file

```python
import logging
import sys

logger = logging.getLogger(__name__)
logger.setLevel(logging.DEBUG)

console = logging.StreamHandler(sys.stdout)
console.setLevel(logging.INFO)

file = logging.FileHandler("app.log", encoding="utf-8")
file.setLevel(logging.DEBUG)

fmt = logging.Formatter("%(asctime)s %(levelname)s %(message)s")
console.setFormatter(fmt)
file.setFormatter(fmt)

logger.addHandler(console)
logger.addHandler(file)
```

### `logger.exception`

Ghi log kèm traceback — dùng trong `except`.

```python
try:
    x = 1 / 0
except ZeroDivisionError:
    logger.exception("Chia cho 0")
```

### Lazy format

```python
logger.info("Người dùng %s đăng nhập từ %s", user, ip)   # tốt
logger.info(f"Người dùng {user} đăng nhập từ {ip}")        # format luôn
```

Với `%s`, logging chỉ format khi level được bật.

## 6.11 Đọc file lớn

File 10GB không thể `read()` hết.

```python
def doc_tung_dong(path: str):
    with open(path, encoding="utf-8") as f:
        for dong in f:
            yield dong.rstrip()
```

```python
def doc_theo_chunk(path: str, size: int = 8192):
    with open(path, "rb") as f:
        while chunk := f.read(size):
            yield chunk
```

## 6.12 Nén

```python
import gzip
import zipfile
import tarfile

with gzip.open("data.gz", "wt", encoding="utf-8") as f:
    f.write("nội dung")

with gzip.open("data.gz", "rt", encoding="utf-8") as f:
    print(f.read())

with zipfile.ZipFile("archive.zip", "w") as z:
    z.write("file1.txt")

with zipfile.ZipFile("archive.zip") as z:
    z.extractall("out/")
```

## 6.13 Bài tập

### Bài 1 — Đọc file số, tính tổng
```python
def tinh_tong(path: str) -> float:
    tong = 0.0
    with open(path, encoding="utf-8") as f:
        for dong in f:
            try:
                tong += float(dong.strip())
            except ValueError:
                continue
    return tong
```

### Bài 2 — Log có timestamp
```python
import logging
from datetime import datetime

logging.basicConfig(
    filename="app.log",
    level=logging.INFO,
    format="%(asctime)s %(message)s",
)
logging.info("Chương trình bắt đầu")
```

### Bài 3 — Lọc CSV
```python
import csv

with open("sv.csv", encoding="utf-8") as f_in, \
     open("gioi.csv", "w", newline="", encoding="utf-8") as f_out:
    reader = csv.DictReader(f_in)
    writer = csv.DictWriter(f_out, fieldnames=reader.fieldnames)
    writer.writeheader()
    for row in reader:
        if float(row["diem"]) >= 8:
            writer.writerow(row)
```

### Bài 4 — Copy an toàn
```python
import shutil
from pathlib import Path

def copy_an_toan(src: str, dst: str) -> None:
    src_p = Path(src)
    if not src_p.is_file():
        raise FileNotFoundError(src)
    dst_p = Path(dst)
    dst_p.parent.mkdir(parents=True, exist_ok=True)
    shutil.copy2(src_p, dst_p)
```

### Bài 5 — Đọc JSON cấu hình
```python
import json
from pathlib import Path

def doc_config(path: str) -> dict:
    p = Path(path)
    if not p.is_file():
        raise FileNotFoundError(f"Không tìm thấy {path}")
    try:
        return json.loads(p.read_text(encoding="utf-8"))
    except json.JSONDecodeError as e:
        raise ValueError(f"Cấu hình sai định dạng: {e}") from e
```

## 6.14 Ghi chú kỹ thuật

- File handle là tài nguyên giới hạn. Luôn dùng `with`.
- Đọc/ghi file nhị phân không có encoding — byte thô.
- `json.dump` không hỗ trợ `datetime` mặc định.
- `logging` thread-safe; `print` cũng vậy nhưng không có level.
- `Path` immutable, thread-safe, dùng được làm dict key.
- Ngoại lệ quá rộng (`except Exception`) che lỗi thật — chỉ dùng ở biên hệ thống.

Stashed.