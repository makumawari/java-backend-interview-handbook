---
tags:
  - Java
  - Comparable
  - Comparator
  - Collections
---

# Comparable vs Comparator

> Phase: Phase 2 — Collections
> Chapter slug: `comparable-comparator`

## Metadata

```yaml
Chapter: Comparable vs Comparator
Phase: Phase 2 — Collections
Difficulty: ★★★
Importance: ★★★★
Interview Frequency: 70%
Prerequisites:
  - Chapter 10 — TreeMap
  - Phase 1, Chapter 12 — equals()
Used Later:
  - Collections Utility (Chapter 15) — Collections.sort() dựa trên chính hai interface này
Estimated Reading: 20 phút
Estimated Practice: 20 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Chapter 10](10-treemap.md) — `TreeMap`/`TreeSet` sắp xếp dựa trên
> `Comparable`/`Comparator`, không dựa trên `hashCode()`.

Bạn viết một class `Product` sắp theo giá (`compareTo()` chỉ so sánh `price`), nhưng
`equals()`/`hashCode()` (Phase 1, Chapter 12-13) so sánh **cả** `name` lẫn `price` — hợp lý,
vì hai sản phẩm khác tên rõ ràng là hai sản phẩm khác nhau. Thêm hai sản phẩm **cùng giá,
khác tên** vào cả `TreeSet` và `HashSet`:

```java
TreeSet size (dua tren compareTo): 1   // ⚠️ chỉ còn 1, dù rõ ràng là 2 sản phẩm khác nhau!
HashSet size (dua tren equals): 2      // đúng như kỳ vọng
```

`TreeSet` **âm thầm loại bỏ** một sản phẩm — không phải vì `equals()` nói chúng bằng nhau (nó
không nói vậy), mà vì `compareTo()` trả về `0` cho chúng. Đây là một trong những cạm bẫy tinh
vi nhất của Collection Framework: `Set` (Chapter 05) dựa trên `equals()`, nhưng `TreeSet` **âm
thầm đổi quy tắc** sang `compareTo()`. Chapter này giải thích rõ ràng hai interface
`Comparable`/`Comparator`, và đúng cạm bẫy này.

## Interview Question (Central)

> `Comparable` và `Comparator` khác nhau như thế nào? Vì sao `compareTo()` không nhất quán với
> `equals()` lại có thể gây lỗi nghiêm trọng khi dùng với `TreeSet`/`TreeMap`?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Phân biệt `Comparable` (một thứ tự "tự nhiên" duy nhất, định nghĩa trong chính class) và
      `Comparator` (thứ tự tuỳ chỉnh, định nghĩa bên ngoài, có thể có nhiều bản)
- [ ] Tự tay tái hiện được cạm bẫy "compareTo không nhất quán với equals" gây mất dữ liệu âm
      thầm trong `TreeSet`
- [ ] Dùng thành thạo `Comparator.comparing()` và `thenComparing()` (Java 8+) để sắp xếp theo
      nhiều tiêu chí
- [ ] Biết quy tắc: `compareTo()` trả về số âm/0/dương, không phải `boolean`

## Prerequisites

- Chapter 10 — đã biết `TreeMap`/`TreeSet` dùng thứ tự so sánh, không dùng hash.
- Phase 1, Chapter 12 — hợp đồng `equals()`, để hiểu rõ điểm khác biệt với `compareTo()`.

## Used Later

- **Collections Utility** (Chapter 15) — `Collections.sort()` và các method sắp xếp khác đều
  dựa trực tiếp trên `Comparable`/`Comparator` học ở đây.

## Problem

Có hai nhu cầu sắp xếp khác nhau về bản chất: (1) một class có **một** thứ tự "tự nhiên", hiển
nhiên nhất — số thì so theo giá trị, chuỗi thì so theo alphabet; (2) đôi khi cần sắp xếp theo
tiêu chí **khác** với thứ tự tự nhiên đó, thậm chí cần **nhiều cách sắp** khác nhau cho cùng
một class tuỳ ngữ cảnh (ví dụ sản phẩm: lúc sắp theo giá, lúc sắp theo tên).

## Concept

- **`Comparable<T>`** — class tự implement, định nghĩa **một** thứ tự "tự nhiên" duy nhất, qua
  method `compareTo(T other)`. Ví dụ: `Integer`, `String` đều implement `Comparable`.
- **`Comparator<T>`** — một object **riêng biệt**, độc lập với class `T`, định nghĩa một cách
  so sánh **tuỳ chỉnh** qua method `compare(T a, T b)`. Có thể tạo **nhiều** `Comparator` khác
  nhau cho cùng một class.

Cả hai đều trả về `int`: âm nếu phần tử thứ nhất "nhỏ hơn", `0` nếu "bằng nhau" (theo tiêu chí
so sánh), dương nếu "lớn hơn" — không phải `boolean` như `equals()`.

## Why?

Nếu chỉ có `Comparable`: một class chỉ có được **một** cách sắp xếp cố định, không thể sắp
theo tiêu chí khác mà không sửa chính class đó (đôi khi không thể sửa — ví dụ class đến từ thư
viện bên thứ ba). `Comparator` giải quyết đúng vấn đề này: định nghĩa cách so sánh **bên
ngoài** class, truyền vào lúc cần (`Collections.sort(list, comparator)`), cho phép **nhiều**
cách sắp khác nhau cùng tồn tại mà không đụng vào class gốc.

## How?

```java
class Product implements Comparable<Product> {
    String name; int price;
    // Comparable: MỘT thứ tự tự nhiên, định nghĩa NGAY TRONG class
    public int compareTo(Product other) { return Integer.compare(this.price, other.price); }
}

// Comparator: định nghĩa BÊN NGOÀI, có thể có NHIỀU bản khác nhau
Comparator<Product> byName = Comparator.comparing(p -> p.name);
Comparator<Product> byNameThenPrice = Comparator.comparing((Product p) -> p.name)
                                                 .thenComparing(p -> p.price);
```

## Visualization

```
Comparable<T>                         Comparator<T>
──────────────                        ──────────────
compareTo(T other)                     compare(T a, T b)
Định nghĩa TRONG class T               Định nghĩa BÊN NGOÀI class T
CHỈ MỘT thứ tự "tự nhiên"              CÓ THỂ có NHIỀU Comparator khác nhau
Ví dụ: Integer, String                 Ví dụ: Comparator.comparing(p -> p.name)
```

## Example

**Dùng cả hai** — `Comparable` cho thứ tự mặc định, `Comparator` cho thứ tự tuỳ chỉnh với
nhiều tiêu chí:

```java
import java.util.*;

public class ComparableDemo {
    public static void main(String[] args) {
        List<Product> products = new ArrayList<>(List.of(
            new Product("Ao thun", 199000),
            new Product("Giay", 199000),
            new Product("Balo", 500000)
        ));

        Collections.sort(products); // dung Comparable (natural ordering theo gia)
        System.out.println("Sap theo Comparable (gia): " + products);

        products.sort(Comparator.comparing((Product p) -> p.name)
                                 .thenComparing(p -> p.price));
        System.out.println("Sap theo Comparator (ten, roi gia): " + products);
    }
}

class Product implements Comparable<Product> {
    String name; int price;
    Product(String name, int price) { this.name = name; this.price = price; }
    public int compareTo(Product other) { return Integer.compare(this.price, other.price); }
    public String toString() { return name + "(" + price + ")"; }
}
```

Kết quả thật (JDK 17):

```
Sap theo Comparable (gia): [Ao thun(199000), Giay(199000), Balo(500000)]
Sap theo Comparator (ten, roi gia): [Ao thun(199000), Balo(500000), Giay(199000)]
```

`thenComparing()` xử lý "hoà" (tie-breaking): với `Comparable` mặc định, `Ao thun` và `Giay`
cùng giá `199000` giữ nguyên thứ tự tương đối (sort ổn định); với `Comparator` theo tên trước,
thứ tự đổi hẳn theo alphabet.

**Cạm bẫy "compareTo không nhất quán với equals"** — tái hiện đúng hiện tượng ở Story:

```java
TreeSet<Product> set = new TreeSet<>(); // dua tren Comparable (gia)
set.add(new Product("Ao thun", 199000));
set.add(new Product("Quan jean", 199000)); // GIA GIONG HET nhung TEN khac
System.out.println("TreeSet size (dua tren compareTo): " + set.size());

HashSet<Product> hashSet = new HashSet<>(); // dua tren equals()/hashCode()
hashSet.add(new Product("Ao thun", 199000));
hashSet.add(new Product("Quan jean", 199000));
System.out.println("HashSet size (dua tren equals): " + hashSet.size());
```

Kết quả thật (giả sử `equals()` so sánh cả `name` lẫn `price`):

```
TreeSet size (dua tren compareTo): 1
HashSet size (dua tren equals): 2
```

`TreeSet` chỉ giữ **1** sản phẩm — vì `compareTo()` chỉ so `price`, hai sản phẩm "hoà" (trả về
`0`) bị `TreeSet` coi là trùng lặp, dù `equals()` (đúng đắn hơn) khẳng định chúng khác nhau.
`HashSet`, dựa trên `equals()`, giữ đúng cả hai.

## Deep Dive

**Vì sao `TreeSet` lại dùng `compareTo() == 0` để quyết định "trùng lặp", không dùng
`equals()`?** Vì `TreeSet`/`TreeMap` (Chapter 10) được cài đặt trên cây nhị phân tìm kiếm —
việc "tìm xem phần tử đã tồn tại chưa" hoàn toàn dựa trên việc **đi theo cây** bằng phép so
sánh (nhỏ hơn đi trái, lớn hơn đi phải) — nó **không bao gigiờ gọi `equals()`** trong toàn bộ
quá trình chèn/tìm kiếm, vì làm vậy sẽ phá vỡ tính nhất quán giữa cấu trúc cây và phép so sánh
dùng để xây dựng nó. Đây là lý do JLS/Javadoc chính thức của `Comparable` yêu cầu:
**"tốt nhất nên đảm bảo `compareTo()` nhất quán với `equals()`"** (`(x.compareTo(y) == 0) ==
(x.equals(y))`) — không bắt buộc về mặt biên dịch, nhưng vi phạm nó gây đúng hậu quả đã thấy ở
Example khi dùng với `TreeSet`/`TreeMap`.

## Engineering Insight

**Vì sao đây được liệt vào nhóm lỗi nguy hiểm nhất — giống các lỗi hợp đồng đã gặp ở Phase 1?**
Vì nó lặp lại đúng khuôn mẫu đã thấy ở [Phase 1, Chapter 12-13](../phase-01-foundation/12-equals.md):
một hợp đồng **không bắt buộc bởi trình biên dịch**, chỉ là quy ước — vi phạm nó không gây lỗi
biên dịch, không ném exception, chỉ **âm thầm mất dữ liệu** theo cách rất khó phát hiện (chỉ
lộ ra khi dùng đúng cấu trúc dữ liệu nhạy cảm với hợp đồng đó — ở đây là `TreeSet`/`TreeMap`,
giống hệt `equals()`/`hashCode()` chỉ lộ ra khi dùng với `HashSet`/`HashMap`). Bài học chung
xuyên suốt cả hai trường hợp: Java có rất nhiều "hợp đồng mềm" (soft contract) dựa trên tài
liệu và quy ước, không phải kiểu kiểm tra tĩnh — kỷ luật tuân thủ đúng các hợp đồng này là một
kỹ năng quan trọng không kém việc biết cú pháp.

## Historical Note

`Comparable` tồn tại từ Java 2 (1998). `Comparator` cũng vậy, nhưng trở nên mạnh mẽ và dễ dùng
hơn hẳn từ Java 8 (2014) với các static/default method: `Comparator.comparing()`,
`thenComparing()`, `reversed()`, `naturalOrder()`, `nullsFirst()` — trước đó, viết một
`Comparator` yêu cầu tự implement toàn bộ interface bằng anonymous class, dài dòng hơn nhiều so
với cú pháp lambda/method reference hiện đại.

## Myth vs Reality

- **Myth:** "`Set` luôn dựa trên `equals()`/`hashCode()`, bất kể implementation nào."
  **Reality:** Chỉ `HashSet`/`LinkedHashSet` (Chapter 05-06) dựa trên `equals()`/`hashCode()`.
  `TreeSet` dựa hoàn toàn trên `compareTo()`/`Comparator`, có thể cho kết quả khác hẳn nếu hai
  hợp đồng không nhất quán — xem Example.

- **Myth:** "`compareTo()` phải trả về đúng -1, 0, hoặc 1."
  **Reality:** Chỉ cần **dấu** đúng (âm/không/dương) — giá trị tuyệt đối không quan trọng.
  `Integer.compare(a, b)` có thể trả về bất kỳ số âm/dương nào, không chỉ -1/1.

## Common Mistakes

- **Viết `compareTo()` không nhất quán với `equals()`** rồi dùng class đó làm phần tử
  `TreeSet`/key `TreeMap` — đúng lỗi trọng tâm của chapter.
- **Trừ trực tiếp hai số để so sánh** (`return a.price - b.price`) thay vì dùng
  `Integer.compare()` — có thể tràn số (overflow) nếu chênh lệch đủ lớn, cho kết quả sai dấu.
- **Quên rằng `Comparator.comparing()` cần kiểu tham số tường minh** khi dùng lambda đứng một
  mình (như `(Product p) -> p.name` ở Example) — nếu không, trình biên dịch đôi khi không suy
  luận được kiểu, gây lỗi biên dịch khó hiểu.

## Best Practices

- Luôn đảm bảo `compareTo()` nhất quán với `equals()` nếu class có khả năng được dùng làm phần
  tử `TreeSet`/key `TreeMap`.
- Dùng `Integer.compare()`/`Double.compare()` thay vì trừ trực tiếp để tránh lỗi tràn số.
- Dùng `Comparator.comparing().thenComparing()` (Java 8+) cho sắp xếp nhiều tiêu chí, thay vì
  tự viết `compare()` dài dòng với nhiều `if/else`.

## Production Notes

**Vấn đề:** một tính năng "loại bỏ đơn hàng trùng lặp" dùng `TreeSet<Order>` (sắp theo thời
gian tạo) để loại trùng, nhưng một số đơn hàng **khác nhau thật sự** (khác `orderId`) bị loại
bỏ nhầm.

- **Triệu chứng:** số lượng đơn hàng sau khi "loại trùng" ít hơn hẳn số lượng thực tế nên có,
  không có exception hay cảnh báo nào.
- **Root cause:** `compareTo()` của `Order` chỉ so sánh `createdAt` (thời gian tạo) — hai đơn
  hàng tạo cùng lúc (hoặc cùng đến mili-giây, nếu độ chính xác thời gian không đủ cao) bị
  `TreeSet` coi là "trùng lặp" dù `orderId` hoàn toàn khác nhau — đúng lỗi đã học ở Example.
- **Debug:** kiểm tra định nghĩa `compareTo()` của class đang dùng làm phần tử `TreeSet`, so
  sánh với định nghĩa `equals()` — nếu khác tiêu chí, đây chính là nguyên nhân.
- **Solution:** sửa `compareTo()` để nhất quán với `equals()` (ví dụ thêm `orderId` làm tiêu
  chí phụ khi `createdAt` bằng nhau), hoặc đổi hẳn sang `HashSet`/`LinkedHashSet` nếu không
  thực sự cần thứ tự sắp xếp.
- **Prevention:** khi dùng `TreeSet`/`TreeMap` cho mục đích loại trùng, luôn tự hỏi: "hai phần
  tử có `compareTo() == 0` thì có thực sự nên bị coi là trùng lặp về mặt nghiệp vụ không?".

## Debug Checklist

- [ ] `TreeSet`/`TreeMap` mất dữ liệu không rõ nguyên nhân? → kiểm tra `compareTo()` có nhất
      quán với `equals()` không, đây gần như luôn là nguyên nhân.
- [ ] Cần sắp xếp theo nhiều tiêu chí? → dùng `Comparator.comparing().thenComparing()`, tránh
      tự viết logic so sánh phức tạp bằng tay.
- [ ] Kết quả so sánh số sai dấu bất thường với số lớn? → kiểm tra có đang trừ trực tiếp hai số
      thay vì dùng `Integer.compare()`/`Long.compare()` không (nguy cơ tràn số).

## Source Code Walkthrough

`TreeMap.put()` (Chapter 10) gọi `compare(key, existingKey)` tại mỗi node trong lúc đi xuống
cây để tìm vị trí chèn — nếu kết quả là `0` tại bất kỳ node nào, `TreeMap` coi đó là **cùng
key**, ghi đè value thay vì tạo node mới. Đây chính xác là điểm mà hành vi "coi là trùng lặp"
xảy ra — hoàn toàn nằm trong logic tìm kiếm trên cây, không có bước gọi `equals()` nào xen vào
suốt quá trình này.

## Summary

`Comparable` định nghĩa **một** thứ tự tự nhiên ngay trong class (`compareTo()`); `Comparator`
định nghĩa thứ tự tuỳ chỉnh **bên ngoài** class, cho phép nhiều cách sắp khác nhau
(`Comparator.comparing().thenComparing()`, Java 8+). `TreeSet`/`TreeMap` (Chapter 10) dựa hoàn
toàn vào `compareTo()`/`Comparator` để quyết định "trùng lặp" — **không** gọi `equals()` — nên
nếu hai hợp đồng không nhất quán, dữ liệu có thể bị mất âm thầm mà không có bất kỳ cảnh báo
nào, một cạm bẫy cùng bản chất với vi phạm hợp đồng `equals()`/`hashCode()` đã học ở Phase 1.

## Interview Questions

**Junior**

- What is the difference between Comparator and Comparable?
- `compareTo()` trả về kiểu gì? Có ý nghĩa gì khi trả về giá trị âm/0/dương?

**Mid**

- Vì sao `compareTo()` không nhất quán với `equals()` có thể gây mất dữ liệu trong `TreeSet`?
- Dùng `Comparator.comparing().thenComparing()` để sắp xếp theo nhiều tiêu chí — viết một ví dụ
  cụ thể.

**Senior**

- Giải thích tại sao `TreeSet`/`TreeMap` không bao giờ gọi `equals()` trong quá trình chèn/tìm
  kiếm — dựa trên cấu trúc cây nhị phân tìm kiếm.
- Một tính năng loại trùng dữ liệu bằng `TreeSet` bị mất dữ liệu không mong muốn. Trình bày quy
  trình điều tra và giải pháp, liên hệ trực tiếp hợp đồng `compareTo()`/`equals()`.

## Exercises

- [ ] Chạy lại đúng ví dụ `ComparableDemo` ở trên.
- [ ] Tái hiện đúng cạm bẫy `CompareToTrap` — xác nhận `TreeSet` chỉ giữ 1 phần tử trong khi
      `HashSet` giữ đúng 2.
- [ ] Sửa `compareTo()` của `Product` để nhất quán với `equals()` (so sánh cả `name` khi `price`
      bằng nhau) — chạy lại, xác nhận `TreeSet` giờ giữ đúng cả 2 phần tử.

## Cheat Sheet

| | Comparable | Comparator |
| --- | --- | --- |
| Method | `compareTo(T other)` | `compare(T a, T b)` |
| Định nghĩa ở đâu | Trong chính class | Bên ngoài, độc lập |
| Số lượng thứ tự | Một (tự nhiên) | Nhiều, tuỳ ý |
| Dùng bởi | `Collections.sort(list)`, `TreeSet`/`TreeMap` mặc định | `Collections.sort(list, cmp)`, `TreeSet`/`TreeMap` tuỳ chỉnh |

```java
Comparator.comparing((Product p) -> p.name)   // sắp theo tên
          .thenComparing(p -> p.price)        // hoà thì sắp theo giá
          .reversed();                        // đảo ngược toàn bộ
```

## References

- Java SE API Documentation — `java.lang.Comparable`, `java.util.Comparator`.
- Java Language Specification — ghi chú về "tốt nhất nên nhất quán với equals" trong Javadoc
  chính thức của `Comparable`.
