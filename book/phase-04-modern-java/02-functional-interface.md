---
tags:
  - Java
  - FunctionalInterface
  - ModernJava
---

# Functional Interface

> Phase: Phase 4 — Modern Java
> Chapter slug: `functional-interface`

## Metadata

```yaml
Chapter: Functional Interface
Phase: Phase 4 — Modern Java
Difficulty: ★★
Importance: ★★★★
Interview Frequency: 70%
Prerequisites:
  - Chapter 01 — Lambda
Used Later:
  - Chapter 04 — Stream
  - Chapter 05 — Collector
Estimated Reading: 12 phút
Estimated Practice: 12 phút
```

## Story

> Xem lại: [Chapter 01](01-lambda.md) — lambda expression phải được gán cho **một kiểu cụ thể**.
> Nhưng kiểu đó phải thoả điều kiện gì để trình biên dịch chấp nhận một lambda gán vào nó?

Thử khai báo một interface có **hai** phương thức trừu tượng, đánh dấu `@FunctionalInterface`:

```java
@FunctionalInterface
interface TwoMethodInterface {
    void method1();
    void method2();
}
```

Biên dịch thử:

```
error: Unexpected @FunctionalInterface annotation
  TwoMethodInterface is not a functional interface
    multiple non-overriding abstract methods found in interface TwoMethodInterface
```

Trình biên dịch **từ chối ngay lập tức** — không đợi tới lúc gán lambda mới báo lỗi.
`@FunctionalInterface` không chỉ là tài liệu, nó là một ràng buộc được kiểm tra thật.

## Interview Question (Central)

> `@FunctionalInterface` là gì? Điều kiện gì để một interface được coi là functional interface?

## Objectives

- [ ] Giải thích chính xác điều kiện: đúng một phương thức trừu tượng (default/static/method từ
      `Object` không tính)
- [ ] Tự tay chứng minh lỗi biên dịch khi interface có 2 phương thức trừu tượng nhưng vẫn gắn
      `@FunctionalInterface`
- [ ] Dùng thành thạo các functional interface chuẩn của JDK: `Supplier`, `Consumer`,
      `Function`, `Predicate`, `BiFunction`

## Prerequisites

- Chapter 01 — hiểu lambda expression, cú pháp cơ bản.

## Used Later

- **Chapter 04 (Stream)**, **Chapter 05 (Collector)** — hầu hết API của Stream nhận tham số kiểu
  `Function`, `Predicate`, `Consumer`, ...

## Problem

Lambda expression cần một "khuôn" để trình biên dịch biết nó implement interface nào, phương
thức nào, chữ ký tham số/kiểu trả về ra sao. Nếu một interface có **nhiều hơn một** phương thức
trừu tượng, trình biên dịch không thể xác định lambda đang implement phương thức nào trong số
đó — mơ hồ không thể giải quyết.

## Concept

**Functional interface** là một interface có **đúng một phương thức trừu tượng** (SAM — Single
Abstract Method). `@FunctionalInterface` là annotation tuỳ chọn nhưng được khuyến nghị — nó yêu
cầu trình biên dịch **kiểm tra và báo lỗi ngay khi biên dịch** nếu interface không thoả điều
kiện SAM, thay vì để lỗi xuất hiện muộn hơn (khi ai đó cố gán lambda vào một interface không phù
hợp).

## Why?

Ràng buộc "đúng một phương thức trừu tượng" chính là điều kiện toán học cần thiết để cú pháp
lambda (`(tham_so) -> than_logic`) có thể ánh xạ **không mơ hồ** vào đúng một phương thức. Việc
đưa `@FunctionalInterface` thành annotation kiểm tra được (thay vì chỉ là quy ước không chính
thức) giúp phát hiện lỗi thiết kế **sớm nhất có thể** — ngay tại nơi định nghĩa interface, không
phải đợi tới khi có người dùng cố gán lambda và gặp lỗi khó hiểu ở một nơi hoàn toàn khác trong
codebase.

## How?

```java
@FunctionalInterface
interface Validator<T> {
    boolean isValid(T t);

    // default method KHONG tinh vao SAM, van hop le
    default Validator<T> and(Validator<T> other) {
        return t -> this.isValid(t) && other.isValid(t);
    }
}
```

## Visualization

```
Interface hop le (SAM - Single Abstract Method):

  interface Validator<T> {
      boolean isValid(T t);     <- CHI 1 phuong thuc TRUU TUONG
      default Validator<T> and(...) { ... }  <- default KHONG tinh
  }
  => @FunctionalInterface OK, lambda gan duoc

Interface KHONG hop le:

  interface TwoMethodInterface {
      void method1();   <- abstract #1
      void method2();   <- abstract #2   => MO HO, khong biet lambda la cai nao
  }
  => @FunctionalInterface BAO LOI BIEN DICH ngay lap tuc
```

## Example

```java
import java.util.function.*;

public class FunctionalInterfaceDemo {
    @FunctionalInterface
    interface Validator<T> {
        boolean isValid(T t);
        default Validator<T> and(Validator<T> other) {
            return t -> this.isValid(t) && other.isValid(t);
        }
    }

    public static void main(String[] args) {
        Validator<String> notEmpty = s -> !s.isEmpty();
        Validator<String> maxLen10 = s -> s.length() <= 10;
        Validator<String> combined = notEmpty.and(maxLen10);

        System.out.println("combined.isValid(\"\"): " + combined.isValid(""));
        System.out.println("combined.isValid(\"hello\"): " + combined.isValid("hello"));
        System.out.println("combined.isValid(\"qua dai qua dai\"): " + combined.isValid("qua dai qua dai"));

        Supplier<String> supplier = () -> "gia tri tao moi";
        Consumer<String> consumer = s -> System.out.println("Consumer nhan: " + s);
        Function<Integer, String> function = n -> "So la: " + n;
        Predicate<Integer> predicate = n -> n % 2 == 0;
        BiFunction<Integer, Integer, Integer> biFunction = (a, b) -> a * b;

        System.out.println("Supplier: " + supplier.get());
        consumer.accept("test");
        System.out.println("Function: " + function.apply(42));
        System.out.println("Predicate(4 chan?): " + predicate.test(4));
        System.out.println("BiFunction(3*5): " + biFunction.apply(3, 5));
    }
}
```

Kết quả thật (JDK 17):

```
combined.isValid(""): false
combined.isValid("hello"): true
combined.isValid("qua dai qua dai"): false
Supplier: gia tri tao moi
Consumer nhan: test
Function: So la: 42
Predicate(4 chan?): true
BiFunction(3*5): 15
```

`combined = notEmpty.and(maxLen10)` kết hợp đúng hai validator: chuỗi rỗng thất bại (không thoả
`notEmpty`), chuỗi "hello" (5 ký tự) hợp lệ, chuỗi dài hơn 10 ký tự thất bại (không thoả
`maxLen10`) — chứng minh `and()` (default method) hoạt động đúng dù `Validator` chỉ có một
phương thức trừu tượng.

Và với interface vi phạm điều kiện SAM:

```java
@FunctionalInterface
interface TwoMethodInterface {
    void method1();
    void method2();
}
```

Kết quả biên dịch thật:

```
TwoMethodInterface.java:1: error: Unexpected @FunctionalInterface annotation
@FunctionalInterface
^
  TwoMethodInterface is not a functional interface
    multiple non-overriding abstract methods found in interface TwoMethodInterface
1 error
```

## Deep Dive

**Vì sao default method và static method không tính vào điều kiện SAM, nhưng phương thức trừu
tượng trùng tên với `Object` (như `equals`, `hashCode`, `toString`) cũng không tính?** Default
method và static method đã có **sẵn phần thân** (implementation) — chúng không cần lambda cung
cấp logic, nên không góp phần vào sự mơ hồ. Phương thức trừu tượng trùng chữ ký với các phương
thức public của `Object` (`equals(Object)`, `hashCode()`, `toString()`) được coi là "đã có cách
implement" vì **mọi class Java đều kế thừa `Object`**, nên bất kỳ implementation nào của
interface (kể cả implementation ẩn sinh ra cho lambda) cũng tự động có sẵn các phương thức đó —
compiler loại chúng ra khỏi việc đếm SAM vì về bản chất chúng không cần lambda "điền vào".

## Engineering Insight

**Vì sao nên luôn gắn `@FunctionalInterface` dù về mặt kỹ thuật interface vẫn hoạt động được
nếu thiếu nó (miễn thực sự chỉ có một phương thức trừu tượng)?** Không có annotation, không có
gì ngăn một đồng nghiệp sau này vô tình thêm phương thức trừu tượng thứ hai vào interface — lỗi
chỉ xuất hiện ở nơi **hoàn toàn khác** (nơi có người cố gán lambda vào interface đó), với thông
điệp lỗi khó liên hệ ngược lại nguyên nhân gốc. `@FunctionalInterface` biến ràng buộc thiết kế
"interface này PHẢI mãi mãi chỉ có một phương thức trừu tượng" thành một điều kiện được compiler
**tự động bảo vệ** ngay tại nơi định nghĩa — đúng nguyên tắc "fail fast, fail near the source"
(báo lỗi sớm, càng gần nguồn gốc vấn đề càng tốt).

## Historical Note

`@FunctionalInterface` ra đời cùng Java 8 (2014), cùng lambda và Stream API. JDK 8 đồng thời bổ
sung gói `java.util.function` với hơn 40 functional interface chuẩn (`Function`, `BiFunction`,
`Supplier`, `Consumer`, `Predicate`, và các biến thể chuyên biệt cho primitive như
`IntFunction`, `ToIntFunction` để tránh autoboxing) — giảm nhu cầu tự định nghĩa interface riêng
cho hầu hết trường hợp thông dụng.

## Myth vs Reality

- **Myth:** "`@FunctionalInterface` là annotation bắt buộc để một interface dùng được với
  lambda."
  **Reality:** Không bắt buộc về mặt kỹ thuật — bất kỳ interface nào thoả điều kiện SAM (một
  phương thức trừu tượng) đều dùng được với lambda dù có annotation hay không.
  `@FunctionalInterface` chỉ thêm một lớp kiểm tra biên dịch để bảo vệ ràng buộc này.

- **Myth:** "Interface có nhiều default method thì không còn là functional interface."
  **Reality:** Xem Example — số lượng default/static method không giới hạn, chỉ có đúng MỘT
  phương thức trừu tượng là điều kiện duy nhất.

## Common Mistakes

- **Thêm phương thức trừu tượng thứ hai vào một functional interface đang được dùng rộng rãi mà
  quên gắn `@FunctionalInterface` để compiler cảnh báo sớm** — dẫn tới lỗi biên dịch rải rác ở
  nhiều nơi dùng interface đó.
- **Tự định nghĩa functional interface trùng chức năng với interface chuẩn có sẵn trong
  `java.util.function`** — không cần thiết, nên kiểm tra gói này trước khi tự viết.
- **Dùng `Function<Integer, Integer>` cho các phép toán trên `int` thay vì
  `IntUnaryOperator`/`IntFunction`** — gây autoboxing/unboxing không cần thiết, ảnh hưởng hiệu
  năng khi gọi số lượng lớn.

## Best Practices

- Luôn gắn `@FunctionalInterface` khi tự định nghĩa một functional interface, để compiler bảo
  vệ ràng buộc thiết kế.
- Ưu tiên dùng functional interface chuẩn của JDK (`Function`, `Predicate`, `Consumer`,
  `Supplier`, ...) trước khi tự định nghĩa interface mới.
- Với các phép toán trên kiểu nguyên thủy, dùng biến thể chuyên biệt (`IntPredicate`,
  `ToLongFunction`, ...) để tránh chi phí autoboxing.

## Debug Checklist

- [ ] Lỗi "multiple non-overriding abstract methods found"? → kiểm tra interface có đúng một
      phương thức trừu tượng không (default/static/phương thức trùng `Object` không tính).
- [ ] Không chắc nên dùng functional interface chuẩn nào? → tra `java.util.function`: cần trả
      về giá trị và nhận tham số → `Function`; chỉ nhận tham số không trả về → `Consumer`; chỉ
      trả về không nhận tham số → `Supplier`; trả về `boolean` → `Predicate`.

## Summary

Functional interface là interface có đúng một phương thức trừu tượng (SAM) — điều kiện bắt buộc
để lambda có thể ánh xạ không mơ hồ vào nó. `@FunctionalInterface` không phải annotation trang
trí — đã chứng minh bằng thực nghiệm: gắn nó vào một interface có 2 phương thức trừu tượng gây
lỗi biên dịch ngay lập tức, giúp phát hiện lỗi thiết kế sớm thay vì để lỗi xuất hiện mơ hồ ở nơi
khác. default method và static method không tính vào điều kiện SAM. JDK cung cấp sẵn hơn 40
functional interface chuẩn trong `java.util.function`, nên ưu tiên dùng trước khi tự định
nghĩa.

## Interview Questions

- What is a @FunctionalInterface?

**Mid**

- Điều kiện gì để một interface được coi là functional interface? default method có tính vào
  điều kiện đó không?

## Exercises

- [ ] Chạy lại `FunctionalInterfaceDemo` và thử biên dịch `TwoMethodInterface` ở trên, xác nhận
      lỗi biên dịch đúng như mô tả.
- [ ] Tự viết một functional interface `TriFunction<A, B, C, R>` (nhận 3 tham số, trả về 1 giá
      trị) — kiểu chưa có sẵn trong `java.util.function`.
- [ ] Thử thêm một phương thức trùng chữ ký với `Object.toString()` vào một functional
      interface đã có, xác nhận nó vẫn biên dịch được (không tính vào SAM).

## Cheat Sheet

| Interface chuẩn | Chữ ký | Dùng khi |
| --- | --- | --- |
| `Supplier<T>` | `T get()` | Chỉ cần tạo/trả về giá trị |
| `Consumer<T>` | `void accept(T t)` | Chỉ cần thực hiện hành động với giá trị |
| `Function<T,R>` | `R apply(T t)` | Biến đổi giá trị T thành R |
| `Predicate<T>` | `boolean test(T t)` | Kiểm tra điều kiện, trả về boolean |
| `BiFunction<T,U,R>` | `R apply(T t, U u)` | Biến đổi 2 giá trị thành 1 kết quả |

## References

- JLS §9.8 — Functional Interfaces.
- Java SE API Documentation — `java.util.function`.
