---
tags:
  - Spring
  - SpringCore
---

# Spring Core

> Phase: Phase 5 — Spring
> Chapter slug: `spring-core`

## Metadata

```yaml
Chapter: Spring Core
Phase: Phase 5 — Spring
Difficulty: ★★
Importance: ★★★★★
Interview Frequency: 85%
Prerequisites:
  - Phase 1, Chapter 08 — OOP Fundamentals
  - Phase 1, Chapter 18 — Annotation
  - Phase 1, Chapter 19 — Reflection
Used Later:
  - Chapter 02 — IoC
  - Chapter 03 — DI
  - Chapter 04 — Bean
Estimated Reading: 15 phút
Estimated Practice: 10 phút
```

## Story

> Xem lại: Phase 1, Chapter 19 (Reflection) — annotation tự nó không làm gì cả, nó chỉ là
> **metadata**. Một class đánh dấu `@Service` không tự động "trở thành" bất cứ thứ gì đặc biệt —
> trừ khi có một cơ chế khác **đọc** annotation đó bằng Reflection rồi hành động dựa trên nó.

Thử đăng ký 4 class, mỗi class đánh dấu một stereotype annotation khác nhau
(`@Component`, `@Repository`, `@Service`, `@Controller`), rồi hỏi container "bạn có những bean
nào":

```
--- Cac bean duoc dang ky ---
springCoreDemoTest.GenericComponent -> GenericComponent
springCoreDemoTest.FakeRepository -> FakeRepository
springCoreDemoTest.FakeService -> FakeService
springCoreDemoTest.FakeController -> FakeController
```

Cả 4 class đều được container nhận diện và tạo instance — vì tất cả các stereotype annotation
này (`@Repository`, `@Service`, `@Controller`) thực chất chỉ là **`@Component` được đặt tên lại**
(meta-annotation), và Spring dùng chính cơ chế Reflection đã học ở Phase 1 để quét, đọc, và
hành động dựa trên chúng.

## Interview Question (Central)

> What are the commonly used Spring annotations?

## Objectives

- [ ] Giải thích Spring Framework là gì, vấn đề cốt lõi nó giải quyết (quản lý vòng đời và mối
      quan hệ giữa các object trong một ứng dụng lớn)
- [ ] Phân biệt các stereotype annotation phổ biến: `@Component`, `@Repository`, `@Service`,
      `@Controller`/`@RestController`
- [ ] Tự tay chứng minh bằng thực nghiệm: `@Repository`/`@Service`/`@Controller` chỉ là
      `@Component` được đặt tên ngữ nghĩa lại

## Prerequisites

- Phase 1, Chapter 08 — hiểu OOP cơ bản (interface, dependency giữa các object).
- Phase 1, Chapter 18-19 — hiểu annotation là metadata, Reflection là cách đọc nó tại runtime —
  chính cơ chế Spring dùng để hoạt động.

## Used Later

- **Chapter 02 (IoC)**, **Chapter 03 (DI)**, **Chapter 04 (Bean)** — Spring Core là nền tảng
  khái niệm cho toàn bộ các chapter còn lại của Phase 5.

## Problem

Một ứng dụng backend thực tế có hàng trăm, hàng nghìn object phối hợp với nhau: tầng
controller gọi tầng service, tầng service gọi tầng repository, repository cần kết nối database.
Nếu mỗi object tự chịu trách nhiệm tạo ra các object nó phụ thuộc (`new` trực tiếp), toàn bộ
codebase trở thành một mạng lưới phụ thuộc chặt chẽ (tightly coupled) — khó test (không thể thay
thế một dependency bằng mock), khó thay đổi (đổi một implementation ảnh hưởng dây chuyền tới mọi
nơi gọi `new` trực tiếp nó), và khó quản lý vòng đời (ai chịu trách nhiệm đảm bảo một
`DataSource` chỉ được tạo một lần, đóng đúng lúc ứng dụng tắt?).

## Concept

**Spring Framework** là một framework quản lý toàn bộ vòng đời và mối quan hệ giữa các object
(gọi là **bean**) trong một ứng dụng Java, thông qua một **container** (IoC Container, Chapter
02) — container chịu trách nhiệm tạo, cấu hình, kết nối (dependency injection, Chapter 03), và
quản lý vòng đời (Chapter 05) của các bean, thay vì để từng object tự lo cho chính nó.

## Why?

Bằng cách tập trung hoá trách nhiệm "tạo và kết nối object" vào một container duy nhất (thay vì
rải rác khắp codebase), Spring cho phép: (1) **tách rời** logic nghiệp vụ khỏi logic khởi tạo —
một class chỉ cần khai báo "tôi cần một `PaymentService`", không cần biết implementation cụ thể
nào hay cách tạo ra nó; (2) **dễ test** — container có thể được cấu hình để tiêm vào một mock
thay vì implementation thật khi chạy test; (3) **quản lý vòng đời nhất quán** — container đảm
bảo mỗi bean singleton chỉ được tạo đúng một lần, đúng thứ tự phụ thuộc, và được dọn dẹp đúng lúc
ứng dụng tắt.

## How?

```java
@Service // danh dau "day la mot bean tang service"
public class PaymentService {
    public String charge(double amount) { return "Da charge: " + amount; }
}

@RestController // danh dau "day la mot bean xu ly HTTP request"
public class PaymentController {
    private final PaymentService paymentService; // Spring TU DONG tiem vao (Chapter 03)
    PaymentController(PaymentService paymentService) { this.paymentService = paymentService; }
}
```

## Visualization

```
Khong co Spring (tu quan ly):        Co Spring (container quan ly):

  Controller                            Controller
     │ new Service()                       │ (container TIEM san)
     ▼                                      ▼
  Service                               Service <──┐
     │ new Repository()                    │       │ IoC Container
     ▼                                      ▼       │ (tao, ket noi,
  Repository                            Repository ─┘  quan ly vong doi)
     │ new DataSource()                    │
     ▼                                      ▼
  DataSource                            DataSource

  Moi object TU chiu trach nhiem         Container chiu trach nhiem
  tao & quan ly dependency cua no        TOAN BO viec tao & ket noi
```

## Example

```java
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.stereotype.*;

public class SpringCoreDemoTest {
    @Component  static class GenericComponent {}
    @Repository static class FakeRepository {}
    @Service    static class FakeService {}
    @Controller static class FakeController {}

    public static void main(String[] args) {
        try (var ctx = new AnnotationConfigApplicationContext(
                GenericComponent.class, FakeRepository.class, FakeService.class, FakeController.class)) {
            for (String name : ctx.getBeanDefinitionNames()) {
                if (!name.startsWith("org.springframework")) {
                    System.out.println(name + " -> " + ctx.getBean(name).getClass().getSimpleName());
                }
            }
        }
    }
}
```

Kết quả thật (Spring Framework 6.1.13, Spring Boot 3.3.4):

```
Spring Framework version: 6.1.13
--- Cac bean duoc dang ky ---
springCoreDemoTest.GenericComponent -> GenericComponent
springCoreDemoTest.FakeRepository -> FakeRepository
springCoreDemoTest.FakeService -> FakeService
springCoreDemoTest.FakeController -> FakeController
```

Cả 4 class — dù đánh dấu 4 annotation khác nhau — đều được container nhận diện và khởi tạo thành
bean, xác nhận chúng cùng chung một cơ chế nền tảng.

## Deep Dive

**Vì sao `@Repository`, `@Service`, `@Controller` "chỉ là" `@Component` được đặt tên lại — kiểm
tra thật bằng cách nào?** Mở mã nguồn (hoặc `javap`/IDE "go to definition") của
`org.springframework.stereotype.Service`:

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component // <- CHINH LA @Component, duoc "gan nhan" y nghia lai
public @interface Service {
    @AliasFor(annotation = Component.class)
    String value() default "";
}
```

`@Service` bản thân nó được đánh dấu `@Component` (meta-annotation, đã học ở Phase 1, Chapter
18) — nghĩa là bộ quét component-scan của Spring, khi tìm annotation `@Component` **hoặc bất kỳ
annotation nào được đánh dấu `@Component`**, đều coi class đó là một candidate bean. Về mặt kỹ
thuật thuần tuý, `@Repository`/`@Service`/`@Controller` **hoán đổi cho nhau được** mà không ảnh
hưởng tới việc bean có được tạo hay không — sự khác biệt thực sự (nếu có) nằm ở **hành vi bổ
sung** mà mỗi annotation kích hoạt riêng (ví dụ `@Repository` kích hoạt dịch exception của tầng
persistence sang `DataAccessException` thống nhất của Spring), không nằm ở cơ chế đăng ký bean cơ
bản.

## Engineering Insight

**Vì sao vẫn nên dùng đúng stereotype annotation theo tầng (Controller/Service/Repository) dù về
mặt kỹ thuật `@Component` cho mọi nơi vẫn hoạt động?** Dù tương đương về mặt đăng ký bean, các
annotation theo tầng mang **ý nghĩa kiến trúc** rõ ràng cho người đọc code (và cho một số công cụ
phân tích tĩnh/IDE) — `@Repository` báo hiệu "class này thuộc tầng truy cập dữ liệu", giúp người
đọc mới vào codebase định vị nhanh vai trò của một class mà không cần đọc hết nội dung. Đây là
ví dụ điển hình về annotation mang giá trị **tài liệu hoá qua kiểu dữ liệu** (tương tự triết lý
đã gặp ở `Optional`, Phase 4 Chapter 06) — hiệu quả kỹ thuật giống nhau, nhưng giá trị giao tiếp
với con người khác nhau đáng kể.

## Historical Note

Spring Framework ra đời năm 2003 (Rod Johnson), ban đầu như một phản ứng lại sự phức tạp của
Java EE/J2EE thời kỳ đầu (đặc biệt là EJB 2.x, vốn nổi tiếng cồng kềnh). Cấu hình XML là cách
duy nhất ở các phiên bản đầu; annotation-based configuration (như `@Component`, `@Autowired`)
chỉ xuất hiện từ Spring 2.5 (2007), và Java-based configuration hoàn toàn không cần XML
(`@Configuration`, Chapter 06) từ Spring 3.0 (2009) — dẫn tới Spring Boot (Chapter 08) năm 2014,
gần như loại bỏ hoàn toàn nhu cầu cấu hình XML thủ công.

## Myth vs Reality

- **Myth:** "`@Service`, `@Repository`, `@Controller` có cơ chế đăng ký bean khác nhau về bản
  chất."
  **Reality:** Đã chứng minh bằng thực nghiệm và mã nguồn — chúng đều là `@Component` với ý
  nghĩa ngữ nghĩa khác nhau; cơ chế đăng ký bean nền tảng giống hệt nhau.

- **Myth:** "Spring là một framework, chỉ dùng được cho web application."
  **Reality:** Spring Framework lõi (IoC Container, DI) hoàn toàn độc lập với web — có thể dùng
  cho ứng dụng CLI, batch job, desktop. Spring Boot (Chapter 08) và Spring MVC/WebFlux mới là
  các module chuyên cho web.

## Common Mistakes

- **Dùng `@Component` cho mọi class thay vì stereotype đúng tầng** — mất đi giá trị tài liệu hoá
  kiến trúc đã nêu ở Engineering Insight.
- **Nhầm lẫn Spring Framework và Spring Boot là một** — Spring Boot (Chapter 08) là lớp
  "auto-configuration + embedded server" xây trên Spring Framework, không phải cùng một thứ.
- **Không hiểu annotation chỉ là metadata thụ động** — như Phase 1 Chapter 18-19 đã nhấn mạnh,
  bản thân `@Service` không "làm" gì — chính bộ quét component-scan của Spring (dùng Reflection)
  mới là thứ tạo ra hành vi.

## Best Practices

- Dùng đúng stereotype annotation theo vai trò kiến trúc của class (`@Repository` cho tầng data
  access, `@Service` cho logic nghiệp vụ, `@RestController`/`@Controller` cho tầng web).
- Hiểu rõ ranh giới Spring Framework (lõi, độc lập web) và Spring Boot (auto-config, embedded
  server) để tránh nhầm lẫn khi đọc tài liệu hoặc trả lời phỏng vấn.

## Debug Checklist

- [ ] Một class đánh dấu stereotype annotation nhưng không được nhận diện thành bean? → kiểm
      tra class có nằm trong phạm vi quét của `@ComponentScan` (hoặc package con của class có
      `@SpringBootApplication`, Chapter 08) không.
- [ ] Không chắc một annotation có phải stereotype của Spring hay không? → kiểm tra annotation
      đó có được đánh dấu (trực tiếp hoặc gián tiếp) bởi `@Component` hay không.

## Summary

Spring Framework quản lý vòng đời và kết nối giữa các object (bean) trong ứng dụng thông qua IoC
Container, thay vì để từng object tự `new` và quản lý dependency của chính nó. Các stereotype
annotation phổ biến — `@Component`, `@Repository`, `@Service`, `@Controller`/`@RestController` —
đã chứng minh bằng thực nghiệm và mã nguồn: đều cùng cơ chế nền tảng (`@Component`
meta-annotation), khác nhau chủ yếu ở ý nghĩa kiến trúc và một số hành vi bổ sung theo tầng.
Spring ra đời 2003 như phản ứng lại sự phức tạp của J2EE/EJB thời kỳ đầu, dần chuyển từ cấu hình
XML sang annotation rồi Java-based configuration, cuối cùng dẫn tới Spring Boot (2014).

## Interview Questions

- What are the commonly used Spring annotations?

**Junior**

- Spring Framework giải quyết vấn đề gì trong một ứng dụng backend lớn?
- `@Component`, `@Service`, `@Repository`, `@Controller` khác nhau như thế nào?

## Exercises

- [ ] Chạy lại `SpringCoreDemoTest` ở trên, xác nhận cả 4 stereotype đều được nhận diện thành
      bean.
- [ ] Mở mã nguồn của `@Repository` và `@Controller` (qua IDE "go to definition" hoặc
      `javap`), xác nhận cả hai đều được đánh dấu `@Component`.
- [ ] Thử tạo một annotation tuỳ chỉnh `@MyComponent` đánh dấu `@Component`, xác nhận Spring
      vẫn nhận diện được class dùng annotation tuỳ chỉnh này thành bean.

## Cheat Sheet

| Annotation | Tầng | Hành vi bổ sung |
| --- | --- | --- |
| `@Component` | Chung, không xác định tầng cụ thể | Không |
| `@Repository` | Data access | Dịch exception tầng persistence sang `DataAccessException` |
| `@Service` | Logic nghiệp vụ | Không (thuần ngữ nghĩa) |
| `@Controller` | Web (trả về view) | Kích hoạt xử lý MVC |
| `@RestController` | Web (trả về JSON/data trực tiếp) | `@Controller` + `@ResponseBody` |
| `@Configuration` | Khai báo bean thủ công | Xem Chapter 06 |
| `@Autowired` | Đánh dấu điểm tiêm dependency | Xem Chapter 03 |

## References

- Spring Framework Documentation — https://docs.spring.io/spring-framework/reference/
- Spring Framework GitHub — mã nguồn `org.springframework.stereotype.*`.
