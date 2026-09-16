# Bộ Sách Python Từ A Đến Z

Bộ tài liệu tự học Python hoàn chỉnh. Mỗi file là một chương độc lập, viết dày với ví dụ chạy được, bài tập kèm lời giải, và ghi chú kỹ thuật.

## Cấu trúc thư mục

```
python-a-z/
├── README.md
├── 01-moi-bat-dau/
├── 02-co-ban/
├── 03-trung-cap/
├── 04-nang-cao/
├── 05-chuyen-gia/
├── 06-an-ninh-mang/
├── 07-chuyen-de/
└── 08-phu-luc/
```

## Lộ trình theo cấp độ

### Cấp 1 — Mới bắt đầu (`01-moi-bat-dau/`)

| File | Nội dung |
|------|----------|
| `01-nhap-mon-python.md` | Python là gì, cài đặt, venv, REPL, cú pháp nền tảng, lỗi thường gặp |
| `02-bien-kieu-du-lieu.md` | Biến, kiểu dữ liệu, số học, float/Decimal/Fraction, chuỗi, f-string, bytes |
| `03-toan-tu-dieu-khien.md` | Toán tử, truthy/falsy, match/case, vòng lặp, break/continue/else |

**Mục tiêu:** viết được script đơn giản, đọc hiểu code cơ bản, biết debug lỗi thường gặp.

### Cấp 2 — Cơ bản (`02-co-ban/`)

| File | Nội dung |
|------|----------|
| `04-chuoi-collections.md` | List, tuple, set, dict, collections, regex, copy, độ phức tạp |
| `05-ham-module.md` | Hàm, tham số, scope LEGB, đệ quy, module, package, argparse, click |
| `06-file-exception.md` | File I/O, pathlib, JSON, CSV, ngoại lệ, logging, nén |

**Mục tiêu:** tổ chức code thành hàm/module, xử lý file và lỗi thành thạo.

### Cấp 3 — Trung cấp (`03-trung-cap/`)

| File | Nội dung |
|------|----------|
| `07-oop.md` | Class, property, kế thừa, MRO, ABC, magic methods, dataclass, slots |
| `08-generator-decorator-context.md` | Iterator, generator, closure, decorator, context manager, contextlib |
| `09-comprehension-functional.md` | Comprehension, lambda, map/filter/reduce, functools, itertools |
| `10-type-hints-tooling.md` | Type hints, Protocol, Generic, venv, ruff, mypy, pre-commit, pyproject |

**Mục tiêu:** viết code có cấu trúc, biết type hints và công cụ chất lượng code.

### Cấp 4 — Nâng cao (`04-nang-cao/`)

| File | Nội dung |
|------|----------|
| `11-concurrency-asyncio.md` | GIL, threading, multiprocessing, asyncio, aiohttp, chọn mô hình |
| `12-testing-packaging-performance.md` | pytest, fixture, mock, coverage, đóng gói, profiling, tối ưu, CI |
| `13-design-patterns-database-web.md` | Design patterns, SQLite, SQLAlchemy, FastAPI, Pydantic |

**Mục tiêu:** làm được backend, viết test, tối ưu hiệu năng, áp dụng design pattern.

### Cấp 5 — Chuyên gia (`05-chuyen-gia/`)

| File | Nội dung |
|------|----------|
| `14-data-ml-security-internals.md` | NumPy, Pandas, scikit-learn, PyTorch, bảo mật, CPython internals, C extension |

**Mục tiêu:** hiểu sâu runtime, viết extension native, làm data/ML.

### An ninh mạng (`06-an-ninh-mang/`)

| File | Nội dung |
|------|----------|
| `15-nang-cao-networking.md` | Socket nâng cao, framing, HTTP tay, TLS, asyncio sockets |
| `16-recon-scanning.md` | Recon, quét cổng, banner grabbing, fingerprinting, phòng thủ |
| `17-packet-crafting.md` | Scapy, gói tin thô, sniffing, pcap, ARP, DNS |
| `18-cryptography.md` | Hash, HMAC, AES-GCM, RSA, chữ ký, trao đổi khóa, lỗi thường gặp |
| `19-web-security.md` | Injection, XSS, CSRF, SSRF, deserialization, headers, kiểm thử |
| `20-reverse-engineering.md` | PE/ELF, bytecode Python, obfuscation, phân tích malware, entropy |
| `21-defensive-secure-coding.md` | Validation, secrets, logging an toàn, SAST, rate limiting, upload |
| `22-ctf-methodology.md` | Forensics, crypto, pwn, web CTF, writeup, nền tảng luyện tập |

**Mục tiêu:** hiểu tấn công để phòng thủ, luyện trên lab có phép.

### Chuyên đề (`07-chuyen-de/`)

| File | Nội dung |
|------|----------|
| `18-chuyen-de-async-nang-cao.md` | TaskGroup, structured concurrency, cancellation, backpressure, anyio |
| `19-chuyen-de-typing-nang-cao.md` | Variance, NewType, Protocol, ParamSpec, TypeGuard, Annotated, mypy strict |
| `20-chuyen-de-cli-dong-goi.md` | argparse/click/typer, pyproject, semver, build wheel, publish PyPI, PyInstaller |
| `21-chuyen-de-du-lieu-lon.md` | Arrow, Parquet, Polars lazy, DuckDB, ETL, streaming |
| `22-chuyen-de-observability.md` | Structured log, correlation ID, Prometheus, OpenTelemetry, health check |
| `23-chuyen-de-cpython-hieu-nang.md` | Profiler, specializing interpreter, free-threading, subinterpreters, tối ưu vi mô |
| `24-chuyen-de-testing-nang-cao.md` | Hypothesis, property-based, mutation testing, fuzzing, golden file, benchmark |
| `25-chuyen-de-caching-background-jobs.md` | lru_cache, TTL, Redis cache, stampede, Celery, APScheduler, rate limiting |
| `26-chuyen-de-auth-oauth.md` | Session, JWT, refresh token, OAuth2/OIDC, RBAC, ABAC, API key, 2FA |
| `27-chuyen-de-grpc-message-queue.md` | Protobuf, gRPC 4 kiểu RPC, RabbitMQ, Redis Streams, Kafka, outbox, idempotency |
| `28-chuyen-de-automation-scraping.md` | requests/BS4, Playwright, Selenium, pyautogui, watchdog, subprocess |
| `29-chuyen-de-media-processing.md` | Pillow, OpenCV, ffmpeg, pydub, librosa, validator upload, batch xử lý |

**Mục tiêu:** đào sâu từng mảng chuyên môn khi đã vững nền tảng.

### Phụ lục (`08-phu-luc/`)

| File | Nội dung |
|------|----------|
| `15-phu-luc-regex.md` | Regex cheat sheet đầy đủ, mẫu thường dùng, cạm bẫy |
| `16-phu-luc-debugging.md` | Đọc traceback, pdb, logging, cProfile, tracemalloc, debug async |
| `17-phu-luc-git.md` | Git cho Python: branch, merge, rebase, bisect, hook, PR |

## Lộ trình gợi ý

**Người mới hoàn toàn:** `01-moi-bat-dau/` → `02-co-ban/`. Gõ lại từng ví dụ, đừng copy-paste.

**Đã biết ít:** `02-co-ban/` → `03-trung-cap/`. Làm hết bài tập cuối mỗi chương.

**Có kinh nghiệm:** `03-trung-cap/` → `04-nang-cao/`.

**An ninh mạng:** đọc nền tảng trước, rồi `06-an-ninh-mang/`.

**Backend đi làm:** 13 (web/DB) → 18 (async) → 19 (typing) → 26 (auth) → 25 (cache/jobs) → 22 (observability) → 27 (gRPC/MQ).

**Data / phân tích:** 14 → 21 (dữ liệu lớn) → 29 (media).

**Automation / scraping:** 06 → 15 → 28.

**Chất lượng code:** 12 (test) → 24 (testing nâng cao) → 19 (typing) → 23 (hiệu năng).

**Mạnh nhất:** 13 → 14 → 23 → 27, đọc kèm source CPython, viết C extension / Rust extension.

## Lưu ý phần an ninh mạng

Các chương trong `06-an-ninh-mang/` mang tính giáo dục. Kỹ thuật tấn công chỉ áp dụng cho:

- Hệ thống bạn sở hữu.
- Hệ thống bạn được ủy quyền kiểm thử bằng văn bản.
- Nền tảng luyện tập có chủ đích: OverTheWire, picoCTF, HackTheBox, TryHackMe, PortSwigger Web Security Academy, lab VM.

Truy cập trái phép hệ thống là vi phạm pháp luật.

## Cài đặt nhanh

```bash
python3 --version          # cần >= 3.10, khuyến nghị 3.12+
python3 -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate
pip install --upgrade pip
```

## Thư viện dùng trong sách

```bash
# Nền tảng
pip install requests pydantic

# Web
pip install fastapi "uvicorn[standard]" sqlalchemy beautifulsoup4

# Data / ML
pip install numpy pandas scikit-learn matplotlib joblib
pip install pyarrow polars duckdb
pip install torch  # tùy chọn, nặng

# Dev tools
pip install pytest pytest-cov pytest-asyncio pytest-benchmark \
            hypothesis mutmut ruff mypy pre-commit build twine

# CLI
pip install click typer rich

# Observability
pip install prometheus-client opentelemetry-api opentelemetry-sdk \
            opentelemetry-exporter-otlp

# Cache / job / queue
pip install redis celery apscheduler arq cachetools slowapi
pip install pika confluent-kafka
pip install grpcio grpcio-tools

# Auth
pip install pyjwt argon2-cffi bcrypt pyotp authlib

# Automation / scraping
pip install httpx playwright selenium lxml watchdog pyautogui

# Media
pip install Pillow opencv-python pydub librosa
# ffmpeg cài qua hệ thống: apt/brew

# Hiệu năng
pip install numba py-spy pyperf pympler memory-profiler line_profiler

# An ninh mạng (chỉ dùng trong lab)
pip install cryptography scapy pwntools pefile pyelftools bandit pip-audit

# Debugging
pip install ipython icecream objgraph
```

## Quy ước trong sách

- Code chạy trên Python 3.10+; một số chỗ cần 3.11 hoặc 3.12+.
- `>>>` là REPL; phần còn lại là file `.py`.
- Mọi ví dụ đều chạy được — gõ lại, đừng chỉ đọc.
- Comment trong code chỉ giải thích *vì sao*, không giải thích *cái gì*.

Stashed.