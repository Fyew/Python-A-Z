# Chương 18 — Mật Mã Học Trong Python

## Nguyên tắc vàng

**Không tự viết thuật toán mật mã.** Dùng thư viện đã được kiểm định. Tự chế crypto là cách nhanh nhất để mất dữ liệu.

```bash
pip install cryptography
```

## Hàm băm (hash)

Hàm băm ánh xạ dữ liệu tùy ý thành chuỗi cố định, một chiều.

```python
import hashlib

h = hashlib.sha256(b"hello").hexdigest()
print(h)

# Băm theo khối cho file lớn
def hash_file(path: str, algo=hashlib.sha256) -> str:
    digest = algo()
    with open(path, "rb") as f:
        for chunk in iter(lambda: f.read(65536), b""):
            digest.update(chunk)
    return digest.hexdigest()
```

SHA-256 cho toàn vẹn. MD5/SHA-1 đã bị phá — chỉ dùng cho checksum không bảo mật.

### HMAC — xác thực thông điệp

```python
import hmac
import hashlib

def sign(key: bytes, message: bytes) -> bytes:
    return hmac.new(key, message, hashlib.sha256).digest()

def verify(key: bytes, message: bytes, tag: bytes) -> bool:
    expected = sign(key, message)
    return hmac.compare_digest(expected, tag)
```

`compare_digest` so sánh thời gian hằng, chống timing attack.

## Băm mật khẩu

Không bao giờ băm mật khẩu bằng SHA trực tiếp — quá nhanh, dễ brute force.

```python
from argon2 import PasswordHasher

ph = PasswordHasher()
hashed = ph.hash("matkhau123")
ph.verify(hashed, "matkhau123")     # True, ném ngoại lệ nếu sai
```

Argon2 là chuẩn hiện tại (khuyến nghị OWASP). bcrypt là lựa chọn tốt khác.

```python
import bcrypt

hashed = bcrypt.hashpw(b"matkhau123", bcrypt.gensalt(rounds=12))
bcrypt.checkpw(b"matkhau123", hashed)
```

Salt tự động sinh và nhúng trong hash. Cost factor điều chỉnh độ chậm.

## Mã hóa đối xứng — AES

```python
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import os

key = AESGCM.generate_key(bit_length=256)
aes = AESGCM(key)

nonce = os.urandom(12)
ciphertext = aes.encrypt(nonce, b"thông điệp bí mật", b"associated-data")
plaintext = aes.decrypt(nonce, ciphertext, b"associated-data")
```

AES-GCM vừa mã hóa vừa xác thực toàn vẹn. **Không bao giờ tái sử dụng nonce** với cùng key — thảm họa bảo mật.

### Cảnh báo về mode

- Dùng GCM hoặc ChaCha20-Poly1305 (AEAD).
- Không dùng ECB — lộ pattern.
- CBC cần padding đúng và không có xác thực toàn vẹn (dễ padding oracle).

## Mã hóa bất đối xứng — RSA

```python
from cryptography.hazmat.primitives.asymmetric import rsa, padding
from cryptography.hazmat.primitives import hashes

private_key = rsa.generate_private_key(public_exponent=65537, key_size=3072)
public_key = private_key.public_key()

ciphertext = public_key.encrypt(
    b"bí mật",
    padding.OAEP(mgf=padding.MGF1(hashes.SHA256()), algorithm=hashes.SHA256(), label=None),
)
plaintext = private_key.decrypt(ciphertext, ...)
```

RSA chỉ mã hóa được dữ liệu nhỏ. Thực tế dùng RSA để trao đổi khóa, rồi AES mã hóa dữ liệu.

## Chữ ký số

```python
from cryptography.hazmat.primitives.asymmetric import ec
from cryptography.hazmat.primitives import hashes

private_key = ec.generate_private_key(ec.SECP256R1())
signature = private_key.sign(b"dữ liệu", ec.ECDSA(hashes.SHA256()))
private_key.public_key().verify(signature, b"dữ liệu", ec.ECDSA(hashes.SHA256()))
```

Chữ ký chứng minh nguồn gốc và toàn vẹn, không che giấu nội dung.

## Trao đổi khóa — Diffie-Hellman

```python
from cryptography.hazmat.primitives.asymmetric import x25519

alice_priv = x25519.X25519PrivateKey.generate()
bob_priv = x25519.X25519PrivateKey.generate()

alice_shared = alice_priv.exchange(bob_priv.public_key())
bob_shared = bob_priv.exchange(alice_priv.public_key())
assert alice_shared == bob_shared
```

Hai bên tạo khóa chung qua kênh công khai mà không ai nghe lén biết được.

## Dẫn xuất khóa

```python
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC
from cryptography.hazmat.primitives import hashes
import os

salt = os.urandom(16)
kdf = PBKDF2HMAC(algorithm=hashes.SHA256(), length=32, salt=salt, iterations=600_000)
key = kdf.derive(b"matkhau")
```

Dùng để biến mật khẩu thành khóa. Iterations cao chống brute force.

## Sinh số ngẫu nhiên an toàn

```python
import secrets

token = secrets.token_urlsafe(32)
number = secrets.randbelow(100)
```

`secrets` dùng CSPRNG. `random` KHÔNG an toàn cho bảo mật — có thể đoán được seed.

## Lưu trữ khóa

- Không hardcode khóa trong source.
- Dùng biến môi trường hoặc secret manager (Vault, AWS Secrets Manager).
- File khóa cần quyền `600`.
- Xoay khóa định kỳ.

## Lỗi thường gặp

| Lỗi | Hậu quả |
|-----|---------|
| Tái dùng nonce AES-GCM | Lộ plaintext, mất toàn vẹn |
| Dùng `random` cho token | Token đoán được |
| So sánh bằng `==` | Timing attack |
| ECB mode | Lộ pattern dữ liệu |
| Băm mật khẩu bằng SHA | Brute force nhanh |
| Padding oracle | Giải mã không cần khóa |

## Bài tập

1. Viết công cụ mã hóa/giải mã file bằng AES-GCM với mật khẩu.
2. Viết hệ thống ký và xác minh chứng chỉ đơn giản.
3. Viết password manager lưu Argon2 hash.
4. Viết chương trình trao đổi tin nhắn mã hóa đầu cuối bằng X25519 + AES.
5. So sánh tốc độ băm của MD5, SHA-256, Argon2 bằng timeit.

Stashed.