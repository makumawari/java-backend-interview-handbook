---
tags:
  - Java
  - OOP
  - Object
  - Foundation
---

# Object

> Phase: Phase 1 — Java Foundation
> Chapter slug: `object`

## Metadata

```yaml
Chapter: Object
Phase: Phase 1 — Java Foundation
Difficulty: ★★
Importance: ★★★★
Interview Frequency: 70%
Prerequisites:
  - Chapter 08 — OOP Fundamentals
Used Later:
  - String (Chapter 10) — String cũng kế thừa Object, override toString/equals/hashCode
  - equals()/hashCode() (Chapter 11-12) — đào sâu 2 method quan trọng nhất giới thiệu ở đây
  - Reflection (Chapter 17) — getClass() là cửa ngõ vào Reflection API
  - Thread, Concurrency (Phase 3) — wait()/notify() được đào sâu ở Chapter Monitor
Estimated Reading: 20 phút
Estimated Practice: 20 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Chapter 08](08-oop-fundamentals.md) — bạn đã viết `class Product { ... }` mà
> không hề viết `extends` gì cả.

Vậy mà đoạn code sau vẫn chạy được, dù bạn chưa từng định nghĩa method `toString()`,
`equals()`, hay `hashCode()` cho `Product`:

```java
Product p = new Product("Áo thun", 199000);
System.out.println(p);          // in ra được gì đó, không lỗi
System.out.println(p.equals(p)); // chạy được, trả về true
System.out.println(p.hashCode()); // chạy được, trả về một số
```

Những method này đến từ đâu? Bạn không viết chúng, `Product` cũng không `extends` class nào
tường minh — vậy làm sao chúng "có sẵn"? Câu trả lời: **mọi class trong Java, dù bạn có viết
`extends` hay không, đều ngầm kế thừa từ một class duy nhất: `java.lang.Object`.**

## Interview Question (Central)

> Vì sao mọi class trong Java, kể cả class bạn tự viết mà không `extends` gì, đều "ngầm" kế
> thừa từ `Object`? `Object` cung cấp sẵn những method nào, và vì sao chúng lại quan trọng?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Biết `class Product {}` thực chất tương đương `class Product extends Object {}`
- [ ] Liệt kê được các method chính của `Object`: `toString()`, `equals()`, `hashCode()`,
      `getClass()`, `clone()`, `wait()`/`notify()`/`notifyAll()`
- [ ] Hiểu hành vi **mặc định** của `toString()`/`equals()`/`hashCode()` khi chưa override —
      và vì sao hành vi mặc định đó thường không phải điều bạn muốn
- [ ] Biết vì sao `finalize()` bị deprecated, không nên dùng

## Prerequisites

- Chapter 08 — hiểu Inheritance và Overriding, vì chapter này liên tục nói về việc override
  method của `Object`.

## Used Later

- **String** (Chapter 10) — `String` cũng kế thừa `Object`, và là ví dụ điển hình của một
  class override đúng cách cả `toString()`, `equals()`, `hashCode()`.
- **equals()/hashCode()** (Chapter 11-12) — chapter này chỉ giới thiệu ở mức tổng quan; hai
  method quan trọng nhất của `Object` (vì ảnh hưởng trực tiếp tới `HashMap`, `HashSet` ở
  Phase 2) sẽ có chapter riêng đào sâu.
- **Reflection** (Chapter 17) — `getClass()` trả về một object `Class`, chính là điểm khởi
  đầu của toàn bộ Reflection API.
- **Thread, Concurrency** (Phase 3) — `wait()`/`notify()`/`notifyAll()` chỉ giới thiệu tên ở
  đây; cơ chế đầy đủ (gắn với `synchronized`, monitor) thuộc về Phase 3.

## Problem

Nếu mỗi class tự định nghĩa từ đầu cách in ra màn hình (`toString`), cách so sánh
(`equals`)... sẽ không có **hành vi mặc định thống nhất** nào cả — mọi class hoàn toàn độc
lập, framework/thư viện (như `println`, `HashMap`) sẽ không biết cách xử lý một object bất kỳ
mà chúng chưa từng biết trước class cụ thể là gì.

## Concept

Java giải quyết bằng cách quy định: **mọi class đều ngầm `extends Object`** nếu không tường
minh `extends` class nào khác. `Object` cung cấp sẵn một bộ method tối thiểu, đảm bảo bất kỳ
object nào — dù thuộc class nào — cũng đều "biết" cách tự in ra chuỗi, tự so sánh, tự cung cấp
mã băm... dù chỉ theo cách rất cơ bản.

```java
class Product { }
// tương đương với:
class Product extends Object { }
```

## Why?

Nếu không có `Object` làm gốc chung: `System.out.println(anyObject)` sẽ không thể hoạt động
tổng quát cho **mọi** object (nó cần gọi được `toString()` trên bất kỳ object nào, kể cả class
chưa từng tồn tại lúc `println` được viết ra). `HashMap`/`HashSet` (Phase 2) cũng không thể
hoạt động tổng quát nếu không có `hashCode()`/`equals()` được đảm bảo tồn tại trên mọi object.
`Object` chính là "hợp đồng tối thiểu" mà mọi class trong hệ sinh thái Java cam kết tuân theo.

## How?

Các method chính của `Object` (tên đầy đủ `java.lang.Object`):

| Method | Việc gì |
| --- | --- |
| `toString()` | Trả về biểu diễn dạng chuỗi của object |
| `equals(Object o)` | So sánh hai object có "bằng nhau" không |
| `hashCode()` | Trả về mã băm (int) — dùng bởi `HashMap`/`HashSet`, xem Phase 2 |
| `getClass()` | Trả về object `Class` mô tả class thật của object (mở đường Reflection) |
| `clone()` | Tạo bản sao object (yêu cầu implement `Cloneable`, nhiều cạm bẫy — xem Deep Dive) |
| `wait()`/`notify()`/`notifyAll()` | Cơ chế đồng bộ hoá thread (chi tiết ở Phase 3) |
| `finalize()` | **Deprecated** — JVM gọi trước khi thu hồi object (xem Historical Note) |

Khi bạn không override, hành vi **mặc định** của mỗi method là:

- `toString()` mặc định trả về `TênClass@hexHashCode`.
- `equals()` mặc định so sánh **địa chỉ tham chiếu** (`==`) — hai object khác nhau dù nội
  dung field giống hệt nhau vẫn bị coi là "không bằng nhau".
- `hashCode()` mặc định trả về một số liên quan tới địa chỉ bộ nhớ/định danh nội bộ của object
  (không đảm bảo là chính địa chỉ vật lý, JVM tự quyết định thuật toán).

## Visualization

```
                    Object
                    + toString()
                    + equals(Object)
                    + hashCode()
                    + getClass()
                    + clone()
                    + wait()/notify()/notifyAll()
                    + finalize()  [deprecated]
                         ▲
                         │  extends (ngầm định, không cần viết)
                         │
                      Product
                    (không override gì cả
                     → dùng nguyên hành vi mặc định của Object)
```

## Example

Xác nhận hành vi mặc định thật (chưa override gì cả):

```java
public class ObjDemo {
    public static void main(String[] args) {
        Object o = new Object();
        System.out.println(o.toString());
        System.out.println(Integer.toHexString(o.hashCode()));

        Product p1 = new Product("Ao thun", 199000);
        Product p2 = new Product("Ao thun", 199000);
        System.out.println(p1.toString());
        System.out.println(p1.equals(p2));
        System.out.println(p1 == p2);
    }
}

class Product {
    String name;
    int price;
    Product(String name, int price) { this.name = name; this.price = price; }
}
```

Kết quả thật (JDK 17):

```
java.lang.Object@7ad041f3
7ad041f3
Product@7344699f
false
false
```

Ba điều đáng chú ý, đúng khớp phần How?:

1. `toString()` mặc định đúng format `TênClass@hexHashCode` — `7ad041f3` (in bằng
   `Integer.toHexString`) khớp chính xác với phần hex sau dấu `@` trong dòng `toString()`.
2. `p1.equals(p2)` trả về `false` dù `p1` và `p2` có `name`/`price` giống hệt nhau — vì
   `equals()` mặc định chỉ so sánh tham chiếu, đúng như `p1 == p2` cũng là `false`.
3. `Product` chưa override gì, nên thừa hưởng **nguyên xi** hành vi của `Object`.

## Deep Dive

**`clone()` là method gây nhiều tranh cãi nhất của `Object`.** Để dùng được, class phải
implement marker interface `Cloneable` (một interface rỗng, không có method nào — chỉ mang ý
nghĩa "đánh dấu") — nếu không, gọi `clone()` sẽ ném `CloneNotSupportedException`. Ngay cả khi
implement đúng, `clone()` mặc định chỉ tạo **shallow copy** (sao chép nông): nếu object chứa
field kiểu tham chiếu (ví dụ một `List` bên trong), bản sao và bản gốc sẽ **cùng trỏ tới một
List** — sửa list ở bản sao vô tình ảnh hưởng bản gốc. Vì sự rắc rối và dễ gây lỗi này,
*Effective Java* (Joshua Bloch — tham chiếu chính thức của handbook, xem References) khuyến
nghị **tránh dùng `clone()`**, thay bằng constructor sao chép (copy constructor) hoặc static
factory method tường minh hơn.

## Engineering Insight

**Vì sao `getClass()` không thể bị override, nhưng `toString()`/`equals()`/`hashCode()` thì
có?** `getClass()` được khai báo `final` trong `Object` — có chủ đích: nó là nguồn thông tin
đáng tin cậy duy nhất về "class thật sự" của một object lúc runtime, phục vụ Reflection
(Chapter 17) và cả JVM nội bộ (ví dụ cơ chế `instanceof`, cast kiểu). Nếu cho phép override,
một class có thể "nói dối" về chính danh tính của nó — phá vỡ toàn bộ các cơ chế dựa trên
`getClass()` một cách không an toàn. Ngược lại, `toString()`/`equals()`/`hashCode()` mô tả
**hành vi nghiệp vụ** (object này "trông như thế nào", "bằng nhau nghĩa là gì") — đây là điều
mỗi class có quyền và thường **nên** tự định nghĩa lại, nên Java để chúng ở dạng override
được (không `final`).

## Historical Note

```
Từ Java 1.0
    ↓
finalize() — JVM gọi method này TRƯỚC KHI thu hồi một object bằng GC (Chapter 07),
cho phép "dọn dẹp lần cuối" (đóng file, giải phóng tài nguyên native...)
    ↓
Vấn đề: finalize() không đảm bảo CHẮC CHẮN được gọi, không đảm bảo gọi ĐÚNG LÚC
(phụ thuộc hoàn toàn vào khi nào GC quyết định chạy — Chapter 07), có thể làm
GC chậm đáng kể, và dễ viết sai gây rò rỉ tài nguyên
    ↓
Java 9 (2017)
    ↓
finalize() bị đánh dấu @Deprecated(since="9") — khuyến nghị dùng try-with-resources
(AutoCloseable) hoặc java.lang.ref.Cleaner thay thế, cả hai đều đảm bảo tính
tất định (deterministic) hơn nhiều so với finalize()
```

## Myth vs Reality

- **Myth:** "In một object ra bằng `println` sẽ lỗi nếu class đó chưa định nghĩa gì cả."
  **Reality:** Luôn chạy được — nhờ hành vi mặc định `toString()` kế thừa từ `Object`
  (`TênClass@hexHashCode`), dù kết quả thường không hữu ích để đọc, không phải là lỗi.

- **Myth:** "Hai object có cùng nội dung field thì `equals()` mặc định sẽ trả về `true`."
  **Reality:** Xem Example — `equals()` mặc định của `Object` chỉ so sánh tham chiếu
  (giống `==`), **không** so sánh nội dung field, trừ khi class tự override lại (xem
  Chapter 11).

## Common Mistakes

- **In object trực tiếp để debug** (`System.out.println(product)`) rồi thất vọng vì chỉ thấy
  `Product@7344699f` không có thông tin gì hữu ích — quên rằng cần tự override `toString()`
  mới có output dễ đọc.
- **Giả định `equals()` mặc định đủ dùng** cho object đại diện dữ liệu (như `Product`), dẫn
  tới việc hai object "giống hệt nhau về nghiệp vụ" bị coi là khác nhau khi dùng trong
  `HashSet`/làm key của `HashMap` — nguồn gốc của rất nhiều bug khó hiểu ở Phase 2.
- **Override `equals()` mà quên override `hashCode()` theo** (hoặc ngược lại) — vi phạm hợp
  đồng `equals`/`hashCode`, chi tiết đầy đủ ở Chapter 11-12, chỉ nêu cảnh báo ở đây.

## Best Practices

- Luôn override `toString()` cho các class đại diện dữ liệu nghiệp vụ (như `Product`,
  `Order`) — cực kỳ hữu ích khi debug/log.
- Không dùng `clone()`/`Cloneable` cho code mới; ưu tiên copy constructor hoặc static factory
  method (xem Deep Dive).
- Không dùng `finalize()`; dùng try-with-resources (`AutoCloseable`) cho tài nguyên cần đóng
  tường minh (file, connection...).

## Production Notes

**Vấn đề:** log production đầy những dòng vô nghĩa dạng `Order@4a2d1a3f`,
`Product@1b28cdfa`, không giúp ích gì khi điều tra sự cố.

- **Triệu chứng:** khi debug production qua log, không thể biết object đang log là gì (giá
  trị field cụ thể nào), chỉ thấy địa chỉ hash vô nghĩa với người đọc.
- **Root cause:** các class nghiệp vụ chưa override `toString()`, đang dùng nguyên hành vi
  mặc định của `Object` — đúng như Common Mistakes đã nêu.
- **Solution:** override `toString()` cho mọi class đại diện dữ liệu quan trọng, in ra đủ
  field cần thiết để debug (cẩn thận: không in field nhạy cảm như mật khẩu, số thẻ...).
- **Prevention:** dùng IDE tự sinh `toString()`, hoặc annotation của thư viện (như Lombok
  `@ToString` — sẽ gặp ở Phase 5 khi làm việc với Spring) để không quên bước này khi tạo
  class mới; đưa việc "có `toString()` hữu ích" vào code review checklist.

## Debug Checklist

- [ ] Log/print ra `TênClass@hexHash` không đọc được gì? → class đó chưa override
      `toString()`.
- [ ] Hai object "trông giống hệt nhau" nhưng `equals()` trả `false`, hoặc bị coi là phần tử
      khác nhau trong `HashSet`? → kiểm tra class đã override `equals()`/`hashCode()` đúng
      cặp chưa (chi tiết Chapter 11-12).
- [ ] Dùng `clone()` nhưng sửa bản sao lại ảnh hưởng bản gốc? → nghi ngờ shallow copy (xem
      Deep Dive), kiểm tra field kiểu tham chiếu bên trong.

## Source Code Walkthrough

`java.lang.Object` là một trong số ít class có method native (`hashCode()`, `getClass()`,
`clone()`, `wait()`/`notify()`... đều khai báo `native`) — nghĩa là hiện thực thật nằm trong
code C++ của JVM (HotSpot), không phải bytecode Java thông thường. Đây là lý do bạn không thể
"đọc source Java" của `hashCode()` mặc định để biết chính xác thuật toán sinh ra số nào — nó
phụ thuộc vào hiện thực JVM cụ thể, không phải điều JLS quy định chi tiết (JLS chỉ yêu cầu:
cùng một object, gọi `hashCode()` nhiều lần trong cùng một lần chạy chương trình phải luôn trả
về cùng một giá trị — không yêu cầu gì hơn).

## Summary

Mọi class trong Java, kể cả không viết `extends` tường minh, đều ngầm kế thừa `Object` — đảm
bảo mọi object đều có sẵn `toString()`, `equals()`, `hashCode()`, `getClass()`, `clone()`,
`wait()`/`notify()`/`notifyAll()`. Hành vi **mặc định** của `toString()`/`equals()`/
`hashCode()` thường không phải điều bạn muốn cho class nghiệp vụ (chỉ dựa trên tham chiếu/địa
chỉ, không so sánh nội dung) — đây là lý do gần như luôn cần override chúng, chủ đề của
Chapter 10 (String, ví dụ override chuẩn mực) và Chapter 11-12 (equals/hashCode đào sâu).
`finalize()` đã deprecated từ Java 9, không nên dùng trong code mới.

## Interview Questions

**Junior**

- What are the methods present in the Object class?
- Nếu một class không viết `extends` gì, nó có lớp cha không? Là gì?

**Mid**

- `equals()` và `hashCode()` mặc định của `Object` hoạt động dựa trên điều gì? Vì sao thường
  không đủ dùng cho class nghiệp vụ?
- Vì sao `clone()`/`Cloneable` bị coi là thiết kế có vấn đề, nên tránh dùng trong code mới?

**Senior**

- Giải thích vì sao `getClass()` là `final` trong khi `toString()`/`equals()`/`hashCode()`
  thì không — liên hệ tới mục đích của từng method.
- `finalize()` bị deprecated từ Java 9 vì những lý do gì? Giải pháp thay thế
  (`AutoCloseable`/`Cleaner`) giải quyết vấn đề đó tốt hơn như thế nào?

## Exercises

- [ ] Chạy lại đúng ví dụ `ObjDemo` ở trên. Thêm `@Override public String toString()` cho
      `Product` (tự chọn format), chạy lại, so sánh output trước/sau.
- [ ] Viết một class implement `Cloneable`, có một field kiểu `List`. Gọi `clone()`, sửa
      `List` trong bản sao, quan sát bản gốc có bị ảnh hưởng không — xác nhận hiện tượng
      shallow copy bằng thực nghiệm.
- [ ] Tìm trong JLS hoặc Javadoc chính thức của `Object.hashCode()` — đọc đúng nguyên văn
      "hợp đồng" (contract) mà method này cam kết, chuẩn bị nền tảng cho Chapter 11-12.

## Cheat Sheet

| Method | Mặc định làm gì | Có nên override? |
| --- | --- | --- |
| `toString()` | `TênClass@hexHashCode` | Nên, cho mọi class nghiệp vụ |
| `equals(Object)` | So sánh tham chiếu (`==`) | Nên, nếu cần so sánh theo nội dung |
| `hashCode()` | Số liên quan định danh nội bộ | Bắt buộc nếu đã override `equals()` |
| `getClass()` | Trả về `Class` thật của object | Không thể (`final`) |
| `clone()` | Shallow copy (cần `Cloneable`) | Tránh dùng — ưu tiên copy constructor |
| `wait()`/`notify()`/`notifyAll()` | Đồng bộ hoá thread | Không override, chỉ gọi đúng ngữ cảnh (Phase 3) |
| `finalize()` | Dọn dẹp trước khi GC thu hồi | **Deprecated**, không dùng |

## References

- Java Language Specification (JLS) — Chapter 4.3.2: The Class Object.
- `java.lang.Object` — Java SE API Documentation (Oracle).
- Effective Java (Joshua Bloch) — Item 11 (hashCode), Item 13 (clone), Item 12 (toString).
