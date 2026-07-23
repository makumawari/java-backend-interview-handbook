---
tags:
  - Spring
  - Bean
---

# Bean và Bean Scope

> Phase: Phase 5 — Spring
> Chapter slug: `bean`

## Metadata

```yaml
Chapter: Bean
Phase: Phase 5 — Spring
Difficulty: ★★★
Importance: ★★★★
Interview Frequency: 80%
Prerequisites:
  - Chapter 02 — IoC
  - Chapter 03 — DI
Used Later:
  - Chapter 05 — Bean Lifecycle
  - Chapter 06 — Configuration
Estimated Reading: 15 phút
Estimated Practice: 12 phút
```

## Story

> Xem lại: [Chapter 02](02-ioc.md) — mọi ví dụ trước giờ gọi `ctx.getBean(X.class)` nhiều lần
> và ngầm giả định nhận về **cùng một object**. Điều đó có phải luôn đúng không?

Khai báo hai bean giống hệt nhau về code, chỉ khác annotation `@Scope`, rồi gọi `getBean()` hai
lần cho mỗi loại:

```java
@Bean @Scope("singleton") Counter singletonCounter() { return new Counter(); }
@Bean @Scope("prototype") Counter prototypeCounter() { return new Counter(); }
```

Kết quả thật:

```
--- Singleton: lay 2 lan ---
s1 == s2: true
s2.value (sau khi s1.value=100): 100
--- Prototype: lay 2 lan ---
Counter constructor chay! (instance moi)
Counter constructor chay! (instance moi)
p1 == p2: false
p2.value (sau khi p1.value=999): 0
```

`s1 == s2` là `true` — cùng một object. `p1 == p2` là `false` — hai object hoàn toàn khác nhau,
và **constructor chạy riêng cho mỗi lần** `getBean()`. Cùng một class, cùng cách khai báo
`@Bean`, nhưng hành vi khác nhau hoàn toàn chỉ vì một annotation `@Scope`.

## Interview Question (Central)

> Bean trong Spring là gì? Bean scope `singleton` và `prototype` khác nhau như thế nào?

## Objectives

- [ ] Định nghĩa chính xác "bean" trong ngữ cảnh Spring (khác với khái niệm JavaBean truyền
      thống)
- [ ] Tự tay chứng minh bằng thực nghiệm sự khác biệt giữa scope `singleton` (mặc định) và
      `prototype`
- [ ] Biết các scope khác (`request`, `session`) và ngữ cảnh áp dụng của chúng

## Prerequisites

- Chapter 02, 03 — hiểu IoC container và cách nó tạo/tiêm bean.

## Used Later

- **Chapter 05 (Bean Lifecycle)** — chi tiết các bước container thực hiện khi tạo một bean.
- **Chapter 06 (Configuration)** — cách khai báo bean tường minh qua `@Bean`.

## Problem

Không phải mọi object trong một ứng dụng nên được **chia sẻ dùng chung** một instance duy nhất.
Một `PaymentService` không giữ trạng thái (stateless) hoàn toàn có thể dùng chung một instance
cho mọi request — tiết kiệm bộ nhớ, tránh tạo lại liên tục. Nhưng một object **có trạng thái
riêng cho mỗi lần dùng** (ví dụ một "builder" tích luỹ dữ liệu tạm thời) mà bị chia sẻ dùng chung
nhầm sẽ gây lỗi nghiêm trọng: dữ liệu của lần dùng này "rò rỉ" sang lần dùng khác.

## Concept

**Bean** trong Spring là một object được IoC Container quản lý toàn bộ vòng đời (tạo, cấu hình,
tiêm dependency, huỷ) — khác với khái niệm "JavaBean" truyền thống (chỉ là một quy ước đặt tên:
constructor không tham số, field private + getter/setter). **Bean scope** quyết định **số lượng
instance** và **thời điểm tạo** cho một bean: `singleton` (mặc định) — đúng một instance duy
nhất cho toàn bộ container, chia sẻ dùng chung; `prototype` — một instance **mới hoàn toàn** cho
mỗi lần `getBean()`/được tiêm vào nơi khác.

## Why?

Mặc định `singleton` phù hợp với đa số bean trong một ứng dụng backend thông thường —
`Service`, `Repository`, `Controller` thường **không giữ trạng thái riêng theo từng lần gọi**
(stateless), nên chia sẻ một instance duy nhất vừa tiết kiệm tài nguyên vừa an toàn. `prototype`
tồn tại cho trường hợp ngược lại — bean **có trạng thái** cần tách biệt hoàn toàn giữa các lần
sử dụng, ví dụ một object đại diện cho một "phiên xử lý" tạm thời, tích luỹ dữ liệu riêng mà
không được phép lẫn với phiên khác.

## How?

```java
@Bean
@Scope("singleton") // hoac khong can khai bao, day la MAC DINH
Counter singletonCounter() { return new Counter(); }

@Bean
@Scope("prototype")
Counter prototypeCounter() { return new Counter(); }

// Hoac dung annotation rieng cho ngan gon:
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
```

## Visualization

```
Singleton (MAC DINH):

  getBean() lan 1 ──┐
  getBean() lan 2 ──┼──► CUNG MOT instance (tao 1 lan duy nhat, luc container khoi dong)
  getBean() lan 3 ──┘

Prototype:

  getBean() lan 1 ──► instance MOI #1 (constructor chay)
  getBean() lan 2 ──► instance MOI #2 (constructor chay LAI)
  getBean() lan 3 ──► instance MOI #3 (constructor chay LAI)
```

## Example

```java
import org.springframework.context.annotation.*;

public class BeanScopeTest {
    static class Counter {
        int value = 0;
        Counter() { System.out.println("Counter constructor chay! (instance moi)"); }
    }

    @Configuration
    static class Config {
        @Bean @Scope("singleton") Counter singletonCounter() { return new Counter(); }
        @Bean @Scope("prototype") Counter prototypeCounter() { return new Counter(); }
    }

    public static void main(String[] args) {
        try (var ctx = new AnnotationConfigApplicationContext(Config.class)) {
            Counter s1 = ctx.getBean("singletonCounter", Counter.class);
            Counter s2 = ctx.getBean("singletonCounter", Counter.class);
            s1.value = 100;
            System.out.println("s1 == s2: " + (s1 == s2));
            System.out.println("s2.value (sau khi s1.value=100): " + s2.value);

            Counter p1 = ctx.getBean("prototypeCounter", Counter.class);
            Counter p2 = ctx.getBean("prototypeCounter", Counter.class);
            p1.value = 999;
            System.out.println("p1 == p2: " + (p1 == p2));
            System.out.println("p2.value (sau khi p1.value=999): " + p2.value);
        }
    }
}
```

Kết quả thật (Spring Framework 6.1.13):

```
Counter constructor chay! (instance moi)    <- singleton: chay LUC KHOI DONG container
--- Singleton: lay 2 lan ---
s1 == s2: true
s2.value (sau khi s1.value=100): 100        <- s2 THAY DOI theo s1, vi CUNG mot object
--- Prototype: lay 2 lan ---
Counter constructor chay! (instance moi)    <- prototype: chay MOI LAN getBean()
Counter constructor chay! (instance moi)
p1 == p2: false
p2.value (sau khi p1.value=999): 0          <- p2 KHONG doi, vi la object KHAC hoan toan
```

Bằng chứng rõ ràng: sửa `s1.value` cũng làm thay đổi `s2.value` (vì `s1 == s2`), nhưng sửa
`p1.value` **không hề** ảnh hưởng `p2.value` (vì chúng là hai object độc lập).

## Deep Dive

**Vì sao constructor của singleton bean chỉ chạy MỘT lần dù `getBean()` được gọi nhiều lần?**
Container lưu instance singleton đã tạo vào một cache nội bộ (singleton registry, cũng chính là
"first-level cache" trong cơ chế "three-level cache" đã nhắc ở Chapter 03) — mọi lời gọi
`getBean()` sau lần tạo đầu tiên chỉ **tra cache** và trả về instance đã có, không tạo lại.
Prototype bean **không được cache** theo cách này — container chỉ nhớ **cách tạo** (bean
definition), mỗi lần `getBean()` là một lần thực thi lại toàn bộ quá trình tạo từ đầu (constructor,
điền dependency, các bước khởi tạo ở Chapter 05).

## Engineering Insight

**Tiêm một bean `prototype` vào một bean `singleton` — điều gì thực sự xảy ra, và đây có phải
cạm bẫy phổ biến không?** Đây là một trong những cạm bẫy phổ biến nhất về bean scope: nếu
`SingletonService` (scope mặc định) có một field kiểu `PrototypeBean` được tiêm qua constructor/
field injection thông thường, Spring chỉ tiêm **một lần duy nhất** — đúng vào lúc `SingletonService`
được tạo (vì bản thân `SingletonService` chỉ được tạo một lần). Từ đó về sau, mọi lần dùng
`PrototypeBean` bên trong `SingletonService` đều dùng lại **instance đầu tiên** đó — hoàn toàn
mất đi ý nghĩa "mỗi lần dùng một instance mới" của scope `prototype`. Để tiêm đúng một
`prototype` bean mới mỗi lần cần dùng bên trong một singleton, cần một cơ chế đặc biệt: dùng
`ObjectFactory<PrototypeBean>`/`Provider<PrototypeBean>` (gọi `.getObject()`/`.get()` mỗi lần
cần instance mới) hoặc annotation `@Lookup` — cả hai đều trì hoãn việc gọi `getBean()` tới đúng
thời điểm cần dùng, thay vì gọi một lần duy nhất lúc tiêm.

## Historical Note

Bean scope `singleton`/`prototype` có từ những phiên bản Spring rất sớm. Các scope
`request`/`session`/`application` (chỉ có ý nghĩa trong ứng dụng web, gắn với vòng đời một HTTP
request/session) được bổ sung khi Spring MVC phát triển, phản ánh nhu cầu quản lý trạng thái
riêng cho từng request/session người dùng mà không phải tự tay dùng `ThreadLocal` thủ công.

## Myth vs Reality

- **Myth:** "'Bean' trong Spring nghĩa giống hệt 'JavaBean' truyền thống (class với
  getter/setter theo quy ước)."
  **Reality:** Bean trong Spring chỉ đơn giản là "object được container quản lý" — không yêu
  cầu tuân theo quy ước JavaBean (constructor không tham số, getter/setter) như tên gọi có thể
  gây nhầm lẫn.

- **Myth:** "Tiêm một `prototype` bean vào `singleton` bean sẽ luôn cho instance mới mỗi lần
  dùng."
  **Reality:** Xem Engineering Insight — mặc định chỉ tiêm một lần lúc singleton được tạo, cần
  `ObjectFactory`/`@Lookup` để thực sự có instance mới mỗi lần.

## Common Mistakes

- **Giả định mọi bean đều là singleton mà không kiểm tra khi thiết kế bean có trạng thái riêng
  theo từng lần dùng** — dẫn tới rò rỉ dữ liệu giữa các lần sử dụng nếu vô tình chia sẻ instance.
- **Tiêm `prototype` bean vào `singleton` bean qua constructor/field injection thông thường, kỳ
  vọng có instance mới mỗi lần** — chỉ nhận đúng một instance duy nhất (xem Engineering Insight).
- **Dùng scope `prototype` cho mọi bean "để an toàn"** — mất lợi ích chia sẻ tài nguyên của
  singleton mà không có lý do thực sự cần thiết.

## Best Practices

- Mặc định coi bean là `singleton` — chỉ chuyển sang `prototype` khi bean thực sự cần trạng thái
  riêng biệt cho mỗi lần sử dụng.
- Khi cần `prototype` bean bên trong một `singleton`, dùng `ObjectFactory`/`Provider`/`@Lookup`
  thay vì tiêm trực tiếp.
- Chỉ dùng scope `request`/`session` khi thực sự có ngữ cảnh web và cần trạng thái gắn với vòng
  đời request/session cụ thể.

## Debug Checklist

- [ ] Dữ liệu "rò rỉ" giữa các request/lần gọi khác nhau dù expect mỗi lần độc lập? → kiểm tra
      bean có scope `singleton` (mặc định) trong khi lẽ ra cần `prototype` không.
- [ ] Tiêm `prototype` bean vào `singleton` bean nhưng luôn nhận cùng instance? → dùng
      `ObjectFactory`/`@Lookup` thay vì tiêm trực tiếp — xem Engineering Insight.

## Summary

Bean là object được IoC Container quản lý toàn bộ vòng đời — khác với "JavaBean" truyền thống.
Bean scope quyết định số lượng instance: `singleton` (mặc định, một instance chia sẻ dùng
chung, cache sau lần tạo đầu) và `prototype` (instance mới mỗi lần `getBean()`, không cache). Đã
chứng minh bằng thực nghiệm: sửa giá trị trên singleton ảnh hưởng mọi nơi dùng nó, còn prototype
thì hoàn toàn độc lập. Tiêm `prototype` vào `singleton` qua cách thông thường chỉ tiêm một lần
duy nhất — cần `ObjectFactory`/`@Lookup` để thực sự có instance mới mỗi lần cần dùng.

## Interview Questions

**Mid**

- Bean trong Spring là gì? Bean scope `singleton` và `prototype` khác nhau như thế nào?
- Điều gì xảy ra khi tiêm một `prototype` bean vào một `singleton` bean theo cách thông thường?

## Exercises

- [ ] Chạy lại `BeanScopeTest` ở trên, xác nhận singleton chia sẻ instance còn prototype thì
      không, đúng như mô tả.
- [ ] Viết một `singleton` bean tiêm một `prototype` bean qua `ObjectFactory<T>`, xác nhận mỗi
      lần gọi `.getObject()` cho ra instance khác nhau.
- [ ] Tìm hiểu scope `request`, viết một ví dụ Spring MVC đơn giản minh hoạ bean được tạo mới
      cho mỗi HTTP request.

## Cheat Sheet

| Scope | Số instance | Thời điểm tạo | Dùng khi |
| --- | --- | --- | --- |
| `singleton` (mặc định) | 1, chia sẻ toàn container | Eager, lúc container khởi động | Bean stateless (đa số Service/Repository) |
| `prototype` | Mới mỗi lần `getBean()` | Lazy, mỗi lần cần | Bean có trạng thái riêng từng lần dùng |
| `request` | 1 mỗi HTTP request | Mỗi request | Dữ liệu gắn với 1 request (chỉ web) |
| `session` | 1 mỗi HTTP session | Mỗi session | Dữ liệu gắn với 1 session người dùng (chỉ web) |

## References

- Spring Framework Documentation — Bean Scopes: https://docs.spring.io/spring-framework/reference/core/beans/factory-scopes.html
