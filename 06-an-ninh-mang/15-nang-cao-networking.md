# Chương 15 — Networking Nâng Cao: Sockets, Protocol, Client/Server

## Mô hình TCP/IP ôn lại

Một kết nối TCP được xác định bởi bộ 4: `(src_ip, src_port, dst_ip, dst_port)`. Socket là file descriptor ở tầng OS; Python bọc nó bằng đối tượng `socket`.

```text
Ứng dụng   → HTTP, DNS, SMTP ...
Vận chuyển → TCP, UDP         (port)
Internet   → IP, ICMP         (địa chỉ)
Liên kết   → Ethernet, ARP    (MAC)
```

## socket thô — TCP client

```python
import socket

def tcp_client(host: str, port: int, payload: bytes) -> bytes:
    with socket.create_connection((host, port), timeout=5) as sock:
        sock.sendall(payload)
        sock.shutdown(socket.SHUT_WR)
        chunks = []
        while True:
            chunk = sock.recv(4096)
            if not chunk:
                break
            chunks.append(chunk)
        return b"".join(chunks)
```

`create_connection` xử lý cả IPv4/IPv6 và timeout. `shutdown(SHUT_WR)` gửi FIN để server biết ta gửi xong.

## TCP server

```python
import socket
import threading

def handle(conn: socket.socket, addr: tuple) -> None:
    with conn:
        while True:
            data = conn.recv(4096)
            if not data:
                break
            conn.sendall(data.upper())

def serve(host: str, port: int) -> None:
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as srv:
        srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        srv.bind((host, port))
        srv.listen(128)
        while True:
            conn, addr = srv.accept()
            threading.Thread(target=handle, args=(conn, addr), daemon=True).start()
```

`SO_REUSEADDR` cho phép bind lại ngay sau khi server tắt, tránh `TIME_WAIT`.

## UDP

```python
import socket

def udp_echo_server(port: int) -> None:
    with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as s:
        s.bind(("0.0.0.0", port))
        while True:
            data, addr = s.recvfrom(65535)
            s.sendto(data, addr)
```

UDP không kết nối, không đảm bảo thứ tự. Dùng cho DNS, game, streaming.

## Timeout và non-blocking

```python
sock = socket.socket()
sock.settimeout(3.0)          # timeout cho mọi thao tác
try:
    sock.connect(("example.com", 80))
except socket.timeout:
    print("quá thời gian")

sock.setblocking(False)       # non-blocking
try:
    data = sock.recv(1024)
except BlockingIOError:
    pass                       # chưa có dữ liệu
```

Non-blocking + `selectors` cho phép một luồng quản lý nhiều socket:

```python
import selectors

sel = selectors.DefaultSelector()

def register(sock):
    sel.register(sock, selectors.EVENT_READ, data=None)
```

## Phân giải DNS

```python
import socket

print(socket.gethostbyname("example.com"))       # A record
print(socket.getaddrinfo("example.com", 443))    # mọi họ địa chỉ

# Reverse DNS
print(socket.gethostbyaddr("93.184.216.34"))
```

## Xây dựng giao thức có khung (framing)

TCP là luồng byte, không có ranh giới message. Phải tự đóng khung.

```python
import struct

def send_message(sock: socket.socket, payload: bytes) -> None:
    header = struct.pack("!I", len(payload))
    sock.sendall(header + payload)

def recv_message(sock: socket.socket) -> bytes:
    header = recv_exact(sock, 4)
    length = struct.unpack("!I", header)[0]
    return recv_exact(sock, length)

def recv_exact(sock: socket.socket, n: int) -> bytes:
    buf = bytearray()
    while len(buf) < n:
        chunk = sock.recv(n - len(buf))
        if not chunk:
            raise ConnectionError("kết nối đóng giữa chừng")
        buf.extend(chunk)
    return bytes(buf)
```

`!I` là big-endian uint32. Đây là mẫu chuẩn cho mọi giao thức nhị phân.

## HTTP bằng tay

```python
import socket

def http_get(host: str, path: str = "/") -> str:
    request = (
        f"GET {path} HTTP/1.1\r\n"
        f"Host: {host}\r\n"
        f"Connection: close\r\n"
        f"User-Agent: book-client\r\n"
        f"\r\n"
    )
    with socket.create_connection((host, 80), timeout=5) as s:
        s.sendall(request.encode())
        return s.recv(65535).decode("utf-8", errors="replace")
```

Hiểu HTTP ở mức byte giúp debug mọi vấn đề tầng ứng dụng sau này.

## TLS

```python
import ssl
import socket

ctx = ssl.create_default_context()
with socket.create_connection(("example.com", 443)) as raw:
    with ctx.wrap_socket(raw, server_hostname="example.com") as tls:
        print(tls.version())
        cert = tls.getpeercert()
        print(cert["subject"])
```

`create_default_context()` xác thực chứng chỉ. Không bao giờ dùng `ssl._create_unverified_context()` trong code thật.

## Asyncio sockets

```python
import asyncio

async def handle(reader, writer):
    while data := await reader.read(4096):
        writer.write(data)
        await writer.drain()
    writer.close()

async def main():
    server = await asyncio.start_server(handle, "127.0.0.1", 8888)
    async with server:
        await server.serve_forever()

asyncio.run(main())
```

## Nguyên tắc thiết kế

- Luôn đặt timeout — không có timeout nghĩa là treo vô hạn.
- Đóng socket bằng `with` hoặc `try/finally`.
- Xử lý `ConnectionResetError`, `BrokenPipeError` — chúng xảy ra bình thường.
- Giao thức nhị phân cần version field để tiến hóa tương thích.
- Giới hạn kích thước message để chống tấn công tài nguyên.

## Bài tập

1. Viết chat server nhiều phòng bằng asyncio.
2. Viết client HTTP có hỗ trợ redirect.
3. Viết giao thức nhị phân có header version + CRC32.
4. Viết proxy TCP đơn giản chuyển tiếp byte hai chiều.
5. Viết DNS query bằng tay (gửi UDP tới 8.8.8.8 cổng 53).

Stashed.