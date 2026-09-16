# Chương 20 — Reverse Engineering và Phân Tích Nhị Phân

> Phân tích mã độc, crack binary, hay dịch ngược phần mềm thương mại có thể vi phạm bản quyền hoặc pháp luật tùy ngữ cảnh. Chương này dùng cho nghiên cứu bảo mật, phân tích malware, và học tập.

## Định dạng file thực thi

### PE (Windows)

```python
import pefile

pe = pefile.PE("target.exe")
print(hex(pe.FILE_HEADER.Machine))
print(pe.FILE_HEADER.TimeDateStamp)
for section in pe.sections:
    print(section.Name.decode().rstrip("\x00"), hex(section.VirtualAddress))
for entry in pe.DIRECTORY_ENTRY_IMPORT:
    print(entry.dll.decode())
    for imp in entry.imports:
        print("  ", imp.name)
```

PE header: DOS stub, NT headers, section table, import/export tables. Import table cho biết binary gọi API nào — manh mối về hành vi.

### ELF (Linux)

```python
from elftools.elf.elffile import ELFFile

with open("/bin/ls", "rb") as f:
    elf = ELFFile(f)
    print(elf.header["e_machine"], elf.header["e_type"])
    for section in elf.iter_sections():
        print(section.name, hex(section["sh_addr"]))
```

## Python bytecode

Python biên dịch thành bytecode, chạy trên máy ảo CPython.

```python
import dis

def ham(a, b):
    if a > b:
        return a
    return b

dis.dis(ham)
```

Xuất ra:

```text
LOAD_FAST a
LOAD_FAST b
COMPARE_OP >
POP_JUMP_IF_FALSE
...
```

### Đọc file .pyc

```python
import marshal
import dis
import importlib.util

def load_pyc(path: str):
    with open(path, "rb") as f:
        f.read(16)                       # header (magic, flags, timestamp, size)
        code = marshal.load(f)
    return code

code = load_pyc("module.pyc")
dis.dis(code)
```

### Decompile

Bytecode có thể dịch ngược gần đúng về source. Công cụ: `decompyle3`, `uncompyle6`, `pycdc`. Không cần cài — có thể đọc trực tiếp bằng `dis` và suy luận.

## Phân tích động

### sys.settrace — theo dõi thực thi

```python
import sys

def tracer(frame, event, arg):
    if event == "call":
        print(f"gọi {frame.f_code.co_name} tại {frame.f_code.co_filename}:{frame.f_lineno}")
    elif event == "line":
        pass
    return tracer

sys.settrace(tracer)
# chạy code cần phân tích
sys.settrace(None)
```

### Monkeypatching để quan sát

```python
import builtins

original_open = builtins.open

def traced_open(file, *args, **kwargs):
    print(f"mở file: {file}")
    return original_open(file, *args, **kwargs)

builtins.open = traced_open
```

Kỹ thuật này quan sát code Python không tin cậy gọi gì mà không cần đọc source.

## Chống phân tích (và phòng thủ)

### Obfuscation

```python
import base64
import zlib

def obfuscate(source: str) -> str:
    compressed = zlib.compress(source.encode())
    encoded = base64.b64encode(compressed).decode()
    return f"import zlib,base64;exec(zlib.decompress(base64.b64decode('{encoded}')))"
```

Đây là obfuscation cơ bản — dễ đảo ngược. Reverse: chặn `exec`, in ra thay vì chạy.

### Anti-debug cơ bản

```python
import sys

def check_debugger():
    return sys.gettrace() is not None or "pydevd" in sys.modules
```

Phòng thủ: kiểm tra nhiều dấu hiệu, không tin một cái. Nhưng obfuscation không thay thế được bảo mật thật — code chạy được thì đọc được.

## Phân tích malware tĩnh

```python
import re

SUSPICIOUS_IMPORTS = [
    "CreateRemoteThread", "VirtualAllocEx", "WriteProcessMemory",
    "URLDownloadToFile", "ShellExecute", "WinExec", "RegSetValue",
]

def scan_imports(path: str) -> list[str]:
    import pefile
    pe = pefile.PE(path)
    found = []
    if hasattr(pe, "DIRECTORY_ENTRY_IMPORT"):
        for entry in pe.DIRECTORY_ENTRY_IMPORT:
            for imp in entry.imports:
                name = imp.name.decode() if imp.name else ""
                for sus in SUSPICIOUS_IMPORTS:
                    if sus.lower() in name.lower():
                        found.append(f"{entry.dll.decode()}!{name}")
    return found
```

### Trích xuất chuỗi

```python
def strings(path: str, min_len: int = 4) -> list[str]:
    pattern = rb"[\x20-\x7e]{" + str(min_len).encode() + rb",}"
    with open(path, "rb") as f:
        data = f.read()
    return [m.decode() for m in re.findall(pattern, data)]
```

Tìm URL, địa chỉ IP, đường dẫn registry trong chuỗi.

### Entropy — phát hiện nén/mã hóa

```python
import math
from collections import Counter

def entropy(data: bytes) -> float:
    if not data:
        return 0.0
    counts = Counter(data)
    length = len(data)
    return -sum((c / length) * math.log2(c / length) for c in counts.values())
```

Entropy gần 8.0 nghĩa là dữ liệu mã hóa/nén — dấu hiệu payload ẩn.

## Phòng thủ

- Sandbox mọi file không tin cậy (Cuckoo, Firejail, VM cô lập).
- Không chạy binary lạ trên máy thật.
- Ký số và xác minh nguồn phần mềm.
- EDR phát hiện hành vi (API calls, injection).
- Cập nhật bản vá — phần lớn malware khai thác lỗi đã biết.

## Bài tập

1. Viết script trích xuất import table của PE và tô đậm API nguy hiểm.
2. Dịch ngược một hàm Python bằng `dis`, viết lại source.
3. Viết công cụ tính entropy từng section của binary.
4. Viết tracer in mọi lời gọi hàm của một script không tin cậy.
5. Phân tích một mẫu malware an toàn trong sandbox (dùng mẫu từ các kho học thuật).

Stashed.