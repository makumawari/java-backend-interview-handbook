---
tags:
  - Spring
  - SpringBoot
  - AutoConfiguration
---

# Auto Configuration

> Phase: Phase 5 — Spring
> Chapter slug: `auto-configuration`

## Metadata

```yaml
Chapter: Auto Configuration
Phase: Phase 5 — Spring
Difficulty: ★★★★
Importance: ★★★★★
Interview Frequency: 75%
Prerequisites:
  - Chapter 06 — Configuration
Used Later:
  - Chapter 08 — Spring Boot
Estimated Reading: 18 phút
Estimated Practice: 15 phút
```

## Story

> Xem lại: [Chapter 06](06-configuration.md) — `@Configuration` + `@Bean` cho phép khai báo
> bean thủ công. Nhưng thêm `spring-boot-starter-data-jpa` + driver H2 vào `pom.xml`, một
> `DataSource` đã kết nối sẵn **tự nhiên xuất hiện** trong container — không ai viết
> `@Bean DataSource dataSource()` cả. Ai đã làm việc đó, và làm sao nó biết phải làm?

Kiểm tra file thật bên trong JAR `spring-boot-autoconfigure`:

```
$ unzip -p spring-boot-autoconfigure-3.3.4.jar \
    META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports \
    | grep -i DataSource

org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
org.springframework.boot.autoconfigure.jdbc.JndiDataSourceAutoConfiguration
...
```

Đây chính là một **danh sách** các class `@Configuration` — Spring Boot tự động **thử áp dụng
tất cả** danh sách này mỗi khi khởi động, và mỗi class tự quyết định "tôi có nên tạo bean của
mình hay không" dựa trên những gì đang có sẵn trên classpath và trong container.

## Interview Question (Central)

> Auto Configuration trong Spring Boot hoạt động như thế nào? Làm sao Spring Boot biết nên tạo
> bean nào và bỏ qua bean nào?

## Objectives

- [ ] Giải thích cơ chế `AutoConfiguration.imports` — danh sách các class tự động được thử áp
      dụng khi khởi động
- [ ] Tự tay chứng minh bằng thực nghiệm `@ConditionalOnMissingBean`: auto-configuration tự
      "lùi lại" (back off) khi người dùng đã tự khai báo bean của riêng mình
- [ ] Hiểu các annotation `@Conditional*` phổ biến khác (`@ConditionalOnClass`,
      `@ConditionalOnProperty`)

## Prerequisites

- Chapter 06 — hiểu `@Configuration`/`@Bean`, nền tảng để hiểu auto-configuration cũng chỉ là
  các class `@Configuration` đặc biệt.

## Used Later

- **Chapter 08 (Spring Boot)** — Auto Configuration là một trong ba trụ cột định nghĩa Spring
  Boot (cùng embedded server và starter dependency).

## Problem

Cấu hình thủ công mọi bean cho một ứng dụng Spring truyền thống (trước Spring Boot) đòi hỏi viết
rất nhiều `@Bean` lặp đi lặp lại giống hệt nhau giữa các dự án: `DataSource`, `EntityManagerFactory`,
`TransactionManager`, `ObjectMapper` (JSON), embedded server... — hầu hết đều có một cấu hình
"mặc định hợp lý" giống nhau ở đa số dự án, chỉ khác một vài tham số (URL database, tên
package). Yêu cầu lập trình viên viết lại từng dòng cấu hình này cho mỗi dự án mới là lãng phí
lặp lại không cần thiết.

## Concept

**Auto Configuration** là cơ chế Spring Boot tự động đăng ký một tập hợp lớn các class
`@Configuration` (khai báo trong file `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`) mỗi khi ứng dụng khởi động — mỗi class trong danh sách này dùng các
annotation `@Conditional*` (`@ConditionalOnClass`, `@ConditionalOnMissingBean`,
`@ConditionalOnProperty`, ...) để **tự quyết định** có nên tạo bean của mình hay không, dựa trên
những gì đang có trên classpath và những gì đã tồn tại trong container.

## Why?

Thiết kế "thử áp dụng tất cả, mỗi class tự quyết định có áp dụng hay không" (thay vì một cơ chế
trung tâm quyết định thay) cho phép Auto Configuration **mở rộng được** — bất kỳ thư viện bên
thứ ba nào cũng có thể cung cấp file `AutoConfiguration.imports` riêng, tự động được Spring Boot
gộp vào cùng cơ chế mà không cần sửa code lõi của Spring Boot. `@ConditionalOnMissingBean` cụ
thể giải quyết đúng vấn đề "tôn trọng lựa chọn của người dùng": nếu người dùng đã tự khai báo một
bean (ví dụ `DataSource` tuỳ chỉnh với cấu hình đặc biệt), auto-configuration tương ứng **lùi
lại**, không ghi đè lựa chọn đó — mặc định hợp lý chỉ áp dụng khi người dùng **chưa** có ý kiến
riêng.

## How?

```java
@AutoConfiguration
class GreeterAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean(Greeter.class) // CHI tao neu chua co Greeter bean nao khac
    Greeter defaultGreeter() { return new DefaultGreeter(); }
}
```

## Visualization

```
Khong co user config (Greeter chua ton tai):

  GreeterAutoConfiguration.defaultGreeter()
       │ @ConditionalOnMissingBean(Greeter.class): chua co -> DIEU KIEN THOA
       ▼
  DefaultGreeter duoc tao va dang ky

Co user config (Greeter DA duoc nguoi dung tu khai bao):

  UserConfig.myGreeter() -> CustomGreeter duoc tao TRUOC
       │
  GreeterAutoConfiguration.defaultGreeter()
       │ @ConditionalOnMissingBean(Greeter.class): DA CO roi -> DIEU KIEN KHONG THOA
       ▼
  defaultGreeter() KHONG DUOC GOI, auto-config "LUI LAI"
```

## Example

```java
interface Greeter { String greet(); }
static class DefaultGreeter implements Greeter { public String greet() { return "auto-configured"; } }
static class CustomGreeter implements Greeter { public String greet() { return "nguoi dung tu khai bao"; } }

@AutoConfiguration
static class GreeterAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean(Greeter.class)
    Greeter defaultGreeter() {
        System.out.println("  -> GreeterAutoConfiguration.defaultGreeter() DUOC GOI");
        return new DefaultGreeter();
    }
}

@Configuration
static class UserConfigWithCustomBean {
    @Bean
    Greeter myGreeter() {
        System.out.println("  -> UserConfig.myGreeter() DUOC GOI");
        return new CustomGreeter();
    }
}
```

Kết quả thật (Spring Boot 3.3.4, dùng `ApplicationContextRunner` để giả lập khởi động):

```
=== Khong co user config: auto-configuration TU DONG chay ===
  -> GreeterAutoConfiguration.defaultGreeter() DUOC GOI
Ket qua: Xin chao tu DefaultGreeter (auto-configured)

=== Co user config: auto-configuration TU LUI (back off) ===
  -> UserConfig.myGreeter() DUOC GOI
Ket qua: Xin chao tu CustomGreeter (nguoi dung tu khai bao)
So luong bean Greeter: 1
```

Khi không có cấu hình người dùng, `defaultGreeter()` được gọi bình thường. Khi có
`UserConfigWithCustomBean`, dòng `"GreeterAutoConfiguration.defaultGreeter() DUOC GOI"` **không
hề xuất hiện** — `@ConditionalOnMissingBean` đã ngăn method đó được gọi hoàn toàn (không chỉ là
"tạo ra rồi bỏ đi"), và cuối cùng container chỉ có **đúng một** bean `Greeter` — chính là bean do
người dùng khai báo.

Và bằng chứng thật về cơ chế danh sách `AutoConfiguration.imports` bên trong chính JAR của Spring
Boot:

```
$ unzip -p spring-boot-autoconfigure-3.3.4.jar \
    META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports | grep -i DataSource

org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
org.springframework.boot.autoconfigure.jdbc.JndiDataSourceAutoConfiguration
org.springframework.boot.autoconfigure.jdbc.XADataSourceAutoConfiguration
org.springframework.boot.autoconfigure.jdbc.DataSourceTransactionManagerAutoConfiguration
org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration
```

Đây chính là cách `DataSourceAutoConfiguration` được Spring Boot "biết" để thử áp dụng — nó nằm
sẵn trong một file danh sách text đơn giản, được đóng gói cùng JAR thư viện.

## Deep Dive

**`@ConditionalOnMissingBean` được đánh giá theo thứ tự nào — vì sao thứ tự này quan trọng?**
Spring Boot đảm bảo mọi auto-configuration của **chính Spring Boot** (và các starter chính
thức) được đánh giá **sau cùng**, sau khi tất cả `@Configuration` class do người dùng khai báo
đã được xử lý — đây là lý do `@ConditionalOnMissingBean` trong auto-configuration luôn "nhìn
thấy" đúng bean người dùng đã khai báo (nếu có) trước khi tự quyết định có tạo bean của mình hay
không. Nếu thứ tự bị đảo ngược (auto-configuration chạy trước), `@ConditionalOnMissingBean` sẽ
luôn thấy "chưa có bean nào" và tạo bean mặc định trước, khiến bean của người dùng (tạo sau) trở
thành bean **thừa/trùng lặp** thay vì bean chính. Cơ chế `@AutoConfigureAfter`/`@AutoConfigureBefore` (khai báo thứ tự tương đối giữa các auto-configuration với nhau) tồn tại để xử lý các
trường hợp phức tạp hơn khi nhiều auto-configuration phụ thuộc lẫn nhau (ví dụ
`HibernateJpaAutoConfiguration` phải chạy sau `DataSourceAutoConfiguration`, vì JPA cần
`DataSource` đã tồn tại).

## Engineering Insight

**Vì sao hiểu Auto Configuration lại quan trọng khi debug một ứng dụng Spring Boot "không hoạt
động như mong đợi"?** Triệu chứng phổ biến nhất: một bean nào đó "không được tạo" dù bạn nghĩ nó
phải tự động có, hoặc một bean tuỳ chỉnh bị "bỏ qua" vì có bean auto-configured trùng loại xuất
hiện trước. Spring Boot cung cấp công cụ chẩn đoán chính thức cho đúng vấn đề này: chạy ứng dụng
với `--debug` (hoặc `debug=true` trong `application.properties`) in ra một **báo cáo đầy đủ**
liệt kê từng auto-configuration class, lý do nó **được áp dụng** (matched) hoặc **bị bỏ qua**
(not matched) kèm điều kiện cụ thể nào không thoả — đây là công cụ debug đầu tiên nên dùng khi
nghi ngờ vấn đề liên quan tới auto-configuration, thay vì đoán mò qua thử-sai.

## Historical Note

Trước Spring Boot 2.7 (2022), danh sách auto-configuration được khai báo trong file
`META-INF/spring.factories` (định dạng key-value tổng quát, dùng chung cho nhiều loại
"factories" khác nhau của Spring, không riêng auto-configuration). Từ Spring Boot 2.7, danh sách
này tách riêng thành file `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (định dạng đơn giản hơn — chỉ là danh sách tên class, mỗi dòng một class) — thay
đổi này giúp tách biệt rõ ràng "auto-configuration" khỏi các loại "factories" khác, và tăng tốc
độ xử lý lúc khởi động.

## Myth vs Reality

- **Myth:** "Auto Configuration là một hộp đen bí ẩn, không thể kiểm soát hay debug được."
  **Reality:** Đã chứng minh — nó chỉ là danh sách class `@Configuration` thông thường trong một
  file text đơn giản, dùng các annotation `@Conditional*` công khai, có thể đọc trực tiếp trong
  JAR và debug qua báo cáo `--debug` chính thức.

- **Myth:** "Auto Configuration luôn ghi đè cấu hình của người dùng."
  **Reality:** Đã chứng minh ngược lại bằng thực nghiệm — `@ConditionalOnMissingBean` đảm bảo
  auto-configuration **tôn trọng** và lùi lại trước cấu hình tường minh của người dùng.

## Common Mistakes

- **Không biết dùng `--debug` để chẩn đoán vấn đề liên quan tới auto-configuration** — mất nhiều
  thời gian đoán mò thay vì đọc báo cáo chính thức.
- **Định nghĩa bean tuỳ chỉnh nhưng đặt sai loại/tên khiến `@ConditionalOnMissingBean` (dựa trên
  type) không nhận diện đúng, dẫn tới cả hai bean (auto-configured và tuỳ chỉnh) cùng tồn tại**
  gây lỗi `NoUniqueBeanDefinitionException` khi Spring không biết tiêm bean nào.
- **Viết auto-configuration tuỳ chỉnh (cho thư viện nội bộ công ty) nhưng quên
  `@ConditionalOnMissingBean`** — luôn ghi đè lựa chọn của người dùng, vi phạm đúng nguyên tắc
  cốt lõi của cơ chế này.

## Debug Checklist

- [ ] Một bean auto-configured không xuất hiện dù đã thêm đúng dependency? → chạy với `--debug`,
      xem báo cáo "Negative matches" để biết điều kiện nào không thoả.
- [ ] Lỗi `NoUniqueBeanDefinitionException` giữa bean tự khai báo và bean auto-configured? →
      kiểm tra bean tự khai báo có đúng type mà `@ConditionalOnMissingBean` kiểm tra không.
- [ ] Cần tìm danh sách đầy đủ auto-configuration của một starter? → mở JAR
      `spring-boot-autoconfigure`, đọc file `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`.

## Summary

Auto Configuration hoạt động bằng cách thử áp dụng một danh sách lớn class `@Configuration`
(khai báo trong `AutoConfiguration.imports`, đã xác nhận bằng cách đọc trực tiếp trong JAR) mỗi
khi khởi động — mỗi class tự quyết định có tạo bean hay không qua các annotation
`@Conditional*`. Đã chứng minh bằng thực nghiệm: `@ConditionalOnMissingBean` khiến
auto-configuration **lùi lại hoàn toàn** (method không được gọi) khi người dùng đã tự khai báo
bean cùng loại — tôn trọng lựa chọn tường minh của người dùng thay vì ghi đè nó. Cơ chế `--debug`
cung cấp báo cáo đầy đủ lý do mỗi auto-configuration được áp dụng hay bị bỏ qua, là công cụ chẩn
đoán chính thức khi hành vi không như mong đợi.

## Interview Questions

**Mid**

- Auto Configuration trong Spring Boot hoạt động như thế nào?
- `@ConditionalOnMissingBean` dùng để làm gì? Cho ví dụ tình huống cần dùng nó.

**Senior**

- Làm sao để debug khi một bean auto-configured không xuất hiện như mong đợi?
- Giải thích vì sao thứ tự đánh giá (auto-configuration chạy sau cấu hình người dùng) lại quan
  trọng đối với tính đúng đắn của `@ConditionalOnMissingBean`.

## Exercises

- [ ] Chạy lại `AutoConfigTest` ở trên, xác nhận auto-configuration lùi lại khi có bean người
      dùng, đúng như mô tả.
- [ ] Chạy một ứng dụng Spring Boot thật với `--debug`, đọc báo cáo "CONDITIONS EVALUATION
      REPORT", tìm một auto-configuration bị "not matched" và xác định lý do.
- [ ] Mở JAR `spring-boot-autoconfigure` (bất kỳ phiên bản nào có trong `~/.m2`), tìm file
      `AutoConfiguration.imports`, đếm tổng số auto-configuration class có sẵn.

## Cheat Sheet

| Annotation | Điều kiện |
| --- | --- |
| `@ConditionalOnClass(X.class)` | Chỉ áp dụng nếu class `X` có trên classpath |
| `@ConditionalOnMissingBean(X.class)` | Chỉ áp dụng nếu **chưa** có bean kiểu `X` trong container |
| `@ConditionalOnBean(X.class)` | Chỉ áp dụng nếu **đã** có bean kiểu `X` |
| `@ConditionalOnProperty(...)` | Chỉ áp dụng nếu một property cấu hình có giá trị nhất định |
| `@ConditionalOnMissingClass(...)` | Chỉ áp dụng nếu class **không** có trên classpath |

## References

- Spring Boot Documentation — Auto-configuration: https://docs.spring.io/spring-boot/reference/using/auto-configuration.html
- Spring Boot Documentation — Creating Your Own Auto-configuration.
