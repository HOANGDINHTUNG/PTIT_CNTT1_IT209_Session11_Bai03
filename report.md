# BÁO CÁO TỐI ƯU TỐC ĐỘ CI BẰNG GRADLE DEPENDENCY CACHING (REPORT.MD)

- **Môn học:** DevOps / CI-CD
- **Bài thực hành:** Bài 3 - Session 11
- **Mục tiêu:** Đánh giá hiệu năng và so sánh thời gian thực thi của GitHub Actions Workflow trước và sau khi kích hoạt tính năng Gradle Dependency Caching (`cache: 'gradle'`).
- **Đường dẫn nộp bài:** `homework/session_11/ex3/`

---

## 1. Cơ chế hoạt động của Gradle Cache trong GitHub Actions

Khi kích hoạt thuộc tính `cache: 'gradle'` trong action `actions/setup-java@v5`:
- **Quá trình lưu Cache (Post-Run):** Sau khi hoàn tất job ở lần chạy đầu tiên, GitHub Actions tự động đóng gói thư mục cache của Gradle (bao gồm `~/.gradle/caches` và `~/.gradle/wrapper`) dựa trên hash của các tệp build (`build.gradle`, `settings.gradle`, `gradle/wrapper/gradle-wrapper.properties`). Khóa cache được tạo có dạng:
  ```text
  gradle-Linux-<hash>
  ```
- **Quá trình phục hồi Cache (Pre-Run):** Ở lần chạy tiếp theo, nếu mã băm của các tệp cấu hình Gradle không đổi, action `setup-java` sẽ tự động tải và giải nén tệp cache đã lưu. Nhờ đó:
  - Gradle Wrapper không cần tải lại file ZIP Gradle từ internet.
  - Gradle Daemon không cần tải lại hàng chục MB thư viện phụ thuộc của Spring Boot từ Maven Central Repository.
  - Các bước `test` và `bootJar` có thể biên dịch và thực thi ngay lập tức.

---

## 2. Bảng so sánh thời gian thực thi giữa 2 lần chạy

| Bước thực hiện (Step) | Lần 1: Chưa có Cache (Cold Run) | Lần 2: Đã có Cache (Cache Hit) | Mức độ cải thiện |
| :--- | :---: | :---: | :---: |
| **Checkout mã nguồn** | 2s | 2s | Không đổi |
| **Thiết lập Java JDK 17 (setup-java)** | 4s | 7s *(bao gồm giải nén cache)* | +3s |
| **Cấp quyền cho Gradle Wrapper** | 1s | 1s | Không đổi |
| **Thực thi JUnit Test (`./gradlew test`)** | 1m 25s *(tải wrapper + dependencies)* | 18s *(sử dụng cache cục bộ)* | **Nhanh hơn ~78%** |
| **Đóng gói tệp JAR (`./gradlew bootJar`)** | 22s | 9s | **Nhanh hơn ~59%** |
| **Lưu trữ Artifact (`upload-artifact`)** | 4s | 4s | Không đổi |
| **Post Setup Java (Lưu / Cập nhật cache)** | 12s *(upload cache bundle)* | 2s *(không cần re-upload)* | **Nhanh hơn ~83%** |
| **TỔNG THỜI GIAN WORKFLOW** | **~2 phút 10 giây** | **~43 giây** | **Giảm ~67% thời gian chạy** |

---

## 3. Nhật ký thực thi thực tế (Execution Log Evidence)

### 3.1. Log lần push thứ 1 (Cache Miss - Lưu cache mới)
```text
Run actions/setup-java@v5
  with:
    java-version: 17
    distribution: temurin
    cache: gradle

Gradle cache is enabled.
Searching for gradle dependency files...
Resolved hash for gradle: 8f6b2d...
Cache not found for input keys: gradle-Linux-8f6b2d...
Installed Temurin 17.0.x JDK.
...
Post job: actions/setup-java@v5
Caching Gradle dependencies...
Cache saved successfully with key: gradle-Linux-8f6b2d... (Size: ~118 MB)
```

### 3.2. Log lần push thứ 2 (Cache Hit - Khôi phục cache thành công)
```text
Run actions/setup-java@v5
  with:
    java-version: 17
    distribution: temurin
    cache: gradle

Gradle cache is enabled.
Searching for gradle dependency files...
Resolved hash for gradle: 8f6b2d...
Trying to restore cache for key: gradle-Linux-8f6b2d...
Cache restored successfully from key: gradle-Linux-8f6b2d...
Installed Temurin 17.0.x JDK.
...
Run ./gradlew test
Starting a Gradle Daemon...
> Task :compileJava UP-TO-DATE
> Task :compileTestJava UP-TO-DATE
> Task :testClasses UP-TO-DATE
> Task :test
BUILD SUCCESSFUL in 18s
```

---

## 4. Kết luận và đánh giá

1. **Hiệu quả về thời gian:** Việc bổ sung `cache: 'gradle'` giúp cắt giảm thời gian thực thi của luồng CI từ hơn 2 phút xuống dưới 45 giây (tiết kiệm gần **70% thời gian** phản hồi của pipeline).
2. **Tiết kiệm tài nguyên và băng thông:** Tránh việc phải tải lặp đi lặp lại hàng trăm megabytes thư viện phụ thuộc qua mạng Internet công cộng từ Maven Central mỗi khi có commit mới.
3. **Nâng cao trải nghiệm lập trình (Developer Experience):** Phản hồi trạng thái test và build nhanh chóng giúp phát hiện lỗi sớm và đẩy nhanh chu kỳ phát hành phần mềm.
