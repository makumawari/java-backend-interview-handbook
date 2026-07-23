---
tags:
  - Java
  - Record
  - ModernJava
---

# Record

> Phase: Phase 4 — Modern Java
> Chapter slug: `record`

## Metadata

```yaml
Chapter: Record
Phase: Phase 4 — Modern Java
Difficulty: ★★
Importance: ★★★★★
Interview Frequency: 80%
Prerequisites:
  - Phase 1, Chapter 09 — Object (equals/hashCode/toString)
  - Phase 1, Chapter 12-13 — equals()/hashCode()
Used Later:
  - Chapter 09 — Pattern Matching (record pattern)
  - DTO trong tầng REST API (Phase 5)
Estimated Reading: 15 phút
Estimated Practice: 15 phút
```

## Story

> Xem lại: Phase 1, Chapter 12-13 (equals/hashCode) — để viết đúng một class chỉ đơn thuần
> "gói" vài field bất biến (ví dụ `Point(x, y)`), phải viết tay constructor, getter,
> `equals()`, `hashCode()`, `toString()` — hàng chục dòng boilerplate cho một class chỉ có 2
> field.

So sánh: khai báo một `record Point(int x, int y)` — chỉ một dòng — rồi kiểm tra `javap` xem
trình biên dịch thực sự sinh ra những gì:

```
final class RecordDemo$Point extends java.lang.Record {
  RecordDemo$Point(int, int);
  double distanceFromOrigin();
  public final java.lang.String toString();
  public final int hashCode();
  public final boolean equals(java.lang.Object);
  public int x();
  public int y();
}
```

Chỉ một dòng khai báo, nhưng trình biên dịch tự động sinh ra constructor, `toString()`,
`hashCode()`, `equals()`, và accessor cho từng field — đúng những gì trước đây phải viết tay.

## Interview Question (Central)

> Record trong Java là gì? Nó khác một class thông thường như thế nào, và trình biên dịch tự
> động sinh ra những gì?

## Objectives

- [ ] Giải thích chính xác những gì trình biên dịch tự động sinh ra cho một `record`
- [ ] Tự tay chứng minh bằng `javap`: record ngầm định `final`, `extends java.lang.Record`
- [ ] Dùng compact constructor để validate dữ liệu ngay khi khởi tạo

## Prerequisites

- Phase 1, Chapter 09, 12-13 — hiểu ý nghĩa và cách viết tay `equals()`/`hashCode()`/`toString()`, để thấy rõ giá trị record mang lại.

## Used Later

- **Chapter 09 (Pattern Matching)** — record pattern cho phép "destructure" (tách) trực tiếp
  các field của record trong biểu thức `switch`/`instanceof`.
- **DTO trong tầng REST API** (Phase 5) — record là lựa chọn tự nhiên cho Data Transfer Object
  bất biến.

## Problem

Một class chỉ đơn thuần "gói" một nhóm giá trị bất biến (immutable data carrier — ví dụ toạ độ,
DTO, value object) đòi hỏi rất nhiều boilerplate: constructor gán từng field, getter cho từng
field, `equals()`/`hashCode()` so sánh đúng theo giá trị (Phase 1, Chapter 12-13), và
`toString()` để debug dễ đọc. Viết tay tất cả những thứ này cho mỗi class dữ liệu đơn giản vừa
tốn công vừa dễ mắc lỗi (quên cập nhật `equals()` khi thêm field mới, ví dụ).

## Concept

**`record`** (Java 16+) là một loại khai báo class đặc biệt, tối ưu cho việc "gói" dữ liệu bất
biến — chỉ cần khai báo tên và danh sách field (gọi là "record components"), trình biên dịch tự
động sinh ra: constructor nhận đủ tham số, accessor method cho từng field (tên trùng tên field,
không có tiền tố `get`), `equals()`/`hashCode()` so sánh theo giá trị của tất cả field,
`toString()` hiển thị tên record và giá trị từng field.

## Why?

Record giải quyết đúng vấn đề boilerplate bằng cách đảo ngược mặc định: thay vì lập trình viên
phải **viết ra** từng phần (constructor, getter, equals...), trình biên dịch **tự động suy ra**
tất cả từ một khai báo duy nhất — vì với một data carrier thuần tuý, "cách đúng" để viết
`equals()`/`hashCode()`/`toString()` gần như luôn giống nhau (so sánh/hiển thị tất cả field),
không có lý do gì để bắt lập trình viên viết lại logic lặp đi lặp lại đó mỗi lần. Đồng thời,
record **ngầm định bất biến** (mọi field đều `final`) và **ngầm định `final`** (không thể kế
thừa) — phản ánh đúng bản chất một data carrier không nên bị thay đổi trạng thái sau khi tạo,
hoặc bị mở rộng theo hướng thêm hành vi phức tạp.

## How?

```java
record Point(int x, int y) {
    // compact constructor: them validation, chay TRUOC khi field duoc gan
    Point {
        if (x < 0 || y < 0) throw new IllegalArgumentException("x, y phai >= 0");
    }

    // van co the them phuong thuc instance thong thuong
    double distanceFromOrigin() { return Math.sqrt(x * x + y * y); }
}
```

## Visualization

```
record Point(int x, int y) { ... }
         │
         ▼  trinh bien dich TU DONG sinh ra:

final class Point extends java.lang.Record {
    private final int x;              <- field TU DONG, final
    private final int y;

    Point(int x, int y) { ... }       <- constructor TU DONG

    public int x() { return x; }      <- accessor TU DONG (KHONG co "get")
    public int y() { return y; }

    public boolean equals(Object o) { <- TU DONG, so sanh theo GIA TRI
        // so sanh x va y
    }
    public int hashCode() { ... }     <- TU DONG, tu x va y
    public String toString() { ... }  <- TU DONG: "Point[x=.., y=..]"
}
```

## Example

```java
public class RecordDemo {
    record Point(int x, int y) {
        Point {
            if (x < 0 || y < 0) {
                throw new IllegalArgumentException("x, y phai >= 0, nhan (" + x + ", " + y + ")");
            }
        }

        double distanceFromOrigin() {
            return Math.sqrt(x * x + y * y);
        }
    }

    public static void main(String[] args) {
        Point p1 = new Point(3, 4);
        Point p2 = new Point(3, 4);

        System.out.println("p1 = " + p1);
        System.out.println("p1.x() = " + p1.x() + ", p1.y() = " + p1.y());
        System.out.println("p1.equals(p2): " + p1.equals(p2) + " (so sanh theo GIA TRI)");
        System.out.println("p1 == p2: " + (p1 == p2) + " (khac object)");
        System.out.println("p1.hashCode() == p2.hashCode(): " + (p1.hashCode() == p2.hashCode()));
        System.out.println("p1.distanceFromOrigin() = " + p1.distanceFromOrigin());

        try {
            new Point(-1, 5);
        } catch (IllegalArgumentException e) {
            System.out.println("Compact constructor validation: " + e.getMessage());
        }
    }
}
```

Kết quả thật (JDK 17):

```
p1 = Point[x=3, y=4]
p1.x() = 3, p1.y() = 4
p1.equals(p2): true (so sanh theo GIA TRI, khong phai reference)
p1 == p2: false (khac object)
p1.hashCode() == p2.hashCode(): true
p1.distanceFromOrigin() = 5.0
Compact constructor validation: x, y phai >= 0, nhan (-1, 5)
```

`p1.equals(p2)` trả về `true` dù `p1 == p2` là `false` (hai object khác nhau trên heap) — đúng
hành vi `equals()` tự động sinh, so sánh theo **giá trị** từng field, không phải reference.
Compact constructor chạy validation **trước** khi field được gán — `new Point(-1, 5)` ném
`IllegalArgumentException` ngay lập tức, không tạo ra object với dữ liệu không hợp lệ.

**Record ngầm định `final` — thử kế thừa:**

```java
record Point(int x, int y) {}
class Point3D extends Point { ... } // loi bien dich
```

Kết quả biên dịch thật:

```
error: cannot inherit from final Point
class Point3D extends Point {
                      ^
```

## Deep Dive

**Compact constructor khác constructor thông thường như thế nào?** Constructor thông thường của
record (`Point(int x, int y) { this.x = x; this.y = y; }`) phải tự gán từng field tường minh.
**Compact constructor** (`Point { ... }`, không có danh sách tham số lặp lại) chạy **trước** khi
việc gán field tự động diễn ra — bên trong nó, các tham số vẫn dùng đúng tên field (`x`, `y`)
nhưng chưa phải là field thật sự, chỉ là tham số cục bộ. Sau khi khối lệnh compact constructor
chạy xong (không ném exception), trình biên dịch **tự động chèn thêm** đoạn `this.x = x; this.y
= y;` — đây là lý do compact constructor rất phù hợp cho validation (như ví dụ) hoặc chuẩn hoá
dữ liệu đầu vào (ví dụ `trim()` một chuỗi) mà không cần lặp lại logic gán field.

## Engineering Insight

**Vì sao record đặc biệt phù hợp làm DTO (Data Transfer Object) trong tầng REST API, hơn hẳn
class POJO truyền thống?** DTO về bản chất là dữ liệu **bất biến, chỉ để truyền tải** giữa các
tầng — record khớp hoàn hảo với bản chất này: bất biến ngầm định (không cần `final` field thủ
công), `equals()`/`hashCode()` đúng theo giá trị ngay từ đầu (hữu ích khi so sánh DTO trong test),
và cú pháp cực ngắn gọn giúp định nghĩa hàng chục DTO khác nhau mà không tạo gánh nặng bảo trì.
Tuy nhiên record **không phù hợp** cho JPA Entity (Phase 6) — Entity cần trạng thái thay đổi
được (mutable, để Hibernate theo dõi thay đổi), constructor không tham số (Hibernate proxy cần),
và không được `final` (Hibernate cần tạo proxy class kế thừa) — tất cả đều mâu thuẫn trực tiếp
với bản chất bất biến, ngầm định `final` của record.

## Historical Note

Record được đề xuất từ Project Amber (cùng nhóm chịu trách nhiệm cho `var`, pattern matching),
trải qua 2 vòng preview (JEP 359 ở Java 14, JEP 384 ở Java 15) trước khi chính thức ổn định ở
**Java 16** (JEP 395, tháng 3/2021). Đây là một phần trong chuỗi cải tiến giảm boilerplate của
Java, cùng hướng với Lombok (thư viện bên thứ ba từng phổ biến để giải quyết đúng vấn đề này
trước khi record ra đời chính thức trong ngôn ngữ).

## Myth vs Reality

- **Myth:** "Record chỉ là một class thông thường với cú pháp ngắn gọn hơn, không có ràng buộc
  gì đặc biệt."
  **Reality:** Đã chứng minh bằng thực nghiệm — record ngầm định `final` (không thể kế thừa),
  ngầm định `extends java.lang.Record`, và mọi field đều bất biến — đây là những ràng buộc thực
  sự, không chỉ là cú pháp.

- **Myth:** "Nên dùng record cho mọi Entity JPA để giảm boilerplate."
  **Reality:** Xem Engineering Insight — record không tương thích với các yêu cầu kỹ thuật của
  JPA Entity (mutable state, no-args constructor, không được final).

## Common Mistakes

- **Cố kế thừa một record** — lỗi biên dịch ngay lập tức, vì record ngầm định `final`.
- **Dùng record làm JPA Entity** — gây lỗi hoặc hành vi không mong muốn với Hibernate (không
  tạo được proxy, không track được thay đổi trạng thái).
- **Quên rằng compact constructor phải hoàn tất việc gán field một cách nhất quán** — không thể
  chỉ gán một phần field trong compact constructor rồi để trình biên dịch tự gán phần còn lại
  khác đi; toàn bộ field được gán tự động sau khi compact constructor chạy xong.

## Best Practices

- Dùng record cho DTO, value object, kết quả trả về của phương thức (đặc biệt kết hợp record
  pattern ở Chapter 09) — bất cứ nơi nào cần một "gói dữ liệu" bất biến, so sánh theo giá trị.
- Dùng compact constructor để validate dữ liệu đầu vào ngay khi khởi tạo, tránh tạo ra object ở
  trạng thái không hợp lệ.
- Không dùng record cho JPA Entity hoặc bất kỳ trường hợp nào cần trạng thái thay đổi được sau
  khi tạo.

## Source Code Walkthrough

Kiểm tra bytecode thực tế bằng `javap` xác nhận chính xác những gì trình biên dịch sinh ra:

```
$ javap RecordDemo\$Point.class
final class RecordDemo$Point extends java.lang.Record {
  RecordDemo$Point(int, int);
  double distanceFromOrigin();
  public final java.lang.String toString();
  public final int hashCode();
  public final boolean equals(java.lang.Object);
  public int x();
  public int y();
}
```

`final class ... extends java.lang.Record` xác nhận trực tiếp hai ràng buộc đã nêu ở Concept:
record luôn kế thừa `java.lang.Record` (lớp trừu tượng nền tảng của mọi record, tương tự cách
mọi enum kế thừa `java.lang.Enum` — Phase 1, Chapter 17) và luôn `final`.
`toString()`/`hashCode()`/`equals()` đều là `final` — không thể override một phần, chỉ có thể
override toàn bộ nếu thực sự cần logic khác (hiếm khi cần thiết với đúng bản chất record).

## Summary

Record cho phép khai báo một data carrier bất biến chỉ trong một dòng — trình biên dịch tự động
sinh constructor, accessor, `equals()`/`hashCode()` (so sánh theo giá trị), `toString()`. Đã
chứng minh bằng `javap`: record ngầm định `final class ... extends java.lang.Record`. Compact
constructor cho phép validate dữ liệu trước khi field được gán tự động. Record là lựa chọn lý
tưởng cho DTO/value object nhưng **không phù hợp** cho JPA Entity do bản chất bất biến và ngầm
định `final` mâu thuẫn với yêu cầu kỹ thuật của Hibernate.

## Interview Questions

- What are the new features introduced in Java 21?
- Coding: Create a Record class in Java 21.
- What are the new features introduced in Java 17?
- Coding: Create a Record class in Java 17.

**Mid**

- Record khác một class thông thường như thế nào? Trình biên dịch tự động sinh ra những gì?
- Vì sao không nên dùng record làm JPA Entity?

## Exercises

- [ ] Chạy lại `RecordDemo` ở trên, dùng `javap` để tự xác nhận những gì trình biên dịch sinh
      ra cho record `Point`.
- [ ] Thử biên dịch một record cố tình kế thừa một record khác, xác nhận lỗi biên dịch đúng như
      mô tả.
- [ ] Viết một record `Money(long amountInCents, String currency)` với compact constructor
      validate `amountInCents >= 0` và `currency` có đúng 3 ký tự.

## Cheat Sheet

| Đặc điểm | Record | Class thông thường |
| --- | --- | --- |
| Field | Ngầm định `final` (bất biến) | Tuỳ khai báo |
| Accessor | Tự động sinh (`x()`, không có `get`) | Phải viết tay |
| `equals`/`hashCode`/`toString` | Tự động sinh, theo giá trị | Phải viết tay (hoặc IDE sinh) |
| Kế thừa | Ngầm định `final`, không kế thừa được | Kế thừa được (trừ khi khai báo `final`) |
| Kế thừa class khác | Không thể (chỉ implement interface) | Được |
| Phù hợp cho | DTO, value object | Entity, class có hành vi phức tạp |

## References

- JEP 395: Records — https://openjdk.org/jeps/395
- Java SE API Documentation — `java.lang.Record`.
