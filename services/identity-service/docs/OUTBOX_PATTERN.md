# Cơ chế Transactional Outbox - Identity Service

Tài liệu này giải thích cách Identity Service đảm bảo tính nhất quán dữ liệu giữa Database và Kafka bằng cách sử dụng **Transactional Outbox Pattern**.

## 1. Cấu trúc Database (Table `outbox_events`)

Bảng này đóng vai trò là "hàng đợi tạm thời" trong Database. Mọi Event được phát ra phải được lưu vào bảng này trong cùng một Transaction với dữ liệu nghiệp vụ.

| Cột | Kiểu dữ liệu | Mô tả |
| :--- | :--- | :--- |
| `id` | `UUID` (Primary Key) | Định danh duy nhất cho mỗi event. |
| `event_type` | `VARCHAR(255)` | Loại sự kiện (ví dụ: `com.rentagf.identity.AccountLocked.v1`). |
| `payload` | `JSONB` | Nội dung chi tiết của sự kiện dưới dạng JSON. |
| `published` | `BOOLEAN` | Trạng thái gửi: `false` (chưa gửi), `true` (đã gửi lên Kafka). |
| `created_at` | `TIMESTAMP` | Thời điểm sự kiện được tạo ra. |

---

## 2. Quy trình lưu sự kiện (Publishing Flow)

### Bước 1: Lưu nguyên tử (Atomicity)
Khi một hành động nghiệp vụ xảy ra (ví dụ: Admin khóa tài khoản), hệ thống thực hiện các bước sau **trong cùng một Database Transaction**:
1. Cập nhật trạng thái tài khoản trong bảng `user_accounts`.
2. Chuyển đổi Domain Event thành JSON.
3. Chèn một dòng mới vào bảng `outbox_events` với `published = false`.

> [!NOTE]
> Nếu một trong hai bước trên thất bại, toàn bộ Transaction sẽ bị Rollback. Điều này đảm bảo không bao giờ có chuyện "dữ liệu đã đổi nhưng không có event" hoặc ngược lại.

### Bước 2: Quét và Gửi (Relay/Worker)
Một tiến trình chạy ngầm (**Outbox Worker**) sẽ thực hiện:
1. Quét bảng `outbox_events` để tìm các bản ghi có `published = false`.
2. Đóng gói (Wrap) payload vào định dạng chuẩn **CloudEvent v1.0**.
3. Gửi tin nhắn lên Kafka topic `identity-events`.
4. Nếu gửi thành công, cập nhật bản ghi đó thành `published = true`.

---

## 3. Định dạng CloudEvent trên Kafka

Khi sự kiện được gửi lên Kafka, nó được bọc trong một lớp Metadata chuẩn để các service khác dễ dàng xử lý:

```json
{
  "specversion": "1.0",
  "id": "uuid-tu-outbox",
  "source": "/rent-a-gf/identity-service",
  "type": "com.rentagf.identity.AccountLocked.v1",
  "datacontenttype": "application/json",
  "time": "2026-05-11T04:17:00Z",
  "data": {
    "userId": "...",
    "reason": "...",
    "lockedBy": "..."
  }
}
```

## 4. Ưu điểm của cơ chế này
1. **No Data Loss**: Đảm bảo mọi thay đổi dữ liệu quan trọng đều có sự kiện tương ứng được gửi đi.
2. **Eventual Consistency**: Các service khác (Booking, Notification) sẽ nhận được thông tin để cập nhật trạng thái theo, dù Kafka có thể bị chậm trễ đôi chút.
3. **Resilience**: Nếu Kafka bị sập, Worker sẽ thử lại liên tục cho đến khi gửi được thì thôi.
