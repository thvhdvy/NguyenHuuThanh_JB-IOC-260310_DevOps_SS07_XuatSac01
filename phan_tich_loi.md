# Bài 5: Phân tích lỗi `UnsupportedClassVersionError` trong CI của notification-service

## 1. Hiện tượng

Sau khi lập trình viên push code lên `notification-service`, bước build của pipeline CI thất bại với thông báo:

```
UnsupportedClassVersionError: class file has been compiled by a more recent version
of the Java Runtime (class file version 65.0), this version of the Java Runtime
only recognizes class file versions up to 61.0
```

Thông tin bổ sung: dịch vụ được phát triển và build bằng **Java 21** trên máy lập trình viên, còn phiên bản Java cấu hình trong CI chưa rõ.

## 2. Phân tích nguyên nhân

### 2.1. Class file version là gì?

Mỗi file `.class` do trình biên dịch (`javac`) tạo ra mang trong phần đầu một số hiệu gọi là **major version**, cho biết file được biên dịch cho phiên bản Java nào. Khi nạp class, JVM so sánh số này với mức tối đa mà nó hỗ trợ. Nếu class có version **cao hơn** mức JVM hỗ trợ, JVM từ chối nạp và ném `UnsupportedClassVersionError`.

Quy tắc: **class file version = số phiên bản Java + 44**.

| JDK     | Class file version |
|---------|--------------------|
| Java 8  | 52.0               |
| Java 11 | 55.0               |
| Java 17 | 61.0               |
| Java 21 | 65.0               |

Áp vào thông báo lỗi:

- `class file version 65.0` → mã được biên dịch bởi **Java 21**.
- `only recognizes ... up to 61.0` → JVM đang chạy trong bước build chỉ là **Java 17**.

Nguồn tra cứu: đặc tả JVM (The Java Virtual Machine Specification, mục "The ClassFile Structure" và bảng major version) và tài liệu release notes của từng bản JDK trên Oracle/OpenJDK.

### 2.2. Lỗi xảy ra ở đâu trong pipeline?

Lỗi xảy ra khi JVM trên runner **nạp** các class đã được biên dịch bằng JDK mới hơn, ví dụ khi Gradle chạy build/test, hoặc khi chạy ứng dụng. Nghĩa là Java dùng để biên dịch (Java 21) và Java dùng để chạy trong bước build (Java 17) không đồng nhất.

### 2.3. Vì sao máy lập trình viên không lỗi nhưng GitHub Actions lỗi?

| | Máy lập trình viên | GitHub Actions runner |
|---|---|---|
| Môi trường | Cài sẵn JDK 21, cố định, do chính lập trình viên chọn | Máy ảo sạch, tạo mới cho mỗi lần chạy |
| Java được chọn bởi | JDK đã cài (`JAVA_HOME`) | Bước `actions/setup-java` trong `ci.yml` (hoặc Java mặc định của runner) |
| Biên dịch và chạy | Cùng một JDK 21 | Có thể là hai phiên bản khác nhau |

Khác biệt cốt lõi: trên máy local, phiên bản Java được đồng bộ ngầm vì chỉ có một JDK. Trên CI, phiên bản Java phải được **khai báo tường minh** trong `ci.yml`. Nếu khai báo này lệch với phiên bản mà dự án yêu cầu (ở đây CI đang dùng Java 17 trong khi code build bằng Java 21), lỗi sẽ xuất hiện.

## 3. Giải pháp

### 3.1. Vị trí cần rà soát trong `ci.yml`

Bước `actions/setup-java`, trường `java-version`:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout source
        uses: actions/checkout@v4

      - name: Set up Java 21
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'   # trước đây là '17' -> lệch với phiên bản biên dịch

      - name: Build with Gradle
        run: ./gradlew build
```

**Vì sao sửa như vậy khắc phục được lỗi:** `java-version: '21'` làm JVM trên runner là Java 21, hỗ trợ class file version tới 65.0, nên đọc được các class do Java 21 biên dịch và lỗi không còn.

### 3.2. Nguyên tắc để tránh lặp lại

1. **Một nguồn sự thật cho phiên bản Java.** Khai báo trong `build.gradle` bằng toolchain:
   ```groovy
   java {
       toolchain {
           languageVersion = JavaLanguageVersion.of(21)
       }
   }
   ```
   Phiên bản trong `ci.yml` phải khớp đúng giá trị này.
2. **Cho CI đọc phiên bản từ một file chung** (ví dụ `.java-version`) bằng tham số `java-version-file` của `actions/setup-java`, để chỉ phải sửa ở một nơi.
3. **Đổi Java thì đổi cả hai nơi trong cùng một Pull Request**: `build.gradle` và `ci.yml`, và để reviewer kiểm tra điều này.
4. **Pin phiên bản tường minh**, không dựa vào Java mặc định cài sẵn trên runner vì nó có thể thay đổi theo thời gian.
5. **Đồng bộ toàn bộ chuỗi**: máy local, CI, Dockerfile (ví dụ `eclipse-temurin:21`) và môi trường triển khai dùng cùng một phiên bản Java.

## 4. Kết luận

Nguyên nhân gốc rễ là sự **không đồng nhất giữa phiên bản Java dùng để biên dịch (Java 21, class file 65.0) và phiên bản Java cấu hình trong CI để chạy bước build (Java 17, tối đa 61.0)**. Cách khắc phục là đặt `java-version` trong `actions/setup-java` khớp với phiên bản của dự án, đồng thời quản lý phiên bản này tại một nơi duy nhất để lỗi không tái diễn.
