---
tags:
  - Spring
  - Configuration
---

# @Configuration và CGLIB Proxy

> Phase: Phase 5 — Spring
> Chapter slug: `configuration`

## Metadata

```yaml
Chapter: Configuration
Phase: Phase 5 — Spring
Difficulty: ★★★★
Importance: ★★★★
Interview Frequency: 55%
Prerequisites:
  - Chapter 04 — Bean
  - Chapter 05 — Bean Lifecycle
Used Later:
  - Chapter 07 — Auto Configuration
  - Chapter 12 — AOP (cùng cơ chế proxy CGLIB)
Estimated Reading: 15 phút
Estimated Practice: 12 phút
```

## Story

> Xem lại: [Chapter 04](04-bean.md) — bean scope `singleton` (mặc định) đảm bảo chỉ có **một**
> instance duy nhất. Nhưng nếu một `@Bean` method **gọi trực tiếp** một `@Bean` method khác
> trong cùng class `@Configuration` (thay vì để Spring tiêm qua tham số), điều gì đảm bảo tính
> singleton đó?

```java
@Configuration
class FullConfig {
    @Bean Engine engine() { return new Engine(); }
    @Bean Car car1() { return new Car(engine()); } // goi TRUC TIEP nhu mot method binh thuong
    @Bean Car car2() { return new Car(engine()); } // goi TRUC TIEP LAN NUA
}
```

Nếu `engine()` chỉ là một method Java thông thường, gọi nó hai lần (trong `car1()` và `car2()`)
phải tạo ra **hai `Engine` khác nhau**. Nhưng thử chạy thật:

```
=== @Configuration (full, proxyBeanMethods=true) ===
Engine constructor chay!          <- CHI IN MOT LAN DUY NHAT!
car1.engine == car2.engine: true
car1.engine == container's Engine bean: true
Config class thuc te: ...FullConfig$$SpringCGLIB$$0
```

`Engine constructor chay!` chỉ in **một lần** — dù `engine()` được gọi trực tiếp hai lần trong
mã nguồn. Và class thực tế của `FullConfig` không phải `FullConfig` gốc, mà là một class khác:
`FullConfig$$SpringCGLIB$$0`.

## Interview Question (Central)

> Vì sao gọi trực tiếp một `@Bean` method từ một `@Bean` method khác trong cùng
> `@Configuration` class vẫn đảm bảo tính singleton? `proxyBeanMethods = false` thay đổi điều
> gì?

## Objectives

- [ ] Giải thích cơ chế CGLIB proxy đứng sau `@Configuration` mặc định (`proxyBeanMethods =
      true`)
- [ ] Tự tay chứng minh bằng thực nghiệm: gọi `@Bean` method trực tiếp vẫn cho cùng instance với
      `proxyBeanMethods = true`, nhưng cho instance khác nhau với `proxyBeanMethods = false`
- [ ] Biết khi nào nên dùng `proxyBeanMethods = false` ("Lite" mode)

## Prerequisites

- Chapter 04 — hiểu bean scope singleton.
- Chapter 05 — hiểu `BeanPostProcessor`, nền tảng cho cơ chế proxy nói chung.

## Used Later

- **Chapter 07 (Auto Configuration)** — các class `@Configuration` tự động của Spring Boot cũng
  tuân theo cùng cơ chế này.
- **Chapter 12 (AOP)** — CGLIB proxy là kỹ thuật nền tảng chung cho cả `@Configuration` lẫn AOP.

## Problem

Một class `@Configuration` là **mã Java thông thường** — gọi một method từ method khác trong
cùng class, về nguyên tắc ngôn ngữ Java thuần tuý, luôn thực thi lại toàn bộ thân method đó. Nếu
Spring không can thiệp gì, việc gọi `engine()` hai lần trong `car1()` và `car2()` phải tạo ra
**hai `Engine` riêng biệt** — vi phạm trực tiếp cam kết "chỉ một instance" của bean scope
`singleton` (Chapter 04).

## Concept

Mặc định, mọi class `@Configuration` được Spring **tạo một CGLIB subclass động** (proxy) tại
runtime — subclass này **override** mọi `@Bean` method, chèn logic: "trước khi thực sự chạy
thân method gốc, kiểm tra xem bean này đã tồn tại trong container chưa — nếu có rồi, trả về
instance đã có, **không** chạy lại thân method". Đây gọi là chế độ **"full" `@Configuration`**
(`proxyBeanMethods = true`, mặc định). Chế độ **"lite"** (`proxyBeanMethods = false`) tắt hoàn
toàn cơ chế này — gọi `@Bean` method trực tiếp hoạt động như một lời gọi method Java thông
thường, chạy lại thân method mỗi lần.

## Why?

CGLIB proxy cho phép lập trình viên viết code `@Configuration` theo phong cách Java tự nhiên
(gọi thẳng một method từ method khác, như bất kỳ class Java nào) mà **vẫn giữ đúng cam kết**
"singleton" của Spring — không cần lập trình viên phải tự nhớ "không được gọi trực tiếp, phải
khai báo tham số để Spring tiêm vào". Đây là một ví dụ về việc Spring dùng kỹ thuật proxy để
**điều chỉnh hành vi runtime** của code Java thông thường mà không cần lập trình viên viết code
khác đi so với trực giác tự nhiên.

## How?

```java
@Configuration // MAC DINH proxyBeanMethods = true -> "full" mode
class FullConfig {
    @Bean Engine engine() { return new Engine(); }
    @Bean Car car1() { return new Car(engine()); } // AN TOAN: engine() qua CGLIB proxy
}

@Configuration(proxyBeanMethods = false) // "lite" mode
class LiteConfig {
    @Bean Engine engine() { return new Engine(); }
    @Bean Car car1() { return new Car(engine()); } // NGUY HIEM: tao Engine MOI, khong qua container
}
```

## Visualization

```
@Configuration (full, MAC DINH):

  FullConfig thuc te la CGLIB subclass: FullConfig$$SpringCGLIB$$0

  car1() goi engine() ──► CGLIB CHAN LAI ──► "co Engine trong container chua?"
                                                 │
                                    Chua co ──► chay than method that, luu vao container
                                    Da co  ──► TRA VE luon, KHONG chay lai than method
  => car1.engine == car2.engine == container's Engine (CHI 1 instance)

@Configuration(proxyBeanMethods=false) (lite):

  LiteConfig la class GOC, KHONG proxy

  car1() goi engine() ──► goi THANG method Java binh thuong ──► LUON tao Engine MOI
  car2() goi engine() ──► goi THANG method Java binh thuong ──► LUON tao Engine MOI (KHAC)
  => car1.engine != car2.engine (2 instance KHAC NHAU, pha vo singleton!)
```

## Example

```java
@Configuration
static class FullConfig {
    @Bean Engine engine() { return new Engine(); }
    @Bean Car car1() { return new Car(engine()); }
    @Bean Car car2() { return new Car(engine()); }
}

@Configuration(proxyBeanMethods = false)
static class LiteConfig {
    @Bean Engine engine() { return new Engine(); }
    @Bean Car car1() { return new Car(engine()); }
    @Bean Car car2() { return new Car(engine()); }
}
```

Kết quả thật (Spring Framework 6.1.13):

```
=== @Configuration(proxyBeanMethods=false), Lite mode ===
Engine constructor chay!
Engine constructor chay!
Engine constructor chay!
car1.engine == car2.engine: false
Config class thuc te: com.handbook.demo.ConfigurationProxyTest$LiteConfig

=== @Configuration (full, proxyBeanMethods=true) ===
Engine constructor chay!
car1.engine == car2.engine: true
car1.engine == container's Engine bean: true
Config class thuc te: com.handbook.demo.ConfigurationProxyTest$FullConfig$$SpringCGLIB$$0
```

Với `LiteConfig`: `Engine constructor chay!` in ra **3 lần** (một cho `@Bean engine()` chính
thức, hai lần nữa cho các lời gọi trực tiếp trong `car1()`/`car2()`) — `car1.engine !=
car2.engine`, mỗi `Car` có một `Engine` hoàn toàn riêng, **phá vỡ** tính singleton dự kiến. Với
`FullConfig`: chỉ **một lần duy nhất**, `car1.engine == car2.engine` và cũng chính là bean
`Engine` đăng ký trong container — CGLIB proxy đã chặn các lời gọi trực tiếp, đảm bảo đúng
singleton. Class thực tế của `FullConfig` là `FullConfig$$SpringCGLIB$$0` — xác nhận đây thực
sự là một subclass động do CGLIB sinh ra, không phải `FullConfig` gốc.

## Deep Dive

**Vì sao CGLIB (không phải JDK dynamic proxy) được dùng cho `@Configuration`?** JDK dynamic
proxy (Phase 1, Chapter 19 — Reflection có nhắc gián tiếp qua khái niệm proxy) chỉ hoạt động
dựa trên **interface** — nó tạo một class implement cùng interface với đối tượng gốc. Class
`@Configuration` là một **class cụ thể** (không nhất thiết implement interface nào), nên JDK
dynamic proxy không áp dụng được. CGLIB tạo proxy bằng cách sinh ra một **subclass** kế thừa
trực tiếp từ chính class `@Configuration` đó, override từng `@Bean` method — đây là lý do một
class `@Configuration` **không được khai báo `final`** (subclass không thể kế thừa một class
`final`) và các `@Bean` method **không được khai báo `private`/`final`/`static`** (không override
được).

## Engineering Insight

**"Lite" mode (`proxyBeanMethods = false`) tồn tại để làm gì, nếu "full" mode an toàn hơn?**
Tạo CGLIB proxy có **chi phí runtime** (thời gian sinh class động lúc khởi động, một chút overhead
mỗi lần gọi method do phải đi qua logic kiểm tra container). Với các class `@Configuration`
**không** gọi chéo `@Bean` method lẫn nhau (mỗi `@Bean` độc lập hoàn toàn, dependency được tiêm
qua tham số thay vì gọi trực tiếp — đây thực ra là cách viết được khuyến nghị nói chung), việc
tạo proxy là lãng phí không cần thiết. Đây chính là lý do Spring Boot 2.2+ **tự động** dùng "lite"
mode cho các class `@Configuration` không cần proxy (`@Configuration(proxyBeanMethods = false)`
mặc định trong nhiều auto-configuration nội bộ của Spring Boot, Chapter 07) để tối ưu thời gian
khởi động — một chi tiết hiệu năng nhỏ nhưng có ý nghĩa đáng kể khi một ứng dụng có hàng trăm
class cấu hình.

## Historical Note

Chế độ "lite" `@Configuration` (`proxyBeanMethods = false`) được thêm vào Spring Framework 5.2
(2019), cùng thời điểm cộng đồng Spring bắt đầu tối ưu mạnh cho thời gian khởi động (khởi động
nhanh trở thành tiêu chí quan trọng hơn với xu hướng container hoá, serverless, nơi thời gian
cold-start ảnh hưởng trực tiếp chi phí vận hành).

## Myth vs Reality

- **Myth:** "Gọi trực tiếp một `@Bean` method từ method khác luôn là lỗi thiết kế, phải luôn
  tiêm qua tham số."
  **Reality:** Với "full" mode (mặc định), gọi trực tiếp vẫn đúng đắn nhờ CGLIB proxy — đã
  chứng minh bằng thực nghiệm. Tuy nhiên tiêm qua tham số vẫn là cách viết rõ ràng hơn, không
  phụ thuộc ngầm vào cơ chế proxy.

- **Myth:** "`@Configuration(proxyBeanMethods = false)` chỉ là một tối ưu hiệu năng nhỏ, không
  ảnh hưởng gì tới tính đúng đắn."
  **Reality:** Đã chứng minh bằng thực nghiệm — nó có thể **phá vỡ** tính singleton nếu code
  vẫn gọi trực tiếp `@Bean` method lẫn nhau, dẫn tới lỗi khó phát hiện (nhiều instance thay vì
  một).

## Common Mistakes

- **Đổi `proxyBeanMethods = false` cho một class `@Configuration` đang gọi chéo `@Bean` method
  mà không kiểm tra tác động** — âm thầm phá vỡ tính singleton, gây lỗi khó debug (nhiều
  instance của cùng một bean logic).
- **Khai báo `@Configuration` class là `final`** — CGLIB proxy cần tạo subclass, không thể với
  class `final`; Spring sẽ báo lỗi hoặc bỏ qua tối ưu proxy.
- **Khai báo `@Bean` method là `private`/`final`** — CGLIB không override được, phá vỡ cơ chế
  proxy.

## Best Practices

- Ưu tiên tiêm dependency qua tham số method thay vì gọi trực tiếp `@Bean` method khác, dù "full"
  mode vẫn đúng — cách viết này rõ ràng hơn, không phụ thuộc ngầm vào cơ chế proxy.
- Chỉ chuyển sang `proxyBeanMethods = false` khi chắc chắn class `@Configuration` đó không gọi
  chéo `@Bean` method lẫn nhau.
- Không khai báo class `@Configuration` là `final`, không khai báo `@Bean` method là
  `private`/`final`/`static` (trừ trường hợp đặc biệt cần bean tĩnh, ví dụ
  `BeanPostProcessor`/`BeanFactoryPostProcessor`, cần khai báo `static` theo tài liệu Spring).

## Debug Checklist

- [ ] Một bean tưởng là singleton nhưng có nhiều instance khác nhau khi debug? → kiểm tra class
      `@Configuration` chứa nó có `proxyBeanMethods = false` không, và có đang gọi chéo `@Bean`
      method không.
- [ ] Lỗi liên quan tới CGLIB khi class `@Configuration` là `final` hoặc `@Bean` method là
      `private`? → bỏ `final`/`private` trên class/method đó.

## Summary

`@Configuration` mặc định (`proxyBeanMethods = true`, "full" mode) được Spring bọc trong một
CGLIB subclass động — đã chứng minh bằng thực nghiệm: class thực tế trở thành
`FullConfig$$SpringCGLIB$$0`, và gọi trực tiếp `@Bean` method nhiều lần chỉ chạy thân method
đúng một lần, đảm bảo singleton. `proxyBeanMethods = false` ("lite" mode) tắt cơ chế này — gọi
trực tiếp trở thành lời gọi Java thông thường, tạo instance mới mỗi lần, **phá vỡ** singleton
nếu code vẫn gọi chéo `@Bean` method (đã chứng minh: 3 lần constructor chạy thay vì 1, hai
`Car` có `Engine` khác nhau). CGLIB được chọn (thay vì JDK dynamic proxy) vì nó proxy được class
cụ thể, không cần interface.

## Interview Questions

**Senior**

- Vì sao gọi trực tiếp một `@Bean` method từ một `@Bean` method khác trong cùng `@Configuration`
  class vẫn đảm bảo tính singleton?
- `proxyBeanMethods = false` thay đổi điều gì? Khi nào nên dùng nó?
- Vì sao class `@Configuration` không được khai báo `final`, và `@Bean` method không được khai
  báo `private`?

## Exercises

- [ ] Chạy lại `ConfigurationProxyTest` ở trên, xác nhận kết quả đúng như mô tả cho cả "full" và
      "lite" mode.
- [ ] Thử khai báo một `@Configuration` class là `final`, quan sát Spring xử lý (báo lỗi hoặc
      bỏ qua tối ưu, tuỳ phiên bản).
- [ ] Dùng IDE hoặc `getClass().getName()` để tự xác nhận một `@Configuration` bean thực tế là
      một CGLIB subclass, không phải class gốc.

## Cheat Sheet

| | `proxyBeanMethods = true` (mặc định) | `proxyBeanMethods = false` |
| --- | --- | --- |
| Gọi trực tiếp `@Bean` method | An toàn, CGLIB chặn lại, trả về instance đã có | Chạy như method Java thường, tạo mới mỗi lần |
| Đảm bảo singleton khi gọi chéo | Có | Không |
| Chi phí runtime | Cao hơn (tạo proxy, kiểm tra container mỗi lần gọi) | Thấp hơn |
| Class thực tế | CGLIB subclass (`...$$SpringCGLIB$$0`) | Class gốc |
| Yêu cầu | Class/method không `final`/`private` | Không |

## References

- Spring Framework Documentation — `@Configuration`: https://docs.spring.io/spring-framework/reference/core/beans/java/configuration-annotation.html
- Spring Framework Javadoc — `@Configuration(proxyBeanMethods)`.
