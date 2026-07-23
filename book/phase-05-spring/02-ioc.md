---
tags:
  - Spring
  - IoC
---

# Inversion of Control (IoC)

> Phase: Phase 5 — Spring
> Chapter slug: `ioc`

## Metadata

```yaml
Chapter: IoC
Phase: Phase 5 — Spring
Difficulty: ★★★
Importance: ★★★★★
Interview Frequency: 90%
Prerequisites:
  - Chapter 01 — Spring Core
Used Later:
  - Chapter 03 — DI
  - Chapter 04 — Bean
  - Chapter 05 — Bean Lifecycle
Estimated Reading: 15 phút
Estimated Practice: 12 phút
```

## Story

> Xem lại: [Chapter 01](01-spring-core.md) — `OrderService` cần một `PaymentService` để hoạt
> động. Theo cách viết thông thường, ai là người quyết định **khi nào** và **bằng cách nào**
> `PaymentService` được tạo ra?

So sánh hai cách viết cùng một quan hệ phụ thuộc:

```java
class OrderService {
    private final PaymentService paymentService = new PaymentService(); // Cach 1
}

class OrderService {
    private final PaymentService paymentService;
    OrderService(PaymentService paymentService) { this.paymentService = paymentService; } // Cach 2
}
```

Ở Cách 1, `OrderService` **tự quyết định** tạo ra `PaymentService` cụ thể nào, khi nào. Ở Cách 2,
`OrderService` chỉ khai báo "tôi cần một `PaymentService`" — **ai đó khác** (container) quyết
định đưa instance nào vào, khi nào. Chạy thử với Spring container quản lý theo Cách 2:

```
=== Cach 2: IoC container tu tao va noi cac bean ===
PaymentService constructor chay!
OrderService nhan PaymentService tu BEN NGOAI (khong tu new)
```

`PaymentService` được tạo ra **trước khi** `OrderService` yêu cầu — quyền quyết định "khi nào
tạo, tạo ra sao" đã bị **đảo ngược** từ `OrderService` sang container.

## Interview Question (Central)

> Inversion of Control (IoC) là gì? Nó "đảo ngược" điều gì so với cách lập trình thông thường?

## Objectives

- [ ] Giải thích chính xác "control" nào bị "đảo ngược" trong IoC
- [ ] Tự tay chứng minh bằng thực nghiệm: container tạo bean trước, tiêm vào nơi cần, không phải
      nơi cần tự tạo
- [ ] Phân biệt IoC (nguyên lý) và Dependency Injection (một trong các cách hiện thực hoá IoC,
      Chapter 03)

## Prerequisites

- Chapter 01 — hiểu Spring Framework quản lý bean qua container.

## Used Later

- **Chapter 03 (DI)** — cách cụ thể IoC container "tiêm" dependency vào bean.
- **Chapter 04 (Bean)**, **Chapter 05 (Bean Lifecycle)** — chi tiết cách container tạo và quản lý
  vòng đời bean.

## Problem

Trong lập trình thủ tục thông thường, một object **chủ động** kiểm soát toàn bộ luồng: nó quyết
định khi nào tạo dependency, tạo bằng cách nào, dùng implementation cụ thể nào. Điều này khiến
object đó **gắn chặt** (tightly coupled) với các quyết định khởi tạo — muốn đổi implementation
(ví dụ đổi từ gọi API thật sang mock khi test), phải sửa trực tiếp code bên trong object đó.

## Concept

**Inversion of Control (IoC)** là nguyên lý thiết kế: thay vì một object tự kiểm soát việc tạo
và quản lý các dependency của nó, quyền kiểm soát đó được **chuyển giao** (đảo ngược) cho một
thực thể bên ngoài — trong Spring, đó là **IoC Container** (`ApplicationContext`). Object chỉ
cần khai báo "tôi cần gì" (qua constructor, field, hoặc setter), container chịu trách nhiệm
"cung cấp cái gì, khi nào, bằng cách nào".

## Why?

"Control" bị đảo ngược cụ thể là: quyền quyết định **thời điểm tạo** (khi nào một
`PaymentService` được khởi tạo) và **cách tạo** (dùng implementation nào, cấu hình ra sao) —
trước đây thuộc về chính object cần dependency đó (`OrderService` tự `new`), giờ thuộc về
container. Lợi ích trực tiếp: `OrderService` không còn biết (và không cần biết) `PaymentService`
được tạo ra như thế nào — cho phép **thay thế** hoàn toàn cách tạo (ví dụ dùng implementation
khác khi test) mà không sửa một dòng nào trong `OrderService`. Đây chính là nguyên lý làm nền
tảng cho khả năng test dễ dàng và thiết kế loose-coupling của toàn bộ hệ sinh thái Spring.

## How?

```java
@Configuration
class AppConfig {
    @Bean
    PaymentService paymentService() { return new PaymentService(); }

    @Bean
    OrderService orderService(PaymentService paymentService) { return new OrderService(paymentService); }
}

try (var ctx = new AnnotationConfigApplicationContext(AppConfig.class)) {
    OrderService order = ctx.getBean(OrderService.class); // KHONG tu new, lay tu container
}
```

## Visualization

```
KHONG co IoC (control nam o OrderService):

  OrderService quyet dinh:
    - Khi nao tao PaymentService?     -> luc OrderService duoc tao
    - Tao implementation nao?          -> OrderService tu chon trong code

  new OrderService() ──► ben trong tu goi ──► new PaymentService()


CO IoC (control nam o Container):

  Container quyet dinh:
    - Khi nao tao PaymentService?     -> theo lifecycle cua container (Chapter 05)
    - Tao implementation nao?          -> theo cau hinh (Chapter 06), co the doi khong sua code

  Container: tao PaymentService TRUOC ──► tiem vao ──► OrderService (chi "nhan", khong "tu tao")
```

## Example

```java
import org.springframework.context.annotation.*;

public class IocDemoTest {
    static class PaymentService {
        PaymentService() { System.out.println("PaymentService constructor chay!"); }
    }

    static class OrderService {
        private final PaymentService paymentService;
        OrderService(PaymentService paymentService) {
            this.paymentService = paymentService;
            System.out.println("OrderService nhan PaymentService tu BEN NGOAI (khong tu new)");
        }
    }

    @Configuration
    static class AppConfig {
        @Bean PaymentService paymentService() { return new PaymentService(); }
        @Bean OrderService orderService(PaymentService paymentService) { return new OrderService(paymentService); }
    }

    public static void main(String[] args) {
        try (var ctx = new AnnotationConfigApplicationContext(AppConfig.class)) {
            OrderService order = ctx.getBean(OrderService.class);
            System.out.println("Lay OrderService tu container: " + order);
        }
    }
}
```

Kết quả thật (Spring Framework 6.1.13):

```
=== Cach 2: IoC container tu tao va noi cac bean ===
PaymentService constructor chay!
OrderService nhan PaymentService tu BEN NGOAI (khong tu new)
Lay OrderService tu container: com.handbook.demo.IocDemoTest$OrderService@55dfcc6
```

Thứ tự quan trọng: `"PaymentService constructor chay!"` in ra **trước** khi code gọi
`ctx.getBean(OrderService.class)` — container đã tự quyết định tạo `PaymentService` (và
`OrderService`) ngay khi khởi tạo (`new AnnotationConfigApplicationContext(...)`), không đợi tới
khi có ai gọi `getBean()`. `OrderService` chỉ đơn thuần **nhận** `PaymentService` qua constructor,
không hề gọi `new PaymentService()` ở bất kỳ đâu bên trong chính nó.

## Deep Dive

**Vì sao container tạo bean "ngay khi khởi tạo" thay vì đợi tới lúc `getBean()` được gọi?** Mặc
định, Spring container tạo **eager** (ngay lập tức, không trì hoãn) mọi bean có scope
`singleton` (Chapter 04) khi `ApplicationContext` được refresh (khởi tạo) — không đợi tới khi có
ai thực sự cần dùng bean đó. Quyết định thiết kế này (khác với việc trì hoãn tạo tới khi thực sự
cần, gọi là "lazy") giúp phát hiện lỗi cấu hình (dependency thiếu, circular dependency — Chapter
03) **ngay khi ứng dụng khởi động**, thay vì để lỗi xuất hiện bất ngờ giữa lúc phục vụ request
thực tế — một ứng dụng "khởi động thành công" gần như đảm bảo mọi bean singleton đã được tạo và
kết nối đúng.

## Engineering Insight

**IoC là một nguyên lý (principle), Dependency Injection là một pattern (khuôn mẫu) hiện thực
hoá nguyên lý đó — vì sao phân biệt hai khái niệm này quan trọng khi trả lời phỏng vấn?** IoC
chỉ nói "quyền kiểm soát việc tạo dependency thuộc về bên ngoài", nhưng **không quy định cách cụ
thể** để làm điều đó — có nhiều cách hiện thực hoá IoC khác nhau: Dependency Injection (Chapter
03, phổ biến nhất trong Spring), Service Locator pattern (object chủ động "hỏi" một registry
trung tâm để lấy dependency, ít phổ biến hơn, có nhược điểm che giấu dependency thực sự), hay
Template Method pattern (framework gọi ngược lại code của bạn tại các điểm định sẵn). Nhầm lẫn
"IoC" và "DI" là đồng nghĩa hoàn toàn là một lỗi khái niệm phổ biến — DI chỉ là **một cách cụ
thể** (dù là cách phổ biến nhất trong hệ sinh thái Spring) để đạt được Inversion of Control.

## Historical Note

Thuật ngữ "Inversion of Control" xuất hiện từ trước Spring rất lâu, gắn liền với nguyên lý
"Hollywood Principle" ("Don't call us, we'll call you" — đừng chủ động gọi framework, framework
sẽ gọi lại code của bạn khi cần) trong thiết kế framework nói chung. Martin Fowler viết bài luận
nổi tiếng "Inversion of Control Containers and the Dependency Injection pattern" (2004) — chính
bài viết này góp phần phổ biến và làm rõ ranh giới giữa IoC (nguyên lý) và DI (một pattern cụ
thể) mà ngành công nghiệp dùng tới ngày nay.

## Myth vs Reality

- **Myth:** "IoC và Dependency Injection là hai tên gọi khác nhau cho cùng một khái niệm."
  **Reality:** Xem Engineering Insight — DI chỉ là một trong nhiều cách hiện thực hoá nguyên lý
  IoC tổng quát hơn.

- **Myth:** "IoC container tạo bean khi nào cần thì tạo (lazy), giống hệt việc gọi `new` thông
  thường chỉ là được container làm hộ."
  **Reality:** Đã chứng minh bằng thực nghiệm — mặc định container tạo mọi bean singleton
  **ngay khi khởi tạo**, không đợi tới lúc cần dùng (xem Deep Dive).

## Common Mistakes

- **Đồng nhất hoàn toàn IoC với DI khi trả lời phỏng vấn** — bỏ lỡ cơ hội thể hiện hiểu biết sâu
  hơn về các cách hiện thực hoá IoC khác (Service Locator, Template Method).
- **Nghĩ rằng bean chỉ được tạo khi thực sự cần dùng (`getBean()`)** — mặc định sai với bean
  singleton, gây hiểu lầm khi debug thời điểm một constructor thực sự chạy.
- **Không hiểu vì sao ứng dụng "khởi động chậm hơn"** khi thêm nhiều bean — vì mọi bean singleton
  được tạo eager ngay lúc khởi động, không phải lazy.

## Debug Checklist

- [ ] Constructor của một bean chạy sớm hơn dự kiến (trước khi có request nào tới)? → bình
      thường với bean singleton mặc định eager, xem Deep Dive.
- [ ] Ứng dụng khởi động chậm khi có nhiều bean nặng (kết nối DB, load config lớn)? → cân nhắc
      đánh dấu `@Lazy` cho các bean không cần thiết ngay lúc khởi động.
- [ ] Nhầm lẫn IoC và DI khi thiết kế/trả lời phỏng vấn? → nhớ: IoC là nguyên lý tổng quát, DI là
      một cách cụ thể (phổ biến nhất trong Spring) để đạt được nó.

## Summary

Inversion of Control (IoC) đảo ngược quyền kiểm soát việc tạo và quản lý dependency — từ chính
object cần dependency đó, chuyển sang một thực thể bên ngoài (IoC Container trong Spring). Đã
chứng minh bằng thực nghiệm: `PaymentService` được container tạo ra **trước** khi `OrderService`
yêu cầu, và `OrderService` chỉ nhận, không tự tạo. Mặc định mọi bean singleton được tạo eager
ngay khi container khởi tạo, giúp phát hiện lỗi cấu hình sớm. IoC là nguyên lý tổng quát; Dependency Injection (Chapter 03) chỉ là một trong các cách hiện thực hoá nó.

## Interview Questions

**Junior**

- Inversion of Control (IoC) là gì?

**Mid**

- IoC khác Dependency Injection như thế nào? Vì sao không nên coi chúng là đồng nghĩa?
- Bean singleton được container tạo ra khi nào — ngay lúc khởi động hay khi thực sự cần dùng?

## Exercises

- [ ] Chạy lại `IocDemoTest` ở trên, quan sát thứ tự các dòng in ra để xác nhận container tạo
      bean trước khi có ai gọi `getBean()`.
- [ ] Thêm `@Lazy` vào bean `PaymentService`, chạy lại, quan sát constructor chỉ chạy khi
      `getBean()` (hoặc bean phụ thuộc nó) thực sự được gọi.
- [ ] Tìm hiểu Service Locator pattern, viết một ví dụ đơn giản minh hoạ nó cũng là một cách
      hiện thực hoá IoC nhưng khác Dependency Injection.

## Cheat Sheet

| Khái niệm | Ý nghĩa |
| --- | --- |
| IoC | Nguyên lý: quyền tạo/quản lý dependency thuộc về bên ngoài, không phải chính object cần nó |
| DI | Một pattern cụ thể hiện thực hoá IoC — dependency được "tiêm" vào (constructor/field/setter) |
| IoC Container | Cơ chế cụ thể của Spring quản lý vòng đời và kết nối bean |
| Service Locator | Cách hiện thực hoá IoC khác — object chủ động hỏi registry để lấy dependency |
| Bean singleton mặc định | Được tạo eager, ngay khi container khởi tạo |

## References

- Spring Framework Documentation — IoC Container: https://docs.spring.io/spring-framework/reference/core/beans.html
- Martin Fowler — "Inversion of Control Containers and the Dependency Injection pattern" (2004).
