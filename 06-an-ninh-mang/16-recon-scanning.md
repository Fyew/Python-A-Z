# Chương 16 — Recon và Scanning (Góc nhìn Red Team)

> Chương này mang tính giáo dục. Mọi kỹ thuật chỉ nên chạy trên hệ thống bạn sở hữu hoặc được phép kiểm thử. Quét cổng hệ thống người khác không được phép là vi phạm pháp luật ở hầu hết quốc gia.

## Vòng đời kiểm thử xâm nhập

```text
Recon → Scanning → Enumeration → Exploitation → Post-exploitation → Reporting
```

Recon (trinh sát) là thu thập thông tin trước khi chạm mục tiêu. Scanning là chủ động dò tìm dịch vụ.

## OSINT — thông tin công khai

Trước khi quét, thu thập từ nguồn mở:

- WHOIS: chủ sở hữu domain, nameserver.
- DNS records: A, AAAA, MX, TXT, NS.
- Certificate Transparency logs: tên subdomain lộ qua chứng chỉ.
- Shodan/Censys: thiết bị lộ ra internet.

```python
import socket

def dns_lookup(domain: str) -> dict:
    result = {}
    for family in (socket.AF_INET, socket.AF_INET6):
        try:
            infos = socket.getaddrinfo(domain, None, family)
            result[family.name] = sorted({i[4][0] for i in infos})
        except socket.gaierror:
            result[family.name] = []
    return result
```

## TCP connect scan

Cách đơn giản nhất: thử `connect()` và xem thành công hay không.

```python
import socket
from concurrent.futures import ThreadPoolExecutor

def scan_port(host: str, port: int, timeout: float = 1.0) -> bool:
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.settimeout(timeout)
        return s.connect_ex((host, port)) == 0

def scan_range(host: str, ports: range, workers: int = 200) -> list[int]:
    open_ports = []
    with ThreadPoolExecutor(max_workers=workers) as ex:
        futures = {ex.submit(scan_port, host, p): p for p in ports}
        for fut in futures:
            if fut.result():
                open_ports.append(futures[fut])
    return sorted(open_ports)
```

`connect_ex` trả về mã lỗi thay vì ném ngoại lệ — nhanh hơn trong vòng lặp.

Lưu ý: connect scan tạo log đầy đủ ở phía mục tiêu. SYN scan ("half-open") kín đáo hơn nhưng cần quyền raw socket — xem chương 17.

## Banner grabbing

Nhiều dịch vụ gửi banner ngay khi kết nối.

```python
import socket

def grab_banner(host: str, port: int, timeout: float = 3.0) -> str:
    try:
        with socket.create_connection((host, port), timeout=timeout) as s:
            s.settimeout(timeout)
            return s.recv(1024).decode("utf-8", errors="replace").strip()
    except (OSError, socket.timeout):
        return ""
```

Ví dụ banner SSH: `SSH-2.0-OpenSSH_8.9p1 Ubuntu-3`. Tiết lộ phiên bản để tra CVE.

## Service fingerprinting

```python
def probe_http(host: str, port: int) -> str:
    import socket
    req = b"HEAD / HTTP/1.0\r\n\r\n"
    with socket.create_connection((host, port), timeout=3) as s:
        s.sendall(req)
        return s.recv(4096).decode("utf-8", errors="replace")
```

Header `Server:` tiết lộ phần mềm. `X-Powered-By` tiết lộ framework.

## UDP scan

UDP khó quét hơn vì không có phản hồi khi cổng đóng. Kỹ thuật: gửi gói và chờ ICMP port unreachable.

```python
import socket

def udp_probe(host: str, port: int, timeout: float = 2.0) -> bool:
    with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as s:
        s.settimeout(timeout)
        try:
            s.sendto(b"\x00", (host, port))
            s.recvfrom(1024)
            return True
        except socket.timeout:
            return False
        except ConnectionRefusedError:
            return False
```

## Phân tích kết quả

Chuyển danh sách cổng mở thành bảng dịch vụ:

```python
COMMON_PORTS = {
    21: "ftp", 22: "ssh", 23: "telnet", 25: "smtp", 53: "dns",
    80: "http", 110: "pop3", 143: "imap", 443: "https",
    445: "smb", 3306: "mysql", 3389: "rdp", 5432: "postgres",
    6379: "redis", 8080: "http-alt", 27017: "mongodb",
}
```

## Tìm subdomain

```python
import socket

def enumerate_subdomains(domain: str, wordlist: list[str]) -> list[str]:
    found = []
    for sub in wordlist:
        host = f"{sub}.{domain}"
        try:
            socket.gethostbyname(host)
            found.append(host)
        except socket.gaierror:
            continue
    return found
```

## Phòng thủ

- Đóng mọi cổng không cần thiết. Bề mặt tấn công nhỏ = rủi ro nhỏ.
- Không để banner lộ phiên bản chính xác (SSH `DebianBanner no`, Nginx `server_tokens off`).
- Dùng port knocking hoặc VPN cho dịch vụ quản trị.
- Giám sát log: nhiều connect thất bại liên tiếp = dấu hiệu scan.
- Rate limiting và fail2ban chặn scan tự động.

## Bài tập (trên lab của bạn)

1. Dựng Metasploitable2 trong VM, quét và liệt kê dịch vụ.
2. Viết scanner đọc danh sách host từ file, xuất CSV.
3. So sánh tốc độ connect scan vs SYN scan (chương 17).
4. Viết script phát hiện scan từ log SSH (đếm IP sai liên tiếp).

Stashed.