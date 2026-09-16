# Chương 02 — Biến, Kiểu Dữ Liệu, Chuỗi

## 2.1 Biến là gì (ở mức bộ nhớ)

Trong Python, biến không phải "hộp chứa giá trị" — nó là **tên trỏ tới một đối tượng**. Đối tượng nằm trên heap; biến giữ tham chiếu tới nó.

```python
x = 10
y = x          # y trỏ cùng đối tượng với x
x = 20         # x trỏ đối tượng mới; y vẫn là 10
```

Minh họa bằng `id()`:

```python
a = [1, 2, 3]
b = a
print(id(a) == id(b))   # True — cùng một đối tượng

c = a.copy()
print(id(a) == id(c))   # False — bản sao khác
```

Đây là gốc rễ của mọi lỗi "sao đổi cái này mà cái kia cũng đổi".

## 2.2 Các kiểu dữ liệu cơ bản

| Kiểu | Ví dụ | Ghi chú |
|------|-------|---------|
| `int` | `42`, `-7`, `10**100` | Số nguyên chính xác tùy ý |
| `float` | `3.14`, `1e-9`, `float('inf')` | Số thực IEEE 754, 64-bit |
| `complex` | `2+3j` | Số phức |
| `bool` | `True`, `False` | Lớp con của `int` (True == 1) |
| `str` | `"hello"`, `'a'` | Chuỗi Unicode bất biến |
| `bytes` | `b"abc"` | Dãy byte bất biến |
| `bytearray` | `bytearray(b"abc")` | Dãy byte thay đổi được |
| `NoneType` | `None` | Chỉ có một giá trị |

Kiểm tra kiểu:

```python
type(42)                # <class 'int'>
isinstance(42, int)     # True
isinstance(True, int)   # True — bool là int
isinstance(3.0, int)    # False
```

### Vì sao `bool` là con của `int`

```python
True + True      # 2
sum([True, False, True])   # 2
```

Tiện trong một số trường hợp, nhưng cẩn thận khi so sánh: `True == 1` là `True`.

## 2.3 Số nguyên (int)

Python 3 không có giới hạn kích thước int (ngoài bộ nhớ máy).

```python
print(2 ** 100)
# 1267650600228229401496703205376

print(10 ** 50)
# 100000000000000000000000000000000000000000000000000
```

### Hệ cơ số

```python
0b1010        # 10 (nhị phân)
0o17          # 15 (bát phân)
0xff          # 255 (thập lục phân)
int("ff", 16) # 255
int("1010", 2) # 10
bin(255)      # '0b11111111'
hex(255)      # '0xff'
oct(8)        # '0o10'
```

### Số học

```python
a, b = 7, 2

a + b     # 9
a - b     # 5
a * b     # 14
a / b     # 3.5   (luôn float)
a // b    # 3     (chia lấy nguyên, làm tròn xuống)
a % b     # 1     (chia lấy dư)
a ** b    # 49    (lũy thừa)
divmod(a, b)   # (3, 1) — vừa thương vừa dư
abs(-5)   # 5
pow(2, 10)      # 1024
pow(2, 10, 100) # 24 — (2^10) % 100, hiệu quả cho số mũ lớn
```

### Chia âm — cẩn thận

```python
-7 // 2   # -4 (làm tròn xuống, không phải về 0)
-7 % 2    # 1  (dấu theo mẫu số)
7 % -2    # -1
```

## 2.4 Số thực (float)

Float là số IEEE 754 double precision — 53 bit mantissa, ~15-17 chữ số thập phân chính xác.

```python
print(0.1 + 0.2)           # 0.30000000000000004
print(0.1 + 0.2 == 0.3)    # False
```

### Vì sao — không phải bug, là giới hạn

`0.1` trong nhị phân là số vô hạn tuần hoàn, không biểu diễn chính xác được. Mọi ngôn ngữ dùng IEEE 754 đều gặp.

### Cách so sánh float đúng

```python
import math

math.isclose(0.1 + 0.2, 0.3)             # True
math.isclose(0.1 + 0.2, 0.3, rel_tol=1e-9)
```

### Giá trị đặc biệt

```python
float('inf')      # dương vô cực
float('-inf')     # âm vô cực
float('nan')      # not a number

math.isinf(float('inf'))   # True
math.isnan(float('nan'))   # True

float('nan') == float('nan')   # False (!) — NaN không bằng chính nó
```

### Làm tròn

```python
round(3.14159, 2)    # 3.14
round(2.5)           # 2 — banker's rounding, không phải 3
round(3.5)           # 4

import math
math.floor(3.7)      # 3
math.ceil(3.2)       # 4
math.trunc(3.9)      # 3
int(3.9)             # 3 — cắt phần thập phân
```

### Kiểm tra `round(2.5) == 2`

Python dùng "round half to even" (banker's rounding) để giảm sai số tích lũy. Nếu cần làm tròn theo cách học đường, dùng `Decimal`.

## 2.5 Decimal và Fraction — số chính xác

### Decimal — cho tiền tệ

```python
from decimal import Decimal, getcontext

getcontext().prec = 28   # số chữ số chính xác

Decimal("0.1") + Decimal("0.2")   # Decimal('0.3')
Decimal("1.1") * Decimal("3")      # Decimal('3.3')

# Không tạo Decimal từ float
Decimal(0.1)          # Decimal('0.1000000000000000055511151231257827021181583404541015625')
Decimal("0.1")        # Decimal('0.1')  ← đúng
```

### Fraction — phân số chính xác

```python
from fractions import Fraction

Fraction(1, 3) + Fraction(1, 6)   # Fraction(1, 2)
Fraction(2, 4)                     # Fraction(1, 2) — tự rút gọn
Fraction(0.5)                      # Fraction(1, 2)
```

## 2.6 Ép kiểu

```python
int("42")        # 42
int(3.9)         # 3 (cắt, không làm tròn)
int("0xff", 16)  # 255
float("3.14")    # 3.14
float("inf")     # inf
str(42)          # "42"
str([1, 2])      # "[1, 2]"
bool(0)          # False
bool("")         # False
bool([])         # False
bool("False")    # True — chuỗi không rỗng luôn True
list("abc")      # ['a', 'b', 'c']
tuple([1, 2])    # (1, 2)
set([1, 1, 2])   # {1, 2}
```

### Ép kiểu thất bại

```python
int("abc")       # ValueError
int(None)        # TypeError
```

Nên bắt lỗi khi input không tin cậy:

```python
def nhap_so(prompt: str) -> int | None:
    raw = input(prompt).strip()
    try:
        return int(raw)
    except ValueError:
        print("Không phải số nguyên hợp lệ.")
        return None
```

## 2.7 Chuỗi (str)

Chuỗi là **bất biến** (immutable). Mọi "thay đổi" thực chất tạo chuỗi mới.

```python
s = "Python"
s[0] = "J"   # TypeError — không sửa được
s = "J" + s[1:]   # tạo chuỗi mới
```

### Truy cập và cắt (slicing)

```python
s = "Python"
s[0]      # 'P'
s[-1]     # 'n'
s[1:4]    # 'yth'
s[:3]     # 'Pyt'
s[3:]     # 'hon'
s[::2]    # 'Pto'
s[::-1]   # 'nohtyP'
s[10]     # IndexError
s[1:100]  # 'ython' — slicing không lỗi khi vượt biên
```

### Phương thức chuỗi

```python
s = "  Hello World  "

s.strip()                 # "Hello World"
s.lstrip()                # "Hello World  "
s.rstrip()                # "  Hello World"
s.lower()                 # "  hello world  "
s.upper()                 # "  HELLO WORLD  "
s.title()                 # "  Hello World  "
s.capitalize()            # "  hello world  "
s.swapcase()              # "  hELLO wORLD  "

"a,b,c".split(",")        # ['a', 'b', 'c']
"a  b   c".split()        # ['a', 'b', 'c'] — tách theo khoảng trắng
"-".join(["a", "b"])      # "a-b"
"".join(["a", "b"])       # "ab"

"abc".replace("b", "X")   # "aXc"
"abc".startswith("ab")    # True
"abc".endswith("bc")      # True
"abc".find("b")           # 1
"abc".find("z")           # -1
"abc".index("z")          # ValueError

"42".zfill(5)             # "00042"
"abc".center(7, "-")      # "--abc--"
"abc".ljust(5)            # "abc  "
"abc".rjust(5)            # "  abc"
```

### Kiểm tra nội dung

```python
"abc123".isalnum()   # True
"123".isdigit()      # True
"abc".isalpha()      # True
"abc".islower()      # True
"ABC".isupper()      # True
" \t\n".isspace()    # True
"Hello World".istitle()  # True
```

### Đếm và tìm

```python
"banana".count("a")     # 3
"banana".count("na")    # 2
"banana".find("na")     # 2
"banana".rfind("na")    # 4
"a" in "cat"            # True
"z" not in "cat"        # True
```

## 2.8 f-string (Python 3.6+)

Cách định dạng mạnh nhất, đọc rõ nhất.

```python
ten = "Nam"
tuoi = 25

f"{ten} {tuoi} tuổi"        # "Nam 25 tuổi"
f"{3.14159:.2f}"            # "3.14"
f"{255:08b}"                # "11111111"
f"{255:x}"                  # "ff"
f"{1000000:,}"              # "1,000,000"
f"{0.85:.1%}"               # "85.0%"
f"{'hi':>10}"               # "        hi"
f"{'hi':<10}|"              # "hi        |"
f"{'hi':^10}|"              # "    hi    |"
f"{tuoi=}"                  # "tuoi=25"  (Python 3.8+)
f"{ten.upper()}"            # "NAM"
f"{[x*2 for x in range(3)]}"  # "[0, 2, 4]"
```

### Định dạng số

```python
f"{1234.5678:,.2f}"    # "1,234.57"
f"{0.00012:.2e}"       # "1.20e-04"
f"{-5:+d}"             # "-5"
f"{5:+d}"              # "+5"
f"{42:5d}|"            # "   42|"
```

### f-string lồng

```python
width = 10
value = 3.14
f"{value:{width}.2f}"   # "      3.14"
```

### Khi nào KHÔNG dùng f-string

- Chuỗi template dùng lại nhiều lần → `str.format` hoặc `string.Template`.
- Logging → dùng `%`-style hoặc tham số lazy (logging chỉ format khi cần):

```python
import logging
logging.info("Người dùng %s đăng nhập", ten)   # tốt — lazy
logging.info(f"Người dùng {ten} đăng nhập")    # luôn format, kể cả khi không log
```

## 2.9 Chuỗi nhiều dòng, raw string, escape

```python
text = """Dòng 1
Dòng 2
Dòng 3"""

path = r"C:\Users\Nam\file.txt"    # raw: không xử lý \n, \t

# Escape
print("Dòng 1\nDòng 2")
print("Tab:\there")
print("Trích dẫn: \"hello\"")
print("Backslash: \\")
print("Unicode: \u00e9")     # é
```

### Nối chuỗi

```python
s = "abc" "def"      # "abcdef" — nối literal
s = "abc" + "def"    # "abcdef"

# Nối nhiều phần tử — dùng join, không dùng + trong vòng lặp
parts = ["a", "b", "c"]
"-".join(parts)      # "a-b-c"
```

### Vì sao `+=` trong vòng lặp chậm

```python
# Chậm O(n^2) — mỗi lần tạo chuỗi mới
result = ""
for w in words:
    result += w

# Nhanh O(n)
result = "".join(words)
```

## 2.10 So sánh chuỗi

```python
"abc" == "abc"        # True
"abc" == "ABC"        # False
"abc" < "abd"         # True — so theo mã Unicode từng ký tự
"Z" < "a"             # True — 'Z' là 90, 'a' là 97
"abc" == 'abc'        # True — nháy đơn/kép tương đương
```

So sánh không phân biệt hoa thường:

```python
"Hello".casefold() == "hello".casefold()   # True
```

`casefold()` mạnh hơn `lower()` cho Unicode quốc tế (ví dụ tiếng Đức ß).

## 2.11 None

`None` là singleton — chỉ có một đối tượng `None` trong toàn bộ chương trình.

```python
x = None
x is None         # True  ← luôn dùng is
x == None         # True nhưng không khuyến khích
type(None)        # <class 'NoneType'>
```

Hàm không `return` trả về `None`:

```python
def khong_tra_ve():
    pass

print(khong_tra_ve())   # None
```

## 2.12 bytes và bytearray

```python
b = b"hello"          # bytes — bất biến
b[0]                  # 104 (số nguyên, không phải ký tự)
b.decode("utf-8")     # "hello"

ba = bytearray(b"hello")   # thay đổi được
ba[0] = 72                 # 'H'
bytes(ba)                  # b"Hello"
ba.decode("utf-8")         # "Hello"

"hello".encode("utf-8")    # b"hello"
```

Bytes là giao diện với file nhị phân, mạng, crypto.

## 2.13 Bài tập

### Bài 1 — Đảo chuỗi và đếm
```python
s = input("Chuỗi: ")
print(f"Đảo ngược: {s[::-1]}")
print(f"Số ký tự: {len(s)}")
```

### Bài 2 — Diện tích hình tròn
```python
import math
r = float(input("Bán kính: "))
print(f"Diện tích: {math.pi * r ** 2:.4f}")
```

### Bài 3 — Đổi giây
```python
tong = int(input("Số giây: "))
h, r = divmod(tong, 3600)
m, s = divmod(r, 60)
print(f"{h:02d}:{m:02d}:{s:02d}")
```

### Bài 4 — Palindrome
```python
s = input("Chuỗi: ").strip().casefold()
loc = "".join(c for c in s if c.isalnum())
print("Palindrome" if loc == loc[::-1] else "Không")
```

### Bài 5 — Đếm từ
```python
text = input("Đoạn văn: ")
words = text.split()
print(f"Số từ: {len(words)}")
print(f"Từ dài nhất: {max(words, key=len)}")
```

### Bài 6 — Định dạng tiền
```python
tien = float(input("Số tiền: "))
print(f"{tien:,.2f} VND")
```

### Bài 7 — Kiểm tra số chính phương
```python
import math
n = int(input("Số: "))
print("Chính phương" if math.isqrt(n) ** 2 == n else "Không")
```

## 2.14 Ghi chú kỹ thuật

- `int` trong Python là đối tượng có chi phí bộ nhớ (28 byte cho số nhỏ). Nếu cần mảng số hiệu năng cao, dùng `array` hoặc `numpy`.
- So sánh chuỗi là O(min(len(a), len(b))) và dừng ở ký tự đầu khác.
- `str` trong Python 3 là Unicode (code point). `bytes` là dãy byte thô.
- `hash()` của `str` được làm ngẫu nhiên hóa (hash randomization) để chống tấn công collision — không dựa vào giá trị hash cụ thể.

Stashed.