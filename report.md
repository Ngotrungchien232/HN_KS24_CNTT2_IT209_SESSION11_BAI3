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

## 3. So sánh thời gian thực thi thực tế giữa 2 lần Push liên tiếp trên GitHub Actions

- **Repository:** [Ngotrungchien232/HN_KS24_CNTT2_IT209_SESSION11_BAI3](https://github.com/Ngotrungchien232/HN_KS24_CNTT2_IT209_SESSION11_BAI3)
- **Lần Push 1 (Run #1 - Chưa có Cache):** [Actions Run #38224713813](https://github.com/Ngotrungchien232/HN_KS24_CNTT2_IT209_SESSION11_BAI3/actions/runs/38224713813)
- **Lần Push 2 (Run #2 - Đã có Cache):** [Actions Run #38224885343](https://github.com/Ngotrungchien232/HN_KS24_CNTT2_IT209_SESSION11_BAI3/actions/runs/38224885343)

| Bước thực thi (Step) | Lần Push 1 (Run #1 - Chưa Cache) | Lần Push 2 (Run #2 - Đã Cache) | Chênh lệch / Đánh giá |
| :--- | :---: | :---: | :--- |
| **Set up job** | 1s | 1s | Không đổi |
| **Checkout repository (`actions/checkout@v4`)** | 0s | 1s | Không đáng kể |
| **Set up Java JDK 17 (`actions/setup-java@v5`)** | **1s** *(Cache not found)* | **2s** *(Cache restored)* | Khôi phục cache Gradle thành công từ khóa `setup-java-Linux-gradle-...` |
| **Grant execute permission (`chmod +x gradlew`)** | 0s | 0s | Không đổi |
| **Run unit tests (`./gradlew test`)** | **33s** | **19s** | **Giảm 14s (Nhanh hơn ~42.4%)** do không phải tải lại Gradle Wrapper & Dependencies |
| **Build project (`./gradlew build -x test`)** | **2s** | **1s** | **Giảm 1s (Nhanh hơn 50%)** |
| **Post Set up Java JDK 17 (Lưu Cache)** | **4s** *(Đóng gói & upload cache)* | **0s** *(Cache hit, bỏ qua ghi mới)* | **Giảm 4s** |
| **Tổng thời gian chạy toàn bộ Job** | **43s** | **26s** | **Giảm 17s (Nhanh hơn ~39.5%)** |

*(Ngoài ra, khi đo trên môi trường máy cục bộ từ trạng thái chưa có `gradle-8.8-bin.zip`, thời gian chạy `./gradlew test` giảm từ **1m 58s** xuống còn **9s**).*

---

## 4. Kết quả kiểm tra Log trên GitHub Actions

### 4.1. Lần Push thứ 1 (Run #1 - Cold Run / Chưa có Cache)
- **Log bước `Set up Java JDK 17`:**
  ```text
  Cache not found for input keys: setup-java-Linux-gradle-...
  ```
- **Log bước `Run unit tests with Gradle` (`./gradlew test`):**
  Gradle phải tải mới `gradle-8.8-bin.zip` và toàn bộ dependencies từ Maven Central, mất **33s**.
- **Log bước `Post Set up Java JDK 17`:**
  ```text
  Cache saved successfully with key: setup-java-Linux-gradle-...
  ```

### 4.2. Lần Push thứ 2 (Run #2 - Warm Run / Đã có Cache)
- **Log bước `Set up Java JDK 17`:**
  Xuất hiện dòng thông báo khôi phục cache Gradle thành công:
  ```text
  Cache restored successfully from key: setup-java-Linux-gradle-...
  ```
  *(Bản dịch giao diện tiếng Việt: `Cache được khôi phục thành công từ khóa: setup-java-Linux-gradle-...`)*
- **Log bước `Run unit tests with Gradle` (`./gradlew test`):**
  Gradle sử dụng trực tiếp bộ phân phối và các thư viện đã được khôi phục từ cache, thời gian chạy giảm từ **33s** xuống chỉ còn **19s**.

---

## 5. Kết luận
1. **Bước `setup-java`:** Ở lần push thứ 2, `actions/setup-java@v5` tự động khôi phục thành công bộ nhớ đệm Gradle đã lưu ở lần chạy đầu tiên.
2. **Bước `./gradlew test`:** Thời gian thực thi trên GitHub Actions giảm từ **33s** xuống **19s** (giảm **42.4%**).
3. **Toàn bộ Job CI:** Tổng thời gian hoàn tất Job giảm từ **43s** xuống **26s** (giảm **39.5%**), chứng minh hiệu quả rõ rệt của cơ chế Gradle Dependency Caching.
