# Phụ Lục A — Regex Cheat Sheet

## A.1 Cú pháp cơ bản

| Mẫu | Ý nghĩa |
|-----|---------|
| `.` | bất kỳ ký tự nào trừ newline |
| `\d` | chữ số `[0-9]` |
| `\D` | không phải chữ số |
| `\w` | chữ, số, gạch dưới `[a-zA-Z0-9_]` |
| `\W` | không phải `\w` |
| `\s` | khoảng trắng `[ \t\n\r\f\v]` |
| `\S` | không phải khoảng trắng |
| `\b` | ranh giới từ |
| `\B` | không phải ranh giới từ |
| `^` | đầu chuỗi (hoặc đầu dòng với MULTILINE) |
| `$` | cuối chuỗi (hoặc cuối dòng với MULTILINE) |

## A.2 Lượng từ (quantifier)

| Mẫu | Ý nghĩa |
|-----|---------|
| `*` | 0 hoặc nhiều |
| `+` | 1 hoặc nhiều |
| `?` | 0 hoặc 1 |
| `{n}` | đúng n lần |
| `{n,}` | ít nhất n |
| `{n,m}` | n đến m |
| `*?` | không tham lam (lazy) |
| `+?` | lazy |
| `??` | lazy |

Mặc định là tham lam (greedy) — khớp nhiều nhất có thể.

```python
import re

text = "<a><b>"
re.findall(r"<.+>", text)     # ['<a><b>'] — greedy
re.findall(r"<.+?>", text)    # ['<a>', '<b>'] — lazy
```

## A.3 Nhóm và tham chiếu

| Mẫu | Ý nghĩa |
|-----|---------|
| `(...)` | nhóm bắt |
| `(?:...)` | nhóm không bắt |
| `(?P<name>...)` | nhóm có tên |
| `\1` | tham chiếu nhóm 1 |
| `(?P=name)` | tham chiếu nhóm có tên |
| `(?=...)` | lookahead dương |
| `(?!...)` | lookahead âm |
| `(?<=...)` | lookbehind dương (độ dài cố định) |
| `(?<!...)` | lookbehind âm |

```python
re.match(r"(\d+)-(\d+)", "12-34").groups()   # ('12', '34')

re.sub(r"(\w+)@(\w+)", r"\2.\1", "user@host")   # 'host.user'

# Lookahead: giá theo sau là VND
re.findall(r"\d+(?= VND)", "100 VND, 200 USD")   # ['100']

# Lookbehind: số đứng sau $
re.findall(r"(?<=\$)\d+", "$100 and 200")   # ['100']
```

## A.4 Lớp ký tự

```python
re.findall(r"[aeiou]", "hello world")     # ['e', 'o', 'o']
re.findall(r"[^aeiou]", "hello")          # ['h', 'l', 'l']
re.findall(r"[a-z0-9]", "aB1-")           # ['a', '1']
re.findall(r"[\w.-]+", "file.txt")        # ['file.txt']
```

Bên trong `[]`, hầu hết ký tự đặc biệt mất nghĩa, trừ `^`, `-`, `]`, `\`.

## A.5 Flags

```python
re.IGNORECASE   # re.I — không phân biệt hoa thường
re.MULTILINE    # re.M — ^ $ khớp đầu/cuối dòng
re.DOTALL       # re.S — . khớp cả newline
re.VERBOSE      # re.X — cho phép comment, khoảng trắng trong pattern
re.ASCII        # \w \d \s chỉ ASCII
```

```python
re.findall(r"^\w+", "abc\ndef", re.MULTILINE)   # ['abc', 'def']
re.findall(r"a.b", "a\nb", re.DOTALL)            # ['a\nb']
```

## A.6 Hàm re

```python
import re

re.match(r"\d+", "123abc")      # match ở đầu, trả Match hoặc None
re.search(r"\d+", "abc123")     # tìm khắp chuỗi
re.fullmatch(r"\d+", "123")     # khớp toàn bộ
re.findall(r"\d+", "a1b22")     # ['1', '22']
re.finditer(r"\d+", "a1b22")    # iterator Match
re.split(r"\s+", "a  b   c")    # ['a', 'b', 'c']
re.sub(r"\d+", "#", "a1b22")    # 'a#b#'
re.subn(r"\d+", "#", "a1b22")   # ('a#b#', 2)
```

### Match object

```python
m = re.search(r"(\d+)-(\d+)", "ngày 12-03")
m.group()       # '12-03'
m.group(0)      # '12-03'
m.group(1)      # '12'
m.group(2)      # '03'
m.groups()      # ('12', '03')
m.start(), m.end()   # vị trí
m.span()        # (start, end)
```

## A.7 Biên dịch trước

```python
pattern = re.compile(r"\d{4}-\d{2}-\d{2}")
pattern.findall(text)
```

Compile một lần, dùng nhiều lần — nhanh hơn vì không biên dịch lại.

## A.8 Mẫu thường dùng

### Email (đơn giản)

```python
r"[\w.+-]+@[\w-]+\.[\w.-]+"
```

### URL

```python
r"https?://[\w./%-]+"
```

### Số điện thoại VN

```python
r"0\d{9}"
```

### Ngày ISO

```python
r"\d{4}-\d{2}-\d{2}"
```

### IPv4

```python
r"\b(?:\d{1,3}\.){3}\d{1,3}\b"
```

### Mật khẩu mạnh

```python
r"^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[^\w\s]).{8,}$"
```

Giải thích: ít nhất 1 chữ thường, 1 hoa, 1 số, 1 ký tự đặc biệt, dài ≥ 8.

### Tách tên file

```python
m = re.match(r"(?P<name>.+)\.(?P<ext>\w+)$", "report.final.pdf")
m.group("name")   # 'report.final'
m.group("ext")    # 'pdf'
```

### Số có dấu phân cách

```python
r"\d{1,3}(?:,\d{3})*(?:\.\d+)?"
```

## A.9 Cạm bẫy

### Tham lam vô tình

```python
# SAI — khớp cả đoạn giữa
re.findall(r"<.*>", "<a>text</a>")   # ['<a>text</a>']

# ĐÚNG
re.findall(r"<.*?>", "<a>text</a>")  # ['<a>', '</a>']
```

### Lookbehind độ dài thay đổi

```python
re.search(r"(?<=ab+)c", "abbbc")   # LỖI — lookbehind phải cố định
```

Dùng package `regex` nếu cần lookbehind thay đổi.

### ReDoS — backtracking

```python
# Pattern nguy hiểm
r"(a+)+b"

# Chuỗi dài toàn 'a' không có 'b' → thời gian mũ
```

Tránh lồng lượng từ. Test với input dài.

### Không dùng regex cho HTML

Dùng `BeautifulSoup` hoặc `lxml`. HTML không phải ngôn ngữ chính quy.

## A.10 Ví dụ tổng hợp — parser log

```python
import re
from collections import Counter

LOG = re.compile(
    r"(?P<ip>\d+\.\d+\.\d+\.\d+)\s+-\s+-\s+"
    r"\[(?P<time>[^\]]+)\]\s+"
    r'"(?P<method>\w+)\s+(?P<path>\S+)\s+HTTP/[\d.]+"\s+'
    r"(?P<status>\d{3})\s+(?P<size>\d+)"
)

def phan_tich(path: str):
    status_counter = Counter()
    path_counter = Counter()

    with open(path, encoding="utf-8") as f:
        for line in f:
            m = LOG.match(line)
            if not m:
                continue
            status_counter[m.group("status")] += 1
            path_counter[m.group("path")] += 1

    return status_counter, path_counter.most_common(5)
```

## A.11 Ghi chú

- Luôn dùng raw string `r"..."` cho pattern.
- Compile pattern dùng lại trong vòng lặp.
- Test regex với `re.DEBUG` để xem cách biên dịch.
- Package `regex` hỗ trợ Unicode đầy đủ và lookbehind thay đổi.

Stashed.