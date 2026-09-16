# Chương 21 — Lập Trình An Toàn và Phòng Thủ

## Nguyên tắc thiết kế an toàn

1. **Least privilege** — cấp quyền tối thiểu cần thiết.
2. **Defense in depth** — nhiều lớp, một lớp hỏng vẫn còn lớp khác.
3. **Fail secure** — khi lỗi, mặc định từ chối, không mở.
4. **Never trust input** — mọi dữ liệu từ ngoài đều không đáng tin.
5. **Keep it simple** — hệ thống phức tạp khó bảo mật.

## Validation input

```python
from pydantic import BaseModel, Field, field_validator
from pathlib import Path

class UploadRequest(BaseModel):
    filename: str = Field(min_length=1, max_length=255)
    size: int = Field(gt=0, le=10 * 1024 * 1024)

    @field_validator("filename")
    @classmethod
    def safe_filename(cls, v: str) -> str:
        if any(c in v for c in ("/", "\\", "\x00")):
            raise ValueError("tên file chứa ký tự không hợp lệ")
        if v != Path(v).name:
            raise ValueError("path traversal")
        return v
```

Pydantic validate và chuyển đổi tự động, trả lỗi rõ ràng.

## Chống path traversal

```python
from pathlib import Path

BASE_DIR = Path("/srv/uploads").resolve()

def safe_path(user_path: str) -> Path:
    candidate = (BASE_DIR / user_path).resolve()
    if not candidate.is_relative_to(BASE_DIR):
        raise ValueError("truy cập ngoài thư mục cho phép")
    return candidate
```

`resolve()` xử lý `..` và symlink. `is_relative_to` kiểm tra containment.

## Secrets management

```python
import os
from functools import lru_cache

@lru_cache
def get_secret(name: str) -> str:
    value = os.environ.get(name)
    if not value:
        raise RuntimeError(f"thiếu biến môi trường {name}")
    return value
```

- Không hardcode. Không log secrets. Không commit `.env`.
- Dùng `python-dotenv` cho dev, Vault/AWS Secrets Manager cho production.
- Xoay khóa định kỳ.

```python
def redact(value: str, show: int = 4) -> str:
    if len(value) <= show * 2:
        return "*" * len(value)
    return f"{value[:show]}...{value[-show:]}"
```

## Logging an toàn

```python
import logging
import re

SENSITIVE = re.compile(r"(password|token|secret|api[_-]?key)", re.IGNORECASE)

class RedactingFormatter(logging.Formatter):
    def format(self, record):
        msg = super().format(record)
        return SENSITIVE.sub(r"\1=***", msg)
```

Không log mật khẩu, token, số thẻ. Log đủ để điều tra nhưng không lộ dữ liệu nhạy cảm.

## Xử lý lỗi không lộ thông tin

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

app = FastAPI()

@app.exception_handler(Exception)
async def generic_handler(request: Request, exc: Exception):
    logging.exception("lỗi không xử lý")
    return JSONResponse(status_code=500, content={"error": "lỗi hệ thống"})
```

Stack trace chỉ hiện trong log nội bộ, không trả về client.

## Dependency an toàn

```bash
pip install pip-audit
pip-audit
```

```bash
pip install safety
safety check
```

Ghim phiên bản, cập nhật bản vá định kỳ, theo dõi CVE.

```python
# Kiểm tra dependency trong CI
import subprocess
result = subprocess.run(["pip-audit", "--format", "json"], capture_output=True, text=True)
if result.returncode != 0:
    raise SystemExit("phát hiện lỗ hổng dependency")
```

## Static analysis (SAST)

```bash
pip install bandit
bandit -r src/
```

Bandit phát hiện pattern nguy hiểm: `eval`, `exec`, `pickle`, `subprocess shell=True`, hardcoded password.

```python
# bandit cảnh báo những dòng này
eval(user_input)                     # B307
exec(code)                           # B102
subprocess.run(cmd, shell=True)      # B602
password = "hardcoded"               # B105
```

## Rate limiting

```python
import time
from collections import defaultdict

class RateLimiter:
    def __init__(self, max_requests: int, window: float):
        self.max = max_requests
        self.window = window
        self.hits: dict[str, list[float]] = defaultdict(list)

    def allow(self, key: str) -> bool:
        now = time.monotonic()
        timestamps = self.hits[key]
        while timestamps and timestamps[0] < now - self.window:
            timestamps.pop(0)
        if len(timestamps) >= self.max:
            return False
        timestamps.append(now)
        return True
```

Chống brute force, DoS, scraping.

## Xử lý file upload

```python
import magic
from pathlib import Path

ALLOWED_TYPES = {"image/png", "image/jpeg", "application/pdf"}

def validate_upload(content: bytes, filename: str) -> str:
    detected = magic.from_buffer(content, mime=True)
    if detected not in ALLOWED_TYPES:
        raise ValueError(f"loại file không cho phép: {detected}")
    suffix = Path(filename).suffix.lower()
    if suffix not in (".png", ".jpg", ".jpeg", ".pdf"):
        raise ValueError("đuôi file không hợp lệ")
    return detected
```

Kiểm tra MIME thật từ nội dung, không tin đuôi file. Lưu file ngoài web root, đổi tên ngẫu nhiên.

## Checklist secure coding

- [ ] Validate mọi input ở biên hệ thống.
- [ ] Tham số hóa truy vấn.
- [ ] Escape output.
- [ ] Secrets ngoài source.
- [ ] Log không lộ dữ liệu nhạy cảm.
- [ ] Xử lý lỗi không lộ stack trace.
- [ ] Rate limiting endpoint nhạy cảm.
- [ ] Dependency được audit.
- [ ] SAST trong CI.
- [ ] Nguyên tắc least privilege.

## Bài tập

1. Thêm validation Pydantic cho API có sẵn.
2. Viết middleware redact secrets trong log.
3. Viết rate limiter dùng Redis.
4. Chạy bandit trên project, sửa hết cảnh báo mức HIGH.
5. Viết validator upload file kiểm tra cả magic bytes.

Stashed.