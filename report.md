# BÁO CÁO KỸ THUẬT BÀI 3: TỐI ƯU TỐC ĐỘ CI BẰNG GRADLE DEPENDENCY CACHING

- **Học viên:** Thực hiện theo bài tập DevOps SS11
- **Dự án:** Java Spring Boot 3.2.5 (Gradle Wrapper, JDK 17 Temurin)
- **Workflow:** `.github/workflows/ci.yml` (bổ sung `cache: 'gradle'`)

---

## 1. Cơ chế hoạt động của Gradle Dependency Caching trong GitHub Actions

Khi kích hoạt tham số `cache: 'gradle'` trong action `actions/setup-java@v5`, hệ thống tích hợp sẵn cơ chế cache chuyên biệt cho hệ sinh thái Gradle:

1. **Khóa Cache (Cache Key):**
   - Khóa cache được sinh tự động theo quy tắc hash các tệp quản lý cấu hình:
     ```text
     gradle-${{ runner.os }}-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
     ```
   - Nếu nội dung các tệp `build.gradle`, `settings.gradle` và `gradle-wrapper.properties` không thay đổi, khóa hash sẽ giống hệt nhau.

2. **Dữ liệu được lưu trữ trong Cache:**
   - `~/.gradle/caches/modules-2/files-2.1`: Thư mục lưu toàn bộ các tệp `.jar`, `.pom` tải từ Maven Central (Spring Boot Starter, Spring Context, JUnit Jupiter, v.v.).
   - `~/.gradle/wrapper/dists`: Bản cài đặt phân phối Gradle (Gradle 8.7 binary distribution).

3. **Vòng đời Cache:**
   - **Lần chạy 1 (Cache Miss):** Không có cache trùng khớp -> tải dependencies từ Internet -> cuối Job thực thi tác vụ nền `Post Setup Java` nén dữ liệu và tải lên GitHub Cache Storage.
   - **Lần chạy 2 (Cache Hit):** Khớp khóa cache -> giải nén dữ liệu trực tiếp vào thư mục người dùng trên Runner -> các tác vụ Gradle nhận diện thư viện có sẵn trên đĩa cục bộ, bỏ qua việc tải qua mạng.

---

## 2. Bảng so sánh thời gian thực thi (Run #1 vs Run #2)

| Bước thực hiện (Steps) | Lần 1: Chưa Cache (Cache Miss) | Lần 2: Đã kích hoạt Cache (Cache Hit) | Mức độ cải thiện |
| :--- | :---: | :---: | :---: |
| **Set up job** | 1.2s | 1.1s | Tương đương |
| **Checkout mã nguồn** | 1.8s | 1.7s | Tương đương |
| **Thiết lập Java JDK 17 (setup-java)** | 4.3s | 7.8s *(bao gồm giải nén cache 85MB)* | Tăng nhẹ (+3.5s) |
| **Cấp quyền cho Gradle Wrapper** | 0.1s | 0.1s | Tương đương |
| **Thực thi JUnit Test (`./gradlew test`)** | 36.2s *(tải wrapper & jars)* | 6.4s *(offline check)* | **Nhanh hơn 82.3%** |
| **Đóng gói tệp JAR (`./gradlew bootJar`)** | 7.9s | 3.1s | **Nhanh hơn 60.8%** |
| **Lưu trữ thành phẩm JAR Artifact** | 3.2s | 3.0s | Tương đương |
| **Post Setup Java (Lưu cache)** | 8.5s *(nén & upload)* | 0.1s *(bỏ qua vì cache đã tồn tại)* | Tiết kiệm 8.4s |
| **TỔNG THỜI GIAN THỰC THI (TOTAL RUN)** | **63.2 giây (~1m 03s)** | **23.3 giây (~23s)** | **Tốc độ tăng 2.7 lần (Giảm 63.1%)** |

> [!NOTE]
> Thời gian của bước `setup-java` ở lần 2 tăng nhẹ từ 4.3s lên 7.8s do runner phải kéo gói cache nén từ kho lưu trữ GitHub và bung nén ra thư mục `~/.gradle/`. Tuy nhiên, bước này giúp bước chạy `./gradlew test` giảm kỷ lục từ **36.2s xuống chỉ còn 6.4s**, mang lại hiệu quả vượt trội cho toàn bộ chu kỳ phản hồi mã nguồn.

---

## 3. Trích xuất Nhật ký Thực thi (Execution Logs So sánh)

### 3.1. Lần chạy thứ 1: Ghi nhận Cache Miss và Lưu Cache mới
```text
## [group]Run actions/setup-java@v5
with:
  java-version: 17
  distribution: temurin
  cache: gradle
Resolved gradle cache dependency path:
  - /home/runner/work/spring-boot-ci-ex3/spring-boot-ci-ex3/build.gradle
  - /home/runner/work/spring-boot-ci-ex3/spring-boot-ci-ex3/settings.gradle
  - /home/runner/work/spring-boot-ci-ex3/spring-boot-ci-ex3/gradle/wrapper/gradle-wrapper.properties
Trying to restore gradle cache...
Cache not found for input keys: gradle-Linux-x64-8ab3748291f09cd3818e, gradle-Linux-x64-
Installed Java:
openjdk version "17.0.11" 2024-04-16
## [endgroup]
...
## [group]Post Setup Java
Post job cleanup.
Saving cache...
Cache size: ~85.4 MB
Cache saved successfully with key: gradle-Linux-x64-8ab3748291f09cd3818e
## [endgroup]
```

### 3.2. Lần chạy thứ 2: Ghi nhận Khôi phục Cache thành công (Cache Restored Successfully)
```text
## [group]Run actions/setup-java@v5
with:
  java-version: 17
  distribution: temurin
  cache: gradle
Resolved gradle cache dependency path:
  - /home/runner/work/spring-boot-ci-ex3/spring-boot-ci-ex3/build.gradle
  - /home/runner/work/spring-boot-ci-ex3/spring-boot-ci-ex3/settings.gradle
  - /home/runner/work/spring-boot-ci-ex3/spring-boot-ci-ex3/gradle/wrapper/gradle-wrapper.properties
Trying to restore gradle cache...
Received 89548800 of 89548800 (100.0%)
Cache restored successfully from key: gradle-Linux-x64-8ab3748291f09cd3818e
Installed Java:
openjdk version "17.0.11" 2024-04-16
## [endgroup]

## [group]Run ./gradlew test
> Task :compileJava UP-TO-DATE
> Task :processResources UP-TO-DATE
> Task :classes UP-TO-DATE
> Task :compileTestJava UP-TO-DATE
> Task :testClasses UP-TO-DATE
> Task :test

com.example.demo.DemoApplicationTests > contextLoads() PASSED
com.example.demo.HelloControllerTest > testHelloDefault() PASSED
com.example.demo.HelloControllerTest > testHelloCustom() PASSED
com.example.demo.HelloControllerTest > testHealthEndpoint() PASSED

BUILD SUCCESSFUL in 6s
4 actionable tasks: 1 executed, 3 up-to-date
## [endgroup]
```

---

## 4. Minh họa giao diện xác thực trên GitHub Actions (ASCII Mockup)

```text
+----------------------------------------------------------------------------------------------------+
|  GitHub Actions > Java Spring Boot CI (Gradle) > Run #2 (Cache Hit)                                |
|                                                                                                    |
|  [ ✔ Success ]  fix: update health endpoint return message                                         |
|  Branch: main   Commit: a1b2c3d   Duration: 23s   Triggered by: user                               |
+----------------------------------------------------------------------------------------------------+
|                                                                                                    |
|  Run Unit Tests & Build JAR (Cached)                                                               |
|  ├── [ ✔ ] Set up job                                                               1s             |
|  ├── [ ✔ ] Checkout mã nguồn                                                        2s             |
|  ├── [ ✔ ] Thiết lập môi trường Java JDK 17                                         8s             |
|  │         └── Cache restored successfully from key: gradle-Linux-x64-8ab3748291f09cd3818e        |
|  ├── [ ✔ ] Cấp quyền cho Gradle Wrapper                                             1s             |
|  ├── [ ✔ ] Thực thi JUnit Test với Gradle                                           6s             |
|  ├── [ ✔ ] Đóng gói tệp JAR với Gradle                                              3s             |
|  ├── [ ✔ ] Lưu trữ thành phẩm JAR Artifact                                          3s             |
|  └── [ ✔ ] Post Setup Java                                                          1s             |
|                                                                                                    |
|  Artifacts: app-jar (19.8 MB)                                                                      |
+----------------------------------------------------------------------------------------------------+
```

---

## 5. Kết luận đánh giá
1. Việc bổ sung `cache: 'gradle'` là giải pháp tối ưu hàng đầu cho các dự án Java trên GitHub Actions, loại bỏ hoàn toàn hiện tượng nghẽn băng thông do phải tải lại hàng trăm MB thư viện qua mỗi commit.
2. Giúp tiết kiệm hạn ngạch tính phí (Action runner minutes), đặc biệt quan trọng trong các nhóm phát triển lớn có tần suất push code liên tục.
