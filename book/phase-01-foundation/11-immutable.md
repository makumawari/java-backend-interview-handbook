---
tags:
  - Java
  - Immutable
  - Foundation
---

# Immutable

> Phase: Phase 1 — Java Foundation
> Chapter slug: `immutable`

## Metadata

```yaml
Chapter: Immutable
Phase: Phase 1 — Java Foundation
Difficulty: ★★★
Importance: ★★★★
Interview Frequency: 60%
Prerequisites:
  - Chapter 09 — Object
  - Chapter 10 — String
Used Later:
  - equals()/hashCode() (Chapter 12-13) — object immutable là ứng viên lý tưởng làm key HashMap
  - Immutable Collection (Phase 2) — List.copyOf()/Collections.unmodifiableList() ở đây
  - Record (Phase 4) — cú pháp Java 16+ tự động hoá gần hết những gì chapter này làm bằng tay
  - Concurrency (Phase 3) — object immutable chia sẻ an toàn giữa nhiều thread không cần khoá
Estimated Reading: 20 phút
Estimated Practice: 25 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Chapter 10](10-string.md) — `String` bất biến vì ba lý do: bảo mật,
> thread-safety, String Pool.

Ba lý do đó không phải đặc quyền riêng của `String`. Bất kỳ class nào bạn tự viết cũng có thể
hưởng lợi tương tự — nếu bạn thiết kế nó đúng cách. Nhưng "đúng cách" khó hơn tưởng tượng. Xem
đoạn code sau, trông có vẻ bất biến (toàn field `final`, không có setter):

```java
final class ImmutableProduct {
    private final String name;
    private final List<String> tags;

    ImmutableProduct(String name, List<String> tags) {
        this.name = name;
        this.tags = tags;  // ⚠️ chỉ gán tham chiếu, KHÔNG copy
    }

    List<String> getTags() { return tags; }
}
```

Trông có vẻ an toàn — không có method nào sửa `tags` cả. Nhưng thử đoạn code gọi nó:

```java
List<String> tags = new ArrayList<>(List.of("giam-gia"));
ImmutableProduct p = new ImmutableProduct("Áo thun", tags);
tags.add("HACK");  // sửa list GỐC, không đụng vào field của p...
System.out.println(p.getTags());  // ...nhưng p.getTags() vẫn đổi theo!
```

`ImmutableProduct` **trông** bất biến nhưng thực chất không phải — vì field `tags` chỉ là một
**tham chiếu** dùng chung với list bên ngoài (đúng khái niệm ở
[Chapter 04](04-runtime-data-areas.md)). Chapter này giải quyết chính xác cái bẫy này.

## Interview Question (Central)

> Làm sao thiết kế một class thật sự immutable trong Java? Chỉ cần đánh dấu field `final` đã
> đủ chưa?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Liệt kê đủ 5 quy tắc thiết kế một class immutable đúng cách
- [ ] Hiểu và áp dụng được **defensive copy** — lý do `final` trên field kiểu tham chiếu
      không tự động đảm bảo bất biến
- [ ] Tự tay tái hiện được lỗ hổng "immutable giả" như ở Story, rồi tự sửa đúng
- [ ] Biết `List.copyOf()`/`Collections.unmodifiableList()` khác nhau ở điểm nào

## Prerequisites

- Chapter 09 — biết `Object` là gì, để hiểu class immutable vẫn kế thừa toàn bộ method của
  `Object`.
- Chapter 10 — đã thấy `String` là ví dụ immutable chuẩn mực, chapter này tổng quát hoá cho
  class của bạn.

## Used Later

- **equals()/hashCode()** (Chapter 12-13) — object immutable là lựa chọn an toàn nhất để làm
  key trong `HashMap`/phần tử trong `HashSet`, vì `hashCode()` không bao giờ đổi sau khi tạo
  (đúng lý do `String` cache được `hashCode()` — Chapter 10).
- **Immutable Collection** (Phase 2) — `List.copyOf()` dùng ở chapter này chỉ là một trường
  hợp cụ thể của một nhóm API rộng hơn sẽ học ở Phase 2.
- **Record** (Phase 4) — cú pháp `record` (Java 16+) tự động sinh constructor, field `final`,
  `equals()`/`hashCode()`/`toString()` — gần như tự động hoá toàn bộ quy tắc chapter này dạy
  bằng tay.
- **Concurrency** (Phase 3) — object immutable chia sẻ được giữa nhiều thread mà không cần
  `synchronized`, đúng lý do đã nêu ở Chapter 10 cho `String`.

## Problem

Field `final` chỉ đảm bảo **tham chiếu** không đổi (không gán lại field đó sang trỏ object
khác) — nó **không** đảm bảo **nội dung** object mà tham chiếu đó trỏ tới không đổi. Với field
kiểu nguyên thuỷ (`int`, `boolean`...), `final` là đủ vì bản thân giá trị nằm ngay trong field.
Nhưng với field kiểu tham chiếu (object, mảng, `List`...), `final` chỉ khoá tham chiếu — nội
dung object bên trong vẫn có thể bị sửa **từ bên ngoài** class, như ở Story.

## Concept

Một class thật sự immutable phải thoả đủ 5 quy tắc, không chỉ một:

1. Class là `final` (hoặc mọi constructor đều `private`) — ngăn lớp con phá vỡ tính bất biến
   bằng cách override method và thêm hành vi mutable.
2. Mọi field là `private final`.
3. Không có method nào cho phép sửa trạng thái sau khi khởi tạo (không có setter).
4. Nếu field kiểu tham chiếu tới object mutable (mảng, `List`, `Date`...): **defensive copy**
   khi nhận vào constructor VÀ khi trả ra qua getter.
5. Không để `this` "thoát ra ngoài" trong lúc constructor đang chạy (tránh code khác nắm được
   tham chiếu tới object khi nó chưa khởi tạo xong).

## Why?

Nếu bỏ qua quy tắc 4 (defensive copy) như ở Story: class trông bất biến trên giấy (`final`
khắp nơi) nhưng thực chất không phải — bất kỳ ai giữ tham chiếu tới object gốc được truyền vào
constructor (hoặc tới object được trả ra từ getter) đều có thể sửa "nội dung bên trong" của
một object tưởng như bất biến. Điều này phá vỡ toàn bộ lợi ích đã nêu ở Chapter 10: không còn
an toàn để chia sẻ giữa nhiều thread, không còn an toàn để dùng làm key `HashMap` (nội dung
đổi → hash đổi → mất trong map, giống hệt cảnh báo ở Chapter 10 nếu `String` mutable).

## How?

Sửa đúng `ImmutableProduct` ở Story bằng defensive copy ở cả hai đầu — constructor **và**
getter:

```java
final class ImmutableProduct {
    private final String name;
    private final int price;
    private final List<String> tags;

    ImmutableProduct(String name, int price, List<String> tags) {
        this.name = name;
        this.price = price;
        this.tags = List.copyOf(tags); // copy VÀ bất biến luôn — chặn cả 2 hướng
    }

    String getName() { return name; }
    int getPrice() { return price; }
    List<String> getTags() { return tags; } // an toàn: đã bất biến từ constructor
}
```

`List.copyOf()` (Java 10+) làm cùng lúc hai việc: tạo một bản sao độc lập (chặn sửa từ bên
ngoài sau khi truyền vào constructor) **và** bản sao đó tự nó bất biến (chặn sửa qua chính
getter trả về) — không cần copy thêm lần nữa ở getter.

## Visualization

```
Bên ngoài                    ImmutableProduct
┌───────────────┐
│ List tags      │            constructor:
│ [giam-gia]     │──copy──▶   private final List tags = List.copyOf(tham số)
└───────────────┘                    │
      │                              │  KHÔNG còn tham chiếu chung
      │ sửa list gốc                 ▼
      ▼                    ┌─────────────────────┐
  [giam-gia, HACK]         │ tags: [giam-gia]      │ ← không đổi theo
                           │ (bản sao độc lập,      │
                           │  bản thân cũng bất biến)│
                           └─────────────────────┘
```

## Example

Chạy thật, xác nhận cả hai hướng tấn công (sửa list gốc sau khi tạo, sửa qua getter) đều bị
chặn:

```java
public final class ImmutableDemo {
    public static void main(String[] args) {
        List<String> tags = new ArrayList<>();
        tags.add("giam-gia");
        ImmutableProduct p = new ImmutableProduct("Ao thun", 199000, tags);

        tags.add("HACK-sau-khi-tao"); // sửa list gốc SAU khi đã truyền vào constructor
        System.out.println("Tags cua product: " + p.getTags());

        try {
            p.getTags().add("HACK-qua-getter");
        } catch (UnsupportedOperationException e) {
            System.out.println("Khong sua duoc qua getter: " + e.getClass().getSimpleName());
        }
    }
}

final class ImmutableProduct {
    private final String name;
    private final int price;
    private final List<String> tags;

    ImmutableProduct(String name, int price, List<String> tags) {
        this.name = name;
        this.price = price;
        this.tags = List.copyOf(tags);
    }

    String getName() { return name; }
    int getPrice() { return price; }
    List<String> getTags() { return tags; }
}
```

Kết quả thật (JDK 17):

```
Tags cua product: [giam-gia]
Khong sua duoc qua getter: UnsupportedOperationException
```

Dòng 1 xác nhận: dù list gốc bên ngoài đã bị thêm `"HACK-sau-khi-tao"`, `p.getTags()` vẫn chỉ
có `[giam-gia]` — defensive copy ở constructor hoạt động. Dòng 2 xác nhận: cố sửa trực tiếp
qua giá trị trả về của getter bị chặn bởi `UnsupportedOperationException` — vì `List.copyOf()`
trả về một list tự thân bất biến.

## Deep Dive

**`final` trên class để làm gì, nếu constructor đã `private`/package-private thì đâu ai kế
thừa được?** Câu hỏi hợp lý — nhưng `final` trên class vẫn cần thiết trong trường hợp
constructor **không** bị giới hạn truy cập (như `ImmutableProduct` ở ví dụ, constructor mặc
định package-private nhưng field vẫn `private`). Nếu bỏ `final`, một lớp con
`EvilProduct extends ImmutableProduct` hoàn toàn hợp lệ có thể override `getTags()` để trả về
list mutable riêng của nó, hoặc thêm field mutable mới — phá vỡ "hợp đồng bất biến" mà code
gọi `ImmutableProduct` (qua kiểu tham chiếu `ImmutableProduct`) tin tưởng. `final` trên class
loại bỏ hoàn toàn khả năng này ngay từ compile-time.

## Engineering Insight

**Vì sao "đừng để `this` thoát ra ngoài lúc constructor đang chạy" (quy tắc 5) lại là một quy
tắc thật sự, không phải lý thuyết suông?** Vì Java **không đảm bảo** constructor chạy xong
hoàn toàn trước khi object khác nhìn thấy nó, nếu bạn tự làm rò rỉ `this`. Ví dụ điển hình:

```java
class Leaky {
    Leaky(EventBus bus) {
        bus.register(this); // ⚠️ "this" thoát ra ngoài TRƯỚC KHI constructor chạy xong
        this.value = computeValue(); // field này CHƯA được gán lúc bus.register() chạy
    }
}
```

Nếu `bus.register(this)` khiến một thread khác (hoặc chính callback đồng bộ) đọc `value` ngay
lập tức, nó sẽ đọc được giá trị **mặc định** (`0`/`null`) thay vì giá trị thật — vì trạng thái
object lúc đó chưa hoàn tất. Đây là lỗi tinh vi, thường chỉ lộ ra trong môi trường đa luồng
(Phase 3) — chính xác loại lỗi mà thiết kế immutable đúng cách (không rò rỉ `this`) triệt tiêu
từ gốc.

## Historical Note

```
Trước Java 9
    ↓
Muốn một List bất biến, phải bọc thủ công:
Collections.unmodifiableList(new ArrayList<>(original))
— PHẢI tự nhớ new ArrayList<>() để copy, unmodifiableList() chỉ BỌC
(view) chứ không tự copy, dễ quên bước copy và mắc lại đúng lỗi ở Story
    ↓
Java 9 (2017) — JEP 269: Convenience Factory Methods for Collections
    ↓
List.of(...), List.copyOf(...) ra đời — vừa copy vừa bất biến trong MỘT lời
gọi, khó viết sai hơn hẳn cách cũ
    ↓
Java 16 (2021) — Record ra đời (Phase 4), tự động hoá gần hết quy tắc ở
chapter này cho trường hợp phổ biến nhất: một "data holder" thuần bất biến
```

## Myth vs Reality

- **Myth:** "Đánh dấu mọi field là `final` là đủ để class trở thành immutable."
  **Reality:** Xem Story — `final` trên field kiểu tham chiếu chỉ khoá **tham chiếu**, không
  khoá **nội dung** object mà nó trỏ tới. Cần defensive copy cho field kiểu tham chiếu tới
  object mutable.

- **Myth:** "`Collections.unmodifiableList(list)` biến `list` gốc thành bất biến."
  **Reality:** Nó chỉ tạo ra một **view** (lớp bọc) không cho sửa **qua chính view đó** — list
  gốc `list` vẫn hoàn toàn mutable, sửa trực tiếp qua biến `list` ban đầu vẫn ảnh hưởng tới
  view. Đây là lý do `List.copyOf()` (tạo bản sao độc lập) an toàn hơn hẳn cho mục đích xây
  immutable object.

## Common Mistakes

- **Chỉ copy ở constructor, quên copy/bọc bất biến ở getter** (hoặc ngược lại) — chỉ chặn
  được một trong hai hướng tấn công đã minh hoạ ở Example.
- **Dùng `Collections.unmodifiableList()` trên chính field nội bộ nhưng field đó không phải
  bản copy** — vẫn bị lộ qua tham chiếu gốc như Myth vs Reality đã chỉ ra.
- **Immutable "nửa vời":** để hầu hết field `final` nhưng có một field `mutable` "tạm thời"
  (ví dụ cache tính toán) mà không cẩn thận — dễ gây lỗi khó tái hiện trong môi trường đa
  luồng vì phá vỡ đúng lợi ích "chia sẻ an toàn không cần khoá" đã nêu ở Used Later.

## Best Practices

- Với "data holder" thuần bất biến (không có logic phức tạp), ưu tiên dùng `record` (Phase 4,
  Java 16+) thay vì viết tay đủ 5 quy tắc — trình biên dịch tự đảm bảo đúng.
- Khi chưa dùng được `record` (project còn ở Java 8-15, hoặc cần logic validate phức tạp
  trong constructor), luôn tự hỏi với **mỗi** field kiểu tham chiếu: "nếu code bên ngoài giữ
  tham chiếu gốc, nó có sửa được nội dung qua field này không?" — cả chiều vào (constructor)
  lẫn chiều ra (getter).
- Validate dữ liệu ngay trong constructor (ví dụ `price` không được âm) — vì object immutable
  không có cơ hội "sửa lại sau" nếu tạo sai, validate sớm là tuyến phòng thủ duy nhất.

## Production Notes

Chapter này thuần về thiết kế — không gắn trực tiếp với một sự cố runtime cụ thể như JVM
internals (Chapter 03-07). Ảnh hưởng production thực tế của "immutable giả" (như ở Story)
thường biểu hiện gián tiếp và khó chẩn đoán: một object tưởng bất biến được dùng làm key
`HashMap`/cache, sau đó bị sửa "từ xa" qua một tham chiếu khác còn sót lại đâu đó trong hệ
thống — cache tra cứu sai hoặc "mất" entry mà không có exception nào báo hiệu, triệu chứng gần
giống hệt Myth vs Reality ở [Chapter 10](10-string.md) đã cảnh báo nếu `String` mutable — chỉ
khác đối tượng gây lỗi là class tự viết thay vì `String`.

## Debug Checklist

- [ ] Object "tưởng bất biến" nhưng giá trị đọc ra thay đổi ngoài ý muốn? → kiểm tra từng
      field kiểu tham chiếu, xác nhận có defensive copy ở cả constructor lẫn getter chưa.
- [ ] `UnsupportedOperationException` khi cố `.add()`/`.set()` vào một List trả về từ getter?
      → không phải bug — đây chính là bằng chứng defensive copy đang hoạt động đúng (xem
      Example), sửa ở phía gọi (đừng cố sửa list trả về, cần thì copy riêng ra trước).

## Source Code Walkthrough

`List.copyOf()` (trong `java.util.List`, hiện thực thật bên trong `ImmutableCollections`
package nội bộ của JDK) tối ưu một trường hợp đặc biệt đáng chú ý: nếu list truyền vào **đã
là** một list bất biến do chính `List.of()`/`List.copyOf()` tạo ra trước đó, nó **không copy
lại** — trả về thẳng chính list đó (vì đã bất biến sẵn, copy thêm là lãng phí). Đây là một tối
ưu nhỏ nhưng thể hiện đúng tinh thần "bất biến" được tận dụng ở tầng thư viện chuẩn, không chỉ
là quy tắc thiết kế lý thuyết.

## Summary

Một class immutable thật sự cần đủ 5 quy tắc: class `final`, field `private final`, không
setter, **defensive copy** cho field kiểu tham chiếu (ở cả constructor và getter), và không để
`this` rò rỉ lúc constructor chưa chạy xong. Chỉ đánh dấu `final` trên field là chưa đủ — đây
là cái bẫy phổ biến nhất (immutable giả). `List.copyOf()`/`List.of()` (Java 9+) giúp làm đúng
dễ dàng hơn hẳn so với `Collections.unmodifiableList()` truyền thống, vì vừa copy vừa tự bất
biến trong một bước.

## Interview Questions

**Junior**

- How do you create an immutable class in Java?
- `final` trên một field kiểu `List` có đủ để field đó bất biến không?

**Mid**

- Defensive copy là gì? Vì sao cần áp dụng ở cả constructor lẫn getter, không chỉ một trong
  hai?
- `Collections.unmodifiableList()` khác `List.copyOf()` ở điểm nào?

**Senior**

- Giải thích vì sao để `this` "thoát ra ngoài" trong lúc constructor chưa chạy xong có thể
  gây lỗi tinh vi trong môi trường đa luồng — liên hệ tới các quy tắc bất biến ở chapter này.
- So sánh việc viết tay một immutable class đủ 5 quy tắc với dùng `record` (Phase 4) — trong
  trường hợp nào bạn vẫn phải viết tay thay vì dùng `record`?

## Exercises

- [ ] Chạy lại đúng ví dụ `ImmutableDemo` ở trên, xác nhận kết quả giống hệt.
- [ ] Cố tình bỏ `List.copyOf()` (dùng thẳng `this.tags = tags`), chạy lại, xác nhận tái hiện
      đúng lỗ hổng ở Story.
- [ ] Viết một class `ImmutableOrder` có field kiểu `Product` (không phải kiểu nguyên thuỷ
      hay `List`) — áp dụng defensive copy đúng cách cho trường hợp field là một object thay
      vì collection.

## Cheat Sheet

| Quy tắc | Vì sao |
| --- | --- |
| Class `final` | Chặn lớp con phá vỡ tính bất biến |
| Field `private final` | Chặn sửa trực tiếp, chặn gán lại tham chiếu |
| Không setter | Không có method nào đổi trạng thái |
| Defensive copy ở constructor | Chặn sửa từ bên ngoài qua tham chiếu gốc |
| Defensive copy/bất biến ở getter | Chặn sửa qua chính giá trị trả về |
| Không để `this` rò rỉ trong constructor | Tránh object bị dùng khi chưa khởi tạo xong |

| API | Hành vi |
| --- | --- |
| `List.of(...)` | Tạo list bất biến mới |
| `List.copyOf(list)` | Copy + bất biến trong một bước (Java 10+) |
| `Collections.unmodifiableList(list)` | Chỉ bọc view, KHÔNG copy — `list` gốc vẫn mutable |

## References

- Effective Java (Joshua Bloch) — Item 17: "Minimize mutability".
- Java SE API Documentation — `java.util.List.copyOf()`, `Collections.unmodifiableList()`.
- JEP 269: Convenience Factory Methods for Collections.
