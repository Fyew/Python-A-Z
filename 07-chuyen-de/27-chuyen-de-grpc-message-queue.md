# Chuyên Đề 27 — gRPC và Message Queue

## 27.1 Vì sao không chỉ REST

REST + JSON tốt cho API công khai, dễ debug, hệ sinh thái rộng. Nhưng:

- JSON text tốn băng thông và CPU.
- Không có schema cứng — dễ lệch giữa client/server.
- HTTP/1.1 không hỗ trợ streaming hai chiều tốt.
- Không có code generation mặc định.

gRPC + Protobuf giải quyết: nhị phân, schema cứng, HTTP/2, code gen.

## 27.2 Protobuf

`user.proto`:

```protobuf
syntax = "proto3";

package demo;

message User {
  int64 id = 1;
  string ten = 2;
  string email = 3;
  repeated string vai_tro = 4;
}

message GetUserRequest {
  int64 id = 1;
}

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc ListUsers(ListUsersRequest) returns (stream User);
}
```

Cài:

```bash
pip install grpcio grpcio-tools
python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. user.proto
```

Sinh `user_pb2.py` và `user_pb2_grpc.py`.

## 27.3 Server gRPC

```python
from concurrent import futures
import grpc
import user_pb2
import user_pb2_grpc

class UserServicer(user_pb2_grpc.UserServiceServicer):
    def GetUser(self, request, context):
        u = db.lay_user(request.id)
        if not u:
            context.abort(grpc.StatusCode.NOT_FOUND, "Không tìm thấy")
        return user_pb2.User(
            id=u["id"], ten=u["ten"], email=u["email"], vai_tro=u["vai_tro"]
        )

    def ListUsers(self, request, context):
        for u in db.tat_ca_user():
            yield user_pb2.User(id=u["id"], ten=u["ten"])

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    user_pb2_grpc.add_UserServiceServicer_to_server(UserServicer(), server)
    server.add_insecure_port("[::]:50051")
    server.start()
    server.wait_for_termination()

if __name__ == "__main__":
    serve()
```

## 27.4 Client gRPC

```python
import grpc
import user_pb2
import user_pb2_grpc

def main():
    with grpc.insecure_channel("localhost:50051") as channel:
        stub = user_pb2_grpc.UserServiceStub(channel)
        user = stub.GetUser(user_pb2.GetUserRequest(id=1))
        print(user.ten)

        for u in stub.ListUsers(user_pb2.ListUsersRequest()):
            print(u.id, u.ten)
```

## 27.5 Bốn kiểu RPC

| Kiểu | Mô tả |
|------|-------|
| Unary | 1 request, 1 response |
| Server streaming | 1 request, nhiều response |
| Client streaming | nhiều request, 1 response |
| Bidirectional | nhiều-nhiều |

```protobuf
service Chat {
  rpc SendMessage(Message) returns (Ack);
  rpc Subscribe(SubscribeRequest) returns (stream Message);
  rpc UploadFile(stream Chunk) returns (UploadResult);
  rpc Chat(stream Message) returns (stream Message);
}
```

## 27.6 Interceptor — middleware

```python
import grpc
import logging

class LoggingInterceptor(grpc.ServerInterceptor):
    def intercept_service(self, continuation, handler_call_details):
        logging.info("gọi %s", handler_call_details.method)
        return continuation(handler_call_details)

server = grpc.server(
    futures.ThreadPoolExecutor(max_workers=10),
    interceptors=[LoggingInterceptor()],
)
```

### Deadline và timeout

```python
try:
    user = stub.GetUser(req, timeout=5)
except grpc.RpcError as e:
    print(e.code(), e.details())
```

Luôn đặt deadline. Không có deadline = treo vô hạn.

### Retry

```python
config = {
    "methodConfig": [{
        "name": [{"service": "demo.UserService"}],
        "retryPolicy": {
            "maxAttempts": 4,
            "initialBackoff": "0.1s",
            "maxBackoff": "1s",
            "backoffMultiplier": 2,
            "retryableStatusCodes": ["UNAVAILABLE"],
        },
    }],
}
channel = grpc.insecure_channel("localhost:50051", options=[("grpc.service_config", json.dumps(config))])
```

## 27.7 TLS

```python
creds = grpc.ssl_channel_credentials(
    root_certificates=open("ca.pem", "rb").read()
)
channel = grpc.secure_channel("api.example.com:443", creds)
```

Server:

```python
server_creds = grpc.ssl_server_credentials([(key, cert)])
server.add_secure_port("[::]:50051", server_creds)
```

## 27.8 So sánh gRPC và REST

| Khía cạnh | REST/JSON | gRPC/Protobuf |
|-----------|-----------|---------------|
| Định dạng | Text | Nhị phân |
| Schema | OpenAPI (tùy chọn) | Bắt buộc |
| HTTP | 1.1 hoặc 2 | 2 |
| Streaming | Hạn chế | Đầy đủ |
| Code gen | Tùy | Bắt buộc |
| Debug | curl dễ | Cần grpcurl |
| Trình duyệt | Hỗ trợ trực tiếp | Cần grpc-web |
| Hiệu năng | Khá | Cao |

Kết luận: gRPC cho microservice nội bộ; REST cho API công khai.

## 27.9 Message queue — vì sao

- Tách producer khỏi consumer (decoupling).
- Xử lý bất đồng bộ — request trả về nhanh.
- Chịu tải đỉnh — queue đệm.
- Nhiều consumer cùng đọc (scale).
- Retry tự động khi lỗi.

## 27.10 RabbitMQ

```bash
pip install pika
```

### Producer

```python
import pika
import json

conn = pika.BlockingConnection(pika.ConnectionParameters("localhost"))
ch = conn.channel()

ch.queue_declare(queue="tasks", durable=True)

ch.basic_publish(
    exchange="",
    routing_key="tasks",
    body=json.dumps({"loai": "email", "dia_chi": "a@x.com"}),
    properties=pika.BasicProperties(delivery_mode=2),   # persistent
)
conn.close()
```

### Consumer

```python
import pika
import json
import time

conn = pika.BlockingConnection(pika.ConnectionParameters("localhost"))
ch = conn.channel()
ch.queue_declare(queue="tasks", durable=True)
ch.basic_qos(prefetch_count=1)   # một task một lúc

def callback(ch, method, properties, body):
    du_lieu = json.loads(body)
    try:
        xu_ly(du_lieu)
        ch.basic_ack(delivery_tag=method.delivery_tag)
    except Exception:
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=False)

ch.basic_consume(queue="tasks", on_message_callback=callback)
ch.start_consuming()
```

### Exchange types

- `direct`: routing key khớp chính xác.
- `topic`: routing key theo pattern (`user.*.created`).
- `fanout`: gửi tất cả queue.
- `headers`: dựa vào header.

```python
ch.exchange_declare(exchange="logs", exchange_type="topic")
ch.queue_bind(exchange="logs", queue="error_queue", routing_key="*.error")
```

### Dead letter queue

```python
ch.queue_declare(
    queue="tasks",
    durable=True,
    arguments={
        "x-dead-letter-exchange": "dlx",
        "x-message-ttl": 60000,
    },
)
```

Message hết TTL hoặc bị nack không requeue sẽ vào DLQ để điều tra.

## 27.11 Redis Streams

Nhẹ hơn RabbitMQ, dùng Redis có sẵn.

```python
import redis

r = redis.Redis(decode_responses=True)

# Producer
r.xadd("events", {"loai": "order", "id": "123"})

# Consumer group
try:
    r.xgroup_create("events", "workers", id="0", mkstream=True)
except redis.ResponseError:
    pass

# Đọc
messages = r.xreadgroup(
    "workers", "worker-1",
    {"events": ">"},
    count=10,
    block=5000,
)
for stream, items in messages or []:
    for msg_id, fields in items:
        xu_ly(fields)
        r.xack("events", "workers", msg_id)
```

Redis Streams có consumer group, ack, pending — đủ cho nhiều trường hợp.

## 27.12 Kafka

Cho throughput cao, log bất biến, replay.

```bash
pip install confluent-kafka
```

```python
from confluent_kafka import Producer, Consumer

producer = Producer({"bootstrap.servers": "localhost:9092"})
producer.produce("events", key="user-1", value='{"loai":"login"}')
producer.flush()

consumer = Consumer({
    "bootstrap.servers": "localhost:9092",
    "group.id": "workers",
    "auto.offset.reset": "earliest",
})
consumer.subscribe(["events"])

while True:
    msg = consumer.poll(1.0)
    if msg is None:
        continue
    if msg.error():
        continue
    print(msg.key(), msg.value())
    consumer.commit()
```

Kafka khác queue truyền thống: message không bị xóa sau khi đọc, offset do consumer quản lý. Cho phép replay.

## 27.13 Chọn công cụ

| Nhu cầu | Công cụ |
|---------|---------|
| Task queue đơn giản | Celery + Redis |
| Routing phức tạp, RPC | RabbitMQ |
| Đã có Redis | Redis Streams |
| Throughput cao, replay | Kafka |
| Async, nhẹ | Arq / Dramatiq |
| Pub/sub đơn giản | Redis Pub/Sub |

Redis Pub/Sub không lưu message — consumer offline là mất. Dùng Streams nếu cần bền.

## 27.14 Idempotency

Mọi consumer phải idempotent — message có thể được gửi lại.

```python
def xu_ly_don(don_id: str, du_lieu: dict) -> None:
    if da_xu_ly(don_id):
        return
    with transaction():
        ghi_don(don_id, du_lieu)
        danh_dau_da_xu_ly(don_id)
```

Dùng `don_id` làm khóa. Bảng `processed_messages` với unique constraint.

## 27.15 Outbox pattern

Đảm bảo ghi DB và gửi message cùng thành công.

```python
def tao_don(du_lieu: dict) -> None:
    with db.transaction():
        don_id = db.insert("orders", du_lieu)
        db.insert("outbox", {"event": "order.created", "payload": {"id": don_id}})
    # background worker đọc outbox, gửi message, xóa
```

Không gửi message trong transaction DB — có thể gửi mà DB rollback.

## 27.16 Bài tập

1. Viết service gRPC `Calculator` với unary và server streaming.
2. Viết interceptor gRPC đo thời gian mỗi RPC.
3. Viết producer/consumer RabbitMQ xử lý task giả.
4. Thêm dead letter queue, kiểm tra message lỗi vào DLQ.
5. Dùng Redis Streams làm task queue, hai consumer chia việc.
6. Viết Kafka producer/consumer đơn giản, test replay.
7. Cài đặt idempotency cho consumer dùng bảng `processed_messages`.
8. Viết outbox pattern cho một thao tác ghi DB + publish event.

## 27.17 Ghi chú kỹ thuật

- Protobuf nhỏ hơn JSON 3-10 lần; parse nhanh hơn nhiều.
- gRPC yêu cầu HTTP/2; load balancer phải hỗ trợ.
- RabbitMQ `prefetch_count` điều tiết tải cho consumer.
- Kafka partition key quyết định thứ tự — cùng key vào cùng partition.
- Redis Streams nhẹ nhưng không bằng Kafka về throughput.
- Mọi message queue cần monitoring: độ sâu queue, tỉ lệ lỗi, độ trễ.
- Luôn có DLQ — message lỗi không nên chặn queue.
- Idempotency không tùy chọn — mọi hệ thống phân tán đều gửi lại message.

Stashed.