# Chương 03 — Toán Tử và Điều Khiển Luồng

## 3.1 Toán tử so sánh

```python
3 > 2        # True
3 >= 3       # True
3 < 4        # True
3 <= 3       # True
3 == 3.0     # True
3 != 4       # True
"a" < "b"    # True
[1, 2] < [1, 3]   # True — so từng phần tử
(1, 2) == (1, 2)  # True
```

Python cho phép so sánh chuỗi (xếp theo Unicode) và list/tuple (xếp từ điển).

### So sánh chuỗi (chained comparison)

```python
x = 5
0 < x < 10        # True, tương đương 0 < x and x < 10
1 < x < 3 < 10    # False
```

Python đánh giá `x` một lần duy nhất trong chuỗi so sánh — hiệu quả và tránh side effect.

## 3.2 Toán tử logic

```python
True and False    # False
True or False     # True
not True          # False
```

### Short-circuit

`and` trả về toán hạng đầu tiên falsy, hoặc toán hạng cuối nếu tất cả truthy.
`or` trả về toán hạng đầu tiên truthy, hoặc toán hạng cuối nếu tất cả falsy.

```python
0 and 5           # 0
5 and 0           # 0
5 and 10          # 10
0 or 5            # 5
5 or 0            # 5
None or "default" # "default"
```

Ứng dụng — giá trị mặc định:

```python
ten = input_ten or "Khách"
config = config or {}
```

### Giá trị falsy

`False`, `None`, `0`, `0.0`, `0j`, `""`, `[]`, `()`, `{}`, `set()`, `range(0)`, và bất kỳ object có `__bool__` trả `False` hoặc `__len__` trả `0`.

```python
if []:
    print("không chạy")

class Rong:
    def __len__(self):
        return 0

bool(Rong())   # False
```

## 3.3 Toán tử gán kết hợp

```python
x = 5
x += 3    # 8
x -= 1    # 7
x *= 2    # 14
x /= 2    # 7.0
x //= 2   # 3.0
x %= 2    # 1.0
x **= 2   # 1.0
```

Cũng có `&=`, `|=`, `^=`, `>>=`, `<<=`.

### Walrus operator `:=` (Python 3.8+)

Gán trong biểu thức:

```python
# Không dùng walrus
line = input()
while line != "quit":
    print(line)
    line = input()

# Dùng walrus — gọn hơn
while (line := input()) != "quit":
    print(line)
```

```python
if (n := len(data)) > 10:
    print(f"Quá dài: {n}")
```

## 3.4 Toán tử identity và membership

```python
a = [1, 2]
b = [1, 2]
a == b     # True  — giá trị bằng nhau
a is b     # False — khác đối tượng

a is None       # False
x is not None   # True

1 in a          # True
3 not in a      # True
"a" in "cat"    # True
"key" in {"key": 1}   # True (kiểm tra key)
```

### `is` vs `==`

- `is`: so danh tính đối tượng (cùng một chỗ trong bộ nhớ).
- `==`: so giá trị.

Dùng `is` cho `None`, `True`, `False`. Dùng `==` cho số, chuỗi, collection.

### Bẫy nhỏ — integer caching

```python
a = 256
b = 256
a is b    # True — Python cache số nhỏ

a = 257
b = 257
a is b    # có thể False (tùy implementation)
```

Đừng bao giờ dựa vào hành vi này. Dùng `==` cho số.

## 3.5 Toán tử ba ngôi (conditional expression)

```python
tuoi = 20
trang_thai = "người lớn" if tuoi >= 18 else "trẻ em"
```

Viết lồng (đừng lạm dụng — khó đọc):

```python
loai = "âm" if n < 0 else ("không" if n == 0 else "dương")
```

## 3.6 `if` / `elif` / `else`

```python
diem = 7.5

if diem >= 9:
    print("Xuất sắc")
elif diem >= 7:
    print("Khá")
elif diem >= 5:
    print("Trung bình")
else:
    print("Yếu")
```

### `if` một dòng

```python
if x > 0: print("dương")
```

Chỉ nên dùng khi thân lệnh ngắn và rõ.

### Truthy/falsy trong điều kiện

```python
if items:          # tốt hơn if len(items) > 0
    xu_ly(items)

if not user:       # tốt hơn if user is None or user == ""
    ...
```

Nhưng cẩn thận khi `0` hoặc `""` là giá trị hợp lệ:

```python
if count is not None:   # đúng khi 0 là hợp lệ
    ...
```

## 3.7 `match` / `case` (Python 3.10+)

Cấu trúc so khớp mẫu, mạnh hơn `switch-case` của ngôn ngữ khác.

### Cơ bản

```python
lenh = "start"
match lenh:
    case "start":
        print("Khởi động")
    case "stop" | "halt":
        print("Dừng")
    case _:
        print("Không rõ")
```

### So khớp cấu trúc (destructuring)

```python
diem = (3, 5)

match diem:
    case (0, 0):
        print("Gốc tọa độ")
    case (x, 0):
        print(f"Trên trục X tại {x}")
    case (0, y):
        print(f"Trên trục Y tại {y}")
    case (x, y):
        print(f"Điểm ({x}, {y})")
```

```python
lenh = {"action": "move", "x": 10, "y": 20}

match lenh:
    case {"action": "move", "x": x, "y": y}:
        print(f"Di chuyển tới ({x}, {y})")
    case {"action": "quit"}:
        print("Thoát")
    case _:
        print("Không hiểu")
```

### Guard clause

```python
match diem:
    case (x, y) if x == y:
        print("Trên đường chéo")
    case (x, y):
        print(f"Điểm ({x}, {y})")
```

### Class pattern

```python
class Point:
    __match_args__ = ("x", "y")
    def __init__(self, x, y):
        self.x, self.y = x, y

match Point(1, 2):
    case Point(0, 0):
        print("Gốc")
    case Point(x, y):
        print(f"({x}, {y})")
```

## 3.8 Vòng lặp `for`

```python
for i in range(5):
    print(i)          # 0 1 2 3 4

for i in range(2, 10, 2):
    print(i)          # 2 4 6 8

for i in range(10, 0, -1):
    print(i)          # 10 9 ... 1

for ch in "abc":
    print(ch)         # a b c

for item in [10, 20, 30]:
    print(item)

for k, v in {"a": 1, "b": 2}.items():
    print(k, v)
```

### `enumerate` — vừa index vừa giá trị

```python
for i, ten in enumerate(["An", "Bình", "Cường"]):
    print(i, ten)

for i, ten in enumerate(["An", "Bình"], start=1):
    print(i, ten)     # 1 An, 2 Bình
```

### `zip` — lặp song song

```python
ten = ["An", "Bình"]
tuoi = [20, 25]

for t, u in zip(ten, tuoi):
    print(t, u)

# zip dừng ở iterable ngắn nhất
list(zip([1, 2, 3], ["a", "b"]))   # [(1, 'a'), (2, 'b')]

# zip_longest lấp giá trị thiếu
from itertools import zip_longest
list(zip_longest([1, 2, 3], ["a"], fillvalue="-"))
# [(1, 'a'), (2, '-'), (3, '-')]
```

### `reversed`

```python
for x in reversed([1, 2, 3]):
    print(x)     # 3 2 1
```

### `sorted` với key

```python
ds = [("An", 8), ("Bình", 9), ("Cường", 7)]
for ten, diem in sorted(ds, key=lambda t: -t[1]):
    print(ten, diem)
```

## 3.9 Vòng lặp `while`

```python
n = 0
while n < 5:
    print(n)
    n += 1
```

### Vòng lặp vô hạn có kiểm soát

```python
while True:
    lenh = input("> ").strip()
    if lenh == "quit":
        break
    print("Bạn nhập:", lenh)
```

### `while ... else`

`else` chạy khi vòng lặp kết thúc **bình thường** (không bị `break`):

```python
n = 0
while n < 3:
    print(n)
    n += 1
else:
    print("Xong không break")   # in ra
```

## 3.10 `break`, `continue`, `else` của vòng lặp

```python
for i in range(10):
    if i == 3:
        continue    # bỏ qua phần còn lại của lần lặp này
    if i == 7:
        break       # thoát hẳn vòng lặp
    print(i)        # 0 1 2 4 5 6
```

### Ứng dụng `else` — tìm kiếm

```python
def tim_nguyen_to(ds):
    for n in ds:
        if n > 1 and all(n % i for i in range(2, int(n**0.5)+1)):
            print(f"Tìm thấy: {n}")
            break
    else:
        print("Không có số nguyên tố")
```

`else` chỉ chạy khi không `break` — nghĩa là "không tìm thấy".

## 3.11 `range`

`range` là object lười (lazy), không tạo list.

```python
r = range(1_000_000_000)   # không tốn bộ nhớ
len(r)                      # 1000000000
r[999]                      # 999
```

```python
list(range(5))         # [0, 1, 2, 3, 4]
list(range(2, 8))      # [2, 3, 4, 5, 6, 7]
list(range(10, 0, -2)) # [10, 8, 6, 4, 2]
list(range(0))         # []
```

## 3.12 Ví dụ tổng hợp

### FizzBuzz

```python
for n in range(1, 101):
    if n % 15 == 0:
        print("FizzBuzz")
    elif n % 3 == 0:
        print("Fizz")
    elif n % 5 == 0:
        print("Buzz")
    else:
        print(n)
```

### Số nguyên tố

```python
def la_nguyen_to(n: int) -> bool:
    if n < 2:
        return False
    if n < 4:
        return True
    if n % 2 == 0:
        return False
    i = 3
    while i * i <= n:
        if n % i == 0:
            return False
        i += 2
    return True

print([n for n in range(2, 50) if la_nguyen_to(n)])
```

### Fibonacci

```python
def fib(n: int) -> list[int]:
    a, b = 0, 1
    ket_qua = []
    for _ in range(n):
        ket_qua.append(a)
        a, b = b, a + b
    return ket_qua

print(fib(10))   # [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

### Đoán số

```python
import random

target = random.randint(1, 100)
luot = 0
while True:
    doan = int(input("Đoán 1-100: "))
    luot += 1
    if doan < target:
        print("Nhỏ hơn")
    elif doan > target:
        print("Lớn hơn")
    else:
        print(f"Đúng sau {luot} lượt")
        break
```

### Bảng cửu chương

```python
for i in range(2, 10):
    for j in range(1, 11):
        print(f"{i} x {j} = {i*j}")
    print()
```

### Đảo ngược số

```python
n = 12345
dao = 0
while n > 0:
    dao = dao * 10 + n % 10
    n //= 10
print(dao)   # 54321
```

## 3.13 Bài tập

### Bài 1 — Tổng chẵn
Tính tổng các số chẵn từ 1 đến 100.

```python
print(sum(x for x in range(1, 101) if x % 2 == 0))   # 2550
```

### Bài 2 — Đếm nguyên tố dưới 1000
```python
dem = sum(1 for n in range(2, 1000) if la_nguyen_to(n))
print(dem)
```

### Bài 3 — Tam giác sao
```python
n = 5
for i in range(1, n + 1):
    print("*" * i)
```

### Bài 4 — Bảng cửu chương có định dạng
```python
for i in range(1, 10):
    row = " ".join(f"{i*j:3d}" for j in range(1, 10))
    print(row)
```

### Bài 5 — Kiểm tra năm nhuận
```python
y = int(input("Năm: "))
nhuan = (y % 4 == 0 and y % 100 != 0) or (y % 400 == 0)
print("Nhuận" if nhuan else "Không nhuận")
```

### Bài 6 — Tổng chữ số
```python
n = int(input("Số: "))
tong = 0
while n > 0:
    tong += n % 10
    n //= 10
print(tong)
```

### Bài 7 — Ước chung lớn nhất (Euclid)
```python
def gcd(a, b):
    while b:
        a, b = b, a % b
    return a

print(gcd(48, 18))   # 6
```

## 3.14 Ghi chú kỹ thuật

- `for` trong Python là "for-each" — luôn lặp trên iterable, không phải đếm số như C.
- Không sửa list trong khi lặp trên nó:

```python
# SAI
ds = [1, 2, 3, 4]
for x in ds:
    if x % 2 == 0:
        ds.remove(x)   # bỏ sót phần tử

# ĐÚNG
ds = [x for x in ds if x % 2 != 0]
```

- `range` hỗ trợ `in` và `len` ở O(1).
- Vòng lặp lồng sâu > 3 cấp thường là dấu hiệu cần tách hàm.

Stashed.