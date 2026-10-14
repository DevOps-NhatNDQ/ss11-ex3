# Bài 3: Tối ưu tốc độ CI bằng Gradle Dependency Caching

## 1. Tổng quan bài tập
- **Mục tiêu:** Bật tính năng Caching cho Gradle Dependencies trong kịch bản GitHub Actions để giảm thời gian tải các thư viện cache Gradle từ Internet mỗi lần luồng CI chạy.
- **Thư mục bài tập:** `devops/bt/ss11/ex3/`
- **Tệp kịch bản:** `.github/workflows/ci.yml`
- **Tệp báo cáo phân tích thời gian:** `report.md`

---

## 2. Cấu trúc thư mục dự án
```text
ex3/
├── .github/
│   └── workflows/
│       └── ci.yml               # Workflow đã bổ sung cấu hình cache: 'gradle'
├── gradle/
│   └── wrapper/
│       └── gradle-wrapper.properties
├── src/
│   ├── main/
│   │   ├── java/com/example/demo/
│   │   │   ├── DemoApplication.java
│   │   │   └── controller/
│   │   │       └── HelloController.java
│   │   └── resources/
│   │       └── application.properties
│   └── test/
│       └── java/com/example/demo/
│           ├── DemoApplicationTests.java
│           └── HelloControllerTest.java
├── .gitignore
├── build.gradle
├── settings.gradle
├── gradlew
├── gradlew.bat
├── README.md
└── report.md                    # Báo cáo kỹ thuật so sánh hiệu năng chạy CI
```

---

## 3. Cấu hình kịch bản `.github/workflows/ci.yml`

```yaml
name: Java Spring Boot CI (Gradle)

on:
  push:
    branches: [ "main" ]

jobs:
  test:
    name: Run Unit Tests & Build JAR (Cached)
    runs-on: ubuntu-latest

    steps:
      - name: Checkout mã nguồn
        uses: actions/checkout@v5

      - name: Thiết lập môi trường Java JDK 17
        uses: actions/setup-java@v5
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'gradle'

      - name: Cấp quyền cho Gradle Wrapper
        run: chmod +x gradlew

      - name: Thực thi JUnit Test với Gradle
        run: ./gradlew test

      - name: Đóng gói tệp JAR với Gradle
        run: ./gradlew bootJar

      - name: Lưu trữ thành phẩm JAR Artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-jar
          path: build/libs/*.jar
```

---

## 4. Hướng dẫn thử nghiệm kiểm chứng Caching
1. **Lần Push 1 (Khởi tạo Cache - Cache Miss):**
   - Đẩy commit kích hoạt workflow.
   - Bước `setup-java` chưa tìm thấy cache tương ứng và sẽ ghi nhận trạng thái cache miss.
   - Các bước `./gradlew test` và `./gradlew bootJar` phải tải Gradle distribution và các dependency jars từ Maven Central.
   - Cuối job, action tự động nén thư mục `~/.gradle/caches` và `~/.gradle/wrapper` lưu lên GitHub Cache Storage.
2. **Lần Push 2 (Tận dụng Cache - Cache Hit):**
   - Thực hiện một commit nhỏ (ví dụ chỉnh sửa comment hoặc code) và push lên nhánh `main`.
   - Bước `setup-java` phát hiện cache key trùng khớp và hiển thị log: `Cache restored successfully from key: gradle-...`.
   - Các bước biên dịch và kiểm thử bỏ qua công đoạn tải thư viện từ mạng, tái sử dụng dependencies có sẵn.
   - Xem chi tiết so sánh số liệu đo đạc tại [report.md](file:///d:/PTIT2024B_EXCERCISE_Rikkei/DevOps/bt/ss11/ex3/report.md).
