# Bài 3: Tối Ưu Tốc Độ CI Bằng Gradle Dependency Caching

## 1. Mục tiêu
Bật tính năng **Caching cho Gradle Dependencies** trong GitHub Actions (`actions/setup-java@v5`) để giảm thời gian tải các thư viện cache Gradle từ Internet mỗi lần luồng CI chạy.

---

## 2. Cấu trúc thư mục dự án

```text
SESSION11_BAI3/
├── .github/
│   └── workflows/
│       └── ci.yml                  # Kịch bản GitHub Actions CI (đã bật cache: 'gradle')
├── gradle/
│   └── wrapper/
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── homework/
│   └── session_11/
│       └── ex3/                    # Thư mục sản phẩm nộp bài theo đường dẫn gợi ý
│           ├── .github/
│           │   └── workflows/
│           │       └── ci.yml
│           └── report.md
├── src/
│   ├── main/java/com/example/
│   │   ├── Application.java
│   │   └── Calculator.java
│   └── test/java/com/example/
│       └── CalculatorTest.java     # Unit Test phục vụ bước ./gradlew test
├── .gitattributes                  # Đảm bảo gradlew giữ chuẩn LF trên Linux runner
├── .gitignore
├── build.gradle
├── gradlew
├── gradlew.bat
├── settings.gradle
├── report.md                       # Báo cáo so sánh thời gian thực thi CI
└── README.md
```

---

## 3. Cấu hình GitHub Actions Workflow (`.github/workflows/ci.yml`)

```yaml
name: Java CI with Gradle

on:
  push:
    branches: [ "main", "master" ]
  pull_request:
    branches: [ "main", "master" ]

jobs:
  build-and-test:
    name: Build and Test with Gradle
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Java JDK 17
        uses: actions/setup-java@v5
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'gradle'

      - name: Grant execute permission for gradlew
        run: chmod +x gradlew

      - name: Run unit tests with Gradle
        run: ./gradlew test

      - name: Build project with Gradle
        run: ./gradlew build -x test
```

---

## 4. Báo cáo so sánh thời gian thực thi (Chưa Cache vs. Đã Cache)

Chi tiết xem tại tệp [report.md](./report.md) (và bản sao tại [homework/session_11/ex3/report.md](./homework/session_11/ex3/report.md)):

| Bước thực thi (Step) | Lần Push 1 (Chưa Cache) | Lần Push 2 (Đã Cache) | Chênh lệch |
| :--- | :---: | :---: | :--- |
| **Set up Java JDK 17** | ~3s *(Cache not found)* | ~6s *(Cache restored)* | Khôi phục cache thành công từ khóa `setup-java-Linux-gradle-...` |
| **Run unit tests (`./gradlew test`)** | **1m 58s (118s)** | **9s** | **Giảm 1m 49s (~92.4%)** |
| **Build project (`./gradlew build -x test`)** | **~12s** | **~5s** | **Giảm ~7s (~58.3%)** |
| **Tổng thời gian toàn bộ Job** | **~2m 24s (144s)** | **~24s** | **Giảm ~2m 00s (~83.3%)** |
