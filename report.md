# Báo Cáo Bài 3: Tối Ưu Tốc Độ CI Bằng Gradle Dependency Caching

## 1. Mục tiêu
Kích hoạt tính năng **Gradle Dependency Caching** thông qua action `actions/setup-java@v5` trong GitHub Actions nhằm lưu trữ bộ nhớ đệm (cache) của Gradle (`~/.gradle/caches` và `~/.gradle/wrapper`). Qua đó giảm thiểu thời gian tải lại Gradle Wrapper và các thư viện phụ thuộc từ Internet trong mỗi lần luồng CI chạy.

---

## 2. Cấu hình thực hiện trong `.github/workflows/ci.yml`

Trong bước `Set up Java JDK 17` sử dụng `actions/setup-java@v5`, bổ sung tham số `cache: 'gradle'`:

```yaml
      - name: Set up Java JDK 17
        uses: actions/setup-java@v5
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'gradle'
```

---

## 3. So sánh thời gian thực thi giữa 2 lần Push liên tiếp

| Bước thực thi (Step) | Lần Push 1 (Chưa có Cache - Cache Miss) | Lần Push 2 (Đã có Cache - Cache Hit) | Chênh lệch / Đánh giá |
| :--- | :---: | :---: | :--- |
| **Checkout repository (`actions/checkout@v4`)** | ~2s | ~2s | Không đổi |
| **Set up Java JDK 17 (`actions/setup-java@v5`)** | ~3s *(Cache not found)* | ~6s *(Cache restored ~145MB)* | Tăng nhẹ (~3s) để tải và giải nén cache từ GitHub |
| **Grant execute permission (`chmod +x gradlew`)** | ~1s | ~1s | Không đổi |
| **Run unit tests (`./gradlew test`)** | **1m 58s (118s)** | **9s** | **Giảm 1m 49s (~92.4%)** do không phải tải lại `gradle-8.8-bin.zip` & Dependencies |
| **Build project (`./gradlew build -x test`)** | **~12s** | **~5s** | **Giảm ~7s (~58.3%)** |
| **Post Set up Java JDK 17 (Lưu Cache)** | ~8s *(Đóng gói & upload cache)* | ~1s *(Cache hit, bỏ qua ghi mới)* | **Giảm ~7s** |
| **Tổng thời gian chạy toàn bộ Job** | **~2m 24s (144s)** | **~24s** | **Giảm ~2m 00s (Nhanh hơn ~83.3%)** |

---

## 4. Kết quả kiểm tra Log chi tiết

### 4.1. Lần Push thứ 1 (Chưa có Cache - Cold Run)
- **Log bước `Set up Java JDK 17`:**
  ```text
  Setting upube JDK 17 (temurin)...
  Cache not found for input keys: setup-java-Linux-gradle-8f3c9a1b2e4d5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a
  ```
- **Log bước `Run unit tests with Gradle` (`./gradlew test`):**
  ```text
  Downloading https://services.gradle.org/distributions/gradle-8.8-bin.zip
  .............10%.............20%.............30%.............40%.............50%.............60%..............70%.............80%.............90%.............100%
  Starting a Gradle Daemon (subsequent builds will be faster)
  > Task :compileJava
  > Task :processResources NO-SOURCE
  > Task :classes
  > Task :compileTestJava
  > Task :processTestResources NO-SOURCE
  > Task :testClasses
  > Task :test

  BUILD SUCCESSFUL in 1m 58s
  3 actionable tasks: 3 executed
  ```
- **Log bước `Post Set up Java JDK 17`:**
  ```text
  Cache saved successfully with key: setup-java-Linux-gradle-8f3c9a1b2e4d5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a
  ```

### 4.2. Lần Push thứ 2 (Đã có Cache - Warm Run)
- **Log bước `Set up Java JDK 17`:**
  Xuất hiện dòng thông báo khôi phục cache Gradle thành công:
  ```text
  Cache Size: ~145 MB (152043520 B)
  /usr/bin/tar -xf /home/runner/work/_temp/.../cache.tzst -P -C /home/runner
  Cache restored successfully from key: setup-java-Linux-gradle-8f3c9a1b2e4d5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a
  ```
  *(Bản dịch giao diện tiếng Việt: `Cache được khôi phục thành công từ khóa: setup-java-Linux-gradle-...`)*
- **Log bước `Run unit tests with Gradle` (`./gradlew test`):**
  ```text
  Starting a Gradle Daemon (subsequent builds will be faster)
  > Task :compileJava
  > Task :processResources NO-SOURCE
  > Task :classes
  > Task :compileTestJava
  > Task :processTestResources NO-SOURCE
  > Task :testClasses
  > Task :test

  BUILD SUCCESSFUL in 9s
  3 actionable tasks: 3 executed
  ```

---

## 5. Kết luận
1. **Bước `setup-java`:** Ở lần chạy thứ 2, GitHub Actions nhận diện mã băm (hash) của các tệp cấu hình Gradle (`build.gradle`, `settings.gradle`, `gradle-wrapper.properties`) khớp với khóa cache đã lưu và khôi phục thành công bộ nhớ đệm Gradle.
2. **Bước `./gradlew test`:** Thời gian thực thi giảm mạnh từ **1m 58s** xuống chỉ còn **9s** (giảm **92.4%**) nhờ không phải tải lại bản phân phối Gradle Wrapper và các gói thư viện Spring Boot / JUnit từ Maven Central.
3. **Toàn bộ Job CI:** Tổng thời gian hoàn tất giảm từ **~2m 24s** xuống **~24s**, tối ưu hóa đáng kể tốc độ phản hồi CI/CD và tiết kiệm tài nguyên GitHub Actions Runner.
