# Kỹ Thuật và Pattern trong Identity Service

Tài liệu này ghi lại các công nghệ, kiến trúc và mẫu thiết kế (patterns) được áp dụng trong **Identity Service** để đảm bảo tính mở rộng, bảo mật và khả năng bảo trì.

## 1. Kiến trúc Tổng thể (Architecture)
*   **Hexagonal Architecture (Ports and Adapters)**: Tách biệt logic nghiệp vụ khỏi các chi tiết hạ tầng (Database, gRPC, HTTP).
*   **Domain-Driven Design (DDD)**: Tập trung vào việc mô hình hóa nghiệp vụ cốt lõi.
    *   **Aggregate Root**: `UserAccount` là trung tâm, quản lý các quy tắc nghiệp vụ (invariants).
    *   **Value Objects**: `Email`, `Role`, `AccountStatus` đảm bảo tính toàn vẹn dữ liệu ngay từ cấp độ nguyên tử.
    *   **Domain Events**: Phát đi các sự kiện như `UserRegistered`, `AccountLocked` để các service khác phản ứng theo.
    *   **Domain Services**: Xử lý các logic phức tạp liên quan đến nhiều aggregate hoặc cấu hình hệ thống (ví dụ: `AccountLockPolicyService`).

## 2. Công nghệ Cốt lõi (Tech Stack)
*   **Language**: Go (Golang) - Hiệu năng cao, hỗ trợ concurrency tốt.
*   **Web Framework**: **Gin Gonic** - Sử dụng cho các API REST (Query và OAuth flow).
*   **RPC Framework**: **gRPC** - Sử dụng cho giao tiếp nội bộ giữa các microservice (Command).
*   **ORM**: **GORM** - Tương tác với cơ sở dữ liệu PostgreSQL.
*   **Database**: **PostgreSQL** - Lưu trữ dữ liệu quan hệ bền vững.
*   **Messaging**: **Kafka** - Truyền tin không đồng bộ (Asynchronous messaging).
*   **Migration**: **golang-migrate** - Quản lý phiên bản cấu trúc database.

## 3. Các Pattern Quan trọng
*   **Transactional Outbox Pattern**: Đảm bảo tính nhất quán dữ liệu (Atomicity) khi vừa lưu database vừa gửi Event. Event được lưu vào bảng `outbox` trong cùng một transaction.
*   **Refresh Token Rotation**: Mỗi khi sử dụng Refresh Token để lấy Access Token mới, Refresh Token cũ sẽ bị thu hồi và thay thế bằng một token mới.
*   **Reuse Detection**: Cơ chế phát hiện và ngăn chặn tấn công nếu một Refresh Token cũ đã bị sử dụng lại.
*   **RSA Key Rotation & JWKS**: Sử dụng cặp khóa RSA (Public/Private) để ký JWT. Public key được public qua endpoint JWKS để các service khác (hoặc Istio) verify mà không cần gọi trực tiếp tới Identity Service.
*   **PKCE (Proof Key for Code Exchange)**: Tăng cường bảo mật cho luồng Google OAuth2, chống lại các cuộc tấn công code injection.
*   **SAGA Pattern (Choreography)**: Phối hợp các giao dịch phân tán thông qua Event (ví dụ: luồng nâng cấp Companion).
*   **Auth Offloading (Istio)**: Trách nhiệm verify JWT được đẩy ra tầng Service Mesh (Istio Waypoint), code ứng dụng chỉ tập trung vào logic nghiệp vụ.

## 4. Chiến lược Testing
*   **Unit Testing**: Test logic domain và invariant với độ bao phủ cao.
*   **Fuzz Testing**: Sử dụng công cụ fuzzing của Go để tìm kiếm các lỗ hổng xử lý dữ liệu đầu vào không lường trước được (đặc biệt cho Value Objects).
*   **Infrastructure Testing**: Sử dụng **SQLite (CGO-free)** chạy in-memory để test các adapter database mà không cần setup Postgres thật.
*   **Contract Testing**: Kiểm tra tính hợp lệ của OpenAPI và các Event contract.
*   **E2E Testing**: Kiểm tra luồng chạy thực tế của Service qua endpoint Health Check và API.

## 5. Quy tắc Nghiệp vụ (Business Rules)
*   **Snapshot Policy**: Luôn lưu bản sao các thông số cấu hình (như chính sách khóa tài khoản) tại thời điểm giao dịch thay vì chỉ tham chiếu ID.
*   **Ubiquitous Language**: Sử dụng thống nhất các thuật ngữ từ BRD (Kano-Coin, Scenario, Companion, Client, Escrow).

## 6. Trạng thái Testing (Cập nhật: 10/05/2026)

Dưới đây là báo cáo chi tiết về các loại test đã được thực hiện trong Identity Service, các chức năng tương ứng và kết quả chạy thực tế.

### 6.1. Tổng quan kết quả kiểm thử

| Loại Test | Chức năng kiểm thử | Lệnh chạy | Kết quả (Pass/Total) |
| :--- | :--- | :--- | :--- |
| **Unit Test** | Logic nghiệp vụ (Domain), Invariants, Value Objects, Application Commands (Login, Lock, Violation), Infrastructure logic (JWT Lifecycle). | `make test-unit` | 54/54 |
| **Fuzz Test** | Kiểm tra độ bền (robustness) của Value Objects và JWT decoding với dữ liệu đầu vào ngẫu nhiên. | `make test-fuzz` | 6/6 |
| **Integration Test** | Kiểm tra tương tác thực tế với PostgreSQL (Adapter), Kafka (Outbox), Redis (Cache). Đã fix lỗi cú pháp SQL backticks. | `make test-integration` | 5/5 |
| **Contract Test** | Kiểm tra tính nhất quán Route-Spec và xác thực Response Schema (dùng `kin-openapi`). Đã fix lỗi schema mismatch và API paths. | `make test-contract` | 2/2 |
| **E2E Test** | Kiểm tra luồng chạy thực tế qua HTTP endpoints: Health Check, OAuth Login, Admin Lock, Companion Upgrade. | `go test ./tests/e2e/...` | 4/4 |

### 6.2. Đồng bộ hóa gRPC (Mới)
*   **Auth Endpoints**: Đã bổ sung đầy đủ các phương thức xác thực vào gRPC interface (`identity.proto`) bao gồm: `InitGoogleAuth`, `LoginGoogle`, `RefreshToken`, `Logout`.
*   **Dependency Injection**: Cập nhật `IdentityGRPCHandler` để sử dụng chung logic Application Command Handlers với HTTP layer, đảm bảo tính nhất quán.

### 6.3. Các chức năng chính đã được bao phủ (Test Coverage)

1.  **Quản lý Tài khoản (UserAccount Aggregate)**:
    *   Khởi tạo tài khoản với Role mặc định (`CLIENT`).
    *   Logic khóa/mở khóa tài khoản tuân thủ quy tắc `[INV-ID01]`.
    *   Phê duyệt/Từ chối yêu cầu nâng cấp lên Companion.
2.  **Xác thực & Bảo mật**:
    *   Luồng Login Google (OAuth2 + PKCE) và tự động đăng ký.
    *   Vòng đời JWT: Cấp phát Access/Refresh token, cơ chế Rotation và Revocation.
    *   Validation dữ liệu đầu vào nghiêm ngặt (Email, Role, Status) qua Fuzzing.
3.  **Hệ thống & Tin nhắn (Infrastructure)**:
    *   **Transactional Outbox**: Đảm bảo Event và Dữ liệu được lưu nguyên tử (Atomicity).
    *   **Outbox Worker**: Đẩy tin nhắn lên Kafka và đánh dấu trạng thái đã xuất bản.
    *   **Database Migration**: Kiểm tra tính đúng đắn của schema PostgreSQL.
4.  **Logic Nghiệp vụ (Application)**:
    *   Ghi nhận vi phạm (Violation) và tự động khóa tài khoản khi vượt ngưỡng.
    *   Quản lý cấu hình hệ thống (System Config) từ database.

### 6.4. Hướng dẫn chạy Test
*   **Chạy toàn bộ test nhanh (Unit + Contract)**: `make test-all`
*   **Chạy Integration Test**: Yêu cầu Docker chạy các dependency (DB, Kafka, Redis). Sau đó chạy `make test-integration`.
*   **Chạy E2E Test**:
    1.  `make docker-up` (Khởi chạy hạ tầng).
    2.  `$env:ENABLE_TEST_ROUTES="true"; go run cmd/server/main.go` (Khởi chạy server với route mock).
    3.  Mở terminal mới và chạy: `go test ./tests/e2e/... -v -count=1`.
