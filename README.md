# BÁO CÁO THỰC HÀNH: TỰ ĐỘNG HÓA ĐÓNG GÓI VÀ PHÁT HÀNH DOCKER IMAGE KHI GẮN GIT TAG (v*.*.*)

---

- **Môn học**: DevOps Fundamentals (IT-209)
- **Session**: 12 - Tối ưu hóa CI/CD Pipeline
- **Bài tập**: Bài 2 - Tự động hóa đóng gói và phát hành Docker Image khi gắn Git Tag (v*.*.*)
- **Học viên**: Hồ Quốc Khải
- **Email**: khaiho6040@gmail.com
- **Đường dẫn gợi ý nộp bài**: `homework/session_12/ex2/`

---

## 1. MỤC TIÊU VÀ NGUYÊN LÝ THIẾT KẾ RELEASE PIPELINE

### 1.1. Mục tiêu
- Xây dựng quy trình Phân phối liên tục (Continuous Delivery - CD) tự động đóng gói và phát hành phiên bản Docker Image chính thức khi có sự kiện gán nhãn phát hành (**Git Tag `v*.*.*`**).
- Tự động đồng bộ tên thẻ Docker Image Tag với phiên bản Git Tag bằng công cụ tiêu chuẩn **`docker/metadata-action@v5`**.
- Đẩy trực tiếp Docker Image vào kho lưu trữ **GitHub Container Registry (`ghcr.io`)** và hiển thị trong mục **GitHub Packages** của repository.

### 1.2. Nguyên lý kích hoạt theo Git Tag
- Trong mô hình phát triển phần mềm chuẩn (Git Flow/GitHub Flow), không phải mọi commit lên nhánh chính đều trở thành một bản phát hành chính thức (Release).
- Việc kích hoạt workflow chỉ khi có sự kiện `push` chứa Git Tag tuân thủ định dạng SemVer (`v*.*.*`) giúp:
  1. Tách biệt rõ ràng giữa các commit phát triển hàng ngày và các phiên bản triển khai thực tế.
  2. Ngăn ngừa việc ghi đè vô ý lên các container image sản xuất.
  3. Duy trì tính toàn vẹn và khả năng truy vết nguồn gốc (Traceability) giữa commit Git và Docker Image trong Registry.

---

## 2. CẤU TRÚC THƯ MỤC DỰ ÁN

```
Ss12/2/
├── .github/
│   └── workflows/
│       └── release.yml          # Kịch bản GitHub Actions Release Docker Image khi gắn Git Tag
├── gradle/
│   └── wrapper/
│       ├── gradle-wrapper.jar   # Gradle wrapper executable archive
│       └── gradle-wrapper.properties # Cấu hình phiên bản Gradle (9.5.1)
├── src/
│   ├── main/
│   │   ├── java/com/example/tasks/
│   │   │   ├── controller/
│   │   │   │   └── TaskController.java        # REST Controller quản lý công việc
│   │   │   ├── model/
│   │   │   │   └── Task.java                  # JPA Entity Task
│   │   │   ├── repository/
│   │   │   │   └── TaskRepository.java        # JpaRepository giao tiếp Database
│   │   │   └── TaskApiApplication.java       # Spring Boot Main Application
│   │   └── resources/
│   │       ├── db/
│   │       │   └── taskdb.sql                 # DDL khởi tạo cơ sở dữ liệu
│   │       └── application.properties        # Cấu hình Spring Datasource & JPA
│   └── test/
│       ├── java/com/example/tasks/
│       │   ├── controller/
│       │   │   └── TaskControllerTest.java    # Unit test JUnit 5 & Mockito
│       │   └── TaskApiApplicationTests.java   # Spring Boot Context load test
│       └── resources/
│           └── application.properties        # Cấu hình H2 in-memory DB cho test
├── .env.example.properties       # Mẫu biến môi trường
├── .gitattributes                # Quy định xử lý line endings
├── .gitignore                    # Loại trừ build artifacts, cache, logs
├── build.gradle                  # Cấu hình Gradle build & sourceCompatibility 17
├── docker-compose.yml            # Docker Compose service cho MySQL và App
├── Dockerfile                    # Multi-stage Dockerfile tối ưu kích thước image
├── gradlew                       # Gradle wrapper script (Linux/macOS)
├── gradlew.bat                   # Gradle wrapper script (Windows)
├── settings.gradle               # Tên root project
└── README.md                     # Báo cáo thực hành chi tiết
```

---

## 3. NỘI DUNG KỊCH BẢN GITHUB ACTIONS WORKFLOW

Tệp kịch bản được lưu tại: `.github/workflows/release.yml`

```yaml
name: Release Docker Image to GHCR

on:
  push:
    tags:
      - 'v*.*.*'

permissions:
  contents: read
  packages: write

jobs:
  release-image:
    name: Build & Release Docker Image
    runs-on: ubuntu-latest
    steps:
      - name: Checkout mã nguồn
        uses: actions/checkout@v5

      - name: Thiết lập Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Đăng nhập vào GitHub Container Registry (GHCR)
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.REGISTRY_TOKEN || secrets.GITHUB_TOKEN }}

      - name: Trích xuất Docker Metadata từ Git Tag
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=ref,event=tag
            type=semver,pattern={{version}}

      - name: Đóng gói và đẩy Docker Image lên GHCR
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

### Phân tích chi tiết từng khối chỉ thị:

1. **`on: push: tags: ['v*.*.*']`**:
   - Bộ lọc kích hoạt: Workflow chỉ được kích hoạt khi có sự kiện đẩy Git Tag thỏa mãn mẫu `v*.*.*` (ví dụ `v1.0.0`, `v1.0.1`, `v2.1.0`). Các commit thông thường đẩy lên nhánh `main` hoặc các nhánh khác sẽ không kích hoạt workflow này.

2. **`permissions: packages: write`**:
   - Cấp quyền ghi gói (`write:packages`) trực tiếp cho `secrets.GITHUB_TOKEN` tích hợp sẵn trong phiên chạy, giúp GitHub Actions tự động đăng nhập và đẩy Docker image lên `ghcr.io` mà không phụ thuộc bắt buộc vào Personal Access Token (PAT).

3. **`docker/login-action@v3`**:
   - Đăng nhập vào GitHub Container Registry (`ghcr.io`) bằng tài khoản người dùng (`${{ github.actor }}`) và khóa bí mật (`${{ secrets.REGISTRY_TOKEN || secrets.GITHUB_TOKEN }}`).

4. **`docker/metadata-action@v5`**:
   - Tự động phân tích ngữ cảnh Git Tag từ sự kiện kích hoạt và tạo danh sách Docker Tag chuẩn:
     - `type=ref,event=tag`: Tạo tag khớp chính xác với Git Tag được push, ví dụ `ghcr.io/<owner>/<repo>:v1.0.0`.
     - `type=semver,pattern={{version}}`: Tạo tag chuẩn Semantic Versioning, ví dụ `ghcr.io/<owner>/<repo>:1.0.0`.
   - Tự động chuẩn hóa tên repository về chữ thường (lowercase) để đảm bảo tính hợp lệ theo quy định đặt tên của OCI Container Registry.

5. **`docker/build-push-action@v5`**:
   - Đọc tệp `Dockerfile`, tiến hành multi-stage build ứng dụng Java Spring Boot bằng Gradle bên trong container và đóng gói sản phẩm JAR vào runtime Alpine JRE siêu nhẹ.
   - Gắn các tags và labels vừa trích xuất từ bước `meta` và đẩy trực tiếp lên `ghcr.io`.

---

## 4. HƯỚNG DẪN TỰ THỰC THI VÀ KIỂM TRA

Học viên tự khởi tạo Git, gán Tag và đẩy lên GitHub theo các bước sau:

### Bước 1: Mở Terminal tại thư mục `Ss12/2`
```bash
cd "d:\[IT-209] DevOps Fundamentals\Ss12\2"
```

### Bước 2: Khởi tạo Git và commit mã nguồn
```bash
git init
git checkout -b main
git add .
git commit -m "feat: setup automated docker release workflow on git tag"
```

### Bước 3: Gắn remote repository của bạn
```bash
git remote add origin https://github.com/<your-username>/<your-repo-name>.git
git push -u origin main
```

### Bước 4: Gán Git Tag `v1.0.0` và đẩy lên GitHub
```bash
git tag v1.0.0
git push origin v1.0.0
```

### Bước 5: Kiểm tra kết quả
1. Truy cập tab **Actions** trên GitHub repository:
   - Xác nhận Workflow **`Release Docker Image to GHCR`** được kích hoạt tự động bởi sự kiện tag `v1.0.0`.
   - Job `release-image` hoàn tất thành công và đạt trạng thái tích xanh (**Success**).
2. Truy cập trang chủ Repository trên GitHub, nhìn sang cột bên phải tại mục **Packages** (hoặc truy cập `https://github.com/<your-username>?tab=packages`):
   - Bạn sẽ thấy Package Docker Image xuất hiện với tên: `ghcr.io/<your-username>/<your-repo-name>`
   - Phiên bản được phát hành mang đúng thẻ tag: **`v1.0.0`**.
3. Chụp ảnh màn hình hiển thị gói Image `v1.0.0` trong mục Packages và lưu vào thư mục `Ss12/2/` (ví dụ `packages_v1.0.0.png`) để hoàn tất sản phẩm nộp bài.
