# Chuyên Đề 28 — Automation và Scraping

## 28.1 Nguyên tắc đạo đức và pháp lý

Trước khi scrape bất cứ thứ gì:

- Đọc `robots.txt` và điều khoản sử dụng.
- Kiểm tra `ToS` — nhiều site cấm scraping.
- Không scrape dữ liệu cá nhân không công khai.
- Tôn trọng rate limit — không DOS site.
- Ưu tiên API chính thức nếu có.

Scraping site công khai, tôn trọng robots, cho mục đích cá nhân/hợp pháp — thường chấp nhận được. Vượt qua xác thực, scrape dữ liệu riêng tư, làm quá tải — không.

## 28.2 requests + BeautifulSoup

```bash
pip install requests beautifulsoup4 lxml
```

```python
import requests
from bs4 import BeautifulSoup

r = requests.get(
    "https://example.com",
    headers={"User-Agent": "Mozilla/5.0 (compatible; MyBot/1.0)"},
    timeout=10,
)
r.raise_for_status()

soup = BeautifulSoup(r.text, "lxml")
for link in soup.select("a[href]"):
    print(link["href"], link.get_text(strip=True))
```

### Selector CSS

```python
soup.select("div.card")               # class
soup.select("#main")                  # id
soup.select("article > h2")           # con trực tiếp
soup.select("a[href^='https']")       # thuộc tính bắt đầu
soup.select_one("h1.title").text
```

### Trích xuất có cấu trúc

```python
bai_viet = []
for item in soup.select("article.post"):
    bai_viet.append({
        "tieu_de": item.select_one("h2").get_text(strip=True),
        "tac_gia": item.select_one(".author").get_text(strip=True),
        "ngay": item.select_one("time")["datetime"],
        "tom_tat": item.select_one("p.summary").get_text(strip=True),
    })
```

## 28.3 Session và retry

```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

session = requests.Session()
retry = Retry(
    total=5,
    backoff_factor=0.5,
    status_forcelist=[429, 500, 502, 503, 504],
    respect_retry_after_header=True,
)
session.mount("https://", HTTPAdapter(max_retries=retry))
session.headers.update({"User-Agent": "MyBot/1.0"})

r = session.get("https://example.com", timeout=10)
```

`respect_retry_after_header` tôn trọng `Retry-After` khi bị 429.

## 28.4 Rate limiting phía client

```python
import time

class RateLimiter:
    def __init__(self, giay_moi_request: float):
        self.khoang = giay_moi_request
        self.lan_cuoi = 0.0

    def doi(self):
        con_lai = self.khoang - (time.monotonic() - self.lan_cuoi)
        if con_lai > 0:
            time.sleep(con_lai)
        self.lan_cuoi = time.monotonic()

rl = RateLimiter(1.0)   # 1 giây/request
for url in urls:
    rl.doi()
    r = session.get(url, timeout=10)
```

## 28.5 httpx — sync và async

```bash
pip install httpx
```

```python
import httpx

with httpx.Client(timeout=10, follow_redirects=True) as c:
    r = c.get("https://example.com")
```

### Async

```python
import asyncio
import httpx

async def main():
    async with httpx.AsyncClient(timeout=10) as c:
        tasks = [c.get(f"https://example.com/{i}") for i in range(10)]
        responses = await asyncio.gather(*tasks)
```

Với rate limit:

```python
async def main():
    sem = asyncio.Semaphore(5)
    async with httpx.AsyncClient() as c:
        async def get(url):
            async with sem:
                return await c.get(url)
        await asyncio.gather(*[get(u) for u in urls])
```

## 28.6 Playwright — trình duyệt thật

Cần khi site render bằng JavaScript, hoặc cần tương tác.

```bash
pip install playwright
playwright install chromium
```

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    page = browser.new_page()
    page.goto("https://example.com")
    page.wait_for_selector("article")
    tieu_de = page.inner_text("h1")
    print(tieu_de)
    browser.close()
```

### Async

```python
from playwright.async_api import async_playwright

async def main():
    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=True)
        page = await browser.new_page()
        await page.goto("https://example.com")
        await page.wait_for_selector("article")
        items = await page.query_selector_all("article")
        for it in items:
            print(await it.inner_text())
        await browser.close()
```

### Tương tác

```python
page.click("button#submit")
page.fill("input[name='email']", "a@x.com")
page.press("input[name='email']", "Enter")
page.select_option("select#country", "VN")
page.check("input[type='checkbox']")

# Chờ
page.wait_for_url("**/success")
page.wait_for_response(lambda r: "/api/" in r.url)
```

### Screenshot và PDF

```python
page.screenshot(path="shot.png", full_page=True)
page.pdf(path="out.pdf", format="A4")
```

### Chặn tài nguyên không cần

```python
async def block(route):
    if route.request.resource_type in ("image", "font", "media"):
        await route.abort()
    else:
        await route.continue_()

await page.route("**/*", block)
```

Tăng tốc đáng kể.

### Đăng nhập

```python
await page.goto("https://site/login")
await page.fill("#username", USER)
await page.fill("#password", PASS)
await page.click("button[type=submit]")
await page.wait_for_url("**/dashboard")

# Lưu session để tái sử dụng
await context.storage_state(path="state.json")
```

Lần sau:

```python
context = await browser.new_context(storage_state="state.json")
```

## 28.7 Selenium

Cũ hơn Playwright nhưng hỗ trợ rộng.

```bash
pip install selenium webdriver-manager
```

```python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

driver = webdriver.Chrome()
driver.get("https://example.com")

WebDriverWait(driver, 10).until(
    EC.presence_of_element_located((By.CSS_SELECTOR, "article"))
)
for el in driver.find_elements(By.CSS_SELECTOR, "article"):
    print(el.text)

driver.quit()
```

## 28.8 Parse có hệ thống

### Pydantic để validate

```python
from pydantic import BaseModel, HttpUrl

class BaiViet(BaseModel):
    tieu_de: str
    tac_gia: str
    ngay: str
    noi_dung: str

bai_viet = [
    BaiViet(
        tieu_de=item.select_one("h2").text,
        tac_gia=item.select_one(".author").text,
        ngay=item.select_one("time")["datetime"],
        noi_dung=item.select_one(".content").text,
    )
    for item in soup.select("article")
]
```

### Lưu kết quả

```python
import polars as pl

df = pl.DataFrame([b.model_dump() for b in bai_viet])
df.write_parquet("ket_qua.parquet")
```

### Checkpoint — chống mất tiến độ

```python
import json
from pathlib import Path

def luu_checkpoint(path: Path, da_xu_ly: set[str]) -> None:
    path.write_text(json.dumps(sorted(da_xu_ly)), encoding="utf-8")

def doc_checkpoint(path: Path) -> set[str]:
    if not path.exists():
        return set()
    return set(json.loads(path.read_text(encoding="utf-8")))
```

## 28.9 Automation desktop với pyautogui

Khi không có API, chỉ có GUI.

```bash
pip install pyautogui
```

```python
import pyautogui
import time

pyautogui.click(100, 200)
pyautogui.typewrite("xin chào", interval=0.05)
pyautogui.hotkey("ctrl", "s")
pyautogui.keyDown("shift")
pyautogui.press("f10")
pyautogui.keyUp("shift")

# Ảnh làm anchor
vi_tri = pyautogui.locateCenterOnScreen("button.png", confidence=0.9)
if vi_tri:
    pyautogui.click(vi_tri)
```

Cần `pip install opencv-python` để dùng `confidence`.

### Safety

`pyautogui.FAILSAFE = True` (mặc định) — đưa chuột vào góc trên trái để dừng khẩn cấp.

## 28.10 Automation CLI với subprocess

```python
import subprocess

ket_qua = subprocess.run(
    ["git", "status", "--porcelain"],
    capture_output=True,
    text=True,
    check=True,
    timeout=30,
)
print(ket_qua.stdout)
```

Không dùng `shell=True` với dữ liệu người dùng.

### Pipeline

```python
p1 = subprocess.Popen(["ls", "-l"], stdout=subprocess.PIPE)
p2 = subprocess.Popen(["wc", "-l"], stdin=p1.stdout, stdout=subprocess.PIPE)
p1.stdout.close()
out, _ = p2.communicate()
print(out)
```

## 28.11 Watchdog — theo dõi thay đổi file

```bash
pip install watchdog
```

```python
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler

class Handler(FileSystemEventHandler):
    def on_created(self, event):
        if not event.is_directory:
            print(f"file mới: {event.src_path}")

    def on_modified(self, event):
        if not event.is_directory:
            print(f"sửa: {event.src_path}")

observer = Observer()
observer.schedule(Handler(), ".", recursive=True)
observer.start()
try:
    while True:
        time.sleep(1)
finally:
    observer.stop()
    observer.join()
```

## 28.12 Bài tập

1. Scrape tiêu đề và link từ một trang tin (dùng site cho phép).
2. Thêm retry + rate limiter, tôn trọng robots.txt.
3. Dùng Playwright lấy dữ liệu từ trang render bằng JS.
4. Lưu session Playwright, chạy lại không cần đăng nhập.
5. Validate dữ liệu scrape bằng Pydantic, lưu Parquet.
6. Viết checkpoint để resume khi script dừng giữa chừng.
7. Viết script automation desktop điền form (dùng app của bạn).
8. Dùng watchdog tự build lại mỗi khi sửa file (giống `cargo watch`).

## 28.13 Ghi chú kỹ thuật

- `robots.txt` không ràng buộc pháp lý nhưng là quy ước nên tôn trọng.
- User-Agent thật giúp site biết ai đang gọi và giảm nguy cơ bị chặn.
- Playwright nhanh và ổn định hơn Selenium cho dự án mới.
- Browser headless vẫn tốn RAM — giới hạn số instance.
- Async + semaphore hiệu quả hơn thread cho nhiều request I/O.
- Lưu HTML thô trước khi parse — có thể reparse khi selector đổi.
- Selector dễ vỡ khi site đổi — viết test, ghi log khi selector fail.
- Tôn trọng `Crawl-delay` trong robots.txt.

Stashed.