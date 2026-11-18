# BÁO CÁO PHÂN TÍCH NGUYÊN NHÂN GỐC RỄ (ROOT CAUSE ANALYSIS)
## SỰ CỐ SUY GIẢM HIỆU NĂNG VÀ MẤT CACHE TRONG DOCKER MULTI-STAGE TRÊN RUNNER CI/CD

**Người thực hiện:** Kỹ sư DevOps Junior - Dự án QuickBite  
**Đối tượng phân tích:** Quy trình đóng gói container cho `order-service`  
**Hiện tượng:** Thời gian thực thi pipeline CI/CD tăng từ ~1.5 phút lên 4 - 5 phút sau khi chuyển sang Dockerfile Multi-stage.

---

### 1. Phân tích nguyên nhân cốt lõi (Root Cause)
Khi tiến hành phân tích sự cố, chúng tôi xác định nguyên nhân bắt nguồn từ **Tính cô lập của không gian thực thi trong Container Builder (Container Isolation)**:

1. **Khác biệt về môi trường thực thi:**
   - Trong pipeline cũ (chạy lệnh `./gradlew bootJar` trực tiếp trên Self-hosted Runner host), tiến trình Gradle chạy dưới quyền user của máy chủ và tái sử dụng toàn bộ thư mục dependencies đã được tải về sẵn tại đường dẫn vật lý `~/.gradle/caches`. Nhờ đó, các lần build tiếp theo chỉ mất 30 - 45 giây để kiểm tra tính toàn vẹn (Cache Hit 100%).
   - Trong pipeline Multi-stage mới (Stage 1: `FROM eclipse-temurin:17-jdk-alpine AS builder`), lệnh `./gradlew bootJar` được thực thi **bên trong một container tạm thời hoàn toàn độc lập**.

2. **Giới hạn của mô hình Docker-outside-of-Docker (DooD):**
   - Cấu hình chia sẻ socket vật lý `/var/run/docker.sock` chỉ cung cấp quyền điều khiển Docker Daemon của máy host, chứ **hoàn toàn không tự động mount hệ thống tệp (filesystem)** của máy host vào bên trong container builder.
   - Mỗi khi chạy `docker build`, Docker khởi tạo một container builder mới với filesystem rỗng. Do thư mục `/root/.gradle` bên trong container chưa hề có dữ liệu, Gradle bắt buộc phải kết nối ra Internet và tải lại hàng trăm MB thư viện phụ thuộc (Spring Boot, Hibernate, Jackson,...) từ Maven Central.

---

### 2. Giải pháp kỹ thuật khắc phục (Remediation)
Để khắc phục triệt để vấn đề mất cache mà vẫn giữ vững tính ưu việt của Multi-stage, chúng tôi đề xuất 2 giải pháp:

1. **Giải pháp 1: Khai thác Docker BuildKit Cache Mount (Khuyên dùng)**
   Kích hoạt Docker BuildKit và sử dụng cờ `--mount=type=cache`:
   ```dockerfile
   # syntax=docker/dockerfile:1.4
   FROM eclipse-temurin:17-jdk-alpine AS builder
   WORKDIR /app
   COPY . .
   RUN --mount=type=cache,target=/root/.gradle ./gradlew bootJar --no-daemon
   ```
   Cơ chế này gắn thư mục cache của Docker Engine trực tiếp vào `/root/.gradle` xuyên suốt các lần build, giúp thời gian build giảm trở lại ~1 phút 10 giây.

2. **Giải pháp 2: Tách biệt Layer Dependency Cache trong Dockerfile**
   Copy riêng các file khai báo phụ thuộc trước khi copy toàn bộ mã nguồn:
   ```dockerfile
   COPY gradlew settings.gradle build.gradle ./
   COPY gradle ./gradle
   RUN ./gradlew dependencies --no-daemon
   COPY src ./src
   RUN ./gradlew bootJar --no-daemon
   ```

---
**Kết luận:** Sự suy giảm hiệu năng là do container builder không kế thừa cache của host. Việc áp dụng BuildKit cache mount giải quyết dứt điểm vấn đề mà không làm giảm tính bảo mật của pipeline.
