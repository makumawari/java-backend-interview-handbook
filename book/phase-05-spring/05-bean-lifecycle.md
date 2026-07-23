---
tags:
  - Spring
  - BeanLifecycle
---

# Bean Lifecycle

> Phase: Phase 5 — Spring
> Chapter slug: `bean-lifecycle`

## Metadata

```yaml
Chapter: Bean Lifecycle
Phase: Phase 5 — Spring
Difficulty: ★★★★
Importance: ★★★★
Interview Frequency: 60%
Prerequisites:
  - Chapter 04 — Bean
Used Later:
  - Chapter 12 — AOP (proxy được tạo trong giai đoạn initialization)
Estimated Reading: 15 phút
Estimated Practice: 12 phút
```

## Story

> Xem lại: [Chapter 04](04-bean.md) — một bean singleton chỉ được tạo một lần. Nhưng "tạo" thực
> ra gồm nhiều **bước riêng biệt**: gọi constructor, tiêm dependency, rồi mới thực sự sẵn sàng
> dùng. Có tới **3 cách khác nhau** để "chạy logic khi bean vừa khởi tạo xong"
> (`@PostConstruct`, `InitializingBean`, `init-method` tuỳ chỉnh) — chúng có chạy theo thứ tự
> nào?

Đăng ký cả 3 cách cùng lúc trên một bean, kèm một `BeanPostProcessor` quan sát "trước" và "sau"
giai đoạn khởi tạo:

```
=== KHOI TAO ===
1. Constructor
0. BeanPostProcessor.postProcessBeforeInitialization
2. @PostConstruct
3. InitializingBean.afterPropertiesSet()
4. custom init-method (khai bao trong @Bean)
5. BeanPostProcessor.postProcessAfterInitialization
=== DONG CONTEXT ===
A. @PreDestroy
B. DisposableBean.destroy()
C. custom destroy-method
```

Thứ tự cố định, lặp lại đúng như vậy mỗi lần chạy — không phải ngẫu nhiên.

## Interview Question (Central)

> Vòng đời của một bean trong Spring gồm những giai đoạn nào? `@PostConstruct`,
> `InitializingBean`, và `init-method` tuỳ chỉnh chạy theo thứ tự nào nếu dùng cùng lúc?

## Objectives

- [ ] Liệt kê đúng thứ tự các bước trong vòng đời khởi tạo và huỷ bean
- [ ] Tự tay chứng minh bằng thực nghiệm thứ tự thực thi của `BeanPostProcessor`,
      `@PostConstruct`, `InitializingBean`, `init-method`
- [ ] Hiểu vai trò của `BeanPostProcessor` — điểm móc nối (hook) mà nhiều tính năng nâng cao của
      Spring (bao gồm AOP, Chapter 12) dựa vào

## Prerequisites

- Chapter 04 — hiểu bean và scope, nền tảng để hiểu "khởi tạo" một bean gồm những gì.

## Used Later

- **Chapter 12 (AOP)** — cơ chế tạo proxy (bọc bean gốc để thêm hành vi) diễn ra chính xác trong
  giai đoạn `postProcessAfterInitialization` của một `BeanPostProcessor` đặc biệt.

## Problem

"Tạo bean" nghe có vẻ là một hành động đơn giản, nhưng thực tế cần nhiều bước tách biệt: tạo
object trần (constructor), điền dependency (DI, Chapter 03), rồi mới thực sự "khởi tạo hoàn tất"
(chạy logic setup cần dependency đã sẵn sàng — ví dụ mở kết nối, load cache ban đầu). Nếu không
hiểu rõ thứ tự các bước này, dễ viết logic khởi tạo sai chỗ — ví dụ đặt logic cần dependency vào
constructor (khi dependency **chưa** được tiêm xong với field injection).

## Concept

**Bean Lifecycle** là chuỗi các giai đoạn Spring container thực hiện khi tạo và huỷ một bean:
**Instantiation** (gọi constructor) → **Populate properties** (tiêm dependency) →
**`BeanPostProcessor.postProcessBeforeInitialization`** → **Initialization**
(`@PostConstruct` → `InitializingBean.afterPropertiesSet()` → `init-method` tuỳ chỉnh) →
**`BeanPostProcessor.postProcessAfterInitialization`** → bean sẵn sàng dùng → (khi container
đóng) **Destruction** (`@PreDestroy` → `DisposableBean.destroy()` → `destroy-method` tuỳ chỉnh).

## Why?

Tách rõ "tạo object" (constructor) khỏi "khởi tạo hoàn tất" (initialization callbacks) giải
quyết đúng vấn đề đã nêu ở Problem: constructor chỉ nên tạo object ở trạng thái tối thiểu (chưa
chắc đã có mọi dependency, đặc biệt với field injection — Chapter 03), còn logic setup cần
dependency đầy đủ (ví dụ validate cấu hình, mở kết nối) nên đặt ở giai đoạn initialization — nơi
Spring **đảm bảo** mọi dependency đã được tiêm xong. `BeanPostProcessor` tồn tại như một điểm
móc nối (hook) tổng quát, cho phép framework (hoặc chính lập trình viên) can thiệp vào **mọi**
bean tại đúng thời điểm trước/sau khởi tạo — đây chính là nền tảng kỹ thuật mà nhiều tính năng
nâng cao của Spring (AOP, Chapter 12) dựa vào để hoạt động mà không cần sửa code của từng bean
riêng lẻ.

## How?

```java
@Component
class LifecycleBean implements InitializingBean, DisposableBean {
    LifecycleBean() { /* constructor: object TOI THIEU */ }

    @PostConstruct
    void postConstruct() { /* chay SAU khi dependency da duoc tiem */ }

    @Override
    public void afterPropertiesSet() { /* tuong tu @PostConstruct, cach cu hon */ }

    @Override
    public void destroy() { /* don dep khi container dong */ }
}

@Bean(initMethod = "customInit", destroyMethod = "customDestroy") // cach thu 3, khai bao trong @Bean
```

## Visualization

```
KHOI TAO (thu tu CO DINH):

  1. Constructor
  2. Tiem dependency (DI)
  3. BeanPostProcessor.postProcessBeforeInitialization  <- TRUOC
  4. @PostConstruct
  5. InitializingBean.afterPropertiesSet()
  6. init-method tuy chinh (khai bao trong @Bean)
  7. BeanPostProcessor.postProcessAfterInitialization   <- SAU (AOP proxy tao o day!)
  8. Bean SAN SANG dung

HUY (khi container dong, chi ap dung cho singleton):

  A. @PreDestroy
  B. DisposableBean.destroy()
  C. destroy-method tuy chinh
```

## Example

```java
static class LifecycleBean implements InitializingBean, DisposableBean {
    LifecycleBean() { System.out.println("1. Constructor"); }

    @PostConstruct
    void postConstruct() { System.out.println("2. @PostConstruct"); }

    public void afterPropertiesSet() { System.out.println("3. InitializingBean.afterPropertiesSet()"); }

    void customInit() { System.out.println("4. custom init-method"); }

    @PreDestroy
    void preDestroy() { System.out.println("A. @PreDestroy"); }

    public void destroy() { System.out.println("B. DisposableBean.destroy()"); }

    void customDestroy() { System.out.println("C. custom destroy-method"); }
}

static class LoggingBeanPostProcessor implements BeanPostProcessor {
    public Object postProcessBeforeInitialization(Object bean, String name) {
        if (bean instanceof LifecycleBean) System.out.println("0. BeanPostProcessor.before");
        return bean;
    }
    public Object postProcessAfterInitialization(Object bean, String name) {
        if (bean instanceof LifecycleBean) System.out.println("5. BeanPostProcessor.after");
        return bean;
    }
}

@Configuration
static class Config {
    @Bean static LoggingBeanPostProcessor bpp() { return new LoggingBeanPostProcessor(); }
    @Bean(initMethod = "customInit", destroyMethod = "customDestroy")
    LifecycleBean lifecycleBean() { return new LifecycleBean(); }
}
```

Kết quả thật (Spring Framework 6.1.13):

```
=== KHOI TAO ===
1. Constructor
0. BeanPostProcessor.postProcessBeforeInitialization
2. @PostConstruct
3. InitializingBean.afterPropertiesSet()
4. custom init-method (khai bao trong @Bean)
5. BeanPostProcessor.postProcessAfterInitialization
=== DONG CONTEXT ===
A. @PreDestroy
B. DisposableBean.destroy()
C. custom destroy-method
```

Thứ tự khởi tạo xác nhận chính xác: `BeanPostProcessor.before` chạy **trước** cả
`@PostConstruct`; và trong 3 cách "init callback", `@PostConstruct` chạy trước
`InitializingBean.afterPropertiesSet()`, rồi mới tới `init-method` tuỳ chỉnh khai báo trong
`@Bean`. Thứ tự huỷ cũng theo đúng mẫu tương tự: `@PreDestroy` → `DisposableBean.destroy()` →
`destroy-method` tuỳ chỉnh.

## Deep Dive

**Vì sao có tới 3 cách khai báo "init callback" (`@PostConstruct`, `InitializingBean`,
`init-method`) — chúng có lịch sử phát triển khác nhau ra sao, và thứ tự cố định đó phản ánh
điều gì?** `InitializingBean` (interface) là cách **cũ nhất**, có từ những phiên bản Spring đầu
tiên — nhược điểm là bắt buộc class phải implement một interface của Spring (coupling trực tiếp
vào framework). `init-method` (khai bao qua XML hoặc `@Bean(initMethod=...)`) ra đời để tránh
coupling đó — cho phép dùng bất kỳ phương thức nào, không cần implement interface, nhưng phải
khai báo tên phương thức riêng biệt khỏi chính class đó. `@PostConstruct` (chuẩn Java EE/Jakarta,
không phải riêng của Spring) ra đời sau, tận dụng annotation để đánh dấu ngay trên phương thức mà
không cần coupling vào Spring hay khai báo tách rời — đây là lý do nó chạy **đầu tiên** trong 3
cách: về mặt triết lý thiết kế, `@PostConstruct` được ưu tiên là "chuẩn hiện đại", trong khi
`InitializingBean`/`init-method` được giữ lại chủ yếu vì tương thích ngược với codebase cũ.

## Engineering Insight

**`BeanPostProcessor` là nền tảng kỹ thuật cho AOP (Chapter 12) như thế nào?** Khi Spring tạo
proxy để thêm hành vi cho một bean (ví dụ logging tự động, transaction tự động — Chapter 13),
proxy đó **không** được tạo trong constructor hay trong `@PostConstruct` của chính bean — nó
được tạo bởi một `BeanPostProcessor` đặc biệt
(`AnnotationAwareAspectJAutoProxyCreator`) tại đúng bước
`postProcessAfterInitialization` (bước số 5 trong Example) — **sau khi** bean gốc đã hoàn toàn
khởi tạo xong. Bean trả về từ `postProcessAfterInitialization` có thể là một object **khác hoàn
toàn** với bean gốc (một proxy bọc quanh bean gốc) — đây chính là lý do khi bạn `@Autowired` một
bean có AOP, bạn thực chất nhận về proxy, không phải instance gốc do chính constructor tạo ra —
kiến thức nền tảng bắt buộc phải hiểu trước khi học AOP.

## Historical Note

`@PostConstruct`/`@PreDestroy` là annotation chuẩn từ JSR-250 (Common Annotations for the Java
Platform, 2006) — không phải annotation riêng của Spring, cũng được các framework Java EE khác
hỗ trợ. Từ Java 9 trở đi, các annotation này bị loại khỏi JDK core (do module hoá — Phase 1,
Chapter 22), chuyển sang gói `javax.annotation-api` riêng, và từ Spring Boot 3
(Jakarta EE 9+) đổi thành `jakarta.annotation.PostConstruct`.

## Myth vs Reality

- **Myth:** "Chỉ cần dùng một trong 3 cách init callback là đủ, thứ tự giữa chúng không quan
  trọng vì hiếm khi dùng chung."
  **Reality:** Đúng là hiếm khi cần cả 3 cùng lúc trong thực tế, nhưng hiểu thứ tự chính xác
  quan trọng khi đọc code kế thừa từ một lớp cha đã dùng một cách, còn lớp con dùng cách khác —
  tình huống này xảy ra thường xuyên hơn tưởng tượng trong codebase lớn, lâu năm.

- **Myth:** "`@Autowired` một bean luôn trả về đúng instance gốc do constructor tạo ra."
  **Reality:** Xem Engineering Insight — nếu bean đó có AOP áp dụng, bạn nhận về một proxy, khác
  với instance gốc — điểm này sẽ trở nên cực kỳ quan trọng khi học self-invocation limitation ở
  Chapter 12-14.

## Common Mistakes

- **Đặt logic cần dependency đầy đủ vào constructor thay vì `@PostConstruct`** — với field
  injection, dependency chưa chắc đã sẵn sàng lúc constructor chạy (constructor chạy trước bước
  populate properties).
- **Quên rằng `@PreDestroy` chỉ chạy với bean scope `singleton`** — bean `prototype` (Chapter
  04) không được container quản lý việc huỷ, `@PreDestroy` sẽ không được gọi tự động.
- **Nhầm lẫn thứ tự `BeanPostProcessor` với init callback** — `postProcessBeforeInitialization`
  chạy **trước** mọi init callback, `postProcessAfterInitialization` chạy **sau** tất cả.

## Debug Checklist

- [ ] Logic trong constructor báo lỗi `NullPointerException` liên quan tới một dependency được
      field-inject? → chuyển logic đó sang `@PostConstruct`, vì dependency chưa chắc sẵn sàng
      lúc constructor chạy.
- [ ] `@PreDestroy` không được gọi khi mong đợi? → kiểm tra scope của bean có phải `prototype`
      không — Spring không quản lý việc huỷ bean prototype.
- [ ] Bean nhận được qua `@Autowired` có hành vi khác instance gốc (ví dụ có thêm log tự động)?
      → có thể đây là proxy do `BeanPostProcessor` tạo (AOP, Chapter 12), không phải instance
      gốc.

## Summary

Vòng đời bean gồm các giai đoạn cố định: constructor → tiêm dependency →
`BeanPostProcessor.before` → init callback (`@PostConstruct` → `InitializingBean` →
`init-method`, theo đúng thứ tự này) → `BeanPostProcessor.after` → sẵn sàng dùng → (khi đóng
container) `@PreDestroy` → `DisposableBean.destroy()` → `destroy-method`. Đã chứng minh bằng
thực nghiệm đúng thứ tự này. `BeanPostProcessor` là nền tảng kỹ thuật cho AOP — proxy được tạo
tại bước `postProcessAfterInitialization`, nghĩa là bean bạn nhận qua `@Autowired` có thể là
proxy, không phải instance gốc do constructor tạo ra.

## Interview Questions

**Mid**

- Vòng đời của một bean trong Spring gồm những giai đoạn nào?
- `@PostConstruct`, `InitializingBean`, và `init-method` tuỳ chỉnh chạy theo thứ tự nào nếu dùng
  cùng lúc?

**Senior**

- Giải thích vai trò của `BeanPostProcessor` trong việc Spring hiện thực hoá AOP. Vì sao bean
  nhận được qua `@Autowired` có thể không phải là instance gốc do constructor tạo ra?

## Exercises

- [ ] Chạy lại `BeanLifecycleTest` ở trên, xác nhận đúng thứ tự các bước khởi tạo và huỷ.
- [ ] Viết một `BeanPostProcessor` tuỳ chỉnh in ra tên class của mọi bean được tạo trong
      container, quan sát thứ tự các bean được khởi tạo.
- [ ] Thử tạo một bean scope `prototype` với `@PreDestroy`, xác nhận nó không được gọi khi
      container đóng (khác với singleton).

## Cheat Sheet

| Bước | Thứ tự | Ghi chú |
| --- | --- | --- |
| Constructor | 1 | Object tối thiểu, dependency chưa chắc sẵn sàng (field injection) |
| Tiêm dependency | 2 | DI hoàn tất |
| `BeanPostProcessor.before` | 3 | Trước mọi init callback |
| `@PostConstruct` | 4 | Chuẩn JSR-250, ưu tiên dùng |
| `InitializingBean.afterPropertiesSet()` | 5 | Cách cũ, coupling vào Spring |
| `init-method` tuỳ chỉnh | 6 | Khai báo qua `@Bean(initMethod=...)` |
| `BeanPostProcessor.after` | 7 | AOP proxy được tạo ở đây |
| `@PreDestroy` | Huỷ, bước 1 | Chỉ với singleton |
| `DisposableBean.destroy()` | Huỷ, bước 2 | Chỉ với singleton |
| `destroy-method` tuỳ chỉnh | Huỷ, bước 3 | Chỉ với singleton |

## References

- Spring Framework Documentation — Bean Lifecycle: https://docs.spring.io/spring-framework/reference/core/beans/factory-nature.html
- JSR-250 — Common Annotations for the Java Platform.
