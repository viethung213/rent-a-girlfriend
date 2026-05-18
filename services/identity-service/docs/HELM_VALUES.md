# Identity Service Helm Chart Values Documentation

Tài liệu này mô tả các giá trị cấu hình (values) được sử dụng trong Helm Chart của Identity Service. Các giá trị này sẽ được Helm inject vào các file templates (Deployment, Service, Gateway, ConfigMap, Secret,...) khi thực hiện `helm install` hoặc `upgrade`.

## 1. Core Deployment (Cấu hình lõi)

| Key | Kiểu | Mặc định | Mô tả |
| :--- | :--- | :--- | :--- |
| `replicaCount` | int | `3` | Số lượng pods chạy song song. |
| `image.repository` | string | `your-registry/identity-service` | Đường dẫn image trên container registry. |
| `image.tag` | string | `latest` | Tag của image (thường là commit SHA trong CI/CD). |
| `image.pullPolicy` | string | `IfNotPresent` | Chính sách kéo image. |

## 2. Networking (Mạng & Routing)

| Key | Kiểu | Mặc định | Mô tả |
| :--- | :--- | :--- | :--- |
| `service.type` | string | `ClusterIP` | Loại service trong K8s. |
| `service.ports.http` | int | `8080` | Port cho HTTP API. |
| `service.ports.grpc` | int | `50051` | Port cho gRPC communication. |
| `gateway.enabled` | bool | `true` | Có tạo Istio Gateway không. |
| `gateway.hostname` | string | `identity.example.com` | Hostname chính của service. |
| `httpRoute.paths` | list | `[/api/v1, ...]` | Các đường dẫn (prefix) được expose qua Gateway. |

## 3. Application Configuration (ConfigMap)
*Được inject vào Pod dưới dạng Biến môi trường (Environment Variables)*

| Key | Kiểu | Mô tả |
| :--- | :--- | :--- |
| `config.DB_HOST` | string | Hostname của PostgreSQL (Neon DB). |
| `config.DB_NAME` | string | Tên cơ sở dữ liệu. |
| `config.DB_USER` | string | Username kết nối DB. |
| `config.KAFKA_BROKERS` | string | Danh sách brokers Kafka (phân cách bằng dấu phẩy). |
| `config.JWT_ISSUER` | string | Định danh người phát hành token JWT. |
| `config.GIN_MODE` | string | Chế độ chạy của Gin Framework (`release` hoặc `debug`). |

## 4. Sensitive Information (Secrets)
*Các giá trị bảo mật, cần ghi đè bằng `--set` hoặc dùng CI/CD secrets*

| Key | Kiểu | Ghi chú |
| :--- | :--- | :--- |
| `secrets.DB_PASSWORD` | string | Mật khẩu truy cập DB. |
| `secrets.GOOGLE_CLIENT_ID` | string | ID ứng dụng Google OAuth. |
| `secrets.GOOGLE_CLIENT_SECRET` | string | Secret key của Google OAuth. |
| `secrets.REDIS_URL` | string | URL kết nối Redis (bao gồm cả password). |

## 5. Scaling & Resources (Tài nguyên & Tự động giãn nở)

| Key | Kiểu | Mặc định | Mô tả |
| :--- | :--- | :--- | :--- |
| `resources.requests.cpu` | string | `250m` | CPU tối thiểu cam kết. |
| `resources.limits.memory` | string | `1Gi` | RAM tối đa được phép sử dụng. |
| `hpa.enabled` | bool | `true` | Kích hoạt Horizontal Pod Autoscaler. |
| `hpa.maxReplicas` | int | `5` | Số lượng pods tối đa khi scale up. |

## 6. Security & Service Mesh (Istio)

| Key | Kiểu | Mặc định | Mô tả |
| :--- | :--- | :--- | :--- |
| `istio.peerAuthentication.mtls.mode` | string | `STRICT` | Ép buộc sử dụng mTLS trong mesh. |
| `podSecurityContext.runAsNonRoot` | bool | `true` | Không cho phép chạy pod bằng quyền root. |
| `istio.authorizationPolicy.allowPaths` | list | `[...]` | Danh sách các API public không cần verify JWT (đã offload cho Gateway). |

---

### Cách ghi đè giá trị khi deploy:
Sử dụng cờ `--set` hoặc file `values-env.yaml`:

```bash
helm upgrade --install identity-service ./helm/identity-service \
  --namespace rent-a-girlfriend \
  --set secrets.DB_PASSWORD=$PROD_DB_PASSWORD \
  --set image.tag=v1.2.3
