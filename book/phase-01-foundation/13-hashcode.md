---
tags:
  - Java
  - hashCode
  - Object
  - Foundation
---

# hashCode()

> Phase: Phase 1 — Java Foundation
> Chapter slug: `hashcode`

## Metadata

```yaml
Chapter: hashCode()
Phase: Phase 1 — Java Foundation
Difficulty: ★★★
Importance: ★★★★★
Interview Frequency: 90%
Prerequisites:
  - Chapter 12 — equals()
Used Later:
  - HashMap, HashSet, ConcurrentHashMap (Phase 2) — hashCode() quyết định object rơi vào bucket nào
  - JPA Entity (Phase 6) — hashCode() cho Entity thừa hưởng đúng những cạm bẫy ở đây
Estimated Reading: 20 phút
Estimated Practice: 20 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Chapter 12](12-equals.md) — bạn đã học cách override `equals()` đúng chuẩn theo
> 5 quy tắc của hợp đồng.

Giờ thử một tình huống tưởng như vô hại: bạn override `equals()` đúng chuẩn hoàn toàn (theo
đúng mọi quy tắc ở Chapter 12), nhưng **quên** không override `hashCode()` đi kèm. Code biên
dịch sạch sẽ — không một cảnh báo nào từ trình biên dịch. Rồi bạn dùng class đó với `HashSet`:

```java
Set<Product> cart = new HashSet<>();
cart.add(new Product("Áo thun", 199000));
cart.add(new Product("Áo thun", 199000)); // "giống hệt" theo equals() vừa viết

System.out.println(cart.size()); // bạn đoán: 1 (vì equals() coi hai cái là "bằng nhau")
                                   // thực tế: 2
```

`equals()` của bạn khẳng định hai object này "bằng nhau". Nhưng `HashSet` vẫn lưu cả hai như
thể chúng khác nhau — nó thậm chí **không bao giờ gọi tới `equals()`** để so sánh hai object
này, vì chúng đã bị coi là khác nhau ngay từ một bước trước đó. Đây chính là hậu quả của việc
vi phạm hợp đồng `equals()`/`hashCode()` — chủ đề trung tâm của chapter này.

## Interview Question (Central)

> Tại sao phải luôn override cả `equals()` và `hashCode()` cùng nhau? Điều gì xảy ra nếu chỉ
> override một trong hai?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Thuộc quy tắc cốt lõi: **hai object bằng nhau theo `equals()` bắt buộc phải có cùng
      `hashCode()`**
- [ ] Tự tay tái hiện được lỗi ở Story bằng `HashSet` thật, rồi tự sửa
- [ ] Hiểu vì sao chiều ngược lại (cùng `hashCode()` chưa chắc `equals()`) là **được phép**
- [ ] Dùng đúng `Objects.hash()` để viết `hashCode()` nhất quán với `equals()`

## Prerequisites

- Chapter 12 — đã biết 5 quy tắc của hợp đồng `equals()`, đã biết `getClass()` vs
  `instanceof`.

## Used Later

- **HashMap, HashSet, ConcurrentHashMap** (Phase 2) — chapter này chỉ dùng `HashSet` như một
  "hộp đen" để minh hoạ hậu quả; Phase 2 sẽ mở hộp đen đó ra, giải thích chính xác
  `hashCode()` quyết định object rơi vào bucket nào bên trong `HashMap`.
- **JPA Entity** (Phase 6) — Entity có ID sinh tự động bởi database (là `null` trước khi lưu)
  khiến việc viết `hashCode()` nhất quán trở thành một bài toán riêng, phức tạp hơn ví dụ ở
  đây.

## Problem

`equals()` (Chapter 12) trả lời câu hỏi "hai object có bằng nhau không" bằng cách so sánh đầy
đủ nội dung — nhưng đây là phép so sánh **tốn kém** nếu phải làm với **mọi cặp** phần tử trong
một tập hợp lớn (độ phức tạp O(n) nếu duyệt tuần tự so sánh từng phần tử). `HashSet`/`HashMap`
cần một cách **rất nhanh** để thu hẹp phạm vi "có khả năng bằng nhau" xuống một nhóm nhỏ, trước
khi mới cần gọi tới `equals()` để so sánh chính xác trong nhóm nhỏ đó.

## Concept

`hashCode()` trả về một số nguyên (`int`) đại diện cho object — dùng làm "địa chỉ gợi ý"
(bucket index, sẽ học chi tiết ở Phase 2) để các cấu trúc dữ liệu dựa trên hash tra cứu gần
như tức thì thay vì duyệt tuần tự. Hợp đồng bắt buộc, chỉ có **một chiều**:

> Nếu `a.equals(b)` là `true`, thì bắt buộc `a.hashCode() == b.hashCode()`.

Chiều ngược lại **không** bắt buộc: hai object có cùng `hashCode()` **không nhất thiết**
`equals()` là `true` (gọi là **hash collision** — hai object khác nhau "tình cờ" trùng mã băm,
điều này được phép và không thể tránh hoàn toàn vì `int` chỉ có hữu hạn giá trị trong khi số
object có thể là vô hạn).

## Why?

`HashSet.add()`/`contains()` hoạt động theo hai bước: (1) tính `hashCode()` của object cần
tìm, nhảy thẳng tới đúng "khu vực" (bucket) tương ứng — cực nhanh, O(1) trung bình; (2) **chỉ
trong khu vực đó**, mới dùng `equals()` để so sánh chính xác từng phần tử. Nếu hai object
"bằng nhau" theo `equals()` nhưng có `hashCode()` khác nhau, chúng sẽ bị tính toán rơi vào
**hai khu vực khác nhau** — bước (2) sẽ **không bao giờ được thực hiện** giữa chúng, vì
`HashSet` thậm chí không nghĩ tới việc so sánh hai object ở hai khu vực khác nhau. Đây chính
xác là lý do ở Story: `equals()` đúng nhưng không được dùng tới, vì đã "lạc nhau" từ bước (1).

## How?

Sửa `Product` ở Story bằng cách thêm `hashCode()` **nhất quán** với `equals()` — quy tắc đơn
giản: `hashCode()` phải được tính từ **đúng những field mà `equals()` đã dùng để so sánh**,
không hơn không kém:

```java
class Product {
    String name;
    int price;
    Product(String name, int price) { this.name = name; this.price = price; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Product p = (Product) o;
        return price == p.price && Objects.equals(name, p.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name, price); // ĐÚNG những field equals() đã dùng
    }
}
```

`Objects.hash(name, price)` tự động kết hợp `hashCode()` của từng field thành một `hashCode()`
tổng hợp cho cả object — tương đương viết tay công thức nhân 31 (đã gặp ở
[Chapter 10](10-string.md) khi phân tích `String.hashCode()`).

## Visualization

```
Không override hashCode() (dùng mặc định của Object — dựa trên định danh nội bộ):

  p1 = new Product("Áo thun", 199000)  →  hashCode = 7ad041f3  →  bucket #243
  p2 = new Product("Áo thun", 199000)  →  hashCode = 5b6f7a91  →  bucket #109
                                             (khác nhau dù equals() nói "bằng nhau"!)

  HashSet chỉ so sánh equals() giữa các phần tử CÙNG bucket
  → p1 và p2 ở hai bucket khác nhau → equals() KHÔNG BAO GIỜ được gọi
  → cả hai đều được thêm vào, size() = 2

Sau khi override hashCode() nhất quán:

  p1 = new Product("Áo thun", 199000)  →  hashCode = 1837404  →  bucket #77
  p2 = new Product("Áo thun", 199000)  →  hashCode = 1837404  →  bucket #77
                                             (GIỐNG NHAU, đúng yêu cầu)

  Cùng bucket #77 → HashSet gọi equals(p1, p2) → true → coi là trùng
  → chỉ thêm 1 lần, size() = 1
```

## Example

Tái hiện đúng cả hai tình huống — thiếu override và có override đúng — chạy thật để so sánh:

```java
import java.util.HashSet;
import java.util.Objects;
import java.util.Set;

public class EqualsHashDemo {
    public static void main(String[] args) {
        Set<BadProduct> badSet = new HashSet<>();
        badSet.add(new BadProduct("Ao thun", 199000));
        badSet.add(new BadProduct("Ao thun", 199000));
        System.out.println("BadProduct set size (khong override equals/hashCode): "
            + badSet.size());

        Set<GoodProduct> goodSet = new HashSet<>();
        goodSet.add(new GoodProduct("Ao thun", 199000));
        goodSet.add(new GoodProduct("Ao thun", 199000));
        System.out.println("GoodProduct set size (co override dung): " + goodSet.size());
    }
}

class BadProduct {
    String name; int price;
    BadProduct(String name, int price) { this.name = name; this.price = price; }
    // KHÔNG override equals()/hashCode() — dùng nguyên mặc định của Object
}

class GoodProduct {
    String name; int price;
    GoodProduct(String name, int price) { this.name = name; this.price = price; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof GoodProduct)) return false;
        GoodProduct p = (GoodProduct) o;
        return price == p.price && Objects.equals(name, p.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name, price);
    }
}
```

Kết quả thật (JDK 17):

```
BadProduct set size (khong override equals/hashCode): 2
GoodProduct set size (co override dung): 1
```

Bằng chứng thực nghiệm trực tiếp: cùng một thao tác (`add()` hai object "giống hệt nhau" về
nội dung), kết quả khác hẳn nhau chỉ vì có/không override đúng cặp `equals()`/`hashCode()`.

## Deep Dive

**"Cùng `hashCode()` không nhất thiết `equals()`" — vì sao Java chấp nhận điều này thay vì
đòi hỏi một hàm băm "hoàn hảo"?** Vì điều đó **không thể thực hiện được** về mặt toán học:
`hashCode()` trả về `int` (32-bit, khoảng 4.3 tỷ giá trị khả dĩ), nhưng số lượng object khả dĩ
(ví dụ số chuỗi `String` khác nhau có thể tạo ra) là **vô hạn**. Theo nguyên lý Dirichlet
(pigeonhole principle), nhồi vô hạn giá trị vào một tập hữu hạn kết quả chắc chắn sẽ có nhiều
giá trị "chung một hộp" (hash collision) — không thể tránh khỏi bằng bất kỳ thuật toán băm
nào, dù thiết kế tốt tới đâu. Vì vậy hợp đồng chỉ yêu cầu chiều bắt buộc duy nhất
(equals → cùng hashCode), còn chấp nhận collision xảy ra ở chiều ngược lại như một thực tế
không tránh khỏi — và `equals()` chính là công cụ để phân xử chính xác khi có collision (bước
(2) ở phần Why?).

## Engineering Insight

**Vì sao lỗi ở Story lại đặc biệt nguy hiểm trong thực tế, so với nhiều lỗi Java khác?** Vì nó
**không gây lỗi biên dịch, không ném exception, không crash** — chương trình chạy "bình
thường", chỉ âm thầm cho kết quả sai (như `size()` = 2 thay vì 1 ở Example). Loại bug này
thuộc nhóm nguy hiểm nhất trong kỹ thuật phần mềm: **sai lệch dữ liệu âm thầm** (silent data
corruption) — không có tín hiệu báo động nào để bạn biết cần điều tra, cho tới khi hậu quả
nghiệp vụ đã xảy ra (ví dụ: hệ thống tính tổng giỏ hàng sai vì tưởng đã loại trùng sản phẩm
nhưng thực tế chưa). So với `NullPointerException` (Chapter 15 sắp tới) — vốn ít nhất còn dừng
chương trình và chỉ thẳng dòng lỗi — vi phạm hợp đồng equals/hashCode "lịch sự" hơn nhiều
nhưng lại nguy hiểm hơn chính vì sự im lặng đó.

## Historical Note

`Objects.hash(Object...)` xuất hiện từ Java 7 (cùng đợt với `Objects.equals()`, Chapter 12),
giúp việc viết `hashCode()` đúng chuẩn dễ dàng hơn hẳn so với thời Java 5/6, khi lập trình
viên phải tự tay viết công thức nhân 31 cho từng field (đúng công thức đã phân tích ở
[Chapter 10](10-string.md) cho `String.hashCode()`) — dễ viết sai, dễ quên field, hoặc quên
đồng bộ khi thêm field mới vào class về sau.

## Myth vs Reality

- **Myth:** "Hai object có `hashCode()` giống nhau thì chắc chắn `equals()` cũng là `true`."
  **Reality:** Ngược lại hoàn toàn — đây là hash collision, hoàn toàn hợp lệ và không thể
  tránh khỏi (xem Deep Dive). Hợp đồng chỉ bắt buộc chiều: equals `true` ⟹ hashCode giống
  nhau, không có chiều ngược lại.

- **Myth:** "hashCode() càng phức tạp, càng nhiều field tham gia, càng tốt."
  **Reality:** `hashCode()` phải dùng **đúng và đủ** những field mà `equals()` dùng — không
  thừa, không thiếu. Thêm field không tham gia `equals()` vào `hashCode()` (hoặc ngược lại)
  đều vi phạm nguyên tắc nhất quán, có thể tái tạo đúng lỗi ở Story theo một cách khác.

## Common Mistakes

- **Override `equals()` mà quên `hashCode()`** — đúng lỗi trọng tâm của chapter, hầu hết IDE
  hiện đại (IntelliJ, Eclipse) đều cảnh báo khi phát hiện tình huống này, nhưng vẫn là lỗi rất
  phổ biến khi tự viết tay hoặc copy-paste code cũ.
- **`hashCode()` và `equals()` dùng field khác nhau** — ví dụ `equals()` so sánh `name` và
  `price`, nhưng `hashCode()` chỉ dựa trên `name` — vẫn vi phạm hợp đồng dù cả hai đều "có
  override", chỉ là không nhất quán với nhau.
- **Dùng field `mutable` trong `hashCode()`** rồi sửa field đó **sau khi** object đã được thêm
  vào `HashSet`/dùng làm key `HashMap` — object "biến mất" khỏi đúng bucket cũ, tra cứu lại
  không tìm thấy nữa dù object đó thực sự vẫn còn trong collection (hệ quả trực tiếp liên
  quan tới [Chapter 11 — Immutable](11-immutable.md): object dùng làm hash key nên bất biến).

## Best Practices

- Luôn override `equals()` và `hashCode()` **cùng lúc**, không bao giờ chỉ một trong hai —
  hầu hết IDE có tính năng tự sinh cả cặp cùng lúc, nên tận dụng thay vì viết tay.
- Dùng `Objects.hash(...)` với **đúng và đủ** field đã dùng trong `equals()`.
- Ưu tiên dùng field **bất biến** (Chapter 11) cho những field tham gia `hashCode()` — tránh
  hoàn toàn nguy cơ "object biến mất khỏi bucket cũ" như đã nêu ở Common Mistakes.

## Production Notes

**Vấn đề:** một `Set`/`Map` dùng để loại trùng hoặc cache tra cứu theo object nghiệp vụ hoạt
động sai — số liệu thống kê "số lượng duy nhất" luôn cao hơn thực tế, hoặc tra cứu cache luôn
miss dù rõ ràng đã lưu trước đó.

- **Triệu chứng:** không có exception, không crash — chỉ số liệu/kết quả sai một cách âm
  thầm, đúng như Engineering Insight đã cảnh báo.
- **Root cause:** class dùng làm phần tử `Set`/key `Map` thiếu `hashCode()` nhất quán với
  `equals()` — giống hệt `BadProduct` ở Example.
- **Debug:** viết một test đơn vị đơn giản: tạo hai object "bằng nhau theo nghiệp vụ", đưa vào
  `HashSet`, kiểm tra `size()` có đúng là 1 không — bug loại này rất dễ bắt được bằng một test
  case tối thiểu nếu nghĩ tới, nhưng dễ bị bỏ sót nếu không ai từng nghĩ tới viết nó.
- **Solution:** thêm `hashCode()` nhất quán như ở phần How?.
- **Prevention:** đưa "mọi class override equals() phải có test kiểm tra kèm hashCode() nhất
  quán" thành quy tắc bắt buộc trong code review; bật cảnh báo IDE/static analysis
  (SpotBugs có sẵn detector `HE_EQUALS_USE_HASHCODE`) cho đúng tình huống này.

## Debug Checklist

- [ ] `HashSet`/`HashMap` cho kết quả `size()`/tra cứu sai dù logic `equals()` trông đúng? →
      kiểm tra ngay `hashCode()` có được override và nhất quán không — đây là nghi phạm số
      một.
- [ ] `hashCode()` có override nhưng vẫn sai? → so lại field dùng trong `hashCode()` với field
      dùng trong `equals()`, đảm bảo khớp chính xác.
- [ ] Object "biến mất" khỏi `Set`/`Map` sau một thời gian dùng? → kiểm tra có field mutable
      nào tham gia `hashCode()` bị sửa sau khi object đã được thêm vào collection không.

## Source Code Walkthrough

Giống [Chapter 12](12-equals.md), hợp đồng `equals()`/`hashCode()` là một quy ước mô tả trong
Javadoc, không phải cơ chế có thể trace qua bytecode. Điều **có thể** trace được là cách
`HashSet`/`HashMap` sử dụng `hashCode()` để chọn bucket — nhưng đó là nội dung thuộc hẳn về
Phase 2 (chapter HashMap), nơi có đủ ngữ cảnh để đọc đúng `putVal()`/bucket index calculation
trong `java.util.HashMap`. Chapter này dừng lại ở mức hợp đồng, chưa đi vào cơ chế bucket cụ
thể.

## Summary

Hợp đồng bắt buộc: **`a.equals(b)` là `true` thì `a.hashCode()` phải bằng `b.hashCode()`**
— chiều ngược lại (cùng hashCode nhưng không equals — hash collision) là hợp lệ và không thể
tránh khỏi. Vi phạm hợp đồng (thường gặp nhất: override `equals()` mà quên `hashCode()`)
không gây lỗi biên dịch hay exception — nó âm thầm làm `HashSet`/`HashMap` cho kết quả sai,
vì `equals()` sẽ không bao giờ được gọi tới giữa hai object rơi vào bucket khác nhau. Luôn
override cả hai cùng lúc, dùng `Objects.hash()` với đúng field đã dùng trong `equals()`, và ưu
tiên field bất biến cho những field tham gia tính hash.

## Interview Questions

**Junior**

- Tại sao phải override cả equals() và hashCode()?
- Hai object có cùng `hashCode()` thì có chắc `equals()` là `true` không?

**Mid**

- Giải thích cơ chế `HashSet` dùng `hashCode()` như thế nào để quyết định có gọi `equals()`
  hay không.
- Vì sao dùng field mutable trong `hashCode()` rồi sửa nó sau khi object đã ở trong
  `HashSet` lại gây lỗi "object biến mất"?

**Senior**

- Giải thích bằng nguyên lý Dirichlet (pigeonhole) vì sao Java không thể (và không cần) đòi
  hỏi một hàm `hashCode()` "hoàn hảo" không bao giờ collision.
- Một hệ thống cache dùng object nghiệp vụ làm key `HashMap` báo cache-miss bất thường cao.
  Trình bày quy trình điều tra, liên hệ trực tiếp tới hợp đồng equals/hashCode và tính bất
  biến (Chapter 11).

## Exercises

- [ ] Chạy lại đúng ví dụ `EqualsHashDemo` ở trên, xác nhận kết quả giống hệt (2 và 1).
- [ ] Sửa `GoodProduct` để `hashCode()` chỉ dựa trên `name` (bỏ `price`) trong khi `equals()`
      vẫn so cả hai field — chạy lại, quan sát và giải thích hành vi (gợi ý: đây vẫn là một
      cách vi phạm hợp đồng, dù "nhẹ" hơn không override gì cả).
- [ ] Viết một class có field `mutable` tham gia `hashCode()`, thêm object vào `HashSet`, sau
      đó sửa field đó, rồi gọi `set.contains(sameObject)` — quan sát và giải thích kết quả.

## Cheat Sheet

| Quy tắc | Bắt buộc? |
| --- | --- |
| `a.equals(b)` = `true` ⟹ `a.hashCode() == b.hashCode()` | Bắt buộc |
| `a.hashCode() == b.hashCode()` ⟹ `a.equals(b)` = `true` | KHÔNG bắt buộc (hash collision hợp lệ) |
| `hashCode()` dùng đúng field như `equals()` | Nguyên tắc thực hành, không phải luật cứng nhưng bắt buộc để nhất quán |

| Việc | Công cụ |
| --- | --- |
| Viết `hashCode()` nhất quán | `Objects.hash(field1, field2, ...)` |
| Kiểm tra nhanh lỗi thiếu override | IDE cảnh báo, hoặc SpotBugs `HE_EQUALS_USE_HASHCODE` |

## References

- `java.lang.Object#hashCode` — Java SE API Documentation (Oracle), phần mô tả hợp đồng đầy
  đủ.
- Effective Java (Joshua Bloch) — Item 11: "Always override hashCode when you override
  equals".
- Java Language Specification (JLS) — Chapter 4.3.2.
