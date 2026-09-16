# Chuyên Đề 26 — Authentication và Authorization

## 26.1 Phân biệt

- **Authentication** (authn): bạn là ai?
- **Authorization** (authz): bạn được làm gì?

Hai việc khác nhau. Token hợp lệ không có nghĩa có quyền.

## 26.2 Hash mật khẩu

```python
from argon2 import PasswordHasher
from argon2.exceptions import VerifyMismatchError

ph = PasswordHasher()

def tao_hash(mat_khau: str) -> str:
    return ph.hash(mat_khau)

def kiem_tra(mat_khau: str, hashed: str) -> bool:
    try:
        ph.verify(hashed, mat_khau)
        return True
    except VerifyMismatchError:
        return False
```

Nếu dùng bcrypt:

```python
import bcrypt

hashed = bcrypt.hashpw(b"matkhau", bcrypt.gensalt(rounds=12))
bcrypt.checkpw(b"matkhau", hashed)
```

Không bao giờ lưu mật khẩu thô. Không dùng MD5/SHA1/SHA256 trực tiếp.

### Rehash khi nâng cost

```python
if ph.check_needs_rehash(hashed):
    hashed = ph.hash(mat_khau)
```

## 26.3 Session cookie

Cách cổ điển, đơn giản, an toàn nếu cấu hình đúng.

```python
import secrets
from fastapi import FastAPI, Response, Cookie, HTTPException

app = FastAPI()
sessions: dict[str, int] = {}   # dùng Redis/DB thật

@app.post("/login")
def login(ten: str, mat_khau: str, response: Response):
    uid = xac_thuc(ten, mat_khau)
    if uid is None:
        raise HTTPException(401, "Sai thông tin")
    session_id = secrets.token_urlsafe(32)
    sessions[session_id] = uid
    response.set_cookie(
        "session_id",
        session_id,
        httponly=True,
        secure=True,
        samesite="lax",
        max_age=86400,
    )
    return {"ok": True}

@app.get("/me")
def me(session_id: str | None = Cookie(None)):
    if not session_id or session_id not in sessions:
        raise HTTPException(401)
    return {"uid": sessions[session_id]}
```

Thuộc tính cookie quan trọng:

- `HttpOnly`: JS không đọc được — chống XSS đánh cắp session.
- `Secure`: chỉ gửi qua HTTPS.
- `SameSite=Lax`: chống CSRF cơ bản.
- `Max-Age`/`Expires`: hết hạn.

## 26.4 JWT — JSON Web Token

Token tự chứa, không cần lưu server (stateless).

```bash
pip install pyjwt
```

```python
import jwt
from datetime import datetime, timedelta, timezone

SECRET = "..."   # từ biến môi trường

def tao_token(uid: int, vai_tro: str) -> str:
    payload = {
        "sub": str(uid),
        "role": vai_tro,
        "iat": datetime.now(timezone.utc),
        "exp": datetime.now(timezone.utc) + timedelta(minutes=15),
    }
    return jwt.encode(payload, SECRET, algorithm="HS256")

def giai_token(token: str) -> dict:
    try:
        return jwt.decode(token, SECRET, algorithms=["HS256"])
    except jwt.ExpiredSignatureError:
        raise ValueError("Token hết hạn")
    except jwt.InvalidTokenError:
        raise ValueError("Token không hợp lệ")
```

### Ba phần của JWT

`header.payload.signature`

- Header: thuật toán.
- Payload: claims (sub, exp, role...).
- Signature: HMAC hoặc RSA/ECDSA.

Payload **không được mã hóa** — ai cũng đọc được. Đừng để dữ liệu nhạy cảm trong đó.

### Thuật toán ký

- `HS256`: HMAC đối xứng — một secret cho cả ký và xác minh.
- `RS256`: RSA bất đối xứng — private key ký, public key xác minh.

Microservice nên dùng RS256: service chỉ cần public key.

### Cạm bẫy bảo mật

```python
# SAI — chấp nhận "none" algorithm
jwt.decode(token, options={"verify_signature": False})

# SAI — không chỉ định algorithm, chấp nhận mọi thứ
jwt.decode(token, SECRET)

# ĐÚNG — whitelist algorithm
jwt.decode(token, SECRET, algorithms=["HS256"])
```

### Access token vs refresh token

- **Access token**: ngắn hạn (5-15 phút), gửi mỗi request.
- **Refresh token**: dài hạn (7-30 ngày), chỉ dùng để lấy access token mới, lưu an toàn.

```python
def refresh(refresh_token: str) -> dict:
    payload = giai_token(refresh_token)
    if payload.get("typ") != "refresh":
        raise ValueError("Sai loại token")
    # kiểm tra refresh token chưa bị thu hồi
    if bi_thu_hoi(refresh_token):
        raise ValueError("Token đã bị thu hồi")
    return {"access_token": tao_token(int(payload["sub"]), payload["role"])}
```

### Thu hồi token

JWT stateless nên không thu hồi trực tiếp được. Cách xử lý:

- Danh sách đen (blacklist) token chưa hết hạn — tốn bộ nhớ.
- Dùng refresh token lưu DB, xóa khi logout.
- Access token ngắn hạn — cửa sổ rủi ro nhỏ.

## 26.5 OAuth2 và OpenID Connect

OAuth2 là ủy quyền (delegated authorization), không phải xác thực. OpenID Connect (OIDC) là lớp xác thực trên OAuth2.

### Authorization Code Flow (khuyến nghị cho web)

1. App chuyển hướng user tới provider.
2. User đăng nhập, đồng ý.
3. Provider chuyển hướng lại với `code`.
4. App đổi `code` lấy token (server-to-server).
5. App dùng token gọi API.

```python
from authlib.integrations.starlette_client import OAuth

oauth = OAuth()
oauth.register(
    name="google",
    client_id=CLIENT_ID,
    client_secret=CLIENT_SECRET,
    server_metadata_url="https://accounts.google.com/.well-known/openid-configuration",
    client_kwargs={"scope": "openid email profile"},
)

@app.get("/login")
async def login(request: Request):
    redirect_uri = request.url_for("auth")
    return await oauth.google.authorize_redirect(request, redirect_uri)

@app.get("/auth")
async def auth(request: Request):
    token = await oauth.google.authorize_access_token(request)
    user = token["userinfo"]
    return {"email": user["email"]}
```

### PKCE — cho ứng dụng public

```python
import secrets
import hashlib
import base64

verifier = secrets.token_urlsafe(64)
challenge = base64.urlsafe_b64encode(
    hashlib.sha256(verifier.encode()).digest()
).rstrip(b"=").decode()
```

Gửi `challenge` khi authorize, `verifier` khi đổi token. Ngăn chặn đánh cắp code.

## 26.6 Phân quyền

### Role-based (RBAC)

```python
from enum import Enum

class VaiTro(str, Enum):
    ADMIN = "admin"
    USER = "user"
    GUEST = "guest"

QUYEN = {
    VaiTro.ADMIN: {"doc", "ghi", "xoa"},
    VaiTro.USER: {"doc", "ghi"},
    VaiTro.GUEST: {"doc"},
}

def co_quyen(vai_tro: VaiTro, hanh_dong: str) -> bool:
    return hanh_dong in QUYEN.get(vai_tro, set())
```

### Dependency FastAPI

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

security = HTTPBearer()

def lay_nguoi_dung(
    creds: HTTPAuthorizationCredentials = Depends(security),
) -> dict:
    try:
        payload = giai_token(creds.credentials)
    except ValueError:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED)
    return payload

def yeu_cau_vai_tro(vai_tro: str):
    def checker(user: dict = Depends(lay_nguoi_dung)) -> dict:
        if user.get("role") != vai_tro:
            raise HTTPException(status.HTTP_403_FORBIDDEN)
        return user
    return checker

@app.delete("/admin/users/{uid}")
def xoa_user(uid: int, user: dict = Depends(yeu_cau_vai_tro("admin"))):
    ...
```

### Attribute-based (ABAC)

Phức tạp hơn, kiểm tra thuộc tính của cả chủ thể và tài nguyên.

```python
def co_the_sua(user: dict, bai_viet: dict) -> bool:
    if user["role"] == "admin":
        return True
    return bai_viet["author_id"] == user["sub"]
```

### Kiểm tra quyền sở hữu

```python
@app.get("/posts/{post_id}")
def lay_bai(post_id: int, user: dict = Depends(lay_nguoi_dung)):
    post = db.lay_post(post_id)
    if not post:
        raise HTTPException(404)
    if post["author_id"] != user["sub"] and user["role"] != "admin":
        raise HTTPException(403)
    return post
```

IDOR phổ biến vì quên bước kiểm tra này.

## 26.7 API key

Cho service-to-service hoặc public API.

```python
import secrets
import hmac

def tao_api_key() -> tuple[str, str]:
    raw = secrets.token_urlsafe(32)
    prefix = raw[:8]
    hashed = hash_api_key(raw)
    return raw, hashed   # lưu prefix + hashed

def hash_api_key(raw: str) -> str:
    import hashlib
    return hashlib.sha256(raw.encode()).hexdigest()

def kiem_tra_api_key(raw: str, hashed: str) -> bool:
    return hmac.compare_digest(hash_api_key(raw), hashed)
```

- Không lưu raw key.
- Prefix để tra cứu nhanh.
- `compare_digest` chống timing attack.

## 26.8 Multi-factor authentication

```python
import pyotp

def tao_bi_mat_totp() -> str:
    return pyotp.random_base32()

def tao_uri(bi_mat: str, email: str) -> str:
    return pyotp.totp.TOTP(bi_mat).provisioning_uri(
        name=email, issuer_name="MyApp"
    )

def kiem_tra_totp(bi_mat: str, ma: str) -> bool:
    return pyotp.TOTP(bi_mat).verify(ma, valid_window=1)
```

`valid_window=1` cho phép lệch 30s để tránh lỗi đồng hồ.

### Backup code

```python
def tao_backup_codes(n: int = 10) -> list[str]:
    return [secrets.token_hex(4) for _ in range(n)]
```

Lưu hash, mỗi code dùng một lần.

## 26.9 Bảo mật token ở client

- **Web**: HttpOnly cookie — không lưu trong `localStorage`.
- **Mobile**: keychain/keystore.
- **CLI**: file quyền `600` trong thư mục config.
- **SPA**: nếu phải dùng `localStorage`, access token ngắn hạn và refresh token trong cookie HttpOnly.

## 26.10 Checklist

- [ ] Mật khẩu hash Argon2/bcrypt, không plaintext.
- [ ] HTTPS bắt buộc, cookie `Secure`.
- [ ] Cookie `HttpOnly` + `SameSite`.
- [ ] JWT whitelist algorithm, secret mạnh.
- [ ] Access token ngắn hạn, refresh token có thể thu hồi.
- [ ] Không để dữ liệu nhạy cảm trong JWT payload.
- [ ] Kiểm tra quyền sở hữu mọi endpoint.
- [ ] Rate limiting login và reset mật khẩu.
- [ ] Log lần đăng nhập thất bại.
- [ ] Cảnh báo khi phát hiện bất thường.
- [ ] Xoay secret định kỳ.
- [ ] MFA cho tài khoản quản trị.

## 26.11 Bài tập

1. Viết API đăng ký/đăng nhập dùng Argon2 + session cookie.
2. Thêm JWT access token + refresh token, endpoint refresh.
3. Viết middleware FastAPI kiểm tra Bearer token.
4. Thêm phân quyền admin/user bằng dependency.
5. Viết login Google với Authlib (dùng client test).
6. Thêm TOTP 2FA, sinh QR code.
7. Viết API key service-to-service có prefix + hash.
8. Kiểm tra IDOR: cố truy cập tài nguyên người khác, phải bị chặn.

## 26.12 Ghi chú kỹ thuật

- Session server-side cần store phân tán (Redis) khi scale ngang.
- JWT không phù hợp khi cần thu hồi tức thì.
- Không đặt `SECRET` trong code; dùng secret manager.
- Xoay signing key: hỗ trợ nhiều key khi verify, ký bằng key mới.
- `SameSite=Strict` chặt hơn nhưng phá link từ ngoài vào.
- CSRF vẫn có thể xảy ra với `SameSite=Lax` — dùng token nếu cần.
- Log authentication không nên log mật khẩu/token.
- OAuth callback URI phải khớp chính xác — cấu hình allowlist.

Stashed.