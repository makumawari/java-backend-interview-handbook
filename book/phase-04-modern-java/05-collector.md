---
tags:
  - Java
  - Collector
  - ModernJava
---

# Collector

> Phase: Phase 4 — Modern Java
> Chapter slug: `collector`

## Metadata

```yaml
Chapter: Collector
Phase: Phase 4 — Modern Java
Difficulty: ★★★
Importance: ★★★★
Interview Frequency: 65%
Prerequisites:
  - Chapter 04 — Stream
Used Later:
  - Truy vấn/tổng hợp dữ liệu trong tầng service (Phase 5+)
Estimated Reading: 15 phút
Estimated Practice: 15 phút
```

## Story

> Xem lại: [Chapter 04](04-stream.md) — `collect()` là terminal operation phổ biến nhất để gộp
> kết quả Stream. Nhưng `collect()` cần một tham số kiểu `Collector` — chính xác nó "biết" cách
> gộp một `List<Product>` thành `Map<String, List<Product>>` như thế nào?

Thử nhóm danh sách sản phẩm theo `category`, rồi thử "gộp" theo một field có key **trùng nhau**:

```java
Map<String, String> m = products.stream()
    .collect(Collectors.toMap(Product::category, Product::name));
```

Với 2 sản phẩm cùng category "Electronics":

```
toMap voi key trung nem: Duplicate key Electronics (attempted merging values Laptop and Mouse)
```

`Collectors.toMap()` không tự động "gộp" các giá trị trùng key — nó **ném lỗi ngay lập tức**
trừ khi bạn cung cấp tường minh cách xử lý xung đột.

## Interview Question (Central)

> `Collectors.groupingBy()` khác `Collectors.toMap()` như thế nào? Điều gì xảy ra khi
> `toMap()` gặp key trùng nhau?

## Objectives

- [ ] Dùng thành thạo `groupingBy`, `toMap`, `joining`, `partitioningBy`, downstream collector
      (`counting`, `summingDouble`)
- [ ] Tự tay chứng minh `toMap()` ném `IllegalStateException` với key trùng, và cách sửa bằng
      merge function
- [ ] Hiểu `Collector` là một đối tượng mô tả "cách gộp" (supplier, accumulator, combiner,
      finisher) chứ không phải một thao tác đơn lẻ

## Prerequisites

- Chapter 04 — hiểu `collect()` là terminal operation, cần một `Collector` làm tham số.

## Used Later

- **Truy vấn/tổng hợp dữ liệu trong tầng service** (Phase 5+) — `groupingBy`/`toMap` là công cụ
  tiêu chuẩn để biến đổi kết quả truy vấn database thành cấu trúc dữ liệu phù hợp cho response
  API.

## Problem

`collect()` (Stream, Chapter 04) cần biết chính xác **cách** gộp các phần tử thành kết quả cuối
— gộp thành `List`? `Map`? Nhóm theo tiêu chí nào? Nếu để lập trình viên tự viết logic gộp thủ
công (một biến tích luỹ + vòng lặp) cho mỗi trường hợp, sẽ mất hết lợi ích khai báo (declarative)
mà Stream API mang lại.

## Concept

**`Collector<T, A, R>`** là một đối tượng đóng gói **chiến lược gộp** — gồm 4 thành phần:
`supplier` (tạo cấu trúc chứa kết quả rỗng ban đầu), `accumulator` (thêm một phần tử vào cấu trúc
đó), `combiner` (gộp hai cấu trúc con lại, dùng khi xử lý song song), `finisher` (biến đổi cấu
trúc trung gian thành kết quả cuối cùng). `Collectors` là lớp tiện ích cung cấp sẵn các
`Collector` thông dụng: `toList()`, `toMap()`, `groupingBy()`, `joining()`, `partitioningBy()`.

## Why?

Đóng gói chiến lược gộp thành một đối tượng `Collector` (thay vì để `collect()` nhận nhiều tham
số rời rạc) cho phép **tái sử dụng và kết hợp** các chiến lược một cách linh hoạt — ví dụ
`groupingBy(classifier, downstream)` nhận một `Collector` khác làm tham số thứ hai
(`downstream`), cho phép "nhóm rồi đếm", "nhóm rồi tính tổng" mà không cần viết một
`Collector` hoàn toàn mới cho mỗi tổ hợp. Việc tách 4 thành phần (supplier/accumulator/combiner/
finisher) rõ ràng cũng chính là điều kiện để `Collector` hoạt động đúng cả trong Stream tuần tự
lẫn Stream song song (`combiner` chỉ cần thiết khi chạy song song).

## How?

```java
List<String> names = products.stream().map(Product::name).collect(Collectors.toList());

Map<String, List<Product>> byCategory = products.stream()
    .collect(Collectors.groupingBy(Product::category));

Map<String, Long> countByCategory = products.stream()
    .collect(Collectors.groupingBy(Product::category, Collectors.counting())); // downstream collector
```

## Visualization

```
Collector<T, A, R> = chien luoc gop:

  supplier:    () -> A           (tao cau truc rong ban dau, VD: new ArrayList<>())
  accumulator: (A, T) -> void    (them 1 phan tu T vao cau truc A)
  combiner:    (A, A) -> A       (gop 2 cau truc A lai, dung khi chay song song)
  finisher:    (A) -> R          (bien doi A thanh ket qua cuoi R)

groupingBy + downstream (VD: counting):

  Product 1 (Electronics) ──┐
  Product 2 (Electronics) ──┼──► groupingBy(category) ──► {Electronics: [P1,P2,P3], Furniture: [P4,P5]}
  Product 3 (Electronics) ──┤         │
  Product 4 (Furniture)   ──┤    downstream: counting()
  Product 5 (Furniture)   ──┘         ▼
                                {Electronics: 3, Furniture: 2}
```

## Example

```java
import java.util.*;
import java.util.stream.*;

public class CollectorDemo {
    record Product(String name, String category, double price) {}

    public static void main(String[] args) {
        List<Product> products = List.of(
            new Product("Laptop", "Electronics", 1200),
            new Product("Mouse", "Electronics", 25),
            new Product("Desk", "Furniture", 300),
            new Product("Chair", "Furniture", 150),
            new Product("Monitor", "Electronics", 400)
        );

        Map<String, List<Product>> byCategory = products.stream()
            .collect(Collectors.groupingBy(Product::category));
        System.out.println("groupingBy: " + byCategory.keySet());

        Map<String, Long> countByCategory = products.stream()
            .collect(Collectors.groupingBy(Product::category, Collectors.counting()));
        System.out.println("groupingBy + counting: " + countByCategory);

        Map<String, Double> totalByCategory = products.stream()
            .collect(Collectors.groupingBy(Product::category, Collectors.summingDouble(Product::price)));
        System.out.println("groupingBy + summingDouble: " + totalByCategory);

        String names = products.stream().map(Product::name).collect(Collectors.joining(", ", "[", "]"));
        System.out.println("joining: " + names);

        Map<Boolean, List<Product>> partitioned = products.stream()
            .collect(Collectors.partitioningBy(p -> p.price() > 200));
        System.out.println("partitioningBy (gia > 200): true=" + partitioned.get(true).size()
            + ", false=" + partitioned.get(false).size());
    }
}
```

Kết quả thật (JDK 17):

```
groupingBy: [Electronics, Furniture]
groupingBy + counting: {Electronics=3, Furniture=2}
groupingBy + summingDouble: {Electronics=1625.0, Furniture=450.0}
joining: [Laptop, Mouse, Desk, Chair, Monitor]
partitioningBy (gia > 200): true=3, false=2
```

**`toMap()` với key trùng — lỗi thật:**

```java
List<Product> dup = List.of(
    new Product("Electronics", "Laptop"),
    new Product("Electronics", "Mouse") // trung key!
);
dup.stream().collect(Collectors.toMap(Product::category, Product::name));
```

Kết quả thật:

```
toMap voi key trung nem: Duplicate key Electronics (attempted merging values Laptop and Mouse)
```

Sửa bằng cách cung cấp tường minh **merge function** (tham số thứ 3):

```java
Map<String, String> merged = dup.stream()
    .collect(Collectors.toMap(Product::category, Product::name, (a, b) -> a + ", " + b));
```

```
toMap voi merge function: {Electronics=Laptop, Mouse}
```

## Deep Dive

**Vì sao `toMap()` ném lỗi ngay lập tức thay vì tự động "ghi đè" giá trị cũ (giống hành vi
`HashMap.put()` thông thường)?** Đây là quyết định thiết kế có chủ đích: `Map.put()` ghi đè âm
thầm là một nguồn lỗi tiềm ẩn phổ biến (dữ liệu bị mất mà không có cảnh báo). `Collectors.toMap()`
buộc lập trình viên phải **quyết định tường minh** cách xử lý xung đột key — ném lỗi ngay để lộ
diện vấn đề (dữ liệu có key trùng ngoài dự kiến) thay vì để nó âm thầm gây mất dữ liệu. Đây cùng
triết lý với "fail fast" đã gặp ở `@FunctionalInterface` (Chapter 02) — phát hiện lỗi thiết kế/dữ
liệu càng sớm càng tốt, thay vì để hậu quả xuất hiện mơ hồ ở nơi khác.

## Engineering Insight

**`groupingBy` mặc định trả về `HashMap` — điều này có ý nghĩa gì khi cần thứ tự key ổn định
hoặc cần một loại Map khác (ví dụ `TreeMap` để key được sắp xếp)?** `Collectors.groupingBy(classifier)` (không có thêm tham số) dùng `HashMap` bên dưới — thứ tự các key khi duyệt **không
đảm bảo** (giống hạn chế đã học ở Phase 2, Chapter 08 HashMap). Với nhu cầu cần `Map` được sắp
xếp theo key, JDK cung cấp biến thể 3 tham số:
`groupingBy(classifier, mapFactory, downstream)`, cho phép truyền `TreeMap::new` làm
`mapFactory`. Đây là một ví dụ cụ thể cho thấy kiến thức Phase 2 (đặc tính từng loại Map) vẫn áp
dụng trực tiếp khi làm việc với Collector — Stream API không "thay thế" kiến thức nền tảng về
collection, mà xây dựng trên đó.

## Historical Note

`Collectors` ra đời cùng Stream API ở Java 8 (2014). `Collectors.teeing()` (gộp kết quả từ hai
collector độc lập chạy song song trên cùng Stream) được bổ sung muộn hơn nhiều, ở Java 12
(2019) — phản ánh Stream API tiếp tục được mở rộng dần qua nhiều phiên bản sau khi ra mắt.

## Myth vs Reality

- **Myth:** "`toMap()` tự động xử lý key trùng bằng cách giữ giá trị cuối cùng, giống
  `HashMap.put()`."
  **Reality:** Đã chứng minh bằng thực nghiệm — nó ném `IllegalStateException` ngay lập tức trừ
  khi có merge function tường minh.

- **Myth:** "`groupingBy` luôn trả về kết quả có thứ tự key giống thứ tự xuất hiện trong dữ liệu
  gốc."
  **Reality:** Mặc định dùng `HashMap` bên dưới — không đảm bảo thứ tự, xem Engineering Insight
  để biết cách khắc phục.

## Common Mistakes

- **Dùng `toMap()` mà không lường trước khả năng key trùng** — gây `IllegalStateException` bất
  ngờ ở production khi dữ liệu thực tế có key trùng (dù dữ liệu test không có).
- **Kỳ vọng thứ tự ổn định từ `groupingBy()` mặc định** — cần chỉ định `mapFactory` (ví dụ
  `TreeMap::new`) nếu thực sự cần thứ tự.
- **Viết downstream collector phức tạp thủ công thay vì kết hợp các collector có sẵn** — JDK đã
  cung cấp sẵn `counting()`, `summingDouble()`, `averagingDouble()`, `mapping()`, `teeing()` cho
  hầu hết nhu cầu tổng hợp thông dụng.

## Best Practices

- Luôn cân nhắc khả năng key trùng khi dùng `toMap()` — cung cấp merge function tường minh nếu
  có khả năng xảy ra, thay vì để lỗi bất ngờ ở production.
- Dùng biến thể `groupingBy(classifier, mapFactory, downstream)` khi cần kiểm soát loại `Map`
  trả về (ví dụ `TreeMap` để có thứ tự key).
- Kết hợp `groupingBy` với downstream collector có sẵn (`counting`, `summingDouble`, `mapping`)
  thay vì tự viết logic tổng hợp thủ công sau khi group.

## Debug Checklist

- [ ] `IllegalStateException: Duplicate key` từ `toMap()`? → cung cấp merge function (tham số
      thứ 3) hoặc chuyển sang `groupingBy()` nếu thực sự cần giữ nhiều giá trị cho cùng key.
- [ ] Kết quả `groupingBy()` có thứ tự key không ổn định giữa các lần chạy? → dùng
      `groupingBy(classifier, TreeMap::new, downstream)` nếu cần thứ tự.

## Summary

`Collector` đóng gói chiến lược gộp Stream thành kết quả cuối (supplier/accumulator/combiner/
finisher). `Collectors` cung cấp sẵn các collector thông dụng: `toList`, `toMap`, `groupingBy`
(có thể kết hợp downstream collector như `counting`, `summingDouble`), `joining`,
`partitioningBy`. Đã chứng minh bằng thực nghiệm: `toMap()` ném `IllegalStateException` ngay khi
gặp key trùng (thiết kế "fail fast" có chủ đích) — cần cung cấp merge function tường minh để xử
lý xung đột. `groupingBy` mặc định dùng `HashMap`, không đảm bảo thứ tự key.

## Interview Questions

**Mid**

- `Collectors.groupingBy()` khác `Collectors.toMap()` như thế nào?
- Điều gì xảy ra khi `toMap()` gặp key trùng nhau? Cách khắc phục?

**Senior**

- Giải thích 4 thành phần của `Collector` (supplier, accumulator, combiner, finisher). Thành
  phần nào chỉ thực sự cần thiết khi chạy Stream song song?

## Exercises

- [ ] Chạy lại `CollectorDemo` và `ToMapDuplicateDemo` ở trên, xác nhận kết quả đúng như mô tả.
- [ ] Dùng `groupingBy(Product::category, TreeMap::new, Collectors.toList())` để nhóm sản phẩm
      theo category với thứ tự key được sắp xếp.
- [ ] Thử viết một `Collector` tuỳ chỉnh bằng `Collector.of()` để tính giá trung bình, so sánh
      với `Collectors.averagingDouble()` có sẵn.

## Cheat Sheet

| Collector | Kết quả |
| --- | --- |
| `toList()` / `toSet()` | `List<T>` / `Set<T>` |
| `toMap(keyFn, valueFn)` | `Map<K,V>`, ném lỗi nếu key trùng |
| `toMap(keyFn, valueFn, mergeFn)` | `Map<K,V>`, dùng mergeFn khi key trùng |
| `groupingBy(classifier)` | `Map<K, List<T>>` |
| `groupingBy(classifier, downstream)` | `Map<K, ketQuaDownstream>` |
| `joining(delim, prefix, suffix)` | `String` |
| `partitioningBy(predicate)` | `Map<Boolean, List<T>>` |

## References

- Java SE API Documentation — `java.util.stream.Collectors`.
