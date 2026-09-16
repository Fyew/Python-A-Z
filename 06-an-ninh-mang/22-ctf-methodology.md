# Chương 22 — CTF và Phương Pháp Luận

## CTF là gì

Capture The Flag — cuộc thi bảo mật. Bạn giải thách thức để lấy "cờ" (chuỗi như `flag{...}`). Các dạng chính:

- **Jeopardy** — thách thức theo chủ đề: web, crypto, reverse, pwn, forensics, misc.
- **Attack-Defense** — tấn công đội khác, phòng thủ hệ thống mình.

CTF là cách luyện an toàn nhất: môi trường hợp pháp, có chủ đích, không gây hại thật.

## Quy trình giải một thách thức

```text
1. Đọc kỹ đề — manh mối thường nằm trong tên file, mô tả, metadata
2. Nhận diện dạng — file gì, giao thức gì, ngôn ngữ gì
3. Thu thập thông tin — strings, file, exiftool, hexdump
4. Phân tích — tĩnh trước, động sau
5. Khai thác — viết script tự động hóa
6. Lấy cờ — xác minh
7. Viết writeup — ghi lại cách giải
```

## Phân loại file

```bash
file mystery.bin
xxd mystery.bin | head
binwalk mystery.bin
strings -n 6 mystery.bin
exiftool mystery.jpg
```

```python
import subprocess
from pathlib import Path

def identify(path: str) -> dict:
    result = {}
    result["file"] = subprocess.run(["file", path], capture_output=True, text=True).stdout.strip()
    result["size"] = Path(path).stat().st_size
    with open(path, "rb") as f:
        result["magic"] = f.read(16).hex()
    return result
```

## Forensics — phân tích ảnh, file, memory

### Đọc metadata ảnh

```python
from PIL import Image
from PIL.ExifTags import TAGS

def get_exif(path: str) -> dict:
    img = Image.open(path)
    raw = img.getexif()
    return {TAGS.get(k, k): v for k, v in raw.items()}
```

### Tìm dữ liệu ẩn

```python
def find_appended_data(path: str, expected_end: int) -> bytes:
    with open(path, "rb") as f:
        data = f.read()
    return data[expected_end:]
```

File có dữ liệu thừa sau điểm kết thúc hợp lệ — dấu hiệu steganography hoặc file ẩn.

### Steganography LSB

```python
from PIL import Image

def extract_lsb(path: str, n_bits: int = 8) -> bytes:
    img = Image.open(path).convert("RGB")
    pixels = list(img.getdata())
    bits = []
    for r, g, b in pixels:
        bits.append(r & 1)
    chars = []
    for i in range(0, len(bits) - 7, 8):
        byte = 0
        for bit in bits[i:i+8]:
            byte = (byte << 1) | bit
        chars.append(byte)
    return bytes(chars)
```

## Crypto challenges

Thách thức crypto thường có lỗi triển khai: nonce tái sử dụng, khóa yếu, XOR lặp.

### XOR với khóa lặp

```python
def xor_repeating(data: bytes, key: bytes) -> bytes:
    return bytes(b ^ key[i % len(key)] for i, b in enumerate(data))
```

### Phá XOR một byte (single-byte XOR)

```python
def score_english(text: bytes) -> float:
    freq = b"etaoin shrdlu"
    return sum(text.lower().count(bytes([c])) for c in freq) / max(len(text), 1)

def break_single_xor(ciphertext: bytes) -> tuple[int, bytes]:
    best = (0, b"")
    best_score = 0.0
    for key in range(256):
        candidate = bytes(b ^ key for b in ciphertext)
        score = score_english(candidate)
        if score > best_score:
            best_score = score
            best = (key, candidate)
    return best
```

### RSA yếu

```python
from math import gcd, isqrt

def factor_small_n(n: int) -> tuple[int, int] | None:
    if n % 2 == 0:
        return 2, n // 2
    for p in range(3, isqrt(n) + 1, 2):
        if n % p == 0:
            return p, n // p
    return None

def rsa_small_e_attack(c: int, e: int, n: int) -> bytes:
    # e nhỏ và m^e < n → căn bậc e nguyên
    root = round(c ** (1 / e))
    return root.to_bytes((root.bit_length() + 7) // 8, "big")
```

Khi `e=3` và thông điệp ngắn, `m^3 < n` nên chỉ cần căn bậc ba.

## Pwn — khai thác nhị phân

Công cụ: `pwntools`.

```bash
pip install pwntools
```

```python
from pwn import *

context.log_level = "info"
context.arch = "amd64"

# Kết nối tới service
io = remote("challenge.ctf.example", 1337)
io.recvuntil(b"name?")
io.sendline(b"AAAA")
io.interactive()
```

### Buffer overflow cơ bản

```python
from pwn import *

def find_offset(pattern_size: int = 200) -> int:
    pattern = cyclic(pattern_size)
    # gửi pattern, đọc giá trị ghi đè return address
    crash_value = 0x61616168        # ví dụ
    return cyclic_find(crash_value)
```

`cyclic` tạo chuỗi không lặp cho phép xác định offset chính xác tới thanh ghi điều khiển.

### ROP chain (khái niệm)

```python
from pwn import *

elf = ELF("./vuln")
rop = ROP(elf)
rop.call(elf.symbols["system"], [next(elf.search(b"/bin/sh\x00"))])
print(rop.dump())
```

Return-Oriented Programming ghép các đoạn code có sẵn (`gadget`) để vượt qua NX.

## Web CTF

Các dạng thường gặp:

- SQL injection tới flag trong database.
- LFI/RFI đọc file `/flag.txt`.
- SSTI (Server-Side Template Injection).
- JWT none algorithm, weak secret.
- Deserialization.

### JWT weak secret

```python
import jwt

def crack_jwt(token: str, wordlist: list[str]) -> str | None:
    for secret in wordlist:
        try:
            jwt.decode(token, secret, algorithms=["HS256"])
            return secret
        except jwt.InvalidSignatureError:
            continue
    return None
```

## Automation với pwntools + requests

```python
import requests
from bs4 import BeautifulSoup

def solve_web_challenge(base: str) -> str:
    s = requests.Session()
    r = s.get(f"{base}/", timeout=5)
    soup = BeautifulSoup(r.text, "html.parser")
    form = soup.find("form")
    action = form.get("action", "")
    r = s.post(f"{base}{action}", data={"input": "payload"}, timeout=5)
    return r.text
```

## Viết writeup

Writeup tốt gồm: mô tả thách thức, phân tích, hướng tiếp cận, script giải, bài học. Viết writeup là cách học hiệu quả nhất — dạy lại chính mình.

## Nền tảng luyện tập hợp pháp

- OverTheWire (Bandit, Natas) — Linux và web cơ bản.
- picoCTF — dành cho người mới.
- PortSwigger Web Security Academy — web miễn phí, chất lượng cao.
- HackTheBox, TryHackMe — lab có phép.
- Root-Me, Pwn.College — đa dạng.
- CTFtime — lịch thi đấu.

## Bài tập

1. Giải 10 bài Bandit đầu tiên.
2. Làm một bài forensics từ picoCTF.
3. Viết script phá single-byte XOR không dùng thư viện ngoài.
4. Giải một bài crypto RSA e nhỏ.
5. Viết writeup đầy đủ cho một thách thức bạn đã giải.

Stashed.