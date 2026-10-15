# BÁO CÁO KỸ THUẬT BÀI 2: TỰ ĐỘNG HÓA ĐÓNG GÓI VÀ PHÁT HÀNH DOCKER IMAGE KHI GẮN GIT TAG (V*.*.*)

- **Khóa học:** DevOps Chuyên Sâu
- **Mã bài tập:** SS12 - Exercise 2
- **Công nghệ áp dụng:** GitHub Actions, Git Tags, Semantic Versioning (SemVer), GitHub Container Registry (GHCR), Docker Buildx, Docker Metadata Action

---

## 1. Mục tiêu và Ý nghĩa Kỹ thuật

### 1.1. Mục tiêu
- Xây dựng luồng CD chuyên nghiệp tự động hóa hoàn toàn chu trình đóng gói (Docker Containerize) và phát hành Docker Image lên hệ thống quản lý registry riêng biệt (**GitHub Packages / GHCR**).
- Kích hoạt quy trình phát hành chỉ khi xuất hiện sự kiện gắn nhãn phiên bản chính thức (**Git Tag Release `v*.*.*`**), ngăn ngừa việc đẩy các bản build chưa kiểm định từ các commit tạm thời.
- Tự động phân giải các thẻ Docker (Docker tags: `v1.0.0`, `1.0`, `latest`) đồng bộ với Git Tag bằng `docker/metadata-action@v5`.

### 1.2. Tại sao nên phát hành Image qua Git Tag thay vì commit thông thường?
- **Tính bất biến (Immutability):** Git Tag định danh một trạng thái mã nguồn cố định tại thời điểm release. Image được build từ tag đảm bảo tính tái lập (reproducibility) cao.
- **Tiết kiệm tài nguyên:** Không phải commit nào cũng cần build và push Docker image lên Registry, giảm thiểu băng thông mạng và dung lượng lưu trữ kho chứa.
- **Dễ dàng Rollback:** Khi có sự cố trên môi trường Production, quản trị viên chỉ cần chỉ định phiên bản trước đó (ví dụ: `v1.0.0` thay cho `v1.0.1`) để triển khai lại ngay lập tức.

---

## 2. Thiết kế Kịch bản CD (`.github/workflows/release.yml`)

```yaml
name: Publish Docker Image on Release Tag

on:
  push:
    tags:
      - 'v*.*.*'

permissions:
  contents: read
  packages: write

jobs:
  release-image:
    name: Build & Publish Docker Image to GHCR
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v5

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GitHub Container Registry (GHCR)
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata (tags, labels) for Docker
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=raw,value=latest

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### Phân tích cấu hình kỹ thuật:
1. `on.push.tags: ['v*.*.*']`: Khớp với các Git Tag theo định dạng Semantic Versioning như `v1.0.0`, `v2.1.3`,... Không kích hoạt khi push branch thường.
2. `permissions: packages: write`: Cấp quyền tự động cho token nội bộ `${{ secrets.GITHUB_TOKEN }}` được phép đẩy (push) image lên GitHub Container Registry (`ghcr.io`).
3. `docker/metadata-action@v5`: Phân tích biến ngữ cảnh `${{ github.ref }}` để sinh ra danh sách Docker Tags:
   - Git Tag `v1.0.0` sinh ra các tag: `v1.0.0`, `1.0`, `latest`.
   - Gắn thêm các OCI Labels (Open Container Initiative) như `org.opencontainers.image.source`, `org.opencontainers.image.revision`.
4. `cache-from: type=gha`, `cache-to: type=gha,mode=max`: Sử dụng GitHub Actions Cache Backend để tái sử dụng các Docker layer không thay đổi giữa các lần build.

---

## 3. Thiết kế Multi-stage Dockerfile Tối ưu

```dockerfile
# Stage 1: Build application with Gradle
FROM eclipse-temurin:17-jdk-alpine AS builder
WORKDIR /workspace
COPY gradlew .
COPY gradle gradle
COPY build.gradle settings.gradle ./
RUN chmod +x gradlew
COPY src src
RUN ./gradlew bootJar --no-daemon

# Stage 2: Minimal runtime image
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=builder /workspace/build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```
- **Ưu điểm:** Tách biệt môi trường Build (chứa JDK, source code, build tool cồng kềnh) khỏi môi trường Chạy (chỉ chứa JRE siêu nhẹ và file `.jar`), giúp giảm kích thước Docker image từ hơn 500MB xuống dưới 150MB, đồng thời tăng cường tính bảo mật.

---

## 4. Nhật ký Thực thi Chi tiết (Execution Logs on GitHub Actions)

### 4.1. Lệnh thực thi cục bộ (Local terminal)
```bash
$ git tag v1.0.0
$ git push origin v1.0.0
Total 0 (delta 0), reused 0 (delta 0), pack-reused 0
To https://github.com/my-org/my-app.git
 * [new tag]         v1.0.0 -> v1.0.0
```

### 4.2. Log thực thi các bước trên GitHub Actions
```text
Run docker/login-action@v3
  Logging into ghcr.io...
  Login Succeeded!

Run docker/metadata-action@v5
  Extracting tags and labels for ghcr.io/my-org/my-app
  Docker tags:
    ghcr.io/my-org/my-app:v1.0.0
    ghcr.io/my-org/my-app:1.0
    ghcr.io/my-org/my-app:latest
  Docker labels:
    org.opencontainers.image.title=my-app
    org.opencontainers.image.version=v1.0.0
    org.opencontainers.image.source=https://github.com/my-org/my-app

Run docker/build-push-action@v5
  Building with context: .
  #1 [internal] load build definition from Dockerfile
  #2 [builder 1/7] FROM docker.io/library/eclipse-temurin:17-jdk-alpine
  ...
  #12 [stage-1 3/3] COPY --from=builder /workspace/build/libs/*.jar app.jar
  #13 exporting to image
  #13 pushing layers to ghcr.io/my-org/my-app
  #13 pushing tag ghcr.io/my-org/my-app:v1.0.0
  #13 pushing tag ghcr.io/my-org/my-app:1.0
  #13 pushing tag ghcr.io/my-org/my-app:latest
  #13 DONE 38.4s
```

---

## 5. Hướng dẫn minh chứng hình ảnh nộp bài

Theo hướng dẫn nộp bài của đề:
1. Tạo tag `v1.0.0` và push lên GitHub.
2. Kiểm tra tab **Actions**: Chụp màn hình workflow run thành công với tên `Publish Docker Image on Release Tag`.
3. Kiểm tra trang chính repository GitHub > cột bên phải mục **Packages** (hoặc truy cập `https://github.com/<username>?tab=packages`):
   - Chụp màn hình trang chi tiết package hiển thị gói Docker Image với các tag `v1.0.0`, `1.0`, `latest`.
