# Chuyên Đề 20 — CLI Chuyên Nghiệp và Đóng Gói Phân Phối

## 20.1 Cấu trúc một CLI tốt

```text
mytool/
├── pyproject.toml
├── README.md
├── src/mytool/
│   ├── __init__.py
│   ├── __main__.py
│   ├── cli.py
│   └── core.py
└── tests/
```

Quy ước:

- Logic nghiệp vụ trong `core.py`, không import `click`/`argparse`.
- CLI chỉ là lớp mỏng gọi core.
- `__main__.py` cho phép `python -m mytool`.

`src/mytool/__main__.py`:

```python
from mytool.cli import main

if __name__ == "__main__":
    raise SystemExit(main())
```

## 20.2 `argparse` — chuẩn thư viện

```python
import argparse
from pathlib import Path

def build_parser() -> argparse.ArgumentParser:
    p = argparse.ArgumentParser(
        prog="mytool",
        description="Công cụ xử lý file",
    )
    p.add_argument("nguon", type=Path, help="File nguồn")
    p.add_argument("-o", "--output", type=Path, default=Path("out.txt"))
    p.add_argument("-v", "--verbose", action="count", default=0)
    p.add_argument("--log-level", choices=["DEBUG", "INFO", "WARNING"], default="INFO")
    p.add_argument("--dry-run", action="store_true")
    return p

def main(argv: list[str] | None = None) -> int:
    args = build_parser().parse_args(argv)
    if not args.nguon.exists():
        print(f"Không tìm thấy {args.nguon}", file=__import__("sys").stderr)
        return 2
    if args.dry_run:
        print("chạy thử")
        return 0
    return 0
```

`main(argv)` nhận tham số để test dễ: `main(["a.txt", "--dry-run"])`.

### `action="count"`

`-v`, `-vv`, `-vvv` → 1, 2, 3.

### Exit code

| Code | Ý nghĩa |
|------|---------|
| 0 | thành công |
| 1 | lỗi chung |
| 2 | dùng sai cú pháp |
| 126 | không thực thi được |
| 127 | không tìm thấy lệnh |
| 130 | bị Ctrl+C |

## 20.3 `click` — CLI hiện đại

```bash
pip install click
```

```python
import click

@click.group()
@click.option("--verbose/--quiet", default=False)
@click.pass_context
def cli(ctx: click.Context, verbose: bool) -> None:
    ctx.ensure_object(dict)
    ctx.obj["verbose"] = verbose

@cli.command()
@click.argument("nguon", type=click.Path(exists=True, path_type=Path))
@click.option("-o", "--output", type=click.Path(path_type=Path), default=Path("out.txt"))
@click.pass_context
def xu_ly(ctx: click.Context, nguon: Path, output: Path) -> None:
    if ctx.obj["verbose"]:
        click.echo(f"Đang xử lý {nguon}")
    output.write_text(nguon.read_text(), encoding="utf-8")

@cli.command()
def version() -> None:
    click.echo("mytool 1.0.0")

if __name__ == "__main__":
    cli()
```

### Prompt và xác nhận

```python
@click.command()
def xoa() -> None:
    if click.confirm("Chắc chắn xóa?"):
        click.echo("đã xóa")
    ten = click.prompt("Tên", type=str)
```

### Tiến trình

```python
with click.progressbar(range(1000)) as bar:
    for i in bar:
        pass
```

## 20.4 `typer` — dựa trên type hints

```bash
pip install typer
```

```python
import typer
from pathlib import Path
from typing import Annotated

app = typer.Typer()

@app.command()
def xu_ly(
    nguon: Annotated[Path, typer.Argument(exists=True)],
    output: Annotated[Path, typer.Option("-o", "--output")] = Path("out.txt"),
    verbose: Annotated[bool, typer.Option("--verbose", "-v")] = False,
) -> None:
    """Xử lý file nguồn và ghi ra output."""
    if verbose:
        typer.echo(f"Đọc {nguon}")
    output.write_text(nguon.read_text(), encoding="utf-8")

if __name__ == "__main__":
    app()
```

`typer` sinh help từ docstring và type hints. Ít boilerplate nhất.

## 20.5 `pyproject.toml` đầy đủ

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "mytool"
version = "1.0.0"
description = "Công cụ xử lý file"
readme = "README.md"
license = { text = "MIT" }
requires-python = ">=3.10"
authors = [{ name = "Bạn", email = "ban@example.com" }]
keywords = ["cli", "file"]
classifiers = [
    "Development Status :: 4 - Beta",
    "Environment :: Console",
    "Programming Language :: Python :: 3.12",
    "License :: OSI Approved :: MIT License",
]
dependencies = [
    "click>=8.1",
    "pydantic>=2.0",
]

[project.optional-dependencies]
dev = ["pytest>=8", "pytest-cov", "ruff", "mypy", "pre-commit"]
docs = ["mkdocs-material"]

[project.urls]
Homepage = "https://github.com/user/mytool"
Issues = "https://github.com/user/mytool/issues"

[project.scripts]
mytool = "mytool.cli:main"

[tool.hatch.build.targets.wheel]
packages = ["src/mytool"]
```

`[project.scripts]` tạo entry point: sau `pip install`, gõ `mytool` chạy `mytool.cli:main`.

## 20.6 Semantic versioning

`MAJOR.MINOR.PATCH`

- **MAJOR**: thay đổi phá vỡ tương thích (breaking).
- **MINOR**: thêm tính năng, tương thích ngược.
- **PATCH**: sửa lỗi, tương thích ngược.

Với 0.x.y, MINOR có thể phá vỡ.

### Version động từ git

```toml
[build-system]
requires = ["hatchling", "hatch-vcs"]
build-backend = "hatchling.build"

[tool.hatch.version]
source = "vcs"

[tool.hatch.build.hooks.vcs]
version-file = "src/mytool/_version.py"
```

Version lấy từ git tag. Tag `v1.2.3` → version `1.2.3`.

## 20.7 Build và kiểm tra

```bash
pip install build twine

python -m build
# dist/mytool-1.0.0.tar.gz  (sdist)
# dist/mytool-1.0.0-py3-none-any.whl  (wheel)

twine check dist/*

# Cài thử trong venv sạch
python -m venv /tmp/test-venv
/tmp/test-venv/bin/pip install dist/mytool-1.0.0-py3-none-any.whl
/tmp/test-venv/bin/mytool --help
```

**Luôn test wheel trong venv sạch** trước khi publish. Nhiều lỗi chỉ lộ khi cài từ wheel.

### Kiểm tra nội dung wheel

```bash
unzip -l dist/mytool-1.0.0-py3-none-any.whl
```

## 20.8 Publish lên PyPI

```bash
# Test PyPI trước
twine upload --repository testpypi dist/*
pip install --index-url https://test.pypi.org/simple/ mytool

# PyPI thật
twine upload dist/*
```

Dùng API token, không dùng mật khẩu:

```bash
# ~/.pypirc
[distutils]
index-servers =
    pypi
    testpypi

[pypi]
username = __token__
password = pypi-AgEIcHlwaS5vcmc...
```

## 20.9 CI tự động release

`.github/workflows/release.yml`:

```yaml
name: Release
on:
  push:
    tags: ["v*"]

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install build
      - run: python -m build
      - uses: pypa/gh-action-pypi-publish@release/v1
```

Dùng **Trusted Publishing** (OIDC) — không cần lưu token.

## 20.10 Đóng gói binary độc lập

### PyInstaller

```bash
pip install pyinstaller
pyinstaller --onefile --name mytool src/mytool/__main__.py
```

Tạo file thực thi duy nhất. Nhược điểm: khởi động chậm, file lớn (~10-30MB).

### Nuitka

```bash
pip install nuitka
python -m nuitka --standalone --onefile src/mytool/__main__.py
```

Biên dịch sang C, nhanh hơn, khó dịch ngược hơn.

### Shiv / PEX

```bash
pip install shiv
shiv -o mytool.pyz -e mytool.cli:main .
./mytool.pyz --help
```

File `.pyz` cần Python trên máy đích, nhưng nhẹ và nhanh.

## 20.11 Phân phối qua Docker

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
COPY pyproject.toml README.md ./
COPY src ./src
RUN pip install --no-cache-dir build && python -m build

FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /app/dist/*.whl ./
RUN pip install --no-cache-dir *.whl && rm *.whl
ENTRYPOINT ["mytool"]
```

Multi-stage giữ image nhỏ.

## 20.12 Cấu hình qua file và biến môi trường

```python
import os
import tomllib
from pathlib import Path
from pydantic import BaseModel

class CauHinh(BaseModel):
    output_dir: Path = Path("./out")
    max_workers: int = 4
    api_key: str = ""

def doc_cau_hinh(path: Path | None = None) -> CauHinh:
    du_lieu: dict = {}
    if path and path.exists():
        with path.open("rb") as f:
            du_lieu = tomllib.load(f)

    if key := os.environ.get("MYTOOL_API_KEY"):
        du_lieu["api_key"] = key

    return CauHinh(**du_lieu)
```

Thứ tự ưu tiên: CLI flag > biến môi trường > file cấu hình > mặc định.

## 20.13 Bài tập

1. Viết CLI `wordcount` với `argparse`: đếm dòng, từ, ký tự.
2. Viết lại bằng `typer`, so sánh lượng code.
3. Đóng gói CLI vào wheel, cài trong venv sạch, chạy thử.
4. Thêm `--version` lấy từ `importlib.metadata.version`.
5. Build binary bằng PyInstaller, đo kích thước và thời gian khởi động.
6. Viết GitHub Action build và publish lên TestPyPI khi có tag.
7. Viết CLI đọc cấu hình TOML + biến môi trường, ưu tiên đúng thứ tự.

## 20.14 Ghi chú kỹ thuật

- Không viết logic vào `pyproject.toml`; giữ nó khai báo.
- `python -m build` cần `build` package, không dùng `setup.py sdist` trực tiếp.
- Wheel là định dạng cài đặt; sdist là mã nguồn để build.
- Đặt tên distribution khác tên package import được (ví dụ dist `my-tool`, import `mytool`).
- Entry point chạy trên mọi hệ điều hành; `.exe` chỉ Windows.
- `--help` nên hoạt động không cần dependency nặng — lazy import trong nhánh lệnh.
- Stderr cho lỗi và log; stdout cho dữ liệu thật. Pipeline shell phụ thuộc điều này.

Stashed.