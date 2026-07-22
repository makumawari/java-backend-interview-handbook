---
tags:
  - Java
  - Enum
  - Foundation
---

# Enum

> Phase: Phase 1 — Java Foundation
> Chapter slug: `enum`

## Metadata

```yaml
Chapter: Enum
Phase: Phase 1 — Java Foundation
Difficulty: ★★
Importance: ★★
Interview Frequency: 40%
Prerequisites:
  - Chapter 08 — OOP Fundamentals
  - Chapter 09 — Object
Used Later:
  - Sealed Class, Pattern Matching (Phase 4) — Enum là "phiên bản hạn chế" của ý tưởng đóng (closed) kiểu dữ liệu
  - Entity, JPA (Phase 6) — Enum làm field Entity là tình huống rất phổ biến, có cạm bẫy riêng
Estimated Reading: 15 phút
Estimated Practice: 15 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

Trước khi Java có `enum` (trước Java 5), lập trình viên biểu diễn một tập giá trị cố định
(trạng thái đơn hàng: chờ xử lý, đã giao, đã huỷ...) bằng hằng số `int`:

```java
public static final int STATUS_PENDING = 0;
public static final int STATUS_SHIPPED = 1;
public static final int STATUS_CANCELLED = 2;

void updateStatus(int status) { ... }
```

Cách này có hai lỗ hổng nghiêm trọng: (1) `updateStatus(99)` — một số hoàn toàn vô nghĩa —
biên dịch được bình thường, lỗi chỉ lộ ra lúc runtime, nếu có ai kiểm tra; (2) in ra
`STATUS_SHIPPED` chỉ thấy số `1`, không có thông tin gì tự giải thích nó là gì.

Java 5 (2004) giới thiệu `enum` để giải quyết cả hai vấn đề — nhưng câu hỏi thú vị hơn nhiều
so với cú pháp là: **`enum` thực chất là gì bên dưới?** Chỉ là hằng số `int` được "trang trí"
đẹp hơn, hay là một thứ hoàn toàn khác? Chapter này trả lời bằng cách mở bytecode ra xem trực
tiếp.

## Interview Question (Central)

> Enum trong Java thực chất là gì bên dưới lớp vỏ cú pháp — chỉ là hằng số `int` được đặt tên
> đẹp, hay là một cấu trúc hoàn toàn khác?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Biết chính xác: mỗi `enum` được `javac` biên dịch thành một **class** kế thừa
      `java.lang.Enum`
- [ ] Hiểu mỗi hằng số enum (`PENDING`, `SHIPPED`...) là một **singleton instance** static,
      không phải một số
- [ ] Viết được enum có constructor, field, và method riêng cho từng hằng số
- [ ] Biết vì sao `enum` an toàn hơn hẳn hằng số `int` cho `switch`, so sánh, và làm key
      `HashMap`

## Prerequisites

- Chapter 08 — hiểu `abstract`, `extends`, override method.
- Chapter 09 — hiểu `Object`, vì `Enum` cũng kế thừa từ nó (gián tiếp).

## Used Later

- **Sealed Class, Pattern Matching** (Phase 4) — `enum` là dạng sơ khai nhất của ý tưởng "một
  kiểu dữ liệu chỉ có hữu hạn dạng cố định, biết trước toàn bộ lúc biên dịch" — sealed class
  (Java 17+) tổng quát hoá ý tưởng này cho trường hợp phức tạp hơn hằng số đơn thuần.
- **Entity, JPA** (Phase 6) — dùng `enum` làm field của Entity ánh xạ xuống database là tình
  huống cực kỳ phổ biến, có cạm bẫy riêng (lưu theo `ORDINAL` hay `STRING`) cần hiểu rõ bản
  chất enum ở chapter này trước.

## Problem

Hằng số `int`/`String` không có sự ràng buộc nào ở tầng kiểu dữ liệu — bất kỳ số nguyên hay
chuỗi nào cũng "hợp lệ về mặt cú pháp" dù vô nghĩa về mặt nghiệp vụ. Cần một cách để trình biên
dịch **tự chặn** việc gán giá trị không nằm trong tập hợp lệ, đồng thời giữ được khả năng tự
mô tả (đọc code thấy ngay `OrderStatus.SHIPPED` thay vì con số `1` vô nghĩa).

## Concept

`enum` định nghĩa một tập **hữu hạn, cố định, biết trước lúc biên dịch** các giá trị khả dĩ.
Khác biệt cốt lõi so với hằng số nguyên thuỷ: **mỗi hằng số enum là một object thật**, không
phải một con số — cụ thể là một **singleton instance** của chính class enum đó.

```java
public enum OrderStatus {
    PENDING, SHIPPED, DELIVERED;
}
```

## Why?

Nếu `enum` chỉ là hằng số `int` được đổi tên: trình biên dịch vẫn không thể chặn được việc
truyền một giá trị số ngoài phạm vi hợp lệ (vì bản chất vẫn chỉ là `int`). Bằng cách biến mỗi
hằng số thành một **object thuộc đúng kiểu enum đó**, trình biên dịch có thể áp dụng toàn bộ
cơ chế kiểm tra kiểu thông thường (giống mọi class khác) — tham số khai báo kiểu
`OrderStatus` chỉ chấp nhận đúng object `PENDING`/`SHIPPED`/`DELIVERED`, không chấp nhận bất kỳ
giá trị nào khác, được chặn ngay lúc biên dịch.

## How?

Xác nhận trực tiếp: `enum` được `javac` biên dịch thành gì. Với khai báo:

```java
public enum OrderStatus {
    PENDING, PAID, SHIPPED, DELIVERED;
}
```

Chạy `javap OrderStatus.class`, kết quả thật (JDK 17):

```
public final class OrderStatus extends java.lang.Enum<OrderStatus> {
  public static final OrderStatus PENDING;
  public static final OrderStatus PAID;
  public static final OrderStatus SHIPPED;
  public static final OrderStatus DELIVERED;
  public static OrderStatus[] values();
  public static OrderStatus valueOf(java.lang.String);
  static {};
}
```

Ba điều then chốt:

1. `enum` biên dịch thành một **`class` thật**, `final` (không cho kế thừa thêm — Java không
   hỗ trợ đa kế thừa, mỗi enum đã "dùng hết suất" `extends` để kế thừa `Enum`, đúng giới hạn
   đã học ở [Chapter 08](08-oop-fundamentals.md)), kế thừa `java.lang.Enum<OrderStatus>`.
2. Mỗi hằng số (`PENDING`, `PAID`...) là một field **`public static final OrderStatus`** —
   một singleton instance, không phải số.
3. `javac` tự sinh thêm hai method `values()` (trả về mảng tất cả hằng số) và `valueOf(String)`
   (tra cứu theo tên) — bạn không viết chúng, trình biên dịch tự thêm vào.

## Visualization

```
enum OrderStatus { PENDING, PAID, SHIPPED, DELIVERED }
              │
              │  javac biên dịch thành
              ▼
final class OrderStatus extends Enum<OrderStatus> {
    public static final OrderStatus PENDING  = new OrderStatus("PENDING", 0);
    public static final OrderStatus PAID     = new OrderStatus("PAID", 1);
    public static final OrderStatus SHIPPED  = new OrderStatus("SHIPPED", 2);
    public static final OrderStatus DELIVERED = new OrderStatus("DELIVERED", 3);
    // + values(), valueOf() tự sinh
}
       ▲
       │  mỗi hằng số = MỘT SINGLETON INSTANCE, khởi tạo trong static block
       │  (đúng cơ chế Initialization đã học ở Chapter 14 — CHỈ MỘT LẦN)
```

## Example

Enum với constructor, field riêng, và **method khác nhau cho từng hằng số** (`abstract
method` được mỗi hằng số tự override) — tính năng nâng cao nhưng rất thực dụng:

```java
public enum OrderStatus2 {
    PENDING("Cho xu ly") {
        @Override public boolean canCancel() { return true; }
    },
    SHIPPED("Da giao van chuyen") {
        @Override public boolean canCancel() { return false; }
    };

    private final String label;
    OrderStatus2(String label) { this.label = label; }
    public String getLabel() { return label; }
    public abstract boolean canCancel();

    public static void main(String[] args) {
        for (OrderStatus2 s : values()) {
            System.out.println(s + " (" + s.getLabel() + ") canCancel=" + s.canCancel());
        }
        System.out.println("PENDING == PENDING: " + (PENDING == valueOf("PENDING")));
    }
}
```

Kết quả thật (JDK 17):

```
PENDING (Cho xu ly) canCancel=true
SHIPPED (Da giao van chuyen) canCancel=false
PENDING == PENDING: true
```

Mỗi hằng số enum tự override `canCancel()` theo đúng tinh thần Polymorphism đã học ở
[Chapter 08](08-oop-fundamentals.md) — thay vì một khối `switch` dài kiểm tra từng loại trạng
thái, mỗi hằng số "tự biết" hành vi của chính nó. `PENDING == valueOf("PENDING")` là `true` —
xác nhận: vì mỗi hằng số là **singleton** (chỉ tạo một lần duy nhất trong static block), so
sánh bằng `==` luôn an toàn và chính xác cho enum, không cần `.equals()` như `String` (Chapter
10) — dù `.equals()` vẫn hoạt động đúng (kế thừa từ `Object`, dùng chính `==` bên trong vì đã
là singleton).

## Deep Dive

**Vì sao `enum` được phép dùng trong `switch` mà không cần viết đầy đủ tên enum
(`case PENDING:` thay vì `case OrderStatus.PENDING:`)?** Đây là cú pháp đặc biệt `javac` dành
riêng cho `enum` trong `switch` — vì trình biên dịch **đã biết chắc chắn** kiểu của biểu thức
trong `switch` là `OrderStatus`, nó tự suy ra `case PENDING` nghĩa là
`case OrderStatus.PENDING`. Quan trọng hơn về mặt an toàn: trình biên dịch **kiểm tra được**
liệu `switch` đã xử lý **đủ tất cả** hằng số của enum hay chưa (đặc biệt rõ với `switch`
expression hiện đại, Phase 4) — một lợi ích không thể có với hằng số `int`/`String`, vì
compiler không có cách nào biết "tập giá trị hợp lệ" của một `int` là gì.

## Engineering Insight

**Vì sao mỗi hằng số enum là singleton lại quan trọng hơn nhiều so với vẻ ngoài "chi tiết cài
đặt"?** Vì nó là nền tảng cho tính **an toàn tuyệt đối khi dùng làm key `HashMap`/phần tử
`HashSet`** (Chapter 12-13, Phase 2): `hashCode()` mặc định của `Enum` (kế thừa nguyên xi từ
`Object` — Chapter 09, dựa trên định danh nội bộ) không bao giờ đổi trong suốt vòng đời chương
trình, vì **không thể tạo thêm instance mới** của bất kỳ hằng số nào (constructor enum không
thể gọi từ bên ngoài, chỉ `javac` gọi được trong static block, xem Visualization). Đây là lý
do `EnumMap`/`EnumSet` (một biến thể `Map`/`Set` tối ưu riêng cho `enum`, sẽ gặp lại ở Phase 2)
có thể cài đặt cực nhanh dựa trên chính thứ tự khai báo (`ordinal()`) thay vì hash bucket thông
thường — một tối ưu chỉ khả thi nhờ đặc tính "singleton, số lượng cố định, biết trước lúc biên
dịch" của enum.

## Historical Note

```
Trước Java 5 (2004)
    ↓
"Typesafe enum pattern" — một pattern thủ công (không phải từ khoá riêng),
lập trình viên tự viết class với constructor private + các static final
instance, gần giống hệt những gì javac TỰ ĐỘNG sinh ra cho enum ngày nay
    ↓
Java 5 (2004)
    ↓
Từ khoá enum ra đời — chính thức hoá pattern trên thành cú pháp ngôn ngữ,
javac tự sinh values()/valueOf(), tự đảm bảo singleton, hỗ trợ switch
```

Điều thú vị: `enum` của Java không phải phát minh cú pháp hoàn toàn mới — nó "đóng gói" một
pattern mà lập trình viên Java kinh nghiệm đã tự viết tay từ trước, thành cú pháp ngôn ngữ
chính thức, an toàn hơn và ít code hơn.

## Myth vs Reality

- **Myth:** "`enum` về bản chất chỉ là hằng số `int` được đặt tên đẹp hơn."
  **Reality:** Xem How? — `enum` biên dịch thành một **class thật**, mỗi hằng số là một
  **object singleton**, không phải số. Đây là khác biệt về bản chất, không chỉ về cú pháp.

- **Myth:** "So sánh enum nên dùng `.equals()` để an toàn, giống String (Chapter 10)."
  **Reality:** Ngược lại — vì mỗi hằng số enum là singleton duy nhất (không có "bản sao" như
  `new String(...)` từng gây rắc rối ở Chapter 10), so sánh bằng `==` luôn an toàn và là cách
  được khuyến nghị cho enum, nhanh hơn `.equals()` (không cần gọi method, không sợ
  `NullPointerException` nếu vế trái là hằng số enum cố định:
  `if (OrderStatus.PENDING == status)`).

## Common Mistakes

- **Lưu enum xuống database bằng `ordinal()` (vị trí khai báo, 0/1/2...)** thay vì tên
  (`name()`/`toString()`) — nếu sau này thêm/xoá/đổi thứ tự hằng số, toàn bộ dữ liệu cũ trong
  database sẽ bị đọc sai (ordinal thay đổi ý nghĩa) mà không có bất kỳ lỗi biên dịch/runtime
  nào báo hiệu — một trong những cạm bẫy Enum + JPA (Phase 6) phổ biến và nguy hiểm nhất.
- **Thêm logic `switch` dài dòng cho từng hằng số ở bên ngoài enum**, thay vì tận dụng
  `abstract method` + override riêng cho từng hằng số như ở Example — bỏ lỡ lợi ích chính của
  Polymorphism đã học ở Chapter 08.
- **Cố tạo thêm instance bằng reflection/serialization không đúng cách** — phá vỡ tính chất
  singleton, gây lỗi khó lường ở những chỗ giả định enum luôn singleton (như so sánh `==`).

## Best Practices

- Luôn so sánh enum bằng `==`, không cần `.equals()`.
- Khi lưu enum xuống database (JPA, Phase 6), luôn dùng `@Enumerated(EnumType.STRING)` (lưu
  theo tên) thay vì `EnumType.ORDINAL` (mặc định, lưu theo vị trí) — an toàn hơn hẳn khi cấu
  trúc enum thay đổi theo thời gian.
- Với logic khác nhau đáng kể giữa các hằng số, cân nhắc `abstract method` + override riêng
  (như Example) thay vì `switch` bên ngoài — code gọn hơn, và trình biên dịch tự nhắc bạn nếu
  quên implement cho hằng số mới thêm vào.

## Production Notes

**Vấn đề:** sau một lần deploy thêm một trạng thái đơn hàng mới vào giữa danh sách enum có
sẵn, dữ liệu đơn hàng cũ trong database bỗng nhiên hiển thị sai trạng thái hàng loạt.

- **Triệu chứng:** đơn hàng "PENDING" cũ hiển thị nhầm thành "SHIPPED" hoặc trạng thái khác
  hoàn toàn không liên quan, dữ liệu mới tạo thì vẫn đúng.
- **Root cause:** enum được lưu bằng `EnumType.ORDINAL` (vị trí khai báo); thêm một hằng số
  mới **ở giữa** danh sách (không phải ở cuối) làm dịch chuyển toàn bộ `ordinal()` của các
  hằng số phía sau nó — dữ liệu cũ trong database (lưu số nguyên) giờ trỏ sang **ý nghĩa
  khác** so với lúc được lưu.
- **Debug:** kiểm tra annotation `@Enumerated` trên field enum của Entity liên quan — nếu là
  `ORDINAL` (hoặc không có annotation, mặc định chính là `ORDINAL`), đây gần như chắc chắn là
  nguyên nhân.
- **Solution:** chuyển sang `EnumType.STRING`, kèm theo một bước migrate dữ liệu cũ (chuyển từ
  số đang lưu sang đúng tên hằng số tương ứng) — cần làm cẩn thận vì đây chính là bước dễ sai
  sót nếu thứ tự đã bị dịch chuyển từ trước.
- **Prevention:** quy định bắt buộc dùng `EnumType.STRING` cho mọi enum ánh xạ xuống database
  ngay từ đầu dự án (Phase 6); nếu bắt buộc phải thêm hằng số enum mới, luôn thêm vào **cuối**
  danh sách, không chèn vào giữa, kể cả khi đã dùng `STRING` (thói quen an toàn nói chung).

## Debug Checklist

- [ ] Dữ liệu enum đọc từ database sai sau khi sửa danh sách enum? → kiểm tra `@Enumerated`
      đang dùng `ORDINAL` hay `STRING` (xem Production Notes).
- [ ] So sánh enum cho kết quả không mong đợi? → xác nhận đang dùng `==` (khuyến nghị) hay
      `.equals()`, cả hai đều nên cho cùng kết quả nếu không có gì bất thường về serialization/
      reflection.
- [ ] `IllegalArgumentException: No enum constant ...`? → `valueOf(String)` được gọi với một
      chuỗi không khớp chính xác tên hằng số nào (phân biệt hoa/thường) — kiểm tra nguồn dữ
      liệu đầu vào.

## Source Code Walkthrough

Đã thực hiện trực tiếp ở phần How?/Visualization bằng `javap`: `enum` biên dịch thành class
`final` kế thừa `java.lang.Enum<T>`, các hằng số là field `public static final`, khởi tạo
trong static block (`static {}` xuất hiện trong output `javap`) — đúng cơ chế Initialization
đã học ở [Chapter 14](14-object-lifecycle.md), chỉ chạy một lần duy nhất khi class enum lần
đầu được dùng chủ động.

## Summary

`enum` không phải hằng số `int` được "trang trí" — nó biên dịch thành một **class thật**
(`final`, kế thừa `java.lang.Enum<T>`), mỗi hằng số là một **singleton instance** được khởi
tạo đúng một lần trong static block. Chính vì là singleton, so sánh bằng `==` luôn an toàn
(khuyến nghị hơn `.equals()`); vì `javac` tự sinh `values()`/`valueOf()` và kiểm tra kiểu ở
tầng biên dịch, enum an toàn hơn hẳn hằng số nguyên thuỷ. Enum còn hỗ trợ constructor, field,
và `abstract method` override riêng cho từng hằng số — công cụ mạnh để thay thế các khối
`switch` dài dòng bằng Polymorphism (Chapter 08). Cạm bẫy quan trọng nhất khi dùng thực tế:
luôn lưu enum xuống database bằng tên (`EnumType.STRING`), không bằng vị trí (`ORDINAL`).

## Interview Questions

**Junior**

- Enum trong Java thực chất là gì bên dưới?
- So sánh hai giá trị enum nên dùng `==` hay `.equals()`?

**Mid**

- `javac` tự sinh thêm những gì cho một class enum mà bạn không viết tường minh?
- Vì sao lưu enum xuống database bằng `ordinal()` lại nguy hiểm? Cho một kịch bản cụ thể.

**Senior**

- Giải thích vì sao mỗi hằng số enum là singleton lại là nền tảng cho việc `EnumMap`/`EnumSet`
  có thể tối ưu hơn `HashMap`/`HashSet` thông thường.
- So sánh việc dùng `abstract method` override riêng cho từng hằng số enum (như ví dụ trong
  chapter) với việc dùng `switch` bên ngoài để xử lý logic khác nhau theo từng hằng số — đánh
  đổi giữa hai cách tiếp cận là gì?

## Exercises

- [ ] Chạy `javap OrderStatus.class` trên một enum bạn tự viết, xác nhận cấu trúc giống hệt
      những gì chapter đã mô tả.
- [ ] Chạy lại đúng ví dụ `OrderStatus2` ở trên, thêm một hằng số mới `DELIVERED` với
      `canCancel()` trả về `false`, xác nhận trình biên dịch **bắt buộc** bạn phải implement
      `canCancel()` cho hằng số mới (không implement sẽ báo lỗi biên dịch).
- [ ] Viết một `switch` expression (Phase 4, hoặc `switch` thường nếu chưa học tới) xử lý đầy
      đủ mọi hằng số của một enum — thử xoá một `case`, quan sát cảnh báo/lỗi biên dịch (tuỳ
      cấu hình trình biên dịch).

## Cheat Sheet

| Câu hỏi | Trả lời |
| --- | --- |
| Enum biên dịch thành gì? | Một `final class extends java.lang.Enum<T>` |
| Mỗi hằng số là gì? | Một singleton instance `public static final` |
| So sánh bằng gì? | `==` (khuyến nghị), `.equals()` cũng đúng nhưng không cần thiết |
| Method tự sinh | `values()`, `valueOf(String)` |
| Lưu xuống DB bằng gì? | `EnumType.STRING`, KHÔNG dùng `ORDINAL` |

## References

- Java Language Specification (JLS) — Chapter 8.9: Enum Classes.
- `java.lang.Enum` — Java SE API Documentation (Oracle).
- Effective Java (Joshua Bloch) — Item 34: "Use enums instead of int constants".
