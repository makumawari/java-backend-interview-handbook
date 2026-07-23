---
tags:
  - JPA
  - MultiDataSource
  - SpringBoot
---

# Multi-Datasource Configuration

> Phase: Phase 6 — Persistence
> Chapter slug: `multi-datasource`

## Metadata

```yaml
Chapter: Multi-Datasource Configuration
Phase: Phase 6 — Persistence
Difficulty: ★★★★
Importance: ★★★
Interview Frequency: 50%
Prerequisites:
  - Chapter 01 — JPA
  - Phase 5, Chapter 07 — Auto Configuration
Used Later:
  - Kiến trúc tích hợp nhiều hệ thống dữ liệu (Phase 8)
Estimated Reading: 15 phút
Estimated Practice: 12 phút
```

## Story

> Xem lại: Phase 5, Chapter 07 (Auto Configuration) — mặc định, Spring Boot tự động tạo **một**
> `DataSource` duy nhất từ `spring.datasource.*`. Nhưng một ứng dụng thực tế có thể cần đọc từ
> một database giao dịch (transactional) **và** ghi vào một database báo cáo (reporting) hoàn
> toàn tách biệt — hai kết nối độc lập, trong cùng một ứng dụng.

Cấu hình hai `DataSource` tường minh, mỗi cái trỏ tới một H2 in-memory database khác nhau, rồi
xác nhận chúng thực sự tách biệt:

```
ordersDataSource JDBC URL: jdbc:h2:mem:orders_db
reportingDataSource JDBC URL: jdbc:h2:mem:reporting_db

=== Truy van bang 'local_order' TREN orders_db ===
[{ID=1, NOTE=don hang tu orders_db}]
=== Truy van bang 'summary' TREN reporting_db ===
[{ID=1, NOTE=bao cao tu reporting_db}]
=== Thu truy van bang 'summary' TREN orders_db (SE LOI, vi bang nay o DB KHAC) ===
Loi nhu du kien: BadSqlGrammarException
```

Hai HikariCP pool riêng biệt (`HikariPool-1` cho `orders_db`, `HikariPool-2` cho
`reporting_db`) — dữ liệu hoàn toàn cô lập, cố truy vấn bảng của database này từ kết nối của
database kia gây lỗi ngay lập tức.

## Interview Question (Central)

> How do you connect two different databases in a single Spring Boot application?

## Objectives

- [ ] Cấu hình tường minh nhiều `DataSource` trong cùng một ứng dụng Spring Boot, tránh xung đột
      với Auto Configuration mặc định
- [ ] Tự tay chứng minh bằng thực nghiệm: hai `DataSource` tạo ra hai connection pool độc lập,
      dữ liệu hoàn toàn cô lập
- [ ] Hiểu vai trò của `@Primary` và `@Qualifier` khi có nhiều bean cùng kiểu

## Prerequisites

- Chapter 01 — hiểu JPA/Hibernate cần một `DataSource` để hoạt động.
- Phase 5, Chapter 07 — hiểu Auto Configuration tự động tạo `DataSource` mặc định từ
  `spring.datasource.*`, và cơ chế `@ConditionalOnMissingBean` cho phép ghi đè nó.

## Used Later

- **Kiến trúc tích hợp nhiều hệ thống dữ liệu** (Phase 8) — nền tảng cho các ứng dụng cần giao
  tiếp với nhiều database, hoặc microservice cần đọc dữ liệu chia sẻ từ một database khác ngoài
  database chính của chính nó.

## Problem

Auto Configuration (Phase 5, Chapter 07) mặc định chỉ tạo **một** `DataSource` từ đúng một tập
thuộc tính `spring.datasource.*` — hoàn toàn không có khái niệm "nhiều database" tích hợp sẵn.
Khi một ứng dụng thực sự cần giao tiếp với nhiều database độc lập (transactional DB + reporting
DB, hoặc tích hợp với database của một hệ thống khác), cần **tự tay** cấu hình tường minh, đồng
thời tránh xung đột với `DataSourceAutoConfiguration` mặc định (dễ gây lỗi "không biết bean nào
là primary" khi Spring cần tiêm `DataSource` cho một chỗ không chỉ định rõ).

## Concept

Cấu hình nhiều `DataSource` đòi hỏi: (1) tắt hoặc ghi đè `DataSourceAutoConfiguration` mặc định
bằng cách tự khai báo `@Bean DataSource` tường minh (Spring tự động lùi lại nhờ
`@ConditionalOnMissingBean`, đã học ở Phase 5 Chapter 07); (2) đánh dấu một trong các
`DataSource` là **`@Primary`** (dùng mặc định khi không chỉ định rõ); (3) dùng **`@Qualifier`**
để chỉ định chính xác `DataSource`/`JdbcTemplate`/`EntityManagerFactory` nào cần tiêm vào từng
nơi cụ thể.

## Why?

`@Primary` giải quyết đúng vấn đề khi có **nhiều bean cùng kiểu** (`DataSource`) — nếu không chỉ
định, Spring không biết nên tiêm bean nào cho một điểm tiêm chỉ khai báo kiểu `DataSource` chung
chung, dẫn tới lỗi `NoUniqueBeanDefinitionException`. `@Qualifier` cho phép **vượt qua** quy tắc
mặc định đó tại từng điểm tiêm cụ thể, chỉ định rõ ràng "tôi muốn chính xác bean tên này" — cùng
triết lý tường minh hơn ngầm định đã gặp nhiều lần xuyên suốt Phase 5 (constructor injection rõ
ràng hơn field injection, `@Query` tường minh hơn derived query method khi cần kiểm soát chi
tiết).

## How?

```java
@Configuration
class MultiDataSourceConfig {
    @Primary
    @Bean
    @ConfigurationProperties("app.datasource.orders")
    DataSource ordersDataSource() { return DataSourceBuilder.create().build(); }

    @Bean
    @ConfigurationProperties("app.datasource.reporting")
    DataSource reportingDataSource() { return DataSourceBuilder.create().build(); }

    @Primary
    @Bean
    JdbcTemplate ordersJdbcTemplate(@Qualifier("ordersDataSource") DataSource ds) {
        return new JdbcTemplate(ds);
    }

    @Bean
    JdbcTemplate reportingJdbcTemplate(@Qualifier("reportingDataSource") DataSource ds) {
        return new JdbcTemplate(ds);
    }
}
```

```properties
app.datasource.orders.jdbc-url=jdbc:h2:mem:orders_db
app.datasource.reporting.jdbc-url=jdbc:h2:mem:reporting_db
```

## Visualization

```
Ung dung Spring Boot
     │
     ├── ordersDataSource (@Primary)  ──► HikariPool-1 ──► orders_db
     │        │
     │   ordersJdbcTemplate (@Primary)
     │
     └── reportingDataSource (@Qualifier)  ──► HikariPool-2 ──► reporting_db
              │
         reportingJdbcTemplate (@Qualifier)

Hai connection pool HOAN TOAN doc lap, khong chia se ket noi, khong chia se du lieu.
```

## Example

```java
@Autowired @Qualifier("ordersDataSource") DataSource ordersDs;
@Autowired @Qualifier("reportingDataSource") DataSource reportingDs;
@Autowired @Qualifier("ordersJdbcTemplate") JdbcTemplate ordersJdbc;
@Autowired @Qualifier("reportingJdbcTemplate") JdbcTemplate reportingJdbc;

try (Connection c1 = ordersDs.getConnection(); Connection c2 = reportingDs.getConnection()) {
    System.out.println("ordersDataSource JDBC URL: " + c1.getMetaData().getURL());
    System.out.println("reportingDataSource JDBC URL: " + c2.getMetaData().getURL());
}

ordersJdbc.execute("create table if not exists local_order (id bigint, note varchar(100))");
ordersJdbc.update("insert into local_order (id, note) values (1, 'don hang tu orders_db')");

reportingJdbc.execute("create table if not exists summary (id bigint, note varchar(100))");
reportingJdbc.update("insert into summary (id, note) values (1, 'bao cao tu reporting_db')");
```

Kết quả thật (Spring Boot 3.3.4, HikariCP):

```
HikariPool-1 - Added connection conn0: url=jdbc:h2:mem:orders_db user=
HikariPool-2 - Added connection conn10: url=jdbc:h2:mem:reporting_db user=

ordersDataSource JDBC URL: jdbc:h2:mem:orders_db
reportingDataSource JDBC URL: jdbc:h2:mem:reporting_db

=== Truy van bang 'local_order' TREN orders_db ===
[{ID=1, NOTE=don hang tu orders_db}]
=== Truy van bang 'summary' TREN reporting_db ===
[{ID=1, NOTE=bao cao tu reporting_db}]

=== Thu truy van bang 'summary' TREN orders_db (SE LOI, vi bang nay o DB KHAC) ===
Loi nhu du kien: BadSqlGrammarException
```

Hai `HikariPool` (`HikariPool-1`, `HikariPool-2`) khởi tạo độc lập, kết nối tới đúng hai database
đã cấu hình. Bảng `local_order` chỉ tồn tại trong `orders_db`, bảng `summary` chỉ tồn tại trong
`reporting_db` — cố truy vấn bảng `summary` **qua kết nối `ordersJdbc`** (thuộc `orders_db`) gây
lỗi `BadSqlGrammarException` ngay lập tức, xác nhận hai database hoàn toàn tách biệt, không chia
sẻ schema.

## Deep Dive

**Vì sao dùng full JPA (nhiều `EntityManagerFactory`) cho nhiều database phức tạp hơn hẳn so với
`JdbcTemplate` như ví dụ trên?** Ví dụ trên dùng `JdbcTemplate` (JDBC thuần, không qua ORM) —
đơn giản vì chỉ cần cấu hình `DataSource`. Với JPA đầy đủ (Entity, Repository, Chapter 02-03),
mỗi database cần **riêng biệt hoàn toàn**: một `LocalContainerEntityManagerFactoryBean` riêng
(trỏ đúng `DataSource` và **đúng package** chứa Entity cho database đó), một
`PlatformTransactionManager` riêng (transaction của database này **không thể** tự động bao gồm
database kia — JPA/Hibernate không hỗ trợ transaction phân tán xuyên hai database mà không có
thêm cơ chế XA/JTA phức tạp), và một `@EnableJpaRepositories(basePackages = ..., entityManagerFactoryRef = ..., transactionManagerRef = ...)` riêng cho từng nhóm Repository. Đây là lý do cấu
hình multi-datasource với JPA đầy đủ phức tạp hơn đáng kể so với JDBC thuần — mỗi tầng (DataSource,
EntityManagerFactory, TransactionManager, Repository scanning) đều cần tách riêng tường minh cho
từng database.

## Engineering Insight

**Vì sao một transaction `@Transactional` (Phase 5, Chapter 13) không tự động bao gồm cả hai
database, và điều này ảnh hưởng gì tới thiết kế nghiệp vụ?** `PlatformTransactionManager` mặc
định (`DataSourceTransactionManager`/`JpaTransactionManager`) chỉ quản lý transaction cho
**đúng một** `DataSource`/`EntityManagerFactory` mà nó được gắn vào — nếu một phương thức
`@Transactional` ghi dữ liệu vào cả `orders_db` và `reporting_db`, và có lỗi xảy ra **sau khi**
đã ghi vào `orders_db` nhưng **trước khi** ghi xong `reporting_db`, chỉ có `orders_db` được
rollback đúng — `reporting_db` (nếu đã commit riêng trước đó, tuỳ thứ tự code) **không hề** được
hoàn tác, dẫn tới dữ liệu không nhất quán **giữa hai database**. Giải quyết đúng đắn vấn đề này
đòi hỏi JTA (Java Transaction API, quản lý transaction phân tán qua nhiều resource) — phức tạp
và tốn hiệu năng hơn đáng kể so với transaction cục bộ — nên trong thực tế, kiến trúc microservice
hiện đại (Phase 8) thường **tránh** thiết kế yêu cầu transaction xuyên nhiều database, thay vào
đó dùng các mẫu hình như Saga pattern hoặc eventual consistency.

## Historical Note

Cấu hình đa `DataSource` qua `@ConfigurationProperties` (thay vì hardcode giá trị hoặc dùng XML)
trở thành cách tiếp cận chuẩn từ khi Spring Boot phổ biến property binding tự động (từ Spring
Boot 1.x, 2014) — trước đó, cấu hình nhiều nguồn dữ liệu trong Spring XML truyền thống dài dòng
và dễ lỗi hơn nhiều.

## Myth vs Reality

- **Myth:** "Chỉ cần khai báo hai `@Bean DataSource` là đủ, Spring tự biết cách xử lý mọi thứ
  còn lại."
  **Reality:** Cần thêm `@Primary` để tránh lỗi `NoUniqueBeanDefinitionException`, và với JPA
  đầy đủ còn cần cấu hình riêng `EntityManagerFactory`/`TransactionManager`/repository scanning
  cho từng database (xem Deep Dive).

- **Myth:** "`@Transactional` tự động đảm bảo tính nhất quán khi ghi vào nhiều database trong
  cùng một phương thức."
  **Reality:** Đã giải thích ở Engineering Insight — mỗi `TransactionManager` chỉ quản lý đúng
  một database; cần JTA cho transaction phân tán thực sự, hoặc tránh thiết kế yêu cầu điều này.

## Common Mistakes

- **Quên `@Primary` khi có nhiều `DataSource`** — gây lỗi khởi động
  `NoUniqueBeanDefinitionException` ở bất kỳ nơi nào tiêm `DataSource` không chỉ định
  `@Qualifier`.
- **Giả định `@Transactional` bao phủ cả hai database** — dẫn tới dữ liệu không nhất quán khi có
  lỗi xảy ra giữa chừng (xem Engineering Insight).
- **Dùng sai tên thuộc tính khi bind `@ConfigurationProperties` cho `DataSource` tự tạo qua
  `DataSourceBuilder`** — cần đúng `jdbc-url` (khớp trực tiếp với `HikariConfig`), không phải
  `url` (chỉ đúng khi dùng `spring.datasource.url` chuẩn của Spring Boot `DataSourceProperties`).

## Best Practices

- Luôn đánh dấu một `DataSource` là `@Primary` khi có nhiều hơn một trong cùng ứng dụng.
- Dùng `@Qualifier` tường minh tại mọi điểm tiêm không phải `DataSource` primary.
- Tránh thiết kế nghiệp vụ yêu cầu transaction nhất quán xuyên nhiều database trong cùng một
  phương thức — cân nhắc Saga pattern hoặc eventual consistency (Phase 8) thay vì JTA phức tạp.
- Với JPA đầy đủ, tách riêng package Entity/Repository cho từng database để cấu hình
  `@EnableJpaRepositories(basePackages=...)` rõ ràng, dễ bảo trì.

## Debug Checklist

- [ ] Lỗi `NoUniqueBeanDefinitionException` khi khởi động? → kiểm tra đã đánh dấu `@Primary`
      cho đúng một `DataSource`/bean liên quan chưa.
- [ ] Dữ liệu không nhất quán giữa hai database sau khi có lỗi xảy ra giữa chừng? → xác nhận
      nghi ngờ đây là hệ quả của việc `@Transactional` chỉ quản lý một database (xem Engineering
      Insight).
- [ ] `BadSqlGrammarException`/bảng không tồn tại dù chắc chắn đã tạo? → kiểm tra đang dùng đúng
      `JdbcTemplate`/`DataSource` (đúng `@Qualifier`) tương ứng với database chứa bảng đó không.

## Summary

Cấu hình nhiều `DataSource` trong Spring Boot đòi hỏi tự khai báo tường minh (Spring tự lùi lại
Auto Configuration mặc định), đánh dấu một cái là `@Primary`, dùng `@Qualifier` cho các điểm
tiêm còn lại. Đã chứng minh bằng thực nghiệm: hai `DataSource` (`orders_db`, `reporting_db`) tạo
ra hai `HikariPool` độc lập, dữ liệu hoàn toàn cô lập — cố truy vấn bảng của database này qua kết
nối của database kia gây lỗi `BadSqlGrammarException` ngay lập tức. Với JPA đầy đủ (không chỉ
JDBC thuần), mỗi database cần thêm `EntityManagerFactory`/`TransactionManager`/repository
scanning riêng. `@Transactional` không tự động bao phủ nhiều database — cần JTA cho transaction
phân tán thực sự, hoặc tránh thiết kế yêu cầu điều này trong kiến trúc hiện đại.

## Interview Questions

- How do you connect two different databases in a single Spring Boot application?

**Senior**

- Vì sao cần `@Primary` khi cấu hình nhiều `DataSource`? Điều gì xảy ra nếu thiếu nó?
- Một phương thức `@Transactional` ghi dữ liệu vào hai database khác nhau — điều gì xảy ra nếu
  có lỗi giữa chừng? Đề xuất giải pháp.

## Exercises

- [ ] Chạy lại `MultiDataSourceTest` ở trên, xác nhận hai database hoàn toàn tách biệt đúng như
      mô tả.
- [ ] Mở rộng ví dụ thành full JPA: thêm `LocalContainerEntityManagerFactoryBean` và
      `PlatformTransactionManager` riêng cho `reporting_db`, cấu hình
      `@EnableJpaRepositories` với `basePackages` tách biệt.
- [ ] Viết một test cố tình gây lỗi giữa chừng khi ghi vào cả hai database trong cùng một
      phương thức, quan sát trực tiếp hiện tượng dữ liệu không nhất quán đã mô tả ở Engineering
      Insight.

## Cheat Sheet

| Bước | Mục đích |
| --- | --- |
| `@Bean DataSource` tường minh | Ghi đè Auto Configuration mặc định (chỉ tạo 1 DataSource) |
| `@Primary` | Chỉ định bean mặc định khi không có `@Qualifier` |
| `@Qualifier("beanName")` | Chỉ định chính xác bean nào cần tiêm |
| `@ConfigurationProperties("app.datasource.x")` | Bind cấu hình từ properties vào `DataSourceBuilder` |
| (JPA đầy đủ) `@EnableJpaRepositories(basePackages=..., entityManagerFactoryRef=..., transactionManagerRef=...)` | Tách repository scanning theo từng database |

## References

- Spring Boot Documentation — Configure Two DataSources: https://docs.spring.io/spring-boot/how-to/data-access.html
- Spring Framework Documentation — `@Primary`, `@Qualifier`.
