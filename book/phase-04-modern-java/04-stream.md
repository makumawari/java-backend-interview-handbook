---
tags:
  - Java
  - Stream
  - ModernJava
---

# Stream API

> Phase: Phase 4 — Modern Java
> Chapter slug: `stream`

## Metadata

```yaml
Chapter: Stream
Phase: Phase 4 — Modern Java
Difficulty: ★★★
Importance: ★★★★★
Interview Frequency: 90%
Prerequisites:
  - Chapter 01 — Lambda
  - Chapter 02 — Functional Interface
  - Chapter 03 — Method Reference
Used Later:
  - Chapter 05 — Collector
Estimated Reading: 20 phút
Estimated Practice: 20 phút
```

## Story

> Xem lại: [Chapter 01-03](01-lambda.md) — lambda và method reference cho phép truyền hành vi
> ngắn gọn. Nhưng viết `for` loop lồng nhau để lọc, biến đổi, rồi gộp dữ liệu từ một collection
> vẫn còn khá dài dòng và dễ lẫn logic "làm gì" với logic "lặp như thế nào".

Thử nghiệm thực tế: tạo một Stream, gọi `filter()`, nhưng **chưa** gọi thao tác cuối cùng
(terminal operation):

```java
Stream<Product> lazyStream = products.stream()
    .peek(p -> System.out.println("peek: " + p.name()))
    .filter(p -> p.price() > 100);
System.out.println("Da tao xong stream, peek() CHUA IN GI CA");
```

Kết quả thật:

```
Da tao xong stream, peek() CHUA IN GI CA (lazy)
--- Bay gio goi terminal operation (count()) ---
peek (intermediate) thay: Laptop
peek (intermediate) thay: Mouse
...
count() = 4
```

`peek()` — dù đã được gọi trong code — **không in gì cả** cho tới khi có một **terminal
operation** (`count()`) được gọi. Đây chính là "lazy evaluation" — điểm khác biệt căn bản giữa
Stream và một vòng lặp thông thường.

## Interview Question (Central)

> Stream API có lazy evaluation là gì? Vì sao các thao tác trung gian (`filter`, `map`, `peek`)
> không thực thi ngay khi được gọi?

## Objectives

- [ ] Phân biệt intermediate operation (lazy) và terminal operation (kích hoạt thực thi)
- [ ] Tự tay chứng minh bằng thực nghiệm: `peek()` không chạy tới khi có terminal operation
- [ ] Tự tay chứng minh short-circuiting: `findFirst()` dừng ngay khi tìm thấy, không duyệt hết
      collection

## Prerequisites

- Chapter 01-03 — hiểu lambda, functional interface, method reference (dùng làm tham số cho các
  thao tác Stream).

## Used Later

- **Chapter 05 (Collector)** — `collect()` là terminal operation phổ biến nhất để gộp kết quả
  Stream thành collection/giá trị tổng hợp.

## Problem

Viết `for` loop để lọc + biến đổi + gộp dữ liệu từ collection thường lồng ghép "làm gì" (logic
nghiệp vụ) với "lặp như thế nào" (chi tiết kỹ thuật của vòng lặp) trong cùng một khối code, làm
giảm khả năng đọc hiểu ý định. Ngoài ra, viết tay các thao tác kết hợp (lọc rồi biến đổi rồi
giới hạn số lượng) thường phải duyệt collection nhiều lần hoặc dùng biến trung gian không cần
thiết.

## Concept

**Stream** là một chuỗi các phần tử hỗ trợ thao tác kiểu functional (map, filter, reduce, ...)
được **tính toán lười (lazy)**: các **intermediate operation** (`filter`, `map`, `peek`, `sorted`,
...) chỉ **mô tả** một "công thức xử lý", không thực thi ngay — chúng chỉ thực sự chạy khi có
một **terminal operation** (`collect`, `count`, `forEach`, `findFirst`, `reduce`, ...) được gọi,
và Stream xử lý **từng phần tử một, xuyên suốt toàn bộ chuỗi thao tác** (không xử lý theo từng
giai đoạn riêng biệt cho toàn bộ collection).

## Why?

Lazy evaluation cho phép JVM **tối ưu hoá toàn bộ chuỗi thao tác** trước khi thực sự chạy — ví
dụ kết hợp nhiều intermediate operation thành một lượt duyệt duy nhất (thay vì duyệt riêng cho
`filter`, rồi duyệt riêng cho `map`), và cho phép **short-circuiting**: nếu terminal operation
chỉ cần một phần kết quả (`findFirst`, `limit`, `anyMatch`), Stream có thể dừng xử lý ngay khi
đủ điều kiện, không cần duyệt hết toàn bộ nguồn dữ liệu — điều mà một chuỗi vòng lặp viết tay,
xử lý tuần tự từng giai đoạn, khó đạt được tự nhiên.

## How?

```java
List<Product> result = products.stream()   // 1. tao Stream tu collection
    .filter(p -> p.price() > 100)          // 2. intermediate: LAZY, chi "mo ta"
    .map(Product::name)                    //    intermediate: LAZY
    .sorted()                              //    intermediate: LAZY
    .collect(Collectors.toList());         // 3. terminal: KICH HOAT toan bo chuoi chay
```

## Visualization

```
Khai bao chuoi (KHONG chay gi):

  stream() -> filter() -> map() -> sorted() -> [CHUA CO terminal operation]
     (chi la mo ta "cong thuc", chua thuc thi)

Goi terminal operation (VD: collect()):

  Voi TUNG PHAN TU mot, chay QUA CA CHUOI truoc khi sang phan tu tiep theo:

  phan_tu_1 -> filter? -> map -> sorted (buffer) ...
  phan_tu_2 -> filter? -> map -> sorted (buffer) ...
  ...
  => collect() gop ket qua cuoi cung

Short-circuit (findFirst, limit, anyMatch):

  phan_tu_1 -> filter (khong khop) -> BO QUA
  phan_tu_2 -> filter (khong khop) -> BO QUA
  phan_tu_3 -> filter (KHOP!) -> DUNG NGAY, khong xu ly phan_tu_4, 5, ...
```

## Example

```java
import java.util.*;
import java.util.stream.*;

public class StreamDemo {
    record Product(String name, String category, double price) {}

    public static void main(String[] args) {
        List<Product> products = List.of(
            new Product("Laptop", "Electronics", 1200),
            new Product("Mouse", "Electronics", 25),
            new Product("Desk", "Furniture", 300),
            new Product("Chair", "Furniture", 150),
            new Product("Monitor", "Electronics", 400)
        );

        System.out.println("--- Tao stream, GOI map/filter, CHUA goi terminal op ---");
        Stream<Product> lazyStream = products.stream()
            .peek(p -> System.out.println("peek (intermediate) thay: " + p.name()))
            .filter(p -> p.price() > 100);
        System.out.println("Da tao xong stream, peek() CHUA IN GI CA (lazy)");

        System.out.println("--- Bay gio goi terminal operation (count()) ---");
        long count = lazyStream.count();
        System.out.println("count() = " + count + " (gio peek() moi thuc su chay)");

        double totalElectronics = products.stream()
            .filter(p -> p.category().equals("Electronics"))
            .mapToDouble(Product::price)
            .sum();
        System.out.println("Tong gia Electronics: " + totalElectronics);

        Optional<Product> firstFurniture = products.stream()
            .peek(p -> System.out.println("peek (short-circuit test): " + p.name()))
            .filter(p -> p.category().equals("Furniture"))
            .findFirst();
        System.out.println("San pham Furniture dau tien: " + firstFurniture.get().name());
    }
}
```

Kết quả thật (JDK 17):

```
--- Tao stream, GOI map/filter, CHUA goi terminal op ---
Da tao xong stream, peek() CHUA IN GI CA (lazy)
--- Bay gio goi terminal operation (count()) ---
peek (intermediate) thay: Laptop
peek (intermediate) thay: Mouse
peek (intermediate) thay: Desk
peek (intermediate) thay: Chair
peek (intermediate) thay: Monitor
count() = 4 (gio peek() moi thuc su chay)
Tong gia Electronics: 1625.0
peek (short-circuit test): Laptop
peek (short-circuit test): Mouse
peek (short-circuit test): Desk
San pham Furniture dau tien: Desk
```

Bằng chứng lazy evaluation: dòng `"Da tao xong stream..."` in **trước** bất kỳ dòng `peek` nào —
`filter()` đã được gọi trong code nhưng chưa hề thực thi. Chỉ khi `count()` (terminal operation)
được gọi, cả 5 dòng `peek` mới xuất hiện.

Bằng chứng short-circuiting: `findFirst()` chỉ in ra 3 dòng `peek` (Laptop, Mouse, Desk) rồi
**dừng ngay** khi tìm thấy "Desk" (sản phẩm Furniture đầu tiên) — không hề xử lý "Chair" hay
"Monitor" dù chúng vẫn còn trong danh sách gốc.

## Deep Dive

**Vì sao Stream xử lý "từng phần tử xuyên suốt cả chuỗi" thay vì "từng giai đoạn cho toàn bộ
collection"?** Đây gọi là kỹ thuật **fusion** (hợp nhất các thao tác trung gian). Nếu Stream xử
lý theo giai đoạn (chạy `filter` cho toàn bộ 5 phần tử trước, rồi mới chạy `map` cho kết quả),
nó sẽ cần một cấu trúc dữ liệu trung gian để lưu kết quả sau mỗi giai đoạn — tốn thêm bộ nhớ và
thêm một lượt duyệt cho mỗi thao tác. Bằng cách xử lý từng phần tử đi "xuyên suốt" toàn bộ chuỗi
thao tác trước khi sang phần tử tiếp theo (`filter` → `map` → `sorted`-buffer, tất cả cho
`Laptop` trước khi xét `Mouse`), Stream chỉ cần **một lượt duyệt duy nhất** qua nguồn dữ liệu
gốc (trừ các thao tác cần toàn bộ dữ liệu trước khi tiếp tục, như `sorted()` — buộc phải đợi hết
rồi mới sắp xếp).

## Engineering Insight

**Vì sao "quên gọi terminal operation" là lỗi phổ biến khi mới học Stream, và hậu quả của nó là
gì?** Vì Stream lazy, code như `products.stream().filter(...).map(...)` (không có `.collect()`
hay bất kỳ terminal operation nào ở cuối) **biên dịch được, chạy được, nhưng không làm gì cả** —
không có exception, không có cảnh báo runtime, chỉ đơn giản là logic filter/map không bao giờ
thực thi. Đây là một trong những lỗi khó phát hiện nhất với người mới dùng Stream, vì không có
tín hiệu lỗi rõ ràng nào — hậu quả (dữ liệu "biến mất" một cách âm thầm) chỉ lộ ra khi kiểm tra
kết quả cuối cùng và thấy nó rỗng/sai một cách khó hiểu.

## Historical Note

Stream API ra đời ở Java 8 (2014), cùng lambda (Chapter 01) và `CompletableFuture` (Phase 3,
Chapter 18) — phản ánh làn sóng chuyển dịch sang phong cách lập trình hàm của JDK giai đoạn này.
Trước đó, thư viện bên thứ ba như Google Guava đã có các tiện ích tương tự (`FluentIterable`)
nhưng không tích hợp sâu vào ngôn ngữ như Stream API.

## Myth vs Reality

- **Myth:** "Stream luôn nhanh hơn vòng lặp `for` truyền thống."
  **Reality:** Với dữ liệu nhỏ, Stream thường **chậm hơn** `for` loop do overhead tạo object
  trung gian (mỗi bước `filter`/`map` tạo một `Stream` wrapper mới) — lợi ích thực sự của Stream
  là khả năng đọc hiểu và biểu đạt ý định rõ ràng hơn, không phải luôn luôn nhanh hơn về hiệu
  năng thô. `parallelStream()` (Phase 3, Chapter 17) chỉ có lợi khi dữ liệu đủ lớn và công việc
  mỗi phần tử đủ nặng.

- **Myth:** "Gọi `filter()`/`map()` trên Stream là đã xử lý dữ liệu ngay lập tức."
  **Reality:** Đã chứng minh bằng thực nghiệm ở Example — chúng chỉ thực sự chạy khi có terminal
  operation.

## Common Mistakes

- **Quên gọi terminal operation** — xem Engineering Insight, code chạy "im lặng", không làm gì
  cả mà không có bất kỳ tín hiệu lỗi nào.
- **Tái sử dụng một Stream đã bị "tiêu thụ" (đã gọi terminal operation)** — Stream chỉ dùng được
  **một lần**; gọi lại một terminal operation khác trên cùng Stream ném
  `IllegalStateException: stream has already been operated upon or closed`.
- **Dùng `peek()` cho logic nghiệp vụ thực sự (không chỉ để debug)** — `peek()` được thiết kế
  cho mục đích quan sát/debug, JDK không đảm bảo nó luôn chạy đúng số lần dự kiến trong mọi
  trường hợp tối ưu hoá của JVM.

## Best Practices

- Luôn đảm bảo mỗi chuỗi Stream kết thúc bằng đúng một terminal operation.
- Dùng `peek()` chỉ cho mục đích debug tạm thời, không dùng nó để thực hiện logic nghiệp vụ có
  tác dụng phụ quan trọng.
- Với dữ liệu nhỏ hoặc logic đơn giản, cân nhắc `for` loop truyền thống nếu nó rõ ràng hơn Stream
  — không ép dùng Stream cho mọi trường hợp.

## Debug Checklist

- [ ] Logic trong `filter()`/`map()` "không chạy", không có lỗi nào? → kiểm tra chuỗi Stream có
      terminal operation ở cuối không.
- [ ] `IllegalStateException: stream has already been operated upon`? → kiểm tra có đang tái sử
      dụng một biến Stream đã gọi terminal operation trước đó không — tạo Stream mới từ
      collection gốc thay vì tái sử dụng.
- [ ] Cần dừng xử lý sớm khi tìm thấy kết quả đầu tiên? → dùng `findFirst()`/`anyMatch()`/`limit()` để tận dụng short-circuiting thay vì `collect()` rồi lọc thủ công sau.

## Summary

Stream API xử lý dữ liệu theo mô hình lazy: intermediate operation (`filter`, `map`, `peek`)
chỉ mô tả công thức xử lý, terminal operation (`collect`, `count`, `findFirst`) mới thực sự kích
hoạt toàn bộ chuỗi chạy — đã chứng minh bằng thực nghiệm: `peek()` không in gì cho tới khi có
terminal operation. Stream xử lý từng phần tử xuyên suốt cả chuỗi thao tác (kỹ thuật fusion),
cho phép short-circuiting — `findFirst()` dừng ngay khi tìm thấy kết quả, không duyệt hết
collection (đã chứng minh: chỉ 3/5 phần tử được xử lý). Quên terminal operation là lỗi phổ biến
và khó phát hiện vì không có tín hiệu lỗi nào.

## Interview Questions

- Stream API có lazy evaluation là gì?

**Mid**

- Intermediate operation và terminal operation khác nhau như thế nào? Cho ví dụ mỗi loại.
- Short-circuiting trong Stream là gì? Thao tác nào tận dụng được nó?

**Senior**

- Giải thích vì sao Stream xử lý "từng phần tử xuyên suốt cả chuỗi thao tác" thay vì "từng giai
  đoạn cho toàn bộ dữ liệu". Điều này có lợi ích gì?

## Exercises

- [ ] Chạy lại `StreamDemo` ở trên, xác nhận thứ tự các dòng in ra đúng như mô tả (lazy +
      short-circuit).
- [ ] Viết một đoạn code Stream cố ý quên terminal operation, xác nhận không có lỗi/cảnh báo
      nào xuất hiện khi chạy.
- [ ] Thử gọi `.count()` hai lần trên cùng một biến Stream, quan sát `IllegalStateException`.

## Cheat Sheet

| Loại | Ví dụ | Đặc điểm |
| --- | --- | --- |
| Intermediate | `filter`, `map`, `peek`, `sorted`, `distinct`, `limit` | Lazy, trả về Stream mới |
| Terminal | `collect`, `forEach`, `count`, `reduce`, `findFirst`, `anyMatch` | Kích hoạt thực thi, kết thúc Stream |
| Short-circuit | `findFirst`, `findAny`, `anyMatch`, `limit` | Có thể dừng sớm, không duyệt hết |

## References

- Java SE API Documentation — `java.util.stream.Stream`.
- JEP tổng quan Java 8 — https://openjdk.org/projects/jdk8/
