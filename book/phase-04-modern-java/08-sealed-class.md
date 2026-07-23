---
tags:
  - Java
  - SealedClass
  - ModernJava
---

# Sealed Class/Interface

> Phase: Phase 4 — Modern Java
> Chapter slug: `sealed-class`

## Metadata

```yaml
Chapter: Sealed Class
Phase: Phase 4 — Modern Java
Difficulty: ★★★
Importance: ★★★★
Interview Frequency: 55%
Prerequisites:
  - Chapter 07 — Record
  - Phase 1, Chapter 08 — OOP Fundamentals (inheritance, interface)
Used Later:
  - Chapter 09 — Pattern Matching (exhaustive switch trên sealed hierarchy)
Estimated Reading: 15 phút
Estimated Practice: 15 phút
```

## Story

> Xem lại: Phase 1, Chapter 08 (OOP Fundamentals) — `interface`/class thông thường cho phép
> **bất kỳ ai, ở bất kỳ đâu** implement/kế thừa nó. Điều đó linh hoạt, nhưng với những trường
> hợp bạn biết chắc "tập hợp các loại con là cố định, đã biết trước" (ví dụ một phương thức
> thanh toán chỉ có thể là thẻ tín dụng, chuyển khoản, hoặc ví điện tử — không có loại thứ tư),
> sự linh hoạt đó lại thành một lỗ hổng.

Thử khai báo một interface với danh sách con **cố định**, rồi cố thêm một loại con **không** có
trong danh sách:

```java
sealed interface Shape permits Circle, Square {}
record Circle(double radius) implements Shape {}
record Square(double side) implements Shape {}
record Triangle(double base, double height) implements Shape {} // KHONG co trong permits!
```

Kết quả biên dịch thật:

```
error: class is not allowed to extend sealed class: Shape (as it is not listed in its 'permits' clause)
```

Trình biên dịch **từ chối ngay lập tức** — không đợi tới runtime.

## Interview Question (Central)

> Sealed class/interface là gì? Nó giải quyết vấn đề gì mà interface thông thường không giải
> quyết được?

## Objectives

- [ ] Giải thích cú pháp `sealed ... permits` và ý nghĩa của việc giới hạn tập hợp subtype
- [ ] Tự tay chứng minh lỗi biên dịch khi thêm subtype không có trong `permits`
- [ ] Tự tay chứng minh `switch` trên sealed hierarchy có thể **exhaustive** (đầy đủ) mà không
      cần `default`

## Prerequisites

- Chapter 07 — hiểu record, vì sealed interface thường kết hợp trực tiếp với record để mô hình
  hoá các loại con.
- Phase 1, Chapter 08 — hiểu `interface`, kế thừa.

## Used Later

- **Chapter 09 (Pattern Matching)** — `switch` trên sealed hierarchy kết hợp record pattern là
  một trong những ứng dụng mạnh nhất của cả hai tính năng.

## Problem

Với `interface` thông thường, trình biên dịch **không thể biết** tập hợp đầy đủ các subtype tồn
tại — bất kỳ class nào, ở bất kỳ module nào, cũng có thể implement nó bất cứ lúc nào (kể cả sau
khi code gốc đã được biên dịch và triển khai). Điều này có nghĩa một `switch` xử lý theo từng
loại con **không bao giờ** được coi là "đầy đủ" một cách an toàn — luôn cần một nhánh
`default` phòng hờ cho loại con chưa biết, dù về mặt logic nghiệp vụ, tập hợp loại con thực sự
đã cố định và đầy đủ từ đầu.

## Concept

**Sealed class/interface** (Java 17+) cho phép khai báo tường minh **danh sách đầy đủ** các
subtype được phép (`permits`) — không class/interface nào khác, ở bất kỳ đâu, có thể
implement/kế thừa nó ngoài danh sách đó. Trình biên dịch dùng thông tin này để xác nhận một
`switch` xử lý hết mọi subtype trong `permits` là **exhaustive** (đầy đủ), không cần nhánh
`default`.

## Why?

Sealed hierarchy chuyển một ràng buộc thiết kế ("tập hợp loại con này là cố định, đã biết trước
— không mở rộng thêm") từ **quy ước không chính thức** (comment, tài liệu, hy vọng đồng nghiệp
đọc và tuân theo) sang **ràng buộc được compiler kiểm tra và bảo vệ thật**. Lợi ích trực tiếp
nhất: `switch` exhaustive — nếu sau này có ai thêm một subtype mới vào `permits`, mọi `switch`
xử lý sealed hierarchy đó ở khắp codebase mà **quên** xử lý loại mới sẽ **báo lỗi biên dịch
ngay lập tức**, thay vì âm thầm rơi vào một nhánh `default` xử lý sai hoặc bị bỏ sót hoàn toàn.

## How?

```java
sealed interface PaymentMethod permits CreditCard, BankTransfer, EWallet {}
record CreditCard(String cardNumber) implements PaymentMethod {}
record BankTransfer(String accountNumber) implements PaymentMethod {}
record EWallet(String walletId) implements PaymentMethod {}

static String describe(PaymentMethod method) {
    return switch (method) { // EXHAUSTIVE, khong can default
        case CreditCard c -> "The tin dung: " + c.cardNumber();
        case BankTransfer b -> "Chuyen khoan: " + b.accountNumber();
        case EWallet e -> "Vi dien tu: " + e.walletId();
    };
}
```

## Visualization

```
interface thong thuong:                sealed interface:

  Shape                                   sealed interface Shape
    ▲  ▲  ▲  ▲ (AI CUNG implement duoc)      permits Circle, Square
    │  │  │  └── ??? (khong biet truoc)        ▲         ▲
    │  │  └── Triangle (them sau)          Circle      Square
    │  └── Circle                        (CHI HAI loai nay, KHONG THEM DUOC nua)
    └── Square

  switch CAN default (khong biet het loai con)
                                          switch KHONG CAN default (biet het, exhaustive)
```

## Example

```java
public class SealedDemo {
    sealed interface PaymentMethod permits CreditCard, BankTransfer, EWallet {}
    record CreditCard(String cardNumber) implements PaymentMethod {}
    record BankTransfer(String accountNumber) implements PaymentMethod {}
    record EWallet(String walletId) implements PaymentMethod {}

    static String describe(PaymentMethod method) {
        return switch (method) {
            case CreditCard c -> "The tin dung: " + c.cardNumber();
            case BankTransfer b -> "Chuyen khoan: " + b.accountNumber();
            case EWallet e -> "Vi dien tu: " + e.walletId();
        };
    }

    public static void main(String[] args) {
        System.out.println(describe(new CreditCard("1234-5678")));
        System.out.println(describe(new BankTransfer("VCB-001")));
        System.out.println(describe(new EWallet("MOMO-999")));
    }
}
```

Kết quả thật (chạy trên JDK có switch pattern matching ổn định — xem Chapter 09):

```
The tin dung: 1234-5678
Chuyen khoan: VCB-001
Vi dien tu: MOMO-999
```

`switch (method)` **biên dịch được mà không có nhánh `default`** — trình biên dịch xác nhận 3
`case` đã bao phủ toàn bộ `permits` (`CreditCard`, `BankTransfer`, `EWallet`), không còn khả
năng nào khác.

**Cố thêm subtype không có trong `permits`:**

```java
sealed interface Shape permits Circle, Square {}
record Circle(double radius) implements Shape {}
record Square(double side) implements Shape {}
record Triangle(double base, double height) implements Shape {} // KHONG trong permits!
```

Kết quả biên dịch thật:

```
error: class is not allowed to extend sealed class: Shape (as it is not listed in its 'permits' clause)
record Triangle(double base, double height) implements Shape {}
^
```

## Deep Dive

**Vì sao `switch` trên sealed hierarchy có thể an toàn bỏ qua `default`, trong khi `switch` trên
`enum` (Phase 1, Chapter 17) truyền thống vẫn thường được khuyến nghị có `default`?** Về bản
chất, cả hai đều dựa trên "tập hợp giá trị/loại cố định, biết trước" — nhưng `enum` vẫn có thể
bị thêm **hằng số mới** trong một class file được biên dịch riêng rồi thay thế (JAR cũ có thể bị
thay bằng JAR mới có thêm enum constant) mà không biên dịch lại toàn bộ codebase gọi tới nó —
đây là lý do một số style guide vẫn khuyến nghị `default` cho `enum` switch để phòng ngừa lỗi
runtime khi các module được biên dịch/triển khai độc lập. Sealed hierarchy có ràng buộc chặt hơn
ở cấp **biên dịch**: mọi subtype trong `permits` phải nằm trong cùng module (hoặc cùng package,
tuỳ cấu hình), khiến trình biên dịch có đủ thông tin để đảm bảo tính đầy đủ tại thời điểm biên
dịch của chính module đó — chặt chẽ hơn giả định ngầm của `enum`.

## Engineering Insight

**Sealed hierarchy + record là mô hình gần nhất Java có được với "algebraic data type" (kiểu dữ
liệu đại số) của các ngôn ngữ hàm như Scala/Kotlin/Haskell.** `sealed interface` đóng vai trò
"sum type" (một giá trị chỉ có thể là MỘT trong số các loại con cố định), còn `record` đóng vai
trò "product type" (gói nhiều field lại thành một). Kết hợp cả hai (như `PaymentMethod` ở
Example) cho phép mô hình hoá dữ liệu nghiệp vụ theo cách **an toàn kiểu (type-safe)** hơn hẳn
cách truyền thống dùng một trường "type" dạng String/enum kèm các trường tuỳ chọn khác nhau tuỳ
loại (dễ dẫn tới trạng thái không hợp lệ — ví dụ trường `cardNumber` bị điền dù `type =
"BANK_TRANSFER"`). Với sealed + record, mỗi loại con chỉ có đúng những field thuộc về nó, không
thể tồn tại trạng thái lai không hợp lệ.

## Historical Note

Sealed class/interface trải qua 2 vòng preview (JEP 360 ở Java 15, JEP 397 ở Java 16) trước khi
chính thức ổn định ở **Java 17** (JEP 409, tháng 9/2021) — cùng phiên bản LTS với Record (dù
Record đã ổn định từ Java 16). Cả hai đều thuộc Project Amber, cùng hướng giảm boilerplate và
tăng tính an toàn kiểu của ngôn ngữ Java.

## Myth vs Reality

- **Myth:** "Sealed class chỉ có ý nghĩa về mặt tài liệu (documentation), không có tác dụng kỹ
  thuật thực sự."
  **Reality:** Đã chứng minh bằng thực nghiệm — nó là ràng buộc được trình biên dịch **kiểm
  tra và ép buộc thật**, không chỉ là quy ước, và trực tiếp cho phép `switch` exhaustive không
  cần `default`.

- **Myth:** "Mọi subtype của sealed interface đều phải là record."
  **Reality:** `permits` có thể liệt kê bất kỳ loại nào: `record`, class thông thường (miễn
  khai báo `final`, `sealed`, hoặc `non-sealed`), thậm chí interface khác — record chỉ là lựa
  chọn phổ biến nhất vì phù hợp tự nhiên với mô hình "mỗi loại con là một gói dữ liệu".

## Common Mistakes

- **Cố implement một sealed interface từ một module/package không được phép** — gây lỗi biên
  dịch; các subtype trong `permits` thường phải nằm cùng module hoặc cùng package với sealed
  type.
- **Quên đánh dấu subtype là `final`/`sealed`/`non-sealed`** — mỗi subtype trực tiếp của một
  sealed type bắt buộc phải khai báo rõ nó có cho phép kế thừa tiếp hay không (record ngầm định
  `final`, nên tự động thoả điều kiện này).
- **Vẫn thêm nhánh `default` "cho chắc" trong switch exhaustive** — làm mất chính lợi ích quan
  trọng nhất của sealed hierarchy (compiler tự phát hiện thiếu case khi có subtype mới).

## Best Practices

- Dùng sealed interface/class khi tập hợp loại con thực sự cố định theo logic nghiệp vụ (không
  phải một khiếm khuyết thiết kế cần "đóng" tạm thời).
- Kết hợp sealed interface với record cho các subtype — mô hình "algebraic data type" giúp mã
  nguồn an toàn kiểu hơn (xem Engineering Insight).
- KHÔNG thêm `default` vào switch exhaustive trên sealed hierarchy — để trình biên dịch tự bảo
  vệ tính đầy đủ khi có subtype mới trong tương lai.

## Debug Checklist

- [ ] Lỗi "class is not allowed to extend sealed class ... not listed in its 'permits' clause"?
      → thêm subtype đó vào danh sách `permits`, hoặc xác nhận nó thực sự nên là một loại con
      hợp lệ.
- [ ] `switch` trên sealed hierarchy báo lỗi "the switch statement does not cover all possible
      input values"? → thêm subtype còn thiếu vào chuỗi `case`, không nên "chữa cháy" bằng
      `default`.

## Summary

Sealed class/interface (`sealed ... permits`) giới hạn tường minh, được compiler bảo vệ, tập
hợp đầy đủ các subtype được phép — đã chứng minh bằng thực nghiệm: thêm subtype ngoài `permits`
gây lỗi biên dịch ngay lập tức. Lợi ích chính là cho phép `switch` **exhaustive** trên sealed
hierarchy mà không cần `default` — nếu quên xử lý một loại con, compiler báo lỗi ngay thay vì
để lỗi âm thầm rơi vào nhánh mặc định. Kết hợp sealed interface với record (Chapter 07) tạo ra
mô hình gần với "algebraic data type" của các ngôn ngữ hàm, giúp biểu diễn dữ liệu nghiệp vụ an
toàn kiểu hơn cách dùng trường "type" truyền thống.

## Interview Questions

**Mid**

- Sealed class/interface là gì? Nó giải quyết vấn đề gì mà interface thông thường không giải
  quyết được?
- Vì sao `switch` trên sealed hierarchy có thể bỏ qua nhánh `default`?

**Senior**

- Giải thích mô hình "algebraic data type" khi kết hợp sealed interface với record. Nó giúp gì
  so với cách biểu diễn dữ liệu nghiệp vụ truyền thống (dùng trường "type")?

## Exercises

- [ ] Chạy lại `SealedDemo` ở trên (cần JDK hỗ trợ switch pattern matching ổn định, xem Chapter
      09), xác nhận kết quả đúng như mô tả.
- [ ] Thử biên dịch một subtype ngoài `permits`, xác nhận lỗi biên dịch đúng như mô tả.
- [ ] Mô hình hoá một sealed hierarchy `Notification` với các loại con `EmailNotification`,
      `SmsNotification`, `PushNotification` (dùng record), viết một `switch` exhaustive xử lý
      từng loại.

## Cheat Sheet

| Từ khoá | Ý nghĩa |
| --- | --- |
| `sealed` | Class/interface giới hạn tập hợp subtype qua `permits` |
| `permits A, B, C` | Danh sách đầy đủ, tường minh các subtype được phép |
| `final` (trên subtype) | Subtype này không cho phép kế thừa tiếp |
| `sealed` (trên subtype) | Subtype này tiếp tục giới hạn tập hợp con của chính nó |
| `non-sealed` (trên subtype) | Subtype này mở lại, cho phép kế thừa tự do như bình thường |

## References

- JEP 409: Sealed Classes — https://openjdk.org/jeps/409
- JLS §8.1.1.2, §9.1.1.4 — Sealed Classes and Interfaces.
