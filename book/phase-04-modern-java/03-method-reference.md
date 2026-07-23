---
tags:
  - Java
  - MethodReference
  - ModernJava
---

# Method Reference

> Phase: Phase 4 — Modern Java
> Chapter slug: `method-reference`

## Metadata

```yaml
Chapter: Method Reference
Phase: Phase 4 — Modern Java
Difficulty: ★★
Importance: ★★★
Interview Frequency: 50%
Prerequisites:
  - Chapter 01 — Lambda
  - Chapter 02 — Functional Interface
Used Later:
  - Chapter 04 — Stream
Estimated Reading: 12 phút
Estimated Practice: 12 phút
```

## Story

> Xem lại: [Chapter 01](01-lambda.md) — lambda `x -> square(x)` chỉ đơn giản **gọi lại** một
> phương thức đã tồn tại sẵn, không thêm logic gì mới. Viết `x -> square(x)` có vẻ hơi thừa —
> tại sao phải "bọc" một lời gọi phương thức đã có sẵn vào một lambda mới?

```java
IntUnaryOperator sq1 = MethodReferenceDemo::square; // method reference
IntUnaryOperator sq2 = x -> square(x);               // lambda tuong duong
```

Cả hai hoạt động giống hệt nhau, nhưng `ClassName::square` ngắn gọn hơn và nói rõ ý định:
"dùng thẳng phương thức này", không cần đặt tên tham số giả `x` chỉ để chuyển tiếp nó.

## Interview Question (Central)

> Method reference là gì? Có bao nhiêu loại method reference trong Java, và khi nào nên dùng nó
> thay vì lambda?

## Objectives

- [ ] Phân biệt 4 loại method reference: static, instance trên object cụ thể, instance trên kiểu
      bất kỳ, constructor reference
- [ ] Tự tay chạy được ví dụ thật cho cả 4 loại
- [ ] Biết khi nào method reference làm code rõ ràng hơn, khi nào lambda vẫn là lựa chọn tốt hơn

## Prerequisites

- Chapter 01, 02 — hiểu lambda expression và functional interface.

## Used Later

- **Chapter 04 (Stream)** — method reference thường được dùng trong `map()`, `filter()`,
  `forEach()` khi logic chỉ là gọi lại một phương thức có sẵn.

## Problem

Nhiều lambda chỉ đơn giản chuyển tiếp tham số cho một phương thức đã tồn tại
(`x -> ClassName.method(x)`, `obj -> obj.method()`) — không có logic mới nào được thêm vào.
Viết đầy đủ cú pháp lambda cho trường hợp này (đặt tên tham số giả, viết lại lời gọi) là thừa
thãi, làm giảm khả năng đọc code so với việc nói thẳng "dùng phương thức này".

## Concept

**Method reference** (`::`) là cú pháp rút gọn hơn nữa cho lambda, khi thân lambda chỉ là một
lời gọi phương thức (hoặc constructor) đã tồn tại sẵn — không thêm logic gì khác. Có 4 loại:
`ClassName::staticMethod`, `instance::instanceMethod`, `ClassName::instanceMethod`,
`ClassName::new`.

## Why?

Method reference không thêm khả năng mới nào so với lambda (chương trình vẫn có thể viết bằng
lambda tương đương) — giá trị của nó thuần tuý là **khả năng đọc hiểu (readability)**: khi thân
lambda chỉ là "gọi lại một phương thức có sẵn", việc đặt tên tham số giả rồi viết lại lời gọi là
một tầng gián tiếp không cần thiết. `ClassName::method` nói thẳng "dùng phương thức này" mà
không cần người đọc phải tự suy luận qua tên tham số trung gian.

## How?

```java
ClassName::staticMethod       // 1. static method
instance::instanceMethod      // 2. instance method tren OBJECT CU THE (bound)
ClassName::instanceMethod     // 3. instance method tren KIEU BAT KY (unbound)
ClassName::new                // 4. constructor reference
```

## Visualization

```
1. Static:              ClassName::staticMethod
                         tuong duong: (args) -> ClassName.staticMethod(args)

2. Bound instance:       instance::instanceMethod
                         tuong duong: (args) -> instance.instanceMethod(args)
                         (object "instance" da CO SAN, duoc "bound" san)

3. Unbound instance:     ClassName::instanceMethod
                         tuong duong: (obj, args) -> obj.instanceMethod(args)
                         (object la THAM SO DAU TIEN, chua biet truoc la object nao)

4. Constructor:          ClassName::new
                         tuong duong: (args) -> new ClassName(args)
```

## Example

```java
import java.util.*;
import java.util.function.*;

public class MethodReferenceDemo {
    static int square(int x) { return x * x; }

    static class Product {
        String name;
        Product(String name) { this.name = name; }
        boolean startsWithA() { return name.startsWith("A"); }
    }

    public static void main(String[] args) {
        // 1. Static method reference
        IntUnaryOperator sq1 = MethodReferenceDemo::square;
        System.out.println("Static method ref: square(5) = " + sq1.applyAsInt(5));

        // 2. Bound instance method reference (tren object cu the)
        String prefix = "Order-";
        Supplier<Integer> lenSupplier = prefix::length;
        System.out.println("Bound instance method ref: prefix.length() = " + lenSupplier.get());

        // 3. Unbound instance method reference (tren kieu bat ky)
        List<Product> products = List.of(new Product("Apple"), new Product("Banana"), new Product("Avocado"));
        long countA = products.stream().filter(Product::startsWithA).count();
        System.out.println("Unbound instance method ref: so san pham bat dau bang A = " + countA);

        // 4. Constructor reference
        Function<String, Product> productFactory = Product::new;
        Product p = productFactory.apply("Cherry");
        System.out.println("Constructor reference: tao Product ten = " + p.name);
    }
}
```

Kết quả thật (JDK 17):

```
Static method ref: square(5) = 25
Bound instance method ref: prefix.length() = 6
Unbound instance method ref: so san pham bat dau bang A = 2
Constructor reference: tao Product ten = Cherry
```

`Product::startsWithA` (loại 3, unbound) đếm đúng 2 sản phẩm bắt đầu bằng "A" (Apple, Avocado) —
ở đây `Product` (object) chính là tham số ẩn đầu tiên mà `filter()` truyền vào, khác hẳn loại 2
(`prefix::length`) nơi `prefix` đã là một object cụ thể được "gắn sẵn" từ trước.

## Deep Dive

**Làm sao trình biên dịch phân biệt được loại 2 (bound) và loại 3 (unbound) khi cú pháp
`ClassName::method` và `instance::method` trông tương tự?** Trình biên dịch dựa vào **kiểu của
danh từ đứng trước `::`**: nếu đó là tên một **biến/object** đã tồn tại (ví dụ `prefix`, một
`String` cụ thể) → bound reference, tương đương lambda không tham số bổ sung nào ngoài các tham
số của interface đích. Nếu đó là tên một **class** (ví dụ `Product`) → unbound reference, và
trình biên dịch tự động chèn thêm một tham số ẩn đầu tiên (kiểu `Product`) để "trở thành" object
gọi phương thức — đây là lý do `Product::startsWithA` (chữ ký `boolean startsWithA()`, không
tham số) lại khớp được với `Predicate<Product>` (chữ ký `boolean test(Product p)`, một tham số)
— tham số đó chính là object `Product` sẽ gọi `startsWithA()` lên chính nó.

## Engineering Insight

**Khi nào method reference thực sự làm code rõ ràng hơn, khi nào nó làm code khó đọc hơn?**
Method reference tốt nhất khi tên phương thức đã tự giải thích rõ ý định (`Product::startsWithA`,
`String::trim`) — người đọc hiểu ngay không cần suy nghĩ. Nhưng với method reference loại 3
(unbound) trên phương thức có tên **không rõ ràng về việc "ai gọi lên ai"** (ví dụ
`String::compareTo` dùng trong `Comparator`, chữ ký `int compareTo(String other)` nhưng khi dùng
làm `Comparator<String>` lại dễ nhầm tham số nào là "this", tham số nào là "other"), lambda
tường minh (`(a, b) -> a.compareTo(b)`) đôi khi dễ đọc hơn hẳn — đặc biệt với người mới, chưa
quen quy tắc "tham số ẩn đầu tiên trở thành object gọi phương thức" đã nêu ở Deep Dive.

## Historical Note

Method reference ra đời cùng đợt với lambda (Java 8, 2014, JSR 335) — về bản chất chúng dùng
chung cơ chế biên dịch `invokedynamic`/`LambdaMetafactory` đã học ở Chapter 01, chỉ khác ở cách
trình biên dịch tổng hợp phần thân "lambda ẩn" (gọi thẳng phương thức có sẵn thay vì biên dịch
một khối lệnh mới).

## Myth vs Reality

- **Myth:** "Method reference là một tính năng ngôn ngữ hoàn toàn khác, tách biệt với lambda."
  **Reality:** Method reference chỉ là cú pháp rút gọn hơn nữa cho một trường hợp đặc biệt của
  lambda (thân lambda chỉ gọi lại một phương thức có sẵn) — cùng cơ chế biên dịch
  `invokedynamic` bên dưới.

- **Myth:** "Method reference luôn nhanh hơn lambda tương đương về hiệu năng runtime."
  **Reality:** Không có sự khác biệt hiệu năng đáng kể — cả hai đều biên dịch qua cùng cơ chế
  `LambdaMetafactory`. Lợi ích của method reference thuần tuý là khả năng đọc hiểu code.

## Common Mistakes

- **Nhầm lẫn loại 2 (bound) và loại 3 (unbound)** — đặc biệt khi cả hai đều dùng cú pháp có vẻ
  giống nhau; luôn kiểm tra danh từ trước `::` là tên biến (bound) hay tên class (unbound).
- **Ép dùng method reference cho mọi lambda, kể cả khi làm giảm khả năng đọc** — xem Engineering
  Insight, không phải lúc nào cũng là lựa chọn tốt hơn.
- **Dùng method reference cho phương thức có tác dụng phụ (side effect) không rõ ràng qua tên
  gọi** — làm người đọc khó đoán được hành vi thực sự nếu chỉ nhìn `ClassName::methodName`.

## Best Practices

- Dùng method reference khi tên phương thức tự giải thích rõ ý định và không gây nhầm lẫn về
  "ai là object gọi phương thức".
- Với `Comparator` hoặc các trường hợp thứ tự tham số dễ gây nhầm lẫn, cân nhắc lambda tường
  minh thay vì method reference.
- Ưu tiên constructor reference (`ClassName::new`) khi truyền một factory method cho API nhận
  `Supplier`/`Function` — rõ ràng hơn lambda `() -> new ClassName()`.

## Debug Checklist

- [ ] Method reference không khớp kiểu functional interface đích (lỗi biên dịch)? → kiểm tra
      đang dùng đúng loại (bound/unbound) — số lượng tham số thực tế của interface đích có thể
      khác với số tham số hiển thị của phương thức do quy tắc "tham số ẩn" ở loại 3.
- [ ] Không chắc method reference đang trỏ tới static hay instance method? → kiểm tra danh từ
      trước `::`: tên class không có object cụ thể → có thể là static HOẶC unbound instance,
      tuỳ chữ ký phương thức có khớp static method không trước.

## Summary

Method reference (`::`) là cú pháp rút gọn hơn nữa cho lambda khi thân lambda chỉ gọi lại một
phương thức/constructor có sẵn — có 4 loại: static, bound instance (trên object cụ thể), unbound
instance (trên kiểu bất kỳ, object là tham số ẩn đầu tiên), và constructor reference. Đã chứng
minh bằng thực nghiệm cả 4 loại hoạt động đúng, cùng chung cơ chế biên dịch với lambda (Chapter
01). Lợi ích chính là khả năng đọc hiểu — chỉ nên dùng khi thực sự làm code rõ ràng hơn, không
ép dùng máy móc.

## Interview Questions

**Mid**

- Có bao nhiêu loại method reference trong Java? Cho ví dụ mỗi loại.
- Method reference khác lambda như thế nào về bản chất kỹ thuật?

## Exercises

- [ ] Chạy lại `MethodReferenceDemo` ở trên, xác nhận cả 4 loại method reference hoạt động
      đúng.
- [ ] Viết lại từng method reference trong ví dụ trên dưới dạng lambda tường minh tương đương,
      so sánh khả năng đọc hiểu.
- [ ] Dùng `Comparator.comparing(Product::getName)` (cần thêm getter) để sắp xếp danh sách
      `Product` theo tên.

## Cheat Sheet

| Loại | Cú pháp | Tương đương lambda |
| --- | --- | --- |
| 1. Static | `ClassName::staticMethod` | `(args) -> ClassName.staticMethod(args)` |
| 2. Bound instance | `instance::instanceMethod` | `(args) -> instance.instanceMethod(args)` |
| 3. Unbound instance | `ClassName::instanceMethod` | `(obj, args) -> obj.instanceMethod(args)` |
| 4. Constructor | `ClassName::new` | `(args) -> new ClassName(args)` |

## References

- JLS §15.13 — Method Reference Expressions.
- Java SE API Documentation — `java.util.function`.
