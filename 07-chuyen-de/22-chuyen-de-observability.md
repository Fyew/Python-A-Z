# Chuyên Đề 22 — Observability: Log, Metrics, Tracing

## 22.1 Ba trụ cột

- **Logs**: sự kiện rời rạc, có ngữ cảnh.
- **Metrics**: số liệu tổng hợp theo thời gian.
- **Traces**: đường đi của một request qua hệ thống.

Không có ba thứ này, debug production là đoán mò.

## 22.2 Log có cấu trúc (structured logging)

Log dạng text khó parse. Log JSON dễ query.

```python
import json
import logging
import sys
from datetime import datetime, timezone

class JsonFormatter(logging.Formatter):
    def format(self, record: logging.LogRecord) -> str:
        payload = {
            "ts": datetime.now(timezone.utc).isoformat(),
            "level": record.levelname,
            "logger": record.name,
            "msg": record.getMessage(),
        }
        if record.exc_info:
            payload["exc"] = self.formatException(record.exc_info)
        for key, value in getattr(record, "extra_fields", {}).items():
            payload[key] = value
        return json.dumps(payload, ensure_ascii=False)

handler = logging.StreamHandler(sys.stdout)
handler.setFormatter(JsonFormatter())

logging.basicConfig(level=logging.INFO, handlers=[handler])
logger = logging.getLogger("app")

logger.info("người dùng đăng nhập", extra={"extra_fields": {"user_id": 42}})
```

Kết quả:

```json
{"ts":"2026-09-16T10:00:00+00:00","level":"INFO","logger":"app","msg":"người dùng đăng nhập","user_id":42}
```

### Không log dữ liệu nhạy cảm

```python
import re

SENSITIVE = re.compile(r"(password|token|secret|api[_-]?key|authorization)", re.IGNORECASE)

class RedactingFilter(logging.Filter):
    def filter(self, record: logging.LogRecord) -> bool:
        msg = record.getMessage()
        if SENSITIVE.search(msg):
            record.msg = SENSITIVE.sub(r"\1=***", msg)
            record.args = ()
        return True
```

## 22.3 Correlation ID — xâu chuỗi log

```python
import contextvars
import uuid
import logging

request_id: contextvars.ContextVar[str] = contextvars.ContextVar("request_id", default="-")

class RequestIdFilter(logging.Filter):
    def filter(self, record: logging.LogRecord) -> bool:
        record.request_id = request_id.get()
        return True

logging.getLogger().addFilter(RequestIdFilter())
```

Trong web framework:

```python
from fastapi import FastAPI, Request
import uuid

app = FastAPI()

@app.middleware("http")
async def gan_request_id(request: Request, call_next):
    rid = request.headers.get("X-Request-ID") or str(uuid.uuid4())
    token = request_id.set(rid)
    try:
        response = await call_next(request)
    finally:
        request_id.reset(token)
    response.headers["X-Request-ID"] = rid
    return response
```

Mọi log trong request mang cùng `request_id`. Khi user báo lỗi, tra theo ID.

## 22.4 Metrics với `prometheus_client`

```bash
pip install prometheus-client
```

```python
from prometheus_client import Counter, Histogram, Gauge, start_http_server

REQUESTS = Counter(
    "http_requests_total",
    "Tổng số request",
    ["method", "endpoint", "status"],
)

LATENCY = Histogram(
    "http_request_duration_seconds",
    "Thời gian xử lý",
    ["endpoint"],
    buckets=(0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10),
)

IN_FLIGHT = Gauge("http_in_flight", "Request đang xử lý")

start_http_server(8000)   # /metrics
```

### Ghi metric

```python
import time

def xu_ly(endpoint: str) -> None:
    IN_FLIGHT.inc()
    t0 = time.perf_counter()
    try:
        ...
        REQUESTS.labels("GET", endpoint, "200").inc()
    finally:
        LATENCY.labels(endpoint).observe(time.perf_counter() - t0)
        IN_FLIGHT.dec()
```

### Bốn loại metric

| Loại | Dùng cho | Ví dụ |
|------|----------|-------|
| Counter | chỉ tăng | số request, số lỗi |
| Gauge | lên xuống | connection đang mở, RAM |
| Histogram | phân phối | latency, kích thước |
| Summary | phân phối (client-side) | hiếm dùng hơn histogram |

### PromQL cơ bản

```promql
rate(http_requests_total[5m])
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
sum by (status) (rate(http_requests_total[5m]))
```

### Không dùng nhãn có cardinality cao

```python
# SAI — user_id tạo hàng triệu series
REQUESTS.labels(user_id=user_id).inc()

# ĐÚNG — nhãn hữu hạn
REQUESTS.labels(endpoint=endpoint, status=status).inc()
```

## 22.5 OpenTelemetry — chuẩn mở

```bash
pip install opentelemetry-api opentelemetry-sdk \
            opentelemetry-exporter-otlp \
            opentelemetry-instrumentation-fastapi
```

### Cấu hình

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource

resource = Resource.create({"service.name": "my-service", "service.version": "1.0.0"})
provider = TracerProvider(resource=resource)
provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter(endpoint="http://localhost:4317")))
trace.set_tracer_provider(provider)

tracer = trace.get_tracer(__name__)
```

### Span thủ công

```python
def xu_ly_don(don_id: int) -> None:
    with tracer.start_as_current_span("xu_ly_don") as span:
        span.set_attribute("don.id", don_id)
        try:
            thanh_toan(don_id)
        except Exception as e:
            span.record_exception(e)
            span.set_status(trace.Status(trace.StatusCode.ERROR, str(e)))
            raise
```

### Tự động instrument FastAPI

```python
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor

app = FastAPI()
FastAPIInstrumentor.instrument_app(app)
```

Tự tạo span cho mỗi request, gắn trace context vào log.

### Log có trace_id

```python
from opentelemetry import trace

def format(self, record):
    span = trace.get_current_span()
    ctx = span.get_span_context()
    record.trace_id = format(ctx.trace_id, "032x") if ctx.is_valid else "-"
    return super().format(record)
```

Giờ log liên kết với trace trong Jaeger/Tempo/Grafana.

## 22.6 Health check và readiness

```python
from fastapi import FastAPI, Response, status

app = FastAPI()

@app.get("/healthz")
def healthz() -> dict[str, str]:
    return {"status": "ok"}

@app.get("/readyz")
def readyz(response: Response) -> dict[str, object]:
    kiem_tra = {
        "db": kiem_tra_db(),
        "cache": kiem_tra_cache(),
    }
    if not all(kiem_tra.values()):
        response.status_code = status.HTTP_503_SERVICE_UNAVAILABLE
    return {"ready": all(kiem_tra.values()), "checks": kiem_tra}
```

- `/healthz`: process còn sống (liveness).
- `/readyz`: sẵn sàng nhận traffic (readiness) — kiểm tra dependency.

Kubernetes dùng cả hai. Sai lầm phổ biến: readiness gọi dependency nặng → restart loop.

## 22.7 Đo thời gian đúng cách

```python
import time

# Đo khoảng thời gian
t0 = time.perf_counter()
...
print(time.perf_counter() - t0)

# Thời gian wall-clock (cho timestamp)
from datetime import datetime, timezone
print(datetime.now(timezone.utc).isoformat())
```

- `perf_counter()`: đơn điệu, độ phân giải cao — đo khoảng.
- `monotonic()`: đơn điệu — timeout.
- `time()`: có thể nhảy khi NTP chỉnh — không dùng đo khoảng.
- `datetime.now(timezone.utc)`: timestamp có múi giờ.

Luôn dùng UTC trong log và database.

## 22.8 Cảnh báo có ý nghĩa

Cảnh báo dựa trên triệu chứng, không dựa trên nguyên nhân.

| Tốt | Tệ |
|-----|-----|
| `p99 latency > 1s trong 5 phút` | `CPU > 80%` |
| `tỉ lệ lỗi 5xx > 1%` | `RAM > 90%` |
| `queue backlog > 10000` | `số thread > 100` |

Mỗi cảnh báo phải có: runbook, ngưỡng rõ, người chịu trách nhiệm, cách tắt.

## 22.9 Ví dụ — middleware đầy đủ

```python
import time
import uuid
import logging
from fastapi import FastAPI, Request
from prometheus_client import Counter, Histogram

logger = logging.getLogger("http")
REQUESTS = Counter("http_requests_total", "requests", ["method", "path", "status"])
LATENCY = Histogram("http_request_duration_seconds", "latency", ["path"])

app = FastAPI()

@app.middleware("http")
async def observe(request: Request, call_next):
    rid = request.headers.get("X-Request-ID") or str(uuid.uuid4())
    token = request_id.set(rid)
    t0 = time.perf_counter()
    status = "500"
    try:
        response = await call_next(request)
        status = str(response.status_code)
        response.headers["X-Request-ID"] = rid
        return response
    finally:
        du = time.perf_counter() - t0
        LATENCY.labels(request.url.path).observe(du)
        REQUESTS.labels(request.method, request.url.path, status).inc()
        logger.info(
            "request",
            extra={"extra_fields": {
                "method": request.method,
                "path": request.url.path,
                "status": status,
                "duration_ms": round(du * 1000, 2),
            }},
        )
        request_id.reset(token)
```

## 22.10 Bài tập

1. Viết JsonFormatter, log ra file, đọc lại và parse từng dòng.
2. Thêm RedactingFilter, kiểm tra không lộ token.
3. Thêm middleware gắn `X-Request-ID`, log kèm ID.
4. Expose `/metrics` với Prometheus, viết PromQL tính p95.
5. Cấu hình OpenTelemetry export sang Jaeger local, xem trace một request.
6. Viết `/healthz` và `/readyz` kiểm tra DB thật.
7. Viết cảnh báo Prometheus cho tỉ lệ lỗi và latency.

## 22.11 Ghi chú kỹ thuật

- Ghi log ra stdout/stderr; để hệ thống ngoài thu thập (12-factor).
- Không tự rotate log trong ứng dụng container.
- Histogram bucket phải phù hợp dải latency thực tế.
- Nhãn Prometheus: cardinality thấp, tên ổn định.
- Trace sampling: 1-10% ở production, 100% ở staging.
- Log level nên điều chỉnh được runtime, không cần restart.
- Đo trước, tối ưu sau — observability phục vụ quyết định.

Stashed.