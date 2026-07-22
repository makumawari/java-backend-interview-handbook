---
tags:
  - Java
  - equals
  - Object
  - Foundation
---

# equals()

> Phase: Phase 1 — Java Foundation
> Chapter slug: `equals`

## Metadata

```yaml
Chapter: equals()
Phase: Phase 1 — Java Foundation
Difficulty: ★★★
Importance: ★★★★★
Interview Frequency: 90%
Prerequisites:
  - Chapter 08 — OOP Fundamentals (Overriding)
  - Chapter 09 — Object
Used Later:
  - hashCode() (Chapter 13) — hợp đồng equals/hashCode phải luôn đi cùng nhau
  - HashMap, HashSet (Phase 2) — mọi lỗi vi phạm hợp đồng equals() đều lộ ra ở đây
  - JPA Entity (Phase 6) — equals() cho Entity là một chủ đề gây tranh cãi kinh điển
Estimated Reading: 20 phút
Estimated Practice: 20 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Chapter 09](09-object.md) — `equals()` mặc định của `Object` chỉ so sánh tham
> chiếu (`==`), nên `new Product(...).equals(new Product(...))` luôn `false` dù nội dung
> giống hệt nhau.

Bạn override `equals()` để so sánh theo nội dung — hợp lý, đúng nhu cầu nghiệp vụ. Nhưng
override "tự do" mà không tuân theo quy tắc nào có thể tạo ra một quả bom hẹn giờ. Xét đoạn
code sau:

```java
class Product {
    String name;
    Product(String name) { this.name = name; }

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Product)) return false;
        return name.equalsIgnoreCase(((Product) o).name);
    }
}

class ImportedProduct extends Product {
    String supplier;
    ImportedProduct(String name, String supplier) {
        super(name);
        this.supplier = supplier;
    }

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof ImportedProduct)) return false;
        ImportedProduct other = (ImportedProduct) o;
        return name.equalsIgnoreCase(other.name) && supplier.equals(other.supplier);
    }
}
```

Trông hợp lý — mỗi class tự so sánh theo field của mình. Nhưng thử:

```java
Product a = new Product("Áo thun");
ImportedProduct b = new ImportedProduct("Áo thun", "Việt Nam");

a.equals(b);  // true  — a chỉ so sánh name, không biết/không quan tâm supplier
b.equals(a);  // false — b kiểm tra "o instanceof ImportedProduct", a không phải, nên false
```

**`a.equals(b)` và `b.equals(a)` cho hai kết quả khác nhau** cho cùng một cặp object. Đây
không phải lỗi cú pháp — code biên dịch hoàn toàn bình thường. Nhưng nó vi phạm một quy tắc
nền tảng mà mọi thư viện Java (kể cả `HashMap`, `List.contains()`, `Collections.sort()`...)
đều **ngầm giả định** là đúng. Hậu quả: hành vi chương trình phụ thuộc vào **thứ tự** bạn gọi
`equals()` — một loại bug cực kỳ khó tái hiện và debug.

## Interview Question (Central)

> `equals()` phải tuân theo những quy tắc nào? Vì sao vi phạm các quy tắc đó lại nguy hiểm,
> dù code vẫn biên dịch và chạy được bình thường?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Thuộc và giải thích được 5 quy tắc trong hợp đồng `equals()`: reflexive, symmetric,
      transitive, consistent, non-null
- [ ] Tự tay tái hiện được vi phạm **symmetric** như ở Story, hiểu vì sao nó xảy ra
- [ ] Biết cuộc tranh luận `getClass()` vs `instanceof` trong `equals()` — và khi nào dùng
      cái nào
- [ ] Viết được một `equals()` đúng chuẩn, dùng `Objects.equals()` để code gọn và an toàn
      `null`

## Prerequisites

- Chapter 08 — hiểu overriding, `instanceof`, ép kiểu (cast).
- Chapter 09 — biết `equals()` mặc định của `Object` là gì.

## Used Later

- **hashCode()** (Chapter 13) — hợp đồng `equals()`/`hashCode()` là MỘT hợp đồng, không thể
  học tách rời hoàn toàn; chapter này tập trung riêng `equals()` trước để không dồn quá nhiều
  khái niệm cùng lúc.
- **HashMap, HashSet** (Phase 2) — mọi vi phạm hợp đồng `equals()` (đặc biệt symmetric,
  transitive) đều biểu hiện thành bug khó hiểu khi dùng object làm phần tử/key của các cấu
  trúc dữ liệu này.
- **JPA Entity** (Phase 6) — viết `equals()` đúng cho một Entity (có ID sinh tự động, có thể
  `null` trước khi lưu DB) là một bài toán riêng, phức tạp hơn ví dụ ở chapter này — nền tảng
  bắt buộc phải nắm trước khi học tới đó.

## Problem

`Object.equals()` mặc định chỉ so sánh tham chiếu (Chapter 09) — không đủ khi bạn cần biết
"hai object có cùng nội dung nghiệp vụ không" (ví dụ: hai `Product` có cùng `name` có nên coi
là "giống nhau" không, dù là hai object khác nhau trong bộ nhớ). Nhưng override tự do, không
theo quy tắc, dẫn tới hành vi không nhất quán như ở Story — nguy hiểm hơn cả việc không
override, vì nó "trông như hoạt động đúng" trong phần lớn trường hợp, chỉ lộ ra sai ở tình
huống cụ thể (kế thừa, thứ tự gọi khác nhau).

## Concept

Java Language Specification quy định `equals()` phải tuân theo 5 quy tắc, gọi chung là
**hợp đồng equals() (equals contract)**:

1. **Reflexive** — `x.equals(x)` luôn phải là `true`.
2. **Symmetric** — nếu `x.equals(y)` là `true` thì `y.equals(x)` cũng phải `true` (và ngược
   lại). Đây là quy tắc bị vi phạm ở Story.
3. **Transitive** — nếu `x.equals(y)` và `y.equals(z)` đều `true`, thì `x.equals(z)` cũng
   phải `true`.
4. **Consistent** — gọi `x.equals(y)` nhiều lần (không có gì thay đổi ở giữa) phải luôn trả
   về cùng một kết quả.
5. **Non-null** — `x.equals(null)` luôn phải là `false` (không được ném exception).

## Why?

Toàn bộ thư viện chuẩn Java — `HashMap`, `HashSet`, `List.contains()`, `Collections.sort()`,
`ArrayList.remove(Object)` — đều **ngầm giả định** `equals()` của mọi object tuân theo 5 quy
tắc trên. Chúng không tự kiểm tra lại; chúng tin tưởng tuyệt đối vào hợp đồng. Nếu vi phạm
(như ở Story), hành vi các cấu trúc dữ liệu này trở nên **không xác định** theo nghĩa đen: kết
quả `list.contains(x)` có thể khác nhau tuỳ vào **thứ tự phần tử được thêm vào list trước đó**
— một loại bug gần như không thể tái hiện ổn định, cực khó debug trong hệ thống thực tế.

## How?

Sửa đúng ví dụ ở Story — điểm mấu chốt: **không nên** cho phép so sánh "chéo" giữa lớp cha và
lớp con nếu chúng có field khác nhau về mặt nghiệp vụ. Có hai trường phái xử lý phổ biến:

**Cách 1 — Dùng `getClass()` thay vì `instanceof`** (chặt chẽ, đảm bảo symmetric tuyệt đối):

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (o == null || getClass() != o.getClass()) return false;
    Product p = (Product) o;
    return Objects.equals(name, p.name);
}
```

`getClass() != o.getClass()` buộc hai object phải **cùng chính xác một class** (không chấp
nhận lớp con) — `Product.equals(ImportedProduct)` và `ImportedProduct.equals(Product)` đều
`false` như nhau, symmetric được đảm bảo, đổi lại: không so sánh "bằng nhau" được giữa lớp cha
và lớp con nữa, dù có thể hợp lý về nghiệp vụ trong một số trường hợp.

**Cách 2 — Dùng `instanceof`** (linh hoạt hơn, nhưng rủi ro vi phạm transitive/symmetric nếu
lớp con thêm field so sánh riêng — đúng lỗi ở Story). *Effective Java* khuyến nghị: nếu dùng
`instanceof`, hãy đảm bảo lớp con **không thêm field mới có ảnh hưởng tới equals()**, hoặc tốt
hơn — ưu tiên **composition thay vì inheritance** (đã nhắc ở [Chapter 08](08-oop-fundamentals.md))
cho các trường hợp cần mở rộng field mà vẫn giữ so sánh nhất quán.

## Visualization

```
Vi phạm Symmetric (Story):
   a (Product)  .equals(  b (ImportedProduct)  )  →  true
   b (ImportedProduct)  .equals(  a (Product)  )  →  false
        ▲                                    ▲
        └──────────── KHÁC NHAU ─────────────┘
                 (vi phạm quy tắc 2)

Sửa bằng getClass():
   a.getClass() = Product.class
   b.getClass() = ImportedProduct.class
   a.getClass() != b.getClass()  →  cả 2 chiều đều false, NHẤT QUÁN
```

## Example

Tái hiện đúng lỗi ở Story, rồi sửa bằng `getClass()`, chạy thật để xác nhận:

```java
import java.util.Objects;

public class EqualsContractDemo {
    public static void main(String[] args) {
        Product a = new Product("Áo thun");
        ImportedProduct b = new ImportedProduct("Áo thun", "Việt Nam");

        System.out.println("a.equals(b): " + a.equals(b));
        System.out.println("b.equals(a): " + b.equals(a));
    }
}

class Product {
    String name;
    Product(String name) { this.name = name; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Product p = (Product) o;
        return Objects.equals(name, p.name);
    }
}

class ImportedProduct extends Product {
    String supplier;
    ImportedProduct(String name, String supplier) {
        super(name);
        this.supplier = supplier;
    }
}
```

Kết quả thật sau khi sửa bằng `getClass()`:

```
a.equals(b): false
b.equals(a): false
```

Cả hai chiều giờ nhất quán (`false` — vì khác class) — đúng yêu cầu symmetric, dù trước khi
sửa (dùng `instanceof` với field so sánh riêng ở lớp con như ở Story) sẽ cho ra `true`/`false`
trái ngược nhau.

## Deep Dive

**`Objects.equals(a, b)` (Java 7+) an toàn hơn `a.equals(b)` như thế nào?** Gọi trực tiếp
`a.equals(b)` sẽ ném `NullPointerException` nếu `a` là `null`. `Objects.equals(a, b)` tự kiểm
tra: nếu cả hai `null` → `true`; nếu một trong hai `null` → `false`; ngược lại mới gọi
`a.equals(b)`. Dùng `Objects.equals()` cho từng field khi viết `equals()` (như ở Example)
tránh được việc phải tự viết `if (name == null ? p.name == null : name.equals(p.name))` dài
dòng và dễ viết sai cho mỗi field kiểu tham chiếu có thể `null`.

## Engineering Insight

**Vì sao *Effective Java* khuyến nghị `getClass()` thay vì `instanceof` khi có kế thừa, dù
`instanceof` "linh hoạt hơn"?** Đây là đánh đổi giữa **tính đúng đắn tuyệt đối theo hợp đồng**
và **sự linh hoạt trong thiết kế**. `instanceof` cho phép một `Product` được coi là "bằng"
một `ImportedProduct` nếu chỉ so sánh field chung (`name`) — nghe hợp lý về mặt trực giác
("chúng đều là Product mà"). Nhưng chính "sự linh hoạt" này lại chính là nguồn gốc vi phạm
symmetric ở Story: khi lớp con override `equals()` để so sánh thêm field riêng, hai object có
thể "bằng nhau theo góc nhìn của lớp cha" nhưng "không bằng nhau theo góc nhìn của lớp con" —
một mâu thuẫn logic không thể giải quyết triệt để nếu vẫn muốn giữ `instanceof`. `getClass()`
loại bỏ triệt để mâu thuẫn này bằng cách từ chối so sánh chéo giữa các class khác nhau ngay từ
đầu — đơn giản hơn, nhưng đúng đắn hơn về mặt hình thức toán học của "quan hệ tương đương"
(equivalence relation) mà symmetric/transitive yêu cầu.

## Historical Note

Hợp đồng `equals()` với 5 quy tắc trên tồn tại nguyên vẹn từ `Object` của Java 1.0, không đổi
qua các phiên bản — đây là một trong những phần ổn định nhất của Java. Điều thay đổi là công
cụ hỗ trợ viết `equals()` đúng dễ dàng hơn: `Objects.equals()`/`Objects.hash()` (Java 7, xem
[Chapter 13](13-hashcode.md)) giúp giảm lỗi thủ công, và `record` (Java 16+, Phase 4) tự sinh
`equals()` tuân thủ đúng hợp đồng hoàn toàn tự động cho trường hợp phổ biến nhất.

## Myth vs Reality

- **Myth:** "Chỉ cần `equals()` biên dịch được và trả về đúng `true`/`false` cho vài test case
  tôi tự nghĩ ra là đủ."
  **Reality:** Story chứng minh: code biên dịch hoàn hảo, một vài test case đơn giản (so sánh
  cùng class) vẫn đúng — lỗi chỉ lộ ra khi có kế thừa và so sánh chéo, tình huống dễ bị bỏ sót
  khi tự viết test case.

- **Myth:** "`instanceof` luôn là lựa chọn 'an toàn' hơn `getClass()` vì nó tổng quát hơn."
  **Reality:** Ngược lại — như Engineering Insight đã phân tích, `instanceof` dễ dẫn tới vi
  phạm symmetric khi có kế thừa và lớp con thêm field so sánh riêng, trong khi `getClass()`
  luôn đảm bảo hợp đồng đúng bất kể cấu trúc kế thừa.

## Common Mistakes

- **Vi phạm symmetric khi có kế thừa** — đúng lỗi ở Story, lỗi phổ biến và tinh vi nhất trong
  toàn bộ chapter này.
- **Không kiểm tra `null`** trước khi ép kiểu, dẫn tới `NullPointerException` thay vì trả về
  `false` — vi phạm quy tắc non-null.
- **Không kiểm tra `this == o` trước** — không sai về logic (vẫn ra kết quả đúng) nhưng bỏ lỡ
  một tối ưu rất rẻ: so sánh tham chiếu trước tránh phải so sánh từng field khi rõ ràng là
  cùng một object.
- **Override `equals()` mà quên override `hashCode()` theo** — hậu quả đầy đủ ở
  [Chapter 13](13-hashcode.md).

## Best Practices

- Luôn theo đúng thứ tự kiểm tra chuẩn: `this == o` → `o == null` → kiểm tra kiểu
  (`getClass()` hoặc `instanceof`) → ép kiểu → so sánh từng field bằng `Objects.equals()`.
- Với class có kế thừa và lớp con có thể thêm field ảnh hưởng tới so sánh, ưu tiên
  `getClass()` để đảm bảo hợp đồng tuyệt đối, trừ khi có lý do thiết kế rõ ràng để chấp nhận
  đánh đổi của `instanceof`.
- Để IDE hoặc `record` (Phase 4, khi hợp) tự sinh `equals()` khi có thể — giảm rủi ro tự viết
  sai một trong 5 quy tắc.

## Production Notes

**Vấn đề:** một tính năng "tìm sản phẩm trùng lặp trong giỏ hàng" (dùng
`list.contains(product)` hoặc `Set<Product>`) hoạt động đúng trong hầu hết trường hợp, nhưng
thỉnh thoảng bỏ sót hoặc báo trùng sai — không tái hiện được ổn định trên máy dev.

- **Triệu chứng:** bug "chập chờn" (flaky), phụ thuộc vào dữ liệu cụ thể và **thứ tự** các
  object được thêm vào collection trước đó — dấu hiệu đặc trưng của vi phạm hợp đồng
  `equals()`.
- **Root cause:** class `Product` (hoặc tương tự) có kế thừa, và lớp con vi phạm symmetric/
  transitive giống hệt Story — nhưng lỗi chỉ lộ ra khi dữ liệu thật chứa cả object lớp cha lẫn
  lớp con trộn lẫn trong cùng một collection.
- **Debug:** viết test case tường minh so sánh **cả hai chiều** (`a.equals(b)` VÀ
  `b.equals(a)`) cho mọi cặp class có quan hệ kế thừa liên quan tới `equals()` — bug loại này
  gần như không bao giờ lộ ra nếu chỉ test một chiều.
- **Solution:** áp dụng `getClass()` thay vì `instanceof` (hoặc refactor sang composition) như
  ở phần How?.
- **Prevention:** thêm rule vào static analysis (SpotBugs có sẵn detector cho vi phạm
  equals/hashCode phổ biến) và code review checklist: mọi `equals()` override trên class có
  khả năng bị kế thừa đều phải được review kỹ quy tắc symmetric.

## Debug Checklist

- [ ] Kết quả `list.contains()`/`set.contains()` không nhất quán, phụ thuộc thứ tự dữ liệu?
      → nghi ngờ vi phạm hợp đồng `equals()`, kiểm tra symmetric bằng test hai chiều.
- [ ] `NullPointerException` khi gọi `equals()`? → thiếu kiểm tra `null` đúng chuẩn.
- [ ] Class có kế thừa và override `equals()` ở nhiều tầng? → kiểm tra kỹ có đang dùng
      `instanceof` kèm field riêng ở lớp con không — nguồn lỗi phổ biến nhất.

## Source Code Walkthrough

Không có "source OpenJDK" cụ thể để đọc ở chapter này — hợp đồng `equals()` là một **quy ước**
được mô tả bằng văn bản trong Javadoc chính thức của `Object.equals()` (không phải cơ chế thực
thi có thể trace qua bytecode như các chapter JVM trước đó). Nguồn tham khảo chính xác nhất là
đọc trực tiếp Javadoc của `java.lang.Object#equals` — phần mô tả 5 quy tắc ở đó chính là bản
gốc mà chapter này diễn giải lại bằng tiếng Việt và ví dụ cụ thể.

## Summary

`equals()` phải tuân theo 5 quy tắc: reflexive, **symmetric**, transitive, consistent,
non-null — đây là một hợp đồng mà toàn bộ thư viện chuẩn Java (HashMap, List, Collections...)
tin tưởng tuyệt đối mà không tự kiểm tra lại. Vi phạm hợp đồng (phổ biến nhất: symmetric khi
có kế thừa, như ở Story) tạo ra bug "chập chờn" phụ thuộc thứ tự, cực khó debug. Dùng
`getClass()` thay vì `instanceof` khi so sánh kiểu đảm bảo symmetric tuyệt đối trong mọi cấu
trúc kế thừa, đổi lại mất khả năng so sánh "bằng nhau" giữa lớp cha và lớp con — một đánh đổi
mà *Effective Java* khuyến nghị chấp nhận.

## Interview Questions

**Junior**

- What is the equals() and hashCode() contract in Java?
- `equals()` mặc định của `Object` so sánh dựa trên điều gì?

**Mid**

- Kể tên 5 quy tắc của hợp đồng `equals()`. Quy tắc nào dễ vi phạm nhất khi có kế thừa?
- `getClass()` và `instanceof` trong `equals()` khác nhau như thế nào? Khi nào nên dùng cái
  nào?

**Senior**

- Tự viết một ví dụ vi phạm quy tắc **transitive** (khác với ví dụ symmetric ở chapter này) —
  giải thích vì sao nó xảy ra và cách sửa.
- Một hệ thống báo bug "tìm kiếm trùng lặp chập chờn, không tái hiện ổn định". Trình bày quy
  trình điều tra bạn sẽ làm, liên hệ trực tiếp tới hợp đồng `equals()`.

## Exercises

- [ ] Chạy lại đúng ví dụ ở Story (dùng `instanceof`, không sửa) — tự tay xác nhận
      `a.equals(b)` và `b.equals(a)` cho hai kết quả khác nhau.
- [ ] Sửa lại bằng `getClass()` như ở phần How?/Example, chạy lại, xác nhận cả hai chiều nhất
      quán.
- [ ] Viết một test case kiểm tra đủ cả 5 quy tắc (reflexive, symmetric, transitive,
      consistent, non-null) cho một class `equals()` bạn tự viết — tự làm quen với việc kiểm
      chứng hợp đồng một cách có hệ thống, không chỉ test "cho có".

## Cheat Sheet

| Quy tắc | Định nghĩa | Vi phạm phổ biến |
| --- | --- | --- |
| Reflexive | `x.equals(x)` = `true` | Hiếm khi vi phạm |
| Symmetric | `x.equals(y)` = `y.equals(x)` | Kế thừa + `instanceof` + field riêng ở lớp con |
| Transitive | `x=y`, `y=z` ⟹ `x=z` | Tương tự symmetric, phức tạp hơn |
| Consistent | Gọi nhiều lần, kết quả không đổi | So sánh dựa trên dữ liệu có thể đổi (ví dụ giờ hệ thống) |
| Non-null | `x.equals(null)` = `false` | Quên kiểm tra `null`, ném `NullPointerException` |

| Thứ tự chuẩn viết `equals()` |
| --- |
| 1. `if (this == o) return true;` |
| 2. `if (o == null \|\| getClass() != o.getClass()) return false;` |
| 3. Ép kiểu, so sánh từng field bằng `Objects.equals(...)` |

## References

- `java.lang.Object#equals` — Java SE API Documentation (Oracle), phần mô tả hợp đồng đầy đủ.
- Effective Java (Joshua Bloch) — Item 10: "Obey the general contract when overriding
  equals".
- Java Language Specification (JLS) — Chapter 4.3.2.
