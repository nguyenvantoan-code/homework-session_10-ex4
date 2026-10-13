# Bài 4: Custom Network và Named Volume bằng Docker CLI

## 1. Mục tiêu

- Tạo Custom Bridge Network tên `app-net`.
- Tạo Named Volume tên `db-data`.
- Kết nối Backend và PostgreSQL qua Docker Network.
- Kiểm tra khả năng phân giải DNS bằng hostname `db-server`.
- Xác minh dữ liệu không bị mất khi xóa và tạo lại Database Container.

## 2. Kiến trúc hệ thống

| Thành phần | Tên | Chức năng |
|---|---|---|
| Network | `app-net` | Kết nối nội bộ giữa các container |
| Volume | `db-data` | Lưu trữ dữ liệu PostgreSQL |
| Database | `db-server` | Chạy PostgreSQL 15 |
| Backend | `app-container` | Chạy REST API từ Bài 2 |

## 3. Triển khai

Tạo Network:

```bash
docker network create --driver bridge app-net
```

Tạo Volume:

```bash
docker volume create db-data
```

Chạy PostgreSQL với Volume:

```bash
docker run -d \
  --name db-server \
  --network app-net \
  -e POSTGRES_DB=appdb \
  -e POSTGRES_USER=appuser \
  -e POSTGRES_PASSWORD=apppass \
  -v db-data:/var/lib/postgresql/data \
  postgres:15-alpine
```

Chạy Backend trên cùng Network và cấu hình biến kết nối:

```bash
docker run -d \
  --name app-container \
  --network app-net \
  -p 8080:3000 \
  -e DB_HOST=db-server \
  -e DB_PORT=5432 \
  -e DB_NAME=appdb \
  -e DB_USER=appuser \
  -e DB_PASSWORD=apppass \
  hw-backend:v1
```

## 4. Kiểm tra Network và Volume

```bash
docker network ls
docker volume ls
docker network inspect app-net
docker exec app-container ping -c 2 db-server
```

Kết quả mong đợi: Backend phân giải được hostname `db-server` thành IP nội bộ của container Database.

## 5. Kiểm tra Data Persistence

Tạo bảng và ghi dữ liệu:

```bash
docker exec db-server psql -U appuser -d appdb -c \
"CREATE TABLE persistence_test (
  id SERIAL PRIMARY KEY,
  message VARCHAR(100)
);"

docker exec db-server psql -U appuser -d appdb -c \
"INSERT INTO persistence_test (message)
 VALUES ('Docker Volume Persistence');"
```

Xóa Database Container:

```bash
docker rm -f db-server
```

Tạo lại container `db-server` với cùng Volume `db-data` và cùng thông tin Database. Chờ PostgreSQL sẵn sàng, sau đó truy vấn:

```bash
docker exec db-server psql -U appuser -d appdb -c \
"SELECT * FROM persistence_test;"
```

Kết quả mong đợi: Bản ghi `Docker Volume Persistence` vẫn tồn tại sau khi container được tạo lại.

## 6. Kết luận

Custom Bridge Network cho phép các container giao tiếp với nhau thông qua hostname mà không cần cố định IP. Named Volume lưu dữ liệu bên ngoài vòng đời của container, giúp dữ liệu PostgreSQL vẫn tồn tại khi container bị xóa và tạo lại, miễn là Volume không bị xóa và được gắn lại đúng cách.

Trong môi trường thực tế, cần quản lý mật khẩu bằng secrets hoặc biến môi trường an toàn thay vì ghi trực tiếp thông tin đăng nhập trong script.
