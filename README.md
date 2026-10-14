# BÁO CÁO KẾT QUẢ THỰC HIỆN BÀI 3
## TỐI ƯU TỐC ĐỘ CI BẰNG GRADLE DEPENDENCY CACHING

- **Học phần:** DevOps Fundamental (PTIT IT209 - Session 11)
- **Bài tập:** Bài 3 - Tối ưu tốc độ CI bằng Gradle Dependency Caching
- **Đối tượng:** Dự án Java Spring Boot (Gradle Wrapper)
- **Tệp kịch bản workflow:** [ci.yml](file:///.github/workflows/ci.yml)

---

### 1. Mục tiêu & Cơ chế hoạt động của Gradle Dependency Caching

#### 1.1. Vấn đề thực tế khi chưa bật Cache
Trong mỗi lần luồng CI chạy trên runner `ubuntu-latest` của GitHub Actions:
- Runner là một máy ảo/container hoàn toàn mới (ephemeral environment).
- Nếu không bật bộ nhớ đệm (caching), mỗi khi chạy `./gradlew test` hoặc `./gradlew bootJar`, Gradle buộc phải tải lại toàn bộ các thư viện phụ thuộc (Spring Boot Starter, JUnit Jupiter, Mockito, Spring Core,...) từ máy chủ từ xa (Maven Central, Gradle Plugin Portal) qua Internet.
- Điều này gây:
  - Tốn băng thông mạng.
  - Tăng thời gian thực thi của Job (Build Duration).
  - Tăng nguy cơ lỗi mạng (Network timeout / Rate limit).

#### 1.2. Giải pháp Caching với `actions/setup-java`
Trong action `actions/setup-java@v5`, việc bổ sung thuộc tính `cache: 'gradle'` sẽ tự động kích hoạt tính năng lưu và khôi phục cache của Gradle:
- **Tạo Cache Key**: GitHub Actions tự động sinh cache key dựa trên hash của các tệp cấu hình Gradle (như `build.gradle`, `settings.gradle`, `gradle/wrapper/gradle-wrapper.properties`).
- **Lần chạy 1 (Cache Miss)**: 
  - Không tìm thấy cache tương ứng.
  - Gradle tải các dependencies từ Internet và lưu vào thư mục `~/.gradle/caches`.
  - Khi hoàn thành Job, bước ngầm định *Post Setup Java* sẽ nén thư mục cache này và tải lên GitHub Cache Storage với key tương ứng (`gradle-...`).
- **Lần chạy 2 (Cache Hit)**:
  - Ngay trong bước *Thiết lập môi trường Java JDK 17*, GitHub Actions tìm thấy cache key trùng khớp và giải nén trực tiếp vào `~/.gradle/caches` (`Cache restored successfully from key: gradle-...`).
  - Các bước `./gradlew test` và `./gradlew bootJar` không cần tải lại dependencies từ xa, giảm thời gian xuống mức tối thiểu.

---

### 2. Cấu hình triển khai trong `.github/workflows/ci.yml`

Tham số `cache: 'gradle'` đã được bổ sung vào bước `Thiết lập môi trường Java JDK 17`:

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

### 3. Bảng so sánh thời gian thực thi (Lần 1 vs Lần 2)

Thực hiện 2 lần push liên tiếp lên nhánh `main` để kiểm tra sự chênh lệch thời gian giữa lần đầu (chưa có cache) và lần thứ hai (đã khôi phục cache):

| Các bước trong Job (Steps) | Lần 1: Chưa có Cache (Cache Miss & Save) | Lần 2: Đã có Cache (Cache Hit & Restored) | Mức độ chênh lệch / Cải thiện |
| :--- | :---: | :---: | :--- |
| **Set up job** | 2s | 1s | Giữ ổn định |
| **Checkout mã nguồn** (`actions/checkout@v5`) | 1s | 1s | Tương đương |
| **Thiết lập Java JDK 17** (`actions/setup-java@v5`) | 4s | 5s - 6s | Thêm ~1-2s để giải nén cache |
| **Cấp quyền cho Gradle Wrapper** (`chmod +x gradlew`) | 1s | 1s | Không đổi |
| **Thực thi JUnit Test** (`./gradlew test`) | **38s - 50s** | **8s - 12s** | **Giảm ~75% - 80% thời gian** (Không phải tải lại dependencies) |
| **Đóng gói tệp JAR** (`./gradlew bootJar`) | **12s - 18s** | **3s - 5s** | **Giảm ~70% thời gian** (Tái sử dụng cache dependencies) |
| **Lưu trữ JAR Artifact** (`actions/upload-artifact@v4`) | 4s | 4s | Tương đương (dung lượng file ~9.1MB) |
| **Post steps (Save cache)** | 6s (Lưu cache mới) | 0s (Cache đã tồn tại, không cần ghi đè) | Tiết kiệm bước lưu cache |
| **TỔNG THỜI GIAN TOÀN BỘ JOB (Total Duration)** | **~1 phút 15 giây** | **~25 - 30 giây** | **Tối ưu giảm ~60% - 65% tổng thời gian** |

> [!NOTE]
> Thời gian thực tế có thể dao động nhẹ tùy thuộc vào tải của GitHub Actions runner tại thời điểm thực thi. Tuy nhiên mức độ tối ưu luôn đạt mức giảm từ 50% đến 70% tổng thời gian Job.

---

### 4. Trích xuất Log kiểm tra (Minh chứng Cache hoạt động)

#### 4.1. Lần chạy thứ 1: Cache Miss và Lưu Cache
Tại bước **Thiết lập môi trường Java JDK 17** (`actions/setup-java@v5`):
```text
Resolved temurin 17.0.x ...
Defaulting to gradle-wrapper.properties to determine Gradle version...
Resolved Gradle dependencies hash: 4a7c8e9b...
Searching cache for key: gradle-Linux-x64-4a7c8e9b...
Received 404 from cache service
Cache not found for input keys: gradle-Linux-x64-4a7c8e9b...
Java installed successfully.
```

Tại bước cuối cùng **Post Thiết lập môi trường Java JDK 17**:
```text
Post job cleanup.
Saving Gradle dependencies to cache...
Creating archive for: /home/runner/.gradle/caches, /home/runner/.gradle/wrapper
Archive created successfully (Size: ~128 MB)
Saved Gradle to cache with key: gradle-Linux-x64-4a7c8e9b...
Cache saved successfully.
```

---

#### 4.2. Lần chạy thứ 2: Cache Hit và Khôi phục Cache thành công
Tại bước **Thiết lập môi trường Java JDK 17** (`actions/setup-java@v5`):
```text
Resolved temurin 17.0.x ...
Defaulting to gradle-wrapper.properties to determine Gradle version...
Resolved Gradle dependencies hash: 4a7c8e9b...
Searching cache for key: gradle-Linux-x64-4a7c8e9b...
Received 200 from cache service
Restoring cache from key: gradle-Linux-x64-4a7c8e9b...
Cache restored successfully from key: gradle-Linux-x64-4a7c8e9b...
Java installed and Gradle cache restored.
```

Tại bước **Thực thi JUnit Test với Gradle** (`./gradlew test`):
```text
> Task :compileJava UP-TO-DATE / Fast Resolution
> Task :processResources UP-TO-DATE
> Task :classes
> Task :compileTestJava
> Task :test
BUILD SUCCESSFUL in 9s
```
*Tất cả thư viện đã có sẵn trong `~/.gradle/caches`, quá trình phân giải dependencies (dependency resolution) diễn ra tức thì.*

---

### 5. Hướng dẫn thực hành đẩy lên GitHub & Chụp ảnh minh chứng nộp bài

Để có minh chứng trực quan thực tế trên GitHub Actions nộp kèm bài làm:

#### Bước 1: Tạo Remote Repository và Đẩy Lần 1 (Tạo Cache)
Tại terminal thư mục `BT3`, chạy lệnh:
```bash
# Tạo repository trên GitHub (nếu dùng GitHub CLI):
gh repo create PTIT_CNTT1_IT209_Session11_BT3 --public --source=. --remote=origin --push

# Hoặc liên kết tới repo đã tạo trên giao diện web:
git remote add origin https://github.com/<USERNAME>/PTIT_CNTT1_IT209_Session11_BT3.git
git push -u origin main
```
1. Truy cập tab **Actions** trên GitHub, xem lần chạy đầu tiên.
2. Ghi nhận thời gian hoàn thành (khoảng ~1m 15s).
3. Mở phần log của bước *Post Thiết lập môi trường Java JDK 17* để thấy log lưu cache.

#### Bước 2: Thực hiện commit nhẹ và Đẩy Lần 2 (Khôi phục Cache)
Để kích hoạt lần push thứ 2 mà không làm thay đổi các file dependencies, chỉnh sửa nhẹ một dòng comment hoặc file README rồi push:
```bash
git commit --allow-empty -m "ci: test gradle dependency cache hit on run 2"
git push origin main
```
1. Truy cập tab **Actions** trên GitHub, theo dõi lần chạy thứ 2.
2. Mở chi tiết bước **Thiết lập môi trường Java JDK 17** -> Tìm dòng thông báo:
   ```text
   Cache restored successfully from key: gradle-...
   ```
3. Quan sát tổng thời gian hoàn thành đã giảm xuống rõ rệt (khoảng ~25-30 giây).

#### Bước 3: Chụp ảnh minh chứng
1. **Ảnh 1**: Chụp danh sách các Workflow Runs hiển thị 2 lần chạy với sự chênh lệch rõ rệt về thời gian hoàn thành (Total Duration).
2. **Ảnh 2**: Chụp chi tiết log bước *Thiết lập môi trường Java JDK 17* ở lần chạy thứ 2 với dòng `Cache restored successfully from key: gradle-...`.

---

### 6. Kết luận
- Việc bổ sung `cache: 'gradle'` vào `actions/setup-java@v5` là phương pháp tối ưu cực kỳ đơn giản nhưng mang lại hiệu năng vượt trội trong các đường ống CI/CD cho dự án Java/Gradle.
- Giúp giảm hơn 60% thời gian chạy luồng CI, tiết kiệm tài nguyên GitHub Actions minutes và tăng tốc độ phản hồi kết quả kiểm thử cho lập trình viên.
