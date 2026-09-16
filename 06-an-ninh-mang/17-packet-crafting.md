# Chương 17 — Packet Crafting, Sniffing và Phân Tích

> Kỹ thuật trong chương này cần quyền root và chỉ dùng trên mạng bạn quản lý. Bắt gói tin mạng người khác là vi phạm pháp luật.

## Raw socket và quyền hạn

Đọc/ghi gói tin thô cần `CAP_NET_RAW` (root). Trên Linux:

```bash
sudo setcap cap_net_raw,cap_net_admin+eip $(which python3.12)
```

## Cấu trúc gói tin

Một gói Ethernet/IP/TCP:

```text
[ Ethernet 14B ][ IP header 20B ][ TCP header 20B ][ payload ]
```

Địa chỉ MAC đích/nguồn, EtherType (0x0800 = IPv4). IP header chứa TTL, protocol, checksum. TCP header chứa port, sequence, flags.

## Scapy — thư viện thao tác gói tin

```bash
pip install scapy
```

### Tạo và gửi gói

```python
from scapy.all import IP, ICMP, TCP, sr1, send

packet = IP(dst="1.1.1.1") / ICMP()
reply = sr1(packet, timeout=2)
if reply:
    print(reply.summary())
```

### SYN scan (half-open)

```python
from scapy.all import IP, TCP, sr1, RandShort

def syn_scan(host: str, port: int, timeout: float = 2.0) -> str:
    pkt = IP(dst=host) / TCP(sport=RandShort(), dport=port, flags="S")
    resp = sr1(pkt, timeout=timeout, verbose=0)
    if resp is None:
        return "filtered"
    if resp.haslayer(TCP):
        flags = resp[TCP].flags
        if flags & 0x12 == 0x12:      # SYN-ACK
            sr1(IP(dst=host) / TCP(sport=port, dport=port, flags="R"), timeout=1, verbose=0)
            return "open"
        if flags & 0x14 == 0x14:      # RST-ACK
            return "closed"
    return "unknown"
```

SYN scan không hoàn tất bắt tay ba bước nên ít để lại log ứng dụng hơn connect scan.

### Sniffing

```python
from scapy.all import sniff, IP, TCP

def on_packet(pkt):
    if pkt.haslayer(IP) and pkt.haslayer(TCP):
        ip = pkt[IP]
        tcp = pkt[TCP]
        print(f"{ip.src}:{tcp.sport} → {ip.dst}:{tcp.dport} flags={tcp.flags}")

sniff(iface="eth0", prn=on_packet, store=False, count=20)
```

### Lọc BPF

```python
sniff(filter="tcp port 80", prn=on_packet, count=10)
```

BPF filter chạy ở kernel, hiệu quả hơn lọc trong Python.

### Ghi và đọc pcap

```python
from scapy.all import rdpcap, wrpcap

packets = rdpcap("capture.pcap")
for pkt in packets:
    if pkt.haslayer(TCP):
        print(pkt[TCP].payload)

wrpcap("out.pcap", packets)
```

## ARP và tầng liên kết

```python
from scapy.all import ARP, Ether, srp

def arp_scan(subnet: str) -> list[dict]:
    pkt = Ether(dst="ff:ff:ff:ff:ff:ff") / ARP(pdst=subnet)
    answered, _ = srp(pkt, timeout=2, verbose=0)
    return [{"ip": r[1].psrc, "mac": r[1].hwsrc} for r in answered]
```

## DNS query bằng tay

```python
from scapy.all import IP, UDP, DNS, DNSQR, sr1

def dns_query(name: str, server: str = "8.8.8.8") -> list[str]:
    pkt = IP(dst=server) / UDP(dport=53) / DNS(rd=1, qd=DNSQR(qname=name, qtype="A"))
    resp = sr1(pkt, timeout=3, verbose=0)
    if resp and resp.haslayer(DNS):
        return [r.rdata for r in resp[DNS].an] if resp[DNS].ancount else []
    return []
```

## Traceroute

```python
from scapy.all import IP, ICMP, sr1

def traceroute(host: str, max_hops: int = 30) -> list[str]:
    path = []
    for ttl in range(1, max_hops + 1):
        pkt = IP(dst=host, ttl=ttl) / ICMP()
        resp = sr1(pkt, timeout=2, verbose=0)
        if resp is None:
            path.append("*")
            continue
        path.append(resp.src)
        if resp.type == 0:      # echo reply
            break
    return path
```

## Phân tích gói nâng cao

Đọc pcap và trích luồng HTTP:

```python
from scapy.all import rdpcap, TCP

def extract_http_requests(path: str) -> list[bytes]:
    requests = []
    for pkt in rdpcap(path):
        if pkt.haslayer(TCP) and pkt[TCP].dport == 80 and pkt[TCP].payload:
            payload = bytes(pkt[TCP].payload)
            if payload.startswith((b"GET", b"POST", b"HEAD")):
                requests.append(payload.split(b"\r\n\r\n")[0])
    return requests
```

## Phòng thủ

- Chuyển toàn bộ sang TLS để sniffing không đọc được nội dung.
- Dùng switch thay hub; bật port security chống ARP spoofing.
- 802.1X xác thực thiết bị trước khi vào mạng.
- Phát hiện ARP spoofing bằng cách so khớp bảng ARP định kỳ.
- IDS (Suricata, Zeek) phát hiện pattern scan và exploit.

## Bài tập (lab riêng)

1. Viết công cụ ARP scan và so sánh với `nmap -sn`.
2. Viết sniffer lọc mật khẩu HTTP POST (trên lab của bạn).
3. Viết script phát hiện ARP spoofing bằng so khớp MAC.
4. Viết DNS spoofing demo trong mạng ảo, rồi viết phòng thủ.
5. Đọc pcap mẫu, trích xuất toàn bộ file transfer.

Stashed.