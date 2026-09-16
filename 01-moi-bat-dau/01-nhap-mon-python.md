# Chương 01 — Nhập Môn Python

## 1.1 Python là gì

Python là ngôn ngữ lập trình bậc cao, thông dịch, đa mục đích, do Guido van Rossum khởi tạo năm 1989 và phát hành lần đầu năm 1991. Tên "Python" lấy từ nhóm hài Monty Python, không phải từ loài rắn — chi tiết nhỏ nhưng đúng.

### Vì sao Python phổ biến

- **Cú pháp gần tiếng Anh.** Đọc code Python dễ hơn C, Java, hay Perl. Người mới học nhanh hơn.
- **Thông dịch (interpreted).** Không cần biên dịch thủ công; sửa file, chạy lại, thấy kết quả ngay.
- **Định kiểu động (dynamic typing).** Biến không cần khai báo kiểu. Nhưng từ 3.5+ có type hints tùy chọn để code lớn vẫn rõ ràng.
- **Quản lý bộ nhớ tự động.** Garbage collector dọn object không còn dùng. Không có `free()` thủ công.
- **Hệ sinh thái khổng lồ.** PyPI có hơn 500.000 gói: web, khoa học dữ liệu, AI, DevOps, tự động hóa.
- **Đa nền tảng.** Cùng một file chạy trên Linux, Windows, macOS, và cả vi điều khiển (MicroPython).

### Python dùng để làm gì

| Lĩnh vực | Thư viện / framework tiêu biểu |
|----------|-------------------------------|
| Web backend | Django, Flask, FastAPI |
| Khoa học dữ liệu | NumPy, Pandas, Polars |
| Machine learning | scikit-learn, PyTorch, TensorFlow |
| Tự động hóa | Ansible, Fabric, Playwright |
| DevOps / script | subprocess, pathlib, click |
| API / microservice | FastAPI, gRPC, Celery |
| Phân tích bảo mật | Scapy, pwntools, Volatility |
| Game / đồ họa | pygame, Panda3D |
| Desktop app | Tkinter, PyQt, PySide |

### Python không phù hợp với việc gì

- Lập trình nhúng real-time cứng (dùng C/Rust).
- Game engine hiệu năng cao (dùng C++).
- Nơi cần tiết kiệm bộ nhớ tối đa đến từng byte.
- Số học hiệu năng cao thuần (nhưng có thể gọi C extension, NumPy).

## 1.2 Triết lý ngôn ngữ — Zen of Python

Gõ trong REPL:

```text
>>> import this
The Zen of Python, by Tim Peters

Beautiful is better than ugly.
Explicit is better than implicit.
Simple is better than complex.
Complex is better than complicated.
Flat is better than nested.
Sparse is better than dense.
Readability counts.
...
```

Một vài điểm đáng nhớ:

- **Explicit hơn implicit.** Code nên nói rõ ý định, không để người đọc tự đoán.
- **Simple hơn complex.** Chọn cách đơn giản nhất giải quyết vấn đề.
- **Readability counts.** Code được đọc nhiều hơn viết. Ưu tiên dễ đọc.
- **Errors should never pass silently.** Đừng nuốt lỗi.

## 1.3 Cài đặt Python

### Kiểm tra phiên bản hiện có

```bash
python3 --version
python --version
which python3
```

Sách này dùng Python 3.12+. Không dùng Python 2 (đã ngừng hỗ trợ từ 2020).

### Linux (Debian / Ubuntu)

```bash
sudo apt update
sudo apt install -y python3 python3-pip python3-venv python3-dev build-essential
python3 --version
```

### Fedora / RHEL

```bash
sudo dnf install -y python3 python3-pip python3-devel gcc
```

### macOS

```bash
brew install python@3.12
```

Đừng dùng Python hệ thống của macOS cho project — nó chỉ phục vụ hệ điều hành.

### Windows

1. Tải từ python.org (bản 3.12+).
2. Khi cài, **tick "Add python.exe to PATH"** — quên bước này là lỗi phổ biến nhất của người mới.
3. Kiểm tra:

```powershell
python --version
pip --version
```

Có thể dùng Microsoft Store nhưng dễ gặp vấn đề quyền. Bản từ python.org ổn định hơn.

### Cài nhiều phiên bản

- Linux/macOS: `pyenv` quản lý nhiều bản Python.
- Windows: `py` launcher (`py -3.11 script.py`).

```bash
# pyenv
curl https://pyenv.run | bash
pyenv install 3.12.5
pyenv local 3.12.5
```

## 1.4 Môi trường ảo (virtual environment)

Đây là khái niệm sống còn. Mỗi project có một môi trường ảo riêng để dependency không đụng nhau.

```bash
# Tạo
python3 -m venv .venv

# Kích hoạt
source .venv/bin/activate        # Linux/macOS
.venv\Scripts\activate           # Windows CMD
.venv\Scripts\Activate.ps1       # Windows PowerShell

# Kiểm tra
which python                     # phải trỏ vào .venv/bin/python
python --version

# Cài gói trong môi trường ảo
pip install requests

# Thoát
deactivate
```

Quy tắc: **mỗi project một venv**. Không bao giờ `pip install` toàn cục trừ công cụ CLI dùng chung (như `ruff`, `httpie`).

## 1.5 Chạy chương trình đầu tiên

Tạo file `hello.py`:

```python
print("Xin chào, thế giới!")
```

Chạy:

```bash
python3 hello.py
```

Kết quả:

```text
Xin chào, thế giới!
```

### Cách Python chạy một file

1. Đọc file, mã hóa sang bytecode.
2. Biên dịch bytecode (lưu cache `.pyc` trong `__pycache__/`).
3. Chạy bytecode trên máy ảo CPython.
4. Garbage collector dọn bộ nhớ trong suốt quá trình.

## 1.6 REPL — Read-Eval-Print Loop

REPL là môi trường gõ lệnh trực tiếp, không cần tạo file. Rất tiện để thử nhanh.

```text
$ python3
Python 3.12.5 (main, ...) [GCC ...] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> 2 + 3
5
>>> "hello".upper()
'HELLO'
>>> [x for x in range(5)]
[0, 1, 2, 3, 4]
>>> exit()
```

### Biến đặc biệt `_`

`_` lưu kết quả biểu thức vừa tính:

```text
>>> 10 * 10
100
>>> _ + 1
101
```

### REPL nâng cao — IPython

```bash
pip install ipython
ipython
```

IPython có autocomplete, syntax highlight, `%timeit`, `%paste`, lịch sử lệnh — dùng khi làm việc thật.

```text
In [1]: %timeit sum(range(1000))
```

## 1.7 Cú pháp nền tảng

### Thụt lề (indentation)

Python dùng thụt lề để xác định khối lệnh, khác C/Java dùng `{}`.

```python
if True:
    print("trong khối if")
    print("vẫn trong khối")
print("ngoài khối")
```

Chuẩn PEP 8: **4 dấu cách** mỗi cấp. Không trộn tab và space (Python 3 cấm trộn trong cùng file).

Lỗi:

```python
if True:
print("lỗi")   # IndentationError
```

### Comment

```python
# Comment một dòng

"""
Docstring (chuỗi tài liệu) nhiều dòng.
Dùng để mô tả module, class, hàm.
Không phải comment thật — nó là một chuỗi.
"""

def cong(a, b):
    """Cộng hai số và trả về kết quả."""
    return a + b
```

Xem docstring bằng `help()`:

```text
>>> help(cong)
```

### Biến và gán

```python
ten = "Nam"
tuoi = 25
chieu_cao = 1.75
la_lap_trinh_vien = True
khong_co_gi = None
```

Gán nhiều biến:

```python
x, y, z = 1, 2, 3
a = b = c = 0

# Hoán đổi không cần biến tạm
x, y = y, x
```

### Quy tắc đặt tên

- Bắt đầu bằng chữ cái hoặc `_`, không bắt đầu bằng số.
- Chỉ chứa chữ, số, gạch dưới.
- Phân biệt hoa thường (`Ten` khác `ten`).
- Theo `snake_case` cho biến/hàm: `tinh_tong`, `ten_nguoi_dung`.
- `PascalCase` cho class: `SinhVien`, `TaiKhoan`.
- `UPPER_CASE` cho hằng: `PI`, `MAX_SIZE`.
- Tên rõ nghĩa: `diem_trung_binh` tốt hơn `dtb`, `x` tệ.

### Từ khóa (keyword)

Không dùng làm tên biến:

```python
import keyword
print(keyword.kwlist)
# ['False', 'None', 'True', 'and', 'as', 'assert', 'async', 'await',
#  'break', 'class', 'continue', 'def', 'del', 'elif', 'else', 'except',
#  'finally', 'for', 'from', 'global', 'if', 'import', 'in', 'is',
#  'lambda', 'nonlocal', 'not', 'or', 'pass', 'raise', 'return',
#  'try', 'while', 'with', 'yield']
```

### Nhập xuất

```python
ten = input("Tên bạn là gì? ")
print("Chào", ten)
print(f"Chào {ten}, bạn {len(ten)} ký tự.")

# print nhiều tham số
print("a", "b", "c", sep="-")        # a-b-c
print("không xuống dòng", end=" ")
print("tiếp tục")
```

`input()` luôn trả về `str`. Muốn số phải ép kiểu:

```python
tuoi = int(input("Tuổi: "))
```

## 1.8 Chạy script với tham số dòng lệnh

```python
import sys

print("Tên script:", sys.argv[0])
if len(sys.argv) > 1:
    print("Tham số:", sys.argv[1:])
else:
    print("Không có tham số.")
```

```bash
python3 script.py a b c
# Tên script: script.py
# Tham số: ['a', 'b', 'c']
```

Với CLI phức tạp, dùng `argparse`:

```python
import argparse

parser = argparse.ArgumentParser(description="Công cụ demo")
parser.add_argument("ten", help="Tên người dùng")
parser.add_argument("--tuoi", type=int, default=0)
args = parser.parse_args()
print(f"{args.ten}, {args.tuoi} tuổi")
```

## 1.9 Cấu trúc một file Python chuẩn

```python
#!/usr/bin/env python3
"""Module docstring: mô tả mục đích file."""

from __future__ import annotations

import os
import sys
from pathlib import Path


def main() -> int:
    """Điểm vào chương trình."""
    print("Bắt đầu chương trình")
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

### `if __name__ == "__main__"` là gì

Khi file chạy trực tiếp, `__name__ == "__main__"`. Khi file bị import, `__name__` là tên module. Khối này đảm bảo code chỉ chạy khi gọi trực tiếp.

```python
# utils.py
def cong(a, b):
    return a + b

if __name__ == "__main__":
    # chỉ chạy khi: python utils.py
    print(cong(1, 2))
```

```python
# main.py
import utils
utils.cong(1, 2)     # không in gì từ utils
```

### `raise SystemExit(main())` là gì

`main()` trả về `int` làm exit code. `SystemExit` thoát sạch, chạy `finally`, đóng file. Tốt hơn `sys.exit()` trong code có cleanup.

## 1.10 Kiểm tra cú pháp và chạy không thực thi

```bash
python3 -m py_compile hello.py     # chỉ biên dịch, không chạy
python3 -m compileall .            # biên dịch cả thư mục
```

## 1.11 Lỗi thường gặp của người mới

| Lỗi | Nguyên nhân | Cách sửa |
|-----|-------------|----------|
| `IndentationError` | Thụt lề sai / trộn tab-space | Dùng 4 space, cấu hình editor |
| `SyntaxError` | Thiếu `:`, ngoặc không đóng | Đọc kỹ dòng báo lỗi |
| `NameError` | Dùng biến chưa khai báo | Kiểm tra tên, phạm vi |
| `TypeError` | Sai kiểu (cộng str với int) | Ép kiểu rõ ràng |
| `ModuleNotFoundError` | Chưa cài gói / sai venv | `pip install`, kích hoạt venv |
| `PermissionError` | Thiếu quyền file | Kiểm tra quyền, không chạy root bừa |
| `python` không tìm thấy | Chưa thêm vào PATH | Dùng `python3` hoặc thêm PATH |

Đọc traceback từ **dưới lên**: dòng cuối là lỗi thật, các dòng trên là đường đi.

```text
Traceback (most recent call last):
  File "hello.py", line 3, in <module>
    print(ten)
NameError: name 'ten' is not defined
```

## 1.12 Bài tập

### Bài 1 — Chào hỏi
Hỏi tên và tuổi, in `Xin chào [tên], bạn [tuổi] tuổi.`

```python
ten = input("Tên bạn: ")
tuoi = int(input("Tuổi bạn: "))
print(f"Xin chào {ten}, bạn {tuoi} tuổi.")
```

### Bài 2 — Tổng hai số
```python
a = float(input("Số thứ nhất: "))
b = float(input("Số thứ hai: "))
print(f"Tổng: {a + b}")
```

### Bài 3 — Bình phương
In 10 dòng, dòng thứ n in `n^2`.

```python
for n in range(1, 11):
    print(f"{n} bình phương = {n ** 2}")
```

### Bài 4 — Đổi đơn vị
Nhập số giây, in ra `giờ:phút:giây`.

```python
tong_giay = int(input("Số giây: "))
gio = tong_giay // 3600
phut = (tong_giay % 3600) // 60
giay = tong_giay % 60
print(f"{gio:02d}:{phut:02d}:{giay:02d}")
```

### Bài 5 — Kiểm tra palindrome
```python
s = input("Nhập chuỗi: ").strip().lower()
print("Palindrome" if s == s[::-1] else "Không phải")
```

## 1.13 Ghi chú kỹ thuật

- `print()` ghi ra `sys.stdout`; lỗi ghi ra `sys.stderr`.
- `input()` đọc một dòng từ `stdin`, bỏ ký tự xuống dòng cuối.
- Mã hóa mặc định của file Python là UTF-8 (PEP 3120). Không cần khai báo coding.
- `__pycache__/` chứa bytecode cache — có thể xóa, Python sẽ tạo lại.
- Không đặt tên file trùng module chuẩn (ví dụ `random.py`, `socket.py`) — sẽ che module chuẩn và gây lỗi khó hiểu.

Stashed.