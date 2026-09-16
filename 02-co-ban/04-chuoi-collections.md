# Chương 04 — Chuỗi Nâng Cao và Collections

## 4.1 List — mảng động

List là dãy có thứ tự, thay đổi được, cho phép trùng lặp.

```python
a = [1, 2, 3]
a = list(range(5))
a = [0] * 5              # [0, 0, 0, 0, 0]
a = [x * 2 for x in range(5)]   # [0, 2, 4, 6, 8]
```

### Thêm / xóa

```python
a = [1, 2, 3]
a.append(4)              # [1,2,3,4] — thêm cuối
a.insert(0, 0)           # [0,1,2,3,4] — thêm vị trí
a.extend([5, 6])         # [0,1,2,3,4,5,6]
a.remove(0)              # xóa phần tử ĐẦU TIÊN có giá trị 0
x = a.pop()              # lấy và xóa cuối
y = a.pop(0)             # lấy và xóa theo vị trí
del a[0]                 # xóa theo vị trí
a.clear()                # xóa hết
```

`remove` ném `ValueError` nếu không có. `pop` ném `IndexError` nếu list rỗng và không có index.

### Sắp xếp

```python
a = [3, 1, 4, 1, 5, 9, 2, 6]
a.sort()                     # tại chỗ, tăng dần
a.sort(reverse=True)         # giảm dần
b = sorted(a)                # trả list mới

ds = [("An", 8), ("Bình", 9), ("Cường", 7)]
ds.sort(key=lambda t: t[1])          # sắp theo điểm
ds.sort(key=lambda t: (-t[1], t[0])) # điểm giảm, tên tăng
```

`sort` ổn định (stable): phần tử bằng nhau giữ nguyên thứ tự tương đối.

### Đảo ngược

```python
a.reverse()          # tại chỗ
list(reversed(a))    # trả iterable
a[::-1]              # tạo list mới đảo
```

### Tìm kiếm

```python
a = [1, 2, 3, 2]
a.index(2)      # 1 — vị trí đầu tiên
a.count(2)      # 2
2 in a          # True
5 in a          # False
```

### Slicing

```python
a = [0, 1, 2, 3, 4, 5]
a[1:4]     # [1, 2, 3]
a[:3]      # [0, 1, 2]
a[3:]      # [3, 4, 5]
a[::2]     # [0, 2, 4]
a[1::2]    # [1, 3, 5]
a[::-1]    # [5, 4, 3, 2, 1, 0]
a[-2:]     # [4, 5]
```

### Copy — cạm bẫy lớn nhất

```python
a = [1, 2, 3]
b = a              # b trỏ CÙNG đối tượng
b.append(4)
print(a)           # [1, 2, 3, 4] — a cũng đổi!
```

Ba mức copy:

```python
import copy

a = [1, 2, [3, 4]]

b = a                      # tham chiếu
c = a.copy()               # shallow — list mới, nhưng phần tử lồng vẫn chung
d = list(a)                # shallow
e = copy.copy(a)           # shallow
f = copy.deepcopy(a)       # deep — copy đệ quy toàn bộ

c[2].append(99)
print(a)                   # [1, 2, [3, 4, 99]] — vì list lồng chung
f[2].append(99)
print(a)                   # không đổi — deepcopy tách hoàn toàn
```

### List lồng (ma trận)

```python
matrix = [
    [1, 2, 3],
    [4, 5, 6],
    [7, 8, 9],
]

for hang in matrix:
    for x in hang:
        print(x, end=" ")
    print()

# Tạo ma trận n x m — CẨN THẬN
sai = [[0] * 3] * 3        # SAI — 3 hàng trỏ cùng list
sai[0][0] = 1
print(sai)                 # [[1,0,0],[1,0,0],[1,0,0]]

dung = [[0] * 3 for _ in range(3)]
dung[0][0] = 1
print(dung)                # [[1,0,0],[0,0,0],[0,0,0]]
```

### Phép toán trên list

```python
[1, 2] + [3, 4]      # [1, 2, 3, 4]
[0] * 3              # [0, 0, 0]
len([1, 2, 3])       # 3
min([3, 1, 2])       # 1
max([3, 1, 2])       # 3
sum([1, 2, 3])       # 6
any([False, True])   # True
all([True, True])    # True
```

### Độ phức tạp

| Thao tác | Độ phức tạp |
|----------|-------------|
| `a[i]` | O(1) |
| `a.append(x)` | O(1) khấu hao |
| `a.pop()` | O(1) |
| `a.insert(0, x)` | O(n) |
| `a.pop(0)` | O(n) |
| `x in a` | O(n) |
| `a.sort()` | O(n log n) |

Chèn/xóa đầu list chậm — dùng `collections.deque` nếu cần thao tác hai đầu.

## 4.2 Tuple — bất biến

```python
t = (1, 2, 3)
t = 1, 2, 3          # ngoặc tùy chọn
t = (1,)             # tuple một phần tử — CẦN dấu phẩy
empty = ()

t[0]                 # 1
# t[0] = 9           # TypeError
```

### Vì sao dùng tuple

- Bất biến → an toàn khi truyền qua hàm.
- Hashable → dùng làm key dict hoặc phần tử set.
- Nhanh hơn list một chút.
- Ngữ nghĩa: "bộ giá trị cố định" (tọa độ, RGB, record).

### Unpacking

```python
x, y, z = (1, 2, 3)
a, *rest = (1, 2, 3, 4)      # a=1, rest=[2,3,4]
*init, last = (1, 2, 3, 4)   # init=[1,2,3], last=4
first, *mid, last = (1,2,3,4,5)  # first=1, mid=[2,3,4], last=5
```

### Hoán đổi

```python
a, b = 1, 2
a, b = b, a     # a=2, b=1
```

### Named tuple — tuple có tên trường

```python
from collections import namedtuple

Point = namedtuple("Point", ["x", "y"])
p = Point(1, 2)
print(p.x, p.y)       # 1 2
print(p[0])           # 1
print(p._asdict())    # {'x': 1, 'y': 2}
p2 = p._replace(x=10)
```

Python 3.6+ có `typing.NamedTuple` với type hints:

```python
from typing import NamedTuple

class Point(NamedTuple):
    x: float
    y: float

p = Point(1.0, 2.0)
```

## 4.3 Set — tập hợp

Set không thứ tự, không trùng lặp.

```python
s = {1, 2, 3}
s = set([1, 2, 2, 3])    # {1, 2, 3}
s = set()                 # rỗng — không dùng {} vì đó là dict
```

### Thêm / xóa

```python
s = {1, 2, 3}
s.add(4)
s.discard(1)         # không lỗi nếu không có
s.remove(2)          # ValueError nếu không có
x = s.pop()          # lấy và xóa một phần tử tùy ý
s.clear()
```

### Phép toán tập hợp

```python
a = {1, 2, 3}
b = {3, 4, 5}

a | b        # hợp: {1,2,3,4,5}
a & b        # giao: {3}
a - b        # hiệu: {1,2}
b - a        # {4,5}
a ^ b        # đối xứng: {1,2,4,5}

a.union(b)
a.intersection(b)
a.difference(b)
a.symmetric_difference(b)
```

### Quan hệ

```python
{1, 2} <= {1, 2, 3}     # True — tập con
{1, 2, 3} >= {1, 2}     # True — tập cha
{1, 2}.isdisjoint({3, 4})   # True — không giao
```

### Ứng dụng

```python
# Khử trùng lặp (mất thứ tự)
ds = [1, 2, 2, 3, 3, 3]
duy_nhat = list(set(ds))

# Giữ thứ tự + khử trùng lặp
seen = set()
ket_qua = []
for x in ds:
    if x not in seen:
        seen.add(x)
        ket_qua.append(x)
```

### Độ phức tạp

`in`, `add`, `remove` là O(1) trung bình. Đây là lý do set nhanh hơn list khi kiểm tra tồn tại.

## 4.4 Dict — bảng băm

Dict lưu cặp key-value, key phải hashable.

```python
d = {"a": 1, "b": 2}
d = dict(a=1, b=2)
d = dict([("a", 1), ("b", 2)])
d = {x: x*x for x in range(5)}
```

### Truy cập

```python
d = {"a": 1, "b": 2}

d["a"]              # 1
d["z"]              # KeyError
d.get("z")          # None
d.get("z", 0)       # 0
"a" in d            # True
```

### Thêm / sửa / xóa

```python
d["c"] = 3
d.update({"d": 4, "e": 5})
d.setdefault("f", 6)      # đặt nếu chưa có, trả giá trị hiện tại
del d["a"]
x = d.pop("b")
d.pop("z", None)          # không lỗi
d.clear()
```

### Lặp

```python
d = {"a": 1, "b": 2}
for k in d:               # key
    print(k)
for v in d.values():
    print(v)
for k, v in d.items():
    print(k, v)
```

### Gộp dict

```python
a = {"x": 1}
b = {"y": 2}

c = {**a, **b}      # {'x':1, 'y':2}
c = a | b           # Python 3.9+
a |= b              # cập nhật tại chỗ
a.update(b)
```

### Dict comprehension

```python
bp = {x: x*x for x in range(5)}
# {0:0, 1:1, 2:4, 3:9, 4:16}

dao = {v: k for k, v in {"a": 1, "b": 2}.items()}
# {1: 'a', 2: 'b'}

loc = {k: v for k, v in d.items() if v > 0}
```

### Ứng dụng — đếm tần suất

```python
from collections import Counter

c = Counter("abracadabra")
print(c)                       # Counter({'a': 5, 'b': 2, 'r': 2, 'c': 1, 'd': 1})
print(c.most_common(2))        # [('a', 5), ('b', 2)]
print(c["a"])                  # 5
print(c["z"])                  # 0 — không lỗi
```

### Độ phức tạp

`get`, `set`, `del`, `in` là O(1) trung bình. Worst case O(n) khi hash collision nhiều (hiếm).

## 4.5 `collections` hữu ích

### `defaultdict`

```python
from collections import defaultdict

dd = defaultdict(list)
dd["a"].append(1)      # không cần khởi tạo
dd["a"].append(2)
print(dd)              # defaultdict(<class 'list'>, {'a': [1, 2]})

dd = defaultdict(int)
for ch in "hello":
    dd[ch] += 1
print(dict(dd))
```

### `deque` — hàng đợi hai đầu

```python
from collections import deque

dq = deque([1, 2, 3])
dq.append(4)           # phải
dq.appendleft(0)       # trái
dq.pop()               # lấy phải
dq.popleft()           # lấy trái
dq.rotate(1)           # xoay
dq.extend([5, 6])
dq.extendleft([-1, -2])
```

`deque` thêm/xóa hai đầu O(1). Dùng làm queue thay list.

### `Counter`

```python
from collections import Counter

c = Counter(["a", "b", "a", "c", "a"])
c["a"]                 # 3

c.update(["a", "d"])
c.subtract(["a"])
c.most_common(2)

# Phép toán
c1 = Counter("aab")
c2 = Counter("abc")
c1 + c2    # cộng
c1 - c2    # trừ (bỏ âm)
c1 & c2    # min
c1 | c2    # max
```

### `OrderedDict`

Dict thường từ 3.7 giữ thứ tự chèn. `OrderedDict` vẫn hữu ích khi cần `move_to_end` hoặc so sánh thứ tự:

```python
from collections import OrderedDict

od = OrderedDict([("a", 1), ("b", 2)])
od.move_to_end("a")
```

### `ChainMap`

```python
from collections import ChainMap

defaults = {"color": "red", "size": 10}
user = {"color": "blue"}
cfg = ChainMap(user, defaults)
cfg["color"]    # 'blue' — user ưu tiên
cfg["size"]     # 10
```

## 4.6 Xử lý chuỗi nâng cao — regex

```python
import re

text = "Ngày 12/03/2024, nhiệt độ 30.5 độ C, giá 1,234,567 VND"
```

### Tìm kiếm

```python
re.findall(r"\d+", text)
# ['12', '03', '2024', '30', '5', '1', '234', '567']

re.search(r"\d+/\d+/\d+", text).group()   # '12/03/2024'
re.match(r"Ngày", text)                    # match ở đầu chuỗi
re.fullmatch(r"\d+", "123")                # khớp toàn bộ
```

### Thay thế

```python
re.sub(r"\d+", "#", text)
re.sub(r"(\d+)/(\d+)/(\d+)", r"\3-\2-\1", text)   # đảo ngày
```

### Nhóm có tên

```python
m = re.search(r"(?P<ngay>\d{2})/(?P<thang>\d{2})/(?P<nam>\d{4})", text)
m.group("ngay")     # '12'
m.group("thang")    # '03'
m.group("nam")      # '2024'
m.groupdict()
```

### Tách

```python
re.split(r"\s*,\s*", "a, b,  c")
# ['a', 'b', 'c']
```

### Cờ (flags)

```python
re.findall(r"ngày", text, re.IGNORECASE)
re.findall(r"^\d+", "1\n2\n3", re.MULTILINE)   # ['1','2','3']
re.findall(r"a.b", "a\nb", re.DOTALL)          # '.' khớp cả newline
re.VERBOSE   # cho phép regex nhiều dòng có comment
```

### Biên dịch regex dùng lại

```python
pattern = re.compile(r"\d{4}-\d{2}-\d{2}")
pattern.findall(text)
```

### Ví dụ — trích email

```python
emails = re.findall(r"[\w.+-]+@[\w-]+\.[\w.-]+", text)
```

### Ví dụ — validate số điện thoại VN

```python
def la_sdt_vn(s: str) -> bool:
    return bool(re.fullmatch(r"0\d{9}", s))

print(la_sdt_vn("0912345678"))   # True
print(la_sdt_vn("912345678"))    # False
```

### Cảnh báo regex

- Backtracking có thể gây DoS (ReDoS). Tránh `(a+)+b` với input dài.
- Không dùng regex để parse HTML/JSON — dùng parser chuyên dụng.
- `re` không hỗ trợ lookbehind variable-length (dùng `regex` package nếu cần).

## 4.7 Phương thức chuỗi nâng cao khác

```python
"a b c".split(maxsplit=1)         # ['a', 'b c']
"a,b,,c".split(",")               # ['a', 'b', '', 'c']
"line1\nline2".splitlines()       # ['line1', 'line2']
"abc".partition("b")              # ('a', 'b', 'c')
"abc".rpartition("b")             # ('a', 'b', 'c')

"hello".maketrans({"h": "H"})     # bảng dịch
"hello".translate(str.maketrans("lo", "10"))   # 'he110'

"Hello World".removeprefix("Hello ")   # Python 3.9+
"Hello World".removesuffix(" World")
```

### `str.format` (cách cũ, vẫn dùng cho template)

```python
"{0} {1}".format("a", "b")
"{ten} {tuoi}".format(ten="Nam", tuoi=25)
"{:>10}|".format("hi")
```

## 4.8 Bài tập

### Bài 1 — Đếm từ, top 5
```python
from collections import Counter

text = input("Đoạn văn: ").lower()
words = re.findall(r"\w+", text)
for w, c in Counter(words).most_common(5):
    print(f"{w}: {c}")
```

### Bài 2 — Tách chẵn lẻ
```python
ds = list(range(1, 21))
chan = [x for x in ds if x % 2 == 0]
le = [x for x in ds if x % 2]
```

### Bài 3 — Giao hai list
```python
a = [1, 2, 3, 4]
b = [3, 4, 5, 6]
giao = list(set(a) & set(b))
```

### Bài 4 — Đảo dict
```python
d = {"a": 1, "b": 2}
dao = {v: k for k, v in d.items()}
```

### Bài 5 — Trích email
```python
emails = re.findall(r"[\w.+-]+@[\w-]+\.[\w.-]+", text)
```

### Bài 6 — Điểm trung bình sinh viên
```python
sinh_vien = [
    {"ten": "An", "diem": [8, 9, 7]},
    {"ten": "Bình", "diem": [6, 7, 8]},
]

for sv in sinh_vien:
    tb = sum(sv["diem"]) / len(sv["diem"])
    print(f"{sv['ten']}: {tb:.2f}")
```

### Bài 7 — Nhóm theo chữ cái đầu
```python
from collections import defaultdict

ten = ["An", "Bình", "Cường", "Bảo", "Chi"]
nhom = defaultdict(list)
for t in ten:
    nhom[t[0]].append(t)
print(dict(nhom))
```

### Bài 8 — Ma trận chuyển vị
```python
m = [[1, 2, 3], [4, 5, 6]]
t = [list(row) for row in zip(*m)]
# [[1, 4], [2, 5], [3, 6]]
```

## 4.9 Ghi chú kỹ thuật

- List, dict, set đều là mutable; tuple và str là immutable.
- Chỉ các đối tượng hashable mới làm key dict / phần tử set. List không hashable.
- `dict` từ Python 3.7 giữ thứ tự chèn theo đặc tả ngôn ngữ.
- `set` không có thứ tự — đừng dựa vào thứ tự khi lặp.
- `Counter` trả `0` cho key thiếu, khác dict thường (KeyError).
- Regex nên được compile một lần nếu dùng lại trong vòng lặp lớn.

Stashed.