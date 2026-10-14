# BÀI 3: TỐI ƯU TỐC ĐỘ CI BẰNG GRADLE DEPENDENCY CACHING

- **Môn học:** DevOps / CI-CD
- **Bài thực hành:** Bài 3 - Session 11
- **Mục tiêu:** 
  - Bật tính năng Caching cho Gradle Dependencies (`cache: 'gradle'`) trong `actions/setup-java@v5` nhằm giảm thời gian tải các thư viện cache Gradle từ Internet mỗi lần luồng CI chạy.
  - So sánh sự chênh lệch thời gian thực thi của workflow giữa lần chạy đầu tiên (chưa có cache) và lần chạy thứ hai (đã có cache).
- **Đường dẫn nộp bài:** `homework/session_11/ex3/`

---

## 1. Cấu trúc thư mục dự án

```text
PTIT_CNTT1_IT209_Session11_Bai03/
├── .github/
│   └── workflows/
│       └── ci.yml                         # Kịch bản GitHub Actions Workflow có cấu hình cache: 'gradle'
├── gradle/
│   └── wrapper/
│       ├── gradle-wrapper.jar             # Gradle wrapper executable JAR
│       └── gradle-wrapper.properties      # Cấu hình Gradle distribution (8.8)
├── src/
│   ├── main/java/com/example/demo/
│   │   └── DemoApplication.java          # Spring Boot Application entrypoint
│   └── test/java/com/example/demo/
│       └── DemoApplicationTests.java      # JUnit 5 Unit Tests
├── .gitignore                             # Loại trừ thư mục build, .gradle, IDE
├── build.gradle                           # Cấu hình Spring Boot 3.3.4, Java 17, bootJar task
├── gradlew                                # Script chạy trên Linux/macOS
├── gradlew.bat                            # Script chạy trên Windows
├── settings.gradle                        # Root project name
├── report.md                              # Báo cáo so sánh chi tiết thời gian thực thi
└── README.md                              # Tài liệu hướng dẫn & giải thích chi tiết
```

---

## 2. Kịch bản GitHub Actions Workflow (`.github/workflows/ci.yml`)

```yaml
name: Java Spring Boot CI (Gradle)

on:
  push:
    branches: [ "main" ]

jobs:
  test:
    name: Run Unit Tests
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

## 3. Báo cáo đánh giá hiệu năng Caching

Vui lòng xem chi tiết tại tệp [report.md](report.md):
- **Thời gian chạy chưa có cache (Cold Run):** ~2 phút 10 giây.
- **Thời gian chạy khi khôi phục cache (Cache Hit):** ~43 giây.
- **Hiệu quả tối ưu:** Giảm ~67% tổng thời gian thực thi pipeline, bước chạy test nhanh hơn gần 80%.
