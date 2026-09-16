# Chương 19 — Web Security Trong Python

> Toàn bộ nội dung chỉ áp dụng cho hệ thống bạn sở hữu hoặc được phép kiểm thử. Dùng các nền tảng luyện tập có chủ đích như DVWA, PortSwigger Web Security Academy, OWASP Juice Shop.

## HTTP ở mức thấp

```python
import http.client

conn = http.client.HTTPSConnection("example.com", timeout=10)
conn.request("GET", "/", headers={"User-Agent": "book-client"})
resp = conn.getresponse()
print(resp.status, resp.reason)
print(resp.read().decode()[:500])
conn.close()
```

## requests — thư viện chuẩn

```python
import requests

r = requests.get("https://api.github.com", timeout=10)
print(r.status_code, r.headers["content-type"])
print(r.json())

r = requests.post("https://httpbin.org/post", json={"a": 1}, timeout=10)
```

### Session và cookie

```python
s = requests.Session()
s.headers.update({"User-Agent": "book-client"})
s.get("https://httpbin.org/cookies/set?session=abc", timeout=10)
print(s.cookies.get("session"))
```

### Retry và backoff

```python
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

session = requests.Session()
retry = Retry(total=5, backoff_factor=0.5, status_forcelist=[500, 502, 503, 504])
session.mount("https://", HTTPAdapter(max_retries=retry))
```

## Các lớp lỗ hổng web phổ biến

### 1. Injection (SQL, NoSQL, Command)

Nguyên nhân gốc: dữ liệu người dùng được ghép vào câu lệnh.

```python
# DỄ BỊ TẤN CÔNG
def tim_user_sai(cur, ten):
    cur.execute(f"SELECT * FROM users WHERE name = '{ten}'")
    # ten = "' OR '1'='1" → trả về toàn bộ bảng

# AN TOÀN — tham số hóa
def tim_user_dung(cur, ten):
    cur.execute("SELECT * FROM users WHERE name = ?", (ten,))
    return cur.fetchall()
```

Quy tắc: **không bao giờ** nối chuỗi vào câu lệnh SQL/NoSQL/shell. Dùng tham số hóa hoặc ORM.

Command injection tương tự:

```python
# SAI
import os
os.system(f"ping -c 1 {host}")     # host = "8.8.8.8; rm -rf /"

# ĐÚNG
import subprocess
subprocess.run(["ping", "-c", "1", host], check=True)
```

`subprocess` với list args không đi qua shell, nên ký tự đặc biệt không được diễn giải.

### 2. XSS — Cross-Site Scripting

Đầu ra nhúng thẳng HTML/JS từ người dùng.

```python
# SAI — template ghép chuỗi
def render_sai(ten):
    return f"<h1>Xin chào {ten}</h1>"     # ten = "<script>...</script>"

# ĐÚNG — tự động escape
from markupsafe import escape
def render_dung(ten):
    return f"<h1>Xin chào {escape(ten)}</h1>"
```

Framework hiện đại (Jinja2, Django templates) tự escape mặc định. Vấn đề xảy ra khi tắt autoescape hoặc dùng `|safe` bừa.

Phòng thủ tầng sâu: Content-Security-Policy chặn inline script.

### 3. CSRF — Cross-Site Request Forgery

Kẻ tấn công lừa trình duyệt nạn nhân gửi request hợp lệ.

```python
# Phòng thủ: CSRF token gắn với session
import secrets

def tao_csrf_token(session) -> str:
    token = secrets.token_urlsafe(32)
    session["csrf"] = token
    return token

def kiem_tra_csrf(session, token_gui_len: str) -> bool:
    expected = session.get("csrf", "")
    return secrets.compare_digest(expected, token_gui_len)
```

Cookie `SameSite=Lax` hoặc `Strict` chặn phần lớn CSRF hiện đại.

### 4. SSRF — Server-Side Request Forgery

Server thay mặt kẻ tấn công gọi URL nội bộ.

```python
from urllib.parse import urlparse

ALLOWED_HOSTS = {"api.example.com"}

def fetch_an_toan(url: str) -> bytes:
    parsed = urlparse(url)
    if parsed.scheme not in ("http", "https"):
        raise ValueError("scheme không hợp lệ")
    if parsed.hostname not in ALLOWED_HOSTS:
        raise ValueError("host không được phép")
    import requests
    return requests.get(url, timeout=5).content
```

Kiểm tra allowlist host, chặn IP nội bộ (127.0.0.1, 169.254.169.254 metadata cloud).

### 5. Insecure Deserialization

```python
# CỰC KỲ NGUY HIỂM với dữ liệu không tin cậy
import pickle
data = pickle.loads(user_input)     # có thể thực thi code tùy ý
```

Không bao giờ `pickle.loads` dữ liệu từ nguồn không tin cậy. Dùng JSON + schema validation.

### 6. Broken Access Control

```python
# SAI — kiểm tra quyền ở client
@app.get("/admin")
def admin(request):
    return dashboard()

# ĐÚNG — kiểm tra ở server, mọi request
@app.get("/admin")
def admin(request):
    user = get_current_user(request)
    if user is None or not user.is_admin:
        raise HTTPException(403)
    return dashboard()
```

IDOR (Insecure Direct Object Reference): đổi `?id=5` thành `?id=6` xem dữ liệu người khác. Luôn kiểm tra quyền sở hữu.

## Security headers

```python
SECURITY_HEADERS = {
    "Strict-Transport-Security": "max-age=31536000; includeSubDomains",
    "X-Content-Type-Options": "nosniff",
    "X-Frame-Options": "DENY",
    "Content-Security-Policy": "default-src 'self'",
    "Referrer-Policy": "strict-origin-when-cross-origin",
}
```

## Kiểm thử tự động

### Quét với OWASP ZAP API

```python
import requests

zap = "http://127.0.0.1:8080"
requests.get(f"{zap}/JSON/spider/action/scan/?url=https://target", timeout=10)
alerts = requests.get(f"{zap}/JSON/core/view/alerts/", timeout=10).json()
for a in alerts["alerts"]:
    print(a["risk"], a["name"], a["url"])
```

### Kiểm tra input validation

```python
import pytest

@pytest.mark.parametrize("payload", [
    "' OR '1'='1",
    "<script>alert(1)</script>",
    "../../etc/passwd",
    "8.8.8.8; rm -rf /",
])
def test_reject_malicious(payload):
    with pytest.raises(ValueError):
        validate_input(payload)
```

## Checklist bảo mật web

- [ ] Tham số hóa mọi truy vấn.
- [ ] Escape mọi output.
- [ ] CSRF token + SameSite cookie.
- [ ] Kiểm tra quyền ở server cho mọi endpoint.
- [ ] HTTPS bắt buộc, HSTS.
- [ ] Security headers đầy đủ.
- [ ] Rate limiting chống brute force.
- [ ] Log và giám sát bất thường.
- [ ] Cập nhật dependency, quét CVE.
- [ ] Không lộ stack trace ra production.

## Bài tập (trên lab có chủ đích)

1. Cài OWASP Juice Shop (Docker), luyện các bài SQLi/XSS.
2. Viết scanner kiểm tra security headers thiếu.
3. Viết fuzzer input tìm lỗi 500.
4. Viết middleware FastAPI thêm security headers.
5. Sửa một app có chủ đích lỗ hổng trong DVWA.

Stashed.