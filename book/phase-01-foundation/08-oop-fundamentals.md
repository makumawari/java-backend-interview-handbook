---
tags:
  - Java
  - OOP
  - Polymorphism
  - Encapsulation
  - Foundation
---

# OOP Fundamentals (Polymorphism, Encapsulation, Inheritance, Abstraction, Overloading vs Overriding, final, super)

> Phase: Phase 1 — Java Foundation
> Chapter slug: `oop-fundamentals`

## Metadata

```yaml
Chapter: OOP Fundamentals (Polymorphism, Encapsulation, Inheritance, Abstraction, Overloading vs Overriding, final, super)
Phase: Phase 1 — Java Foundation
Difficulty: ★★
Importance: ★★★★
Interview Frequency: 90%
Prerequisites:
  - Chapter 01 — Một chương trình Java chạy như thế nào? (cú pháp class, method cơ bản)
Used Later:
  - Object (Chapter 09) — Object là gốc của mọi kế thừa
  - equals()/hashCode() (Chapter 11-12) — override đúng cách dựa trên nguyên tắc ở đây
  - Toàn bộ Spring (Phase 5) — DI/AOP dựa trên polymorphism qua interface
  - Toàn bộ Design Pattern, System Design (Phase 9)
Estimated Reading: 25 phút
Estimated Practice: 30 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

Bạn đang xây tính năng thanh toán cho hệ thống E-Commerce. Ban đầu chỉ có một cách thanh
toán: thẻ tín dụng. Bạn viết:

```java
void checkout(String method, int amount) {
    if (method.equals("CREDIT_CARD")) {
        // logic thanh toán thẻ
    }
}
```

Ba tháng sau, sếp yêu cầu thêm Momo. Bạn thêm một `else if`. Rồi thêm ZaloPay, thêm
`else if` nữa. Rồi thêm trả góp, thêm ví điện tử khác... Mỗi lần thêm phương thức thanh toán
mới, bạn phải **sửa lại chính method `checkout()` cũ** — dù logic thanh toán thẻ, Momo,
ZaloPay không hề liên quan gì tới nhau.

Đây chính là triệu chứng của việc thiếu OOP đúng cách. Bốn tính chất
**Encapsulation, Inheritance, Polymorphism, Abstraction** — không phải lý thuyết suông để
trả lời phỏng vấn, mà là bốn công cụ cụ thể để tránh chính xác cái bẫy "chuỗi if-else vô tận"
ở trên. Chapter này sẽ xây lại ví dụ thanh toán trên bằng OOP đúng cách, và giải thích rạch
ròi cả những khái niệm hay bị hỏi nhưng ít khi được giải thích thấu đáo: `final`, `super`, và
sự khác nhau giữa **overloading** với **overriding**.

## Interview Question (Central)

> Bốn tính chất của lập trình hướng đối tượng (Encapsulation, Inheritance, Polymorphism,
> Abstraction) là gì? Vì sao chúng lại quan trọng đến mức trở thành nền tảng thiết kế của gần
> như mọi framework Java (Spring, JPA...)?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Giải thích được 4 tính chất OOP bằng ví dụ cụ thể, không chỉ định nghĩa suông
- [ ] Viết lại được một chuỗi if-else bằng polymorphism qua interface
- [ ] Phân biệt rạch ròi **Overloading** (cùng tên, khác tham số, quyết định lúc compile) và
      **Overriding** (cùng chữ ký, lớp con định nghĩa lại, quyết định lúc runtime)
- [ ] Dùng đúng `final` trên biến/method/class, hiểu rõ từng ngữ cảnh khác nhau
- [ ] Dùng đúng `super` để gọi constructor/method của lớp cha, tránh nhầm lẫn phổ biến

## Prerequisites

- Chapter 01 — biết cú pháp `class`, khai báo method, biến — chưa cần biết gì về kế thừa hay
  interface.

## Used Later

- **Object** (Chapter 09) — mọi class trong chapter này (kể cả không viết `extends` tường
  minh) đều ngầm kế thừa `Object`; Chapter 09 sẽ giải thích tại sao.
- **equals()/hashCode()** (Chapter 11-12) — override hai method này đúng cách đòi hỏi hiểu
  rõ overriding (chapter này) trước.
- **Spring** (Phase 5) — toàn bộ Dependency Injection dựa trên polymorphism: bạn khai báo
  phụ thuộc vào interface (`PaymentService`), Spring tiêm vào implementation cụ thể lúc
  runtime — đúng cơ chế `Payment` ở Example bên dưới, chỉ khác là Spring tự động hoá việc
  chọn implementation.
- **Design Pattern, System Design** (Phase 9) — gần như mọi design pattern (Strategy,
  Factory, Template Method...) đều là ứng dụng cụ thể của 4 tính chất OOP ở chapter này.

## Problem

Xem lại đoạn code ở Story: mỗi lần thêm một phương thức thanh toán mới, ta phải **sửa** một
method đã có sẵn (`checkout()`), thay vì chỉ **thêm** code mới. Đây vi phạm một nguyên tắc
thiết kế cơ bản: code cũ đã chạy ổn định không nên bị động vào chỉ vì thêm tính năng mới —
càng sửa nhiều, càng dễ vô tình làm hỏng logic thanh toán thẻ đang chạy tốt trong lúc chỉ định
thêm Momo.

## Concept

Bốn tính chất OOP, mỗi tính chất giải quyết một khía cạnh của vấn đề trên:

- **Encapsulation (Đóng gói)** — giấu chi tiết triển khai bên trong một class, chỉ lộ ra
  giao diện cần thiết. `private` field + `public` method truy cập có kiểm soát, thay vì để
  field `public` cho ai cũng sửa trực tiếp.
- **Inheritance (Kế thừa)** — một class có thể kế thừa field/method từ class khác
  (`extends`), tái sử dụng code chung, chỉ viết thêm/viết lại phần khác biệt.
- **Polymorphism (Đa hình)** — nhiều class khác nhau có thể được xử lý thông qua **cùng một
  kiểu tham chiếu** (interface hoặc lớp cha), mỗi class tự quyết định hành vi cụ thể của
  riêng mình. Đây chính là công cụ thay thế chuỗi if-else ở Story.
- **Abstraction (Trừu tượng hoá)** — định nghĩa **cái gì** cần làm (qua interface/abstract
  class) mà không quy định **làm như thế nào** — phần "như thế nào" để lớp con tự quyết định.

## Why?

Nếu thiếu Polymorphism: mọi logic rẽ nhánh theo loại đối tượng phải viết bằng if-else/switch
dựa trên một trường "loại" nào đó (như `method.equals("CREDIT_CARD")` ở Story) — mỗi lần thêm
loại mới là một lần sửa code cũ, rủi ro cao, khó test độc lập từng loại.

Nếu thiếu Encapsulation: field `public` cho phép bất kỳ đoạn code nào ở bất kỳ đâu trong dự
án gán giá trị bất hợp lệ trực tiếp (ví dụ set giá sản phẩm âm) — không có điểm nào để kiểm
soát/validate.

## How?

Viết lại ví dụ thanh toán bằng Polymorphism qua interface, đúng tinh thần Abstraction (định
nghĩa "cái gì" — `process()` — không quan tâm "làm thế nào"):

```java
interface Payment {
    void process(int amount);
}

class CreditCardPayment implements Payment {
    public void process(int amount) {
        System.out.println("Thanh toán " + amount + "đ qua thẻ tín dụng");
    }
}

class MomoPayment implements Payment {
    public void process(int amount) {
        System.out.println("Thanh toán " + amount + "đ qua Momo");
    }
}
```

Muốn thêm ZaloPay? **Thêm** một class `ZaloPayPayment implements Payment` mới — không sửa
một dòng nào trong `CreditCardPayment` hay `MomoPayment`. Đây chính là giá trị thực dụng của
polymorphism: đóng (không sửa) với code cũ, mở (thêm được) với tính năng mới.

## Visualization

```
                    Payment (interface — Abstraction)
                    + process(amount)
                         ▲        ▲        ▲
                         │        │        │
          implements     │        │        │     implements
             ┌────────────┘        │        └────────────┐
             │                     │                      │
   CreditCardPayment          MomoPayment           ZaloPayPayment
   process() → "thẻ"        process() → "Momo"     process() → "ZaloPay"

Code gọi:  Payment p = ...;  p.process(amount);
           ↑ cùng một dòng gọi, hành vi thực tế phụ thuộc
             VÀO OBJECT THẬT được gán vào p LÚC RUNTIME — đây là Polymorphism.
```

## Example

Toàn bộ ví dụ chạy được, thể hiện Polymorphism qua interface (Abstraction), thân bài các class
implement là Encapsulation (field `private`, chỉ thao tác qua method):

```java
public class PaymentDemo {
    public static void main(String[] args) {
        Payment[] payments = {
            new CreditCardPayment(),
            new MomoPayment()
        };
        for (Payment p : payments) {
            p.process(199000);
        }
    }
}

interface Payment {
    void process(int amount);
}

class CreditCardPayment implements Payment {
    public void process(int amount) {
        System.out.println("Thanh toán " + amount + "đ qua thẻ tín dụng");
    }
}

class MomoPayment implements Payment {
    public void process(int amount) {
        System.out.println("Thanh toán " + amount + "đ qua Momo");
    }
}
```

Chạy thật, kết quả:

```
Thanh toán 199000đ qua thẻ tín dụng
Thanh toán 199000đ qua Momo
```

Chú ý dòng `for (Payment p : payments)`: **cùng một dòng code** `p.process(amount)` tạo ra
**hai hành vi khác nhau**, tuỳ vào object thật đứng sau tham chiếu `p` — đây chính xác là
Polymorphism ("nhiều hình dạng").

## Deep Dive

**Inheritance + Overriding: khi nào lớp con "định nghĩa lại" hành vi lớp cha.** Xét bài toán
khác: `DiscountedProduct` kế thừa `Product`, tính giá khác đi:

```java
class Product {
    String name;
    int price;
    Product(String name, int price) { this.name = name; this.price = price; }
    int getFinalPrice() { return price; }
}

class DiscountedProduct extends Product {
    int discountPercent;
    DiscountedProduct(String name, int price, int discountPercent) {
        super(name, price); // gọi constructor của Product — bắt buộc là dòng ĐẦU TIÊN
        this.discountPercent = discountPercent;
    }
    @Override
    int getFinalPrice() {
        return price - (price * discountPercent / 100);
    }
}
```

Gọi `new DiscountedProduct("Áo thun", 199000, 10).getFinalPrice()` cho kết quả thật:
`179100` (199000 trừ 10%) — chạy đúng, xác nhận `@Override` đã thay thế hoàn toàn hành vi
`getFinalPrice()` của `Product` gốc.

**`super(...)` ở dòng đầu tiên của constructor không phải quy ước — nó là bắt buộc theo cú
pháp.** Java yêu cầu constructor lớp con phải khởi tạo xong phần "lớp cha" trước khi làm bất
cứ điều gì khác — đảm bảo object luôn ở trạng thái hợp lệ theo đúng thứ tự từ gốc (`Object`)
xuống lá.

## Engineering Insight

**Overloading vs Overriding — vì sao đây là câu hỏi phỏng vấn kinh điển dù nghe "hiển
nhiên"?** Vì chúng giống nhau ở bề mặt (cùng tên method) nhưng khác nhau ở **thời điểm quyết
định** — đây mới là điểm mấu chốt hay bị hỏi sai trọng tâm:

```java
class Calc {
    int sum(int a, int b) { return a + b; }             // Overload #1
    int sum(int a, int b, int c) { return a + b + c; }   // Overload #2
    double sum(double a, double b) { return a + b; }     // Overload #3
}
```

Chạy thật `c.sum(1, 2)`, `c.sum(1, 2, 3)`, `c.sum(1.5, 2.5)` cho kết quả `3`, `6`, `4.0` —
**Overloading được quyết định lúc compile** (`javac` nhìn số lượng/kiểu tham số tại chỗ gọi
để chọn đúng version — gọi là *static/compile-time binding*). Ngược lại, **Overriding được
quyết định lúc runtime** (JVM nhìn object thật đứng sau tham chiếu, như ví dụ
`DiscountedProduct` ở trên — gọi là *dynamic/runtime binding*). Sự khác biệt này không phải
chi tiết vụn vặt: nó chính là cơ chế nền tảng cho Polymorphism hoạt động — nếu overriding
cũng được quyết định lúc compile như overloading, ví dụ `Payment[]` ở phần How? sẽ không thể
nào chạy đúng, vì lúc compile, `javac` không thể biết trước phần tử nào trong mảng là
`CreditCardPayment`, phần tử nào là `MomoPayment`.

## Historical Note

Bốn tính chất OOP không phải phát minh của Java — chúng bắt nguồn từ Simula (1967, ngôn ngữ
mô phỏng đầu tiên có khái niệm class) và được phổ biến rộng rãi bởi Smalltalk (thập niên
1970) và C++ (thập niên 1980). Java (1995) kế thừa gần như nguyên vẹn tư tưởng này, nhưng
loại bỏ đa kế thừa class (multiple inheritance) mà C++ cho phép — một class Java chỉ
`extends` được **một** class cha (dù `implements` được nhiều interface), một quyết định thiết
kế cố ý để tránh sự phức tạp/mơ hồ nổi tiếng của đa kế thừa trong C++ (gọi là "Diamond
Problem").

## Myth vs Reality

- **Myth:** "Kế thừa (Inheritance) luôn là cách tốt để tái sử dụng code."
  **Reality:** Kế thừa tạo ra ràng buộc chặt (tight coupling) giữa lớp cha và lớp con — sửa
  lớp cha có thể vô tình phá vỡ hành vi lớp con. Nguyên tắc phổ biến trong giới kỹ sư có kinh
  nghiệm là **"ưu tiên composition hơn inheritance"** (dùng một object khác làm field, thay
  vì kế thừa nó) khi mối quan hệ không thực sự là "là một loại của" (is-a) rõ ràng.

- **Myth:** "Overloading là một dạng đơn giản của Polymorphism, bản chất giống Overriding."
  **Reality:** Đây là điểm gây nhầm lẫn nhiều nhất. Overloading đôi khi được gọi là
  "compile-time polymorphism" trong một số tài liệu, nhưng cơ chế quyết định (dựa vào kiểu
  khai báo tại chỗ gọi, không phải object thật lúc runtime) hoàn toàn khác overriding — xem
  Engineering Insight.

## Common Mistakes

- **Quên `@Override`.** Không bắt buộc về mặt cú pháp, nhưng thiếu nó khiến trình biên dịch
  không thể cảnh báo nếu bạn gõ sai tên method/tham số — lúc đó bạn vô tình **overload** thay
  vì **override**, và lớp cha vẫn chạy hành vi cũ mà không báo lỗi gì.
- **Nhầm private method là có thể override.** `private` method không tham gia cơ chế
  overriding — lớp con định nghĩa method cùng tên chỉ là một method **hoàn toàn mới**, độc
  lập, không liên quan gì tới method `private` của lớp cha (dù cùng tên).
- **Expose field trực tiếp (`public int price`) thay vì qua getter/setter**, phá vỡ
  Encapsulation, mất khả năng validate hay thay đổi cách lưu trữ nội bộ về sau mà không ảnh
  hưởng code gọi.

## Best Practices

- Mặc định khai báo field là `private`, chỉ mở ra qua method khi thực sự cần — đúng tinh
  thần Encapsulation.
- Luôn thêm `@Override` khi có ý định ghi đè method lớp cha — để trình biên dịch bắt lỗi
  giúp bạn.
- Dùng `final` trên method khi bạn **cố ý không muốn** lớp con override nó (thường vì method
  đó chứa logic bất biến, quan trọng về mặt an toàn/đúng đắn — ví dụ các method trong
  `String`, sẽ gặp lại ở [Chapter 10](10-string.md)); dùng `final` trên biến để đánh dấu
  hằng số hoặc field không đổi sau khi khởi tạo (một bước hướng tới immutable object — xem
  Chapter 11 sau này).
- Cân nhắc `composition` (field kiểu object khác) thay vì `inheritance` khi quan hệ giữa hai
  class không thực sự là "is-a" rõ ràng.

## Production Notes

OOP Fundamentals là kiến thức thiết kế, không gắn trực tiếp với một loại sự cố production cụ
thể như các chapter về JVM internals trước đó. Ảnh hưởng thực tế của chapter này tới
production là **gián tiếp nhưng sâu rộng**: thiết kế polymorphism đúng cách (ví dụ interface
`PaymentService` ở Story) là nền tảng để Spring's Dependency Injection hoạt động — sẽ thấy lại
đầy đủ ở Phase 5 khi một bug production hoá ra bắt nguồn từ việc lạm dụng kế thừa sâu (nhiều
tầng `extends`) khiến không ai còn nhớ hành vi thật sự nằm ở lớp nào.

## Debug Checklist

- [ ] Gọi method trên object nhưng chạy sai hành vi mong đợi? → kiểm tra có đúng đang override
      không (có `@Override`?), hay vô tình overload (sai tên/tham số).
- [ ] Method lớp con không bao giờ được gọi dù đã viết `@Override`? → kiểm tra method lớp cha
      có phải `final`/`private`/`static` không — cả ba trường hợp này đều **không** cho phép
      override đúng nghĩa.
- [ ] `NullPointerException` ngay trong constructor lớp con? → kiểm tra thứ tự gọi `super(...)`
      và các dòng khởi tạo field, đảm bảo không dùng field của lớp cha trước khi `super(...)`
      chạy xong.

## Source Code Walkthrough

Không có "source OpenJDK" cụ thể để đọc ở chapter khái niệm nền tảng này — cơ chế
overriding/dynamic binding được hiện thực ở tầng bytecode bằng lệnh `invokevirtual`
(đã gặp ở [Chapter 01](01-mot-chuong-trinh-java-chay-nhu-the-nao.md) khi đọc bytecode của
`product.getName()`) — chính lệnh này, không phải `invokestatic`, là cơ chế cấp thấp cho phép
JVM tra cứu đúng method của object thật lúc runtime thay vì cố định lúc compile. Đây là mối
liên kết trực tiếp giữa khái niệm OOP trừu tượng ở chapter này với bytecode cụ thể đã học.

## Summary

Bốn tính chất OOP — Encapsulation (giấu chi tiết), Inheritance (tái sử dụng), Polymorphism
(một giao diện, nhiều hành vi), Abstraction (định nghĩa "cái gì" không phải "thế nào") — cùng
nhau giải quyết vấn đề cụ thể: tránh phải sửa code cũ mỗi khi thêm tính năng mới (minh chứng ở
ví dụ thanh toán). **Overloading** (cùng tên khác tham số, quyết định lúc compile) và
**Overriding** (lớp con định nghĩa lại, quyết định lúc runtime nhờ `invokevirtual`) là hai cơ
chế dễ nhầm nhưng hoạt động hoàn toàn khác nhau — overriding chính là nền tảng khiến
polymorphism hoạt động được. `final` khoá lại (biến/method/class) khi bạn cố ý không muốn thay
đổi/override; `super` gọi ngược lên lớp cha, bắt buộc là dòng đầu tiên trong constructor lớp
con.

## Interview Questions

**Junior**

- What is Polymorphism? Explain with examples.
- What is Encapsulation?
- What is Method Overloading?

**Mid**

- What is the use of the `final` keyword?
- What is the use of the `super` keyword?
- Overloading và Overriding khác nhau ở điểm nào? Cái nào quyết định lúc compile, cái nào lúc
  runtime?

**Senior**

- Vì sao nguyên tắc "ưu tiên composition hơn inheritance" lại phổ biến trong giới kỹ sư có
  kinh nghiệm? Cho một ví dụ cụ thể kế thừa sai chỗ gây vấn đề bảo trì.
- Giải thích ở tầng bytecode vì sao overriding hoạt động được lúc runtime (liên hệ
  `invokevirtual` ở Chapter 01) trong khi overloading được giải quyết hoàn toàn lúc compile.

## Exercises

- [ ] Chạy lại đúng ví dụ `PaymentDemo` ở trên. Thêm một class `ZaloPayPayment` mới mà
      **không sửa** bất kỳ dòng nào trong các class đã có.
- [ ] Chạy lại ví dụ `DiscountedProduct`, thử bỏ dòng `super(name, price)` đi — quan sát lỗi
      biên dịch, đọc kỹ thông báo lỗi để hiểu vì sao Java bắt buộc dòng đó.
- [ ] Viết một class `Calc` với 3 method `sum` overload như ví dụ ở Engineering Insight, tự
      gọi `javap -c` xem bytecode — xác nhận `javac` đã chọn đúng method nào tại mỗi chỗ gọi
      (gợi ý: tìm số trong tên method đã bị "đổi tên" ở mức bytecode, gọi là *name mangling*
      tuỳ theo chữ ký).

## Cheat Sheet

| Tính chất | Ý nghĩa | Công cụ Java |
| --- | --- | --- |
| Encapsulation | Giấu chi tiết, kiểm soát truy cập | `private` field + getter/setter |
| Inheritance | Tái sử dụng field/method lớp cha | `extends`, `super` |
| Polymorphism | Một tham chiếu, nhiều hành vi thật | Interface/lớp cha + `@Override` |
| Abstraction | Định nghĩa "cái gì", không phải "thế nào" | `interface`, `abstract class` |

| So sánh | Overloading | Overriding |
| --- | --- | --- |
| Chữ ký method | Khác (số/kiểu tham số) | Giống hệt |
| Quyết định lúc | Compile-time | Runtime |
| Quan hệ class | Cùng một class (hoặc lớp con thêm mới) | Lớp con định nghĩa lại lớp cha |
| Lệnh bytecode liên quan | `invokestatic`/`invokespecial` (cố định) | `invokevirtual` (tra cứu động) |

| `final` dùng trên | Ý nghĩa |
| --- | --- |
| Biến | Không gán lại được sau khi khởi tạo (hằng số) |
| Method | Lớp con không override được |
| Class | Không class nào `extends` được (ví dụ: `String`) |

## References

- Java Language Specification (JLS) — Chapter 8: Classes, Chapter 9: Interfaces.
- Effective Java (Joshua Bloch) — Item 18: "Favor composition over inheritance".
- Java Virtual Machine Specification (Oracle) — mô tả `invokevirtual` vs `invokestatic`/
  `invokespecial` (liên hệ Chapter 01, 05).
