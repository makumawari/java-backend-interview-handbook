---
tags:
  - Spring
  - SpringBoot
---

# Spring Boot

> Phase: Phase 5 — Spring
> Chapter slug: `spring-boot`

## Metadata

```yaml
Chapter: Spring Boot
Phase: Phase 5 — Spring
Difficulty: ★★★
Importance: ★★★★★
Interview Frequency: 85%
Prerequisites:
  - Chapter 07 — Auto Configuration
Used Later:
  - Chapter 09 — REST
  - Chapter 17 — Security
Estimated Reading: 15 phút
Estimated Practice: 12 phút
```

## Story

> Xem lại: [Chapter 07](07-auto-configuration.md) — Auto Configuration là một trong ba trụ cột
> của Spring Boot. Nhưng có một câu hỏi thực tế hơn: chạy `mvn spring-boot:run`, không cài
> Tomcat riêng, không deploy file WAR vào bất kỳ server nào — làm sao ứng dụng lại phục vụ HTTP
> request được ngay lập tức?

Đóng gói ứng dụng thành file JAR rồi kiểm tra **bên trong** nó:

```
$ unzip -l target/demo-0.0.1-SNAPSHOT.jar | grep tomcat-embed-core
  3556312  ...  BOOT-INF/lib/tomcat-embed-core-10.1.30.jar
```

Chính **Tomcat** — cùng server đã chạy hàng triệu ứng dụng Java EE truyền thống — nằm sẵn **bên
trong** file JAR của ứng dụng, dưới dạng một thư viện thông thường. Khởi động ứng dụng:

```
Tomcat initialized with port 8080 (http)
Tomcat started on port 8080 (http) with context path '/'
Started DemoApplication in 1.147 seconds (process running for 1.256)
```

Ứng dụng tự khởi động Tomcat như một phần của chính nó — không cần cài đặt, cấu hình, hay deploy
riêng biệt.

## Interview Question (Central)

> Spring Boot khác Spring Framework như thế nào? Ba trụ cột chính của Spring Boot là gì?

## Objectives

- [ ] Phân biệt rõ Spring Framework (lõi) và Spring Boot (lớp tiện ích xây trên lõi)
- [ ] Tự tay chứng minh bằng thực nghiệm: Tomcat được đóng gói **bên trong** file JAR của ứng
      dụng (embedded server)
- [ ] Giải thích `@SpringBootApplication` thực chất là tổ hợp của 3 annotation khác

## Prerequisites

- Chapter 07 — hiểu Auto Configuration, một trong ba trụ cột của Spring Boot.

## Used Later

- **Chapter 09 (REST)** — REST API xây trên nền Spring Boot với embedded server đã học ở
  chapter này.
- **Chapter 17 (Security)** — Spring Security tích hợp tự động qua chính cơ chế Auto
  Configuration + Spring Boot.

## Problem

Triển khai một ứng dụng Java EE truyền thống (trước Spring Boot) đòi hỏi: cài đặt server (Tomcat/
JBoss/WebLogic) riêng biệt trên máy chủ, đóng gói ứng dụng thành file WAR, deploy WAR vào server
đó, rồi cấu hình khớp giữa phiên bản server và ứng dụng. Quy trình này cồng kềnh, dễ xảy ra lỗi
"chạy được trên máy tôi nhưng không chạy trên server" do khác biệt môi trường/phiên bản giữa máy
phát triển và server triển khai thực tế.

## Concept

**Spring Boot** là một lớp tiện ích xây dựng trên Spring Framework, giải quyết đúng vấn đề triển
khai bằng ba trụ cột: **Embedded Server** (Tomcat/Jetty/Undertow được đóng gói ngay bên trong
JAR ứng dụng, không cần cài server riêng), **Auto Configuration** (Chapter 07, tự động cấu hình
hợp lý dựa trên classpath), và **Starter Dependencies** (các gói phụ thuộc gộp sẵn, ví dụ
`spring-boot-starter-web` kéo theo toàn bộ thư viện cần thiết cho web app mà không cần khai báo
từng thư viện riêng lẻ).

## Why?

Đóng gói server vào cùng JAR ứng dụng (thay vì deploy vào server cài sẵn) đảo ngược mô hình
triển khai truyền thống: thay vì "một server, nhiều ứng dụng deploy vào đó" (dễ xung đột phiên
bản giữa các ứng dụng dùng chung server), Spring Boot theo mô hình "mỗi ứng dụng tự mang server
riêng của mình" — đúng triết lý phù hợp với container hoá (Docker) và microservice: mỗi service
là một tiến trình độc lập hoàn toàn, `java -jar app.jar` là đủ để chạy, không phụ thuộc bất kỳ
phần mềm nào đã cài sẵn trên máy chủ ngoài JVM.

## How?

```java
@SpringBootApplication // = @Configuration + @ComponentScan + @EnableAutoConfiguration
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

```bash
mvn package                          # dong goi thanh 1 file JAR duy nhat
java -jar target/demo-0.0.1-SNAPSHOT.jar   # chay, KHONG can cai Tomcat rieng
```

## Visualization

```
Java EE truyen thong (WAR):              Spring Boot (JAR + embedded server):

  Server (Tomcat) cai san tren may        app.jar
       │                                     ├── BOOT-INF/classes/  (code cua ban)
       ▼ deploy vao                          └── BOOT-INF/lib/
  app.war                                        ├── tomcat-embed-core.jar  <- Tomcat O DAY
       │                                         └── ... (moi thu vien khac)
  Server phai duoc cai, cau hinh
  KHOP phien ban voi ung dung TRUOC       java -jar app.jar
  khi deploy duoc                         => Tomcat TU KHOI DONG BEN TRONG chinh JVM nay,
                                              khong can cai gi them ngoai JVM
```

## Example

```java
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

Đóng gói và kiểm tra nội dung JAR:

```
$ mvn package
$ unzip -l target/demo-0.0.1-SNAPSHOT.jar | grep tomcat-embed-core
  3556312  ...  BOOT-INF/lib/tomcat-embed-core-10.1.30.jar
```

Khởi động:

```
$ java -jar target/demo-0.0.1-SNAPSHOT.jar
...
Tomcat initialized with port 8080 (http)
Tomcat started on port 8080 (http) with context path '/'
Started DemoApplication in 1.147 seconds (process running for 1.256)
```

`tomcat-embed-core-10.1.30.jar` nằm thật sự bên trong file JAR ứng dụng (thư mục
`BOOT-INF/lib/`), xác nhận Tomcat là một **thư viện được đóng gói cùng**, không phải một server
cài đặt riêng biệt trên máy. Ứng dụng khởi động và tự phục vụ HTTP chỉ với `java -jar`, sẵn sàng
sau **1.147 giây**.

## Deep Dive

**`@SpringBootApplication` thực chất "giải nén" thành những gì?** Đây là một
meta-annotation (Phase 1, Chapter 18) tổ hợp của 3 annotation:

```java
@SpringBootConfiguration // ban chat la @Configuration, danh dau day la class cau hinh chinh
@ComponentScan          // tu dong quet package hien tai + cac package con de tim @Component
@EnableAutoConfiguration // kich hoat co che Auto Configuration (Chapter 07)
public @interface SpringBootApplication { ... }
```

Việc gộp 3 annotation thành một giúp class chính (`DemoApplication`) chỉ cần một dòng annotation
duy nhất để kích hoạt đầy đủ: (1) bản thân nó là một `@Configuration`, (2) tự động quét mọi
`@Component` trong package của nó và các package con, (3) kích hoạt toàn bộ cơ chế Auto
Configuration đã học ở Chapter 07. Đây là lý do class chứa `@SpringBootApplication` theo quy
ước **nên đặt ở package gốc** (root package) của ứng dụng — `@ComponentScan` mặc định chỉ quét
package của chính class đó trở xuống, đặt sai vị trí sẽ khiến nhiều bean không được phát hiện.

## Engineering Insight

**Fat JAR ("uber JAR") hoạt động thế nào để một file `.jar` duy nhất chứa được cả code lẫn hàng
trăm thư viện phụ thuộc, trong khi cấu trúc JAR tiêu chuẩn của Java không hỗ trợ JAR lồng trong
JAR?** Định dạng JAR tiêu chuẩn (dựa trên ZIP) và cơ chế classloader mặc định của JVM **không hỗ
trợ trực tiếp** việc nạp class từ một JAR nằm lồng bên trong JAR khác. Spring Boot giải quyết
bằng `spring-boot-maven-plugin`: đóng gói theo cấu trúc đặc biệt (`BOOT-INF/classes/` cho code
của bạn, `BOOT-INF/lib/` cho từng JAR phụ thuộc **nguyên vẹn**, không giải nén), rồi cung cấp một
**custom `ClassLoader`** (`LaunchedURLClassLoader`) biết cách đọc trực tiếp từ các JAR lồng bên
trong mà không cần giải nén ra đĩa trước — toàn bộ cơ chế này được kích hoạt tự động qua
`Main-Class` đặc biệt (`JarLauncher`) khai báo trong `MANIFEST.MF` của fat JAR.

## Historical Note

Spring Boot ra mắt năm 2014 (Spring Boot 1.0), do đội ngũ Pivotal phát triển — ra đời trong bối
cảnh microservice bắt đầu trở thành xu hướng kiến trúc chủ đạo, nơi mô hình "một server dùng
chung nhiều ứng dụng" của Java EE truyền thống trở nên bất tiện (mỗi microservice cần triển khai
độc lập). Embedded server (ban đầu chỉ hỗ trợ Tomcat, sau mở rộng Jetty/Undertow) chính là tính
năng cốt lõi giải quyết đúng nhu cầu đó ngay từ phiên bản đầu tiên.

## Myth vs Reality

- **Myth:** "Spring Boot là một framework hoàn toàn khác, thay thế Spring Framework."
  **Reality:** Spring Boot xây dựng **trên** Spring Framework (dùng `ApplicationContext`, DI,
  AOP y hệt Chapter 02-03, 12), không thay thế nó — Spring Boot chỉ thêm auto-configuration,
  embedded server, và starter dependencies để giảm công sức thiết lập ban đầu.

- **Myth:** "Cần cài Tomcat trên máy chủ trước khi triển khai một ứng dụng Spring Boot."
  **Reality:** Đã chứng minh bằng thực nghiệm — Tomcat nằm sẵn bên trong JAR ứng dụng, chỉ cần
  JVM cài sẵn trên máy chủ, không cần cài thêm bất kỳ server nào khác.

## Common Mistakes

- **Đặt class `@SpringBootApplication` không ở package gốc** — `@ComponentScan` ngầm định chỉ
  quét từ package đó trở xuống, bean ở package "anh em" (sibling) sẽ không được phát hiện.
- **Nhầm lẫn Spring Boot và Spring Framework là một, dẫn tới hiểu sai phạm vi kiến thức khi ôn
  phỏng vấn** — nên phân biệt rõ ràng như Myth vs Reality đã nêu.
- **Cố cài đặt Tomcat riêng rồi deploy WAR của ứng dụng Spring Boot vào đó** — đi ngược lại triết
  lý embedded server, chỉ cần thiết trong số ít trường hợp đặc biệt (yêu cầu hạ tầng doanh
  nghiệp bắt buộc dùng server dùng chung).

## Best Practices

- Đặt class `@SpringBootApplication` ở package gốc của ứng dụng để `@ComponentScan` hoạt động
  đúng phạm vi.
- Tận dụng embedded server và fat JAR cho mô hình triển khai container hoá (Docker) — mỗi
  ứng dụng một tiến trình độc lập, không phụ thuộc phần mềm cài sẵn ngoài JVM.
- Hiểu rõ ranh giới Spring Framework/Spring Boot để trả lời chính xác khi phỏng vấn hỏi "Spring
  Boot khác Spring như thế nào".

## Debug Checklist

- [ ] Một số bean không được phát hiện dù đã đánh dấu đúng stereotype annotation? → kiểm tra vị
      trí package của class `@SpringBootApplication` so với package chứa bean đó.
- [ ] Muốn xác nhận server nào đang chạy bên trong ứng dụng? → kiểm tra log khởi động (dòng
      "Tomcat started..."/"Netty started..." tuỳ starter đang dùng) hoặc mở JAR kiểm tra
      `BOOT-INF/lib/`.

## Summary

Spring Boot xây trên Spring Framework, thêm ba trụ cột: Embedded Server, Auto Configuration
(Chapter 07), Starter Dependencies. Đã chứng minh bằng thực nghiệm: Tomcat được đóng gói **thật
sự bên trong** file JAR ứng dụng (`BOOT-INF/lib/tomcat-embed-core-*.jar`), cho phép
`java -jar app.jar` tự khởi động và phục vụ HTTP mà không cần cài server riêng — khởi động chỉ
mất ~1.1 giây trong thực nghiệm. `@SpringBootApplication` là tổ hợp của
`@SpringBootConfiguration` + `@ComponentScan` + `@EnableAutoConfiguration`. Cơ chế fat JAR dùng
custom `ClassLoader` để nạp trực tiếp từ các JAR lồng bên trong, vượt qua giới hạn của classloader
tiêu chuẩn.

## Interview Questions

**Junior**

- Spring Boot khác Spring Framework như thế nào?

**Mid**

- Ba trụ cột chính của Spring Boot là gì?
- `@SpringBootApplication` thực chất là tổ hợp của những annotation nào?

**Senior**

- Giải thích cơ chế fat JAR cho phép một file `.jar` duy nhất chứa cả code lẫn hàng trăm thư
  viện phụ thuộc, vượt qua giới hạn classloader tiêu chuẩn của JVM như thế nào.

## Exercises

- [ ] Đóng gói một ứng dụng Spring Boot (`mvn package`), mở file JAR bằng `unzip -l`, xác nhận
      cấu trúc `BOOT-INF/classes/` và `BOOT-INF/lib/`.
- [ ] Chạy `java -jar app.jar` trên một máy không cài Tomcat/JDK server nào khác, xác nhận ứng
      dụng vẫn khởi động và phục vụ HTTP bình thường.
- [ ] Đổi `spring-boot-starter-web` thành `spring-boot-starter-webflux` (hoặc thêm dependency
      Jetty/Undertow theo tài liệu chính thức), quan sát dòng log embedded server đổi từ Tomcat
      sang server khác.

## Cheat Sheet

| Trụ cột | Vai trò |
| --- | --- |
| Embedded Server | Tomcat/Jetty/Undertow đóng gói cùng JAR, không cần cài server riêng |
| Auto Configuration | Tự động cấu hình bean hợp lý dựa trên classpath (Chapter 07) |
| Starter Dependencies | Gộp sẵn các thư viện liên quan (`spring-boot-starter-web`, ...) |
| `@SpringBootApplication` | = `@SpringBootConfiguration` + `@ComponentScan` + `@EnableAutoConfiguration` |

## References

- Spring Boot Documentation — https://docs.spring.io/spring-boot/index.html
- Spring Boot Documentation — Packaging Executable Archives.
