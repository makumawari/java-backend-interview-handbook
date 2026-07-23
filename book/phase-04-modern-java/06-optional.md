---
tags:
  - Java
  - Optional
  - ModernJava
---

# Optional

> Phase: Phase 4 — Modern Java
> Chapter slug: `optional`

## Metadata

```yaml
Chapter: Optional
Phase: Phase 4 — Modern Java
Difficulty: ★★
Importance: ★★★★
Interview Frequency: 75%
Prerequisites:
  - Chapter 01 — Lambda
Used Later:
  - Chapter 04 — Stream (findFirst/findAny trả về Optional)
Estimated Reading: 15 phút
Estimated Practice: 12 phút
```

## Story

> Mọi lập trình viên Java đều từng gặp `NullPointerException` — ném ra ở một dòng code hoàn
> toàn không liên quan tới nơi giá trị `null` thực sự được tạo ra, khiến việc debug mất nhiều
> thời gian truy ngược nguồn gốc.

Với hai cách cung cấp giá trị mặc định trông có vẻ tương đương:

```java
present.orElse(expensiveDefault("A"));
present.orElseGet(() -> expensiveDefault("B"));
```

Thử đo xem hàm `expensiveDefault` có thực sự được gọi hay không, ngay cả khi `present` **đã có
giá trị** (không cần dùng tới giá trị mặc định):

```
--- orElse: LUON danh gia tham so, du Optional co gia tri ---
  -> expensiveDefault(orElse, Optional CO gia tri) DUOC GOI!
--- orElseGet: CHI danh gia (lazy) khi Optional RONG ---
  (khong thay dong '-> expensiveDefault' o tren vi orElseGet LAZY)
```

`orElse()` **luôn** gọi `expensiveDefault()`, dù giá trị đó cuối cùng không được dùng tới.
`orElseGet()` thì không — nó chỉ gọi khi thực sự cần.

## Interview Question (Central)

> Optional dùng khi nào, có nên dùng làm field trong Entity không?

## Objectives

- [ ] Hiểu đúng mục đích thiết kế của `Optional`: kiểu trả về, không phải kiểu dữ liệu để lưu
      trữ
- [ ] Tự tay chứng minh khác biệt `orElse()` (luôn đánh giá) và `orElseGet()` (lazy, chỉ đánh
      giá khi cần)
- [ ] Giải thích chính xác vì sao KHÔNG nên dùng `Optional` làm field của Entity/class thông
      thường

## Prerequisites

- Chapter 01 — hiểu lambda, vì `orElseGet`, `map`, `flatMap` đều nhận lambda làm tham số.

## Used Later

- **Chapter 04 (Stream)** — `findFirst()`, `findAny()`, `reduce()` trả về `Optional`.

## Problem

`null` trong Java không phân biệt được hai ý nghĩa hoàn toàn khác nhau: "giá trị chưa được gán"
(lỗi lập trình, cần sửa) và "giá trị hợp lệ nhưng không tồn tại" (trạng thái hợp lệ của dữ liệu,
ví dụ một `Order` chưa có `discountCode`). Vì không có gì trong **kiểu trả về** của một phương
thức báo hiệu "phương thức này có thể trả về null", lập trình viên gọi nó dễ quên kiểm tra, dẫn
tới `NullPointerException` xuất hiện ở một nơi hoàn toàn khác so với nơi giá trị null được tạo
ra.

## Concept

**`Optional<T>`** là một wrapper object biểu diễn tường minh "có thể có hoặc không có giá trị" —
dùng làm **kiểu trả về** của phương thức để báo hiệu rõ ràng ngay tại chữ ký phương thức: "kết
quả này có thể không tồn tại, bạn PHẢI xử lý trường hợp đó". Nó cung cấp các phương thức
(`map`, `flatMap`, `filter`, `orElse`, `orElseGet`, `ifPresent`) để xử lý cả hai trường hợp
(có/không có giá trị) một cách khai báo (declarative), không cần `if (x != null)` tường minh.

## Why?

`Optional` chuyển vấn đề "có thể null" từ một **quy ước ngầm** (phải đọc Javadoc hoặc đoán) sang
một **ràng buộc tường minh trong kiểu dữ liệu** — chữ ký phương thức `Optional<String> findEmail()`
tự nó đã nói rõ "có thể không có email" ngay tại điểm khai báo, thay vì `String findEmail()` mơ
hồ (không rõ có thể trả `null` hay không nếu không đọc tài liệu). Đây là ứng dụng trực tiếp của
nguyên tắc "làm cho lỗi tiềm ẩn không thể biên dịch được" — dù `Optional` không loại bỏ hoàn toàn
khả năng lỗi (vẫn có thể gọi `.get()` mù quáng), nó buộc lập trình viên phải **chủ động** đối mặt
với trường hợp rỗng ngay khi viết code, thay vì vô tình bỏ sót.

## How?

```java
Optional<String> maybeEmail = findEmail(userId);

String email = maybeEmail.orElse("no-email@example.com");       // gia tri mac dinh CO SAN
String email2 = maybeEmail.orElseGet(() -> tinhToanMacDinh());  // gia tri mac dinh TON KEM, lazy
maybeEmail.ifPresent(e -> System.out.println("Email: " + e));   // chi chay neu CO gia tri
String upper = maybeEmail.map(String::toUpperCase).orElse("N/A"); // bien doi neu CO gia tri
```

## Visualization

```
orElse(expensiveDefault()):

  Optional CO gia tri  -> VAN goi expensiveDefault() (lang phi, roi bo ket qua)
  Optional RONG        -> goi expensiveDefault(), dung ket qua

orElseGet(() -> expensiveDefault()):

  Optional CO gia tri  -> KHONG goi expensiveDefault() (lazy, tiet kiem)
  Optional RONG        -> goi expensiveDefault(), dung ket qua
```

## Example

```java
import java.util.*;

public class OptionalDemo {
    static String expensiveDefault(String label) {
        System.out.println("  -> expensiveDefault(" + label + ") DUOC GOI!");
        return "gia tri mac dinh";
    }

    record Product(String name, Optional<String> discountCode) {}

    public static void main(String[] args) {
        Optional<String> present = Optional.of("gia tri that");
        Optional<String> empty = Optional.empty();

        System.out.println("--- orElse: LUON danh gia tham so ---");
        present.orElse(expensiveDefault("orElse, Optional CO gia tri"));

        System.out.println("--- orElseGet: CHI danh gia (lazy) khi RONG ---");
        present.orElseGet(() -> expensiveDefault("orElseGet, Optional CO gia tri"));
        System.out.println("  (khong thay dong '-> expensiveDefault' o tren vi orElseGet LAZY)");

        Product p1 = new Product("Laptop", Optional.of("SALE10"));
        Product p2 = new Product("Mouse", Optional.empty());

        String code1 = p1.discountCode().map(c -> "Ma: " + c).orElse("Khong co ma giam gia");
        String code2 = p2.discountCode().map(c -> "Ma: " + c).orElse("Khong co ma giam gia");
        System.out.println(p1.name() + " -> " + code1);
        System.out.println(p2.name() + " -> " + code2);

        try {
            empty.get();
        } catch (NoSuchElementException e) {
            System.out.println("get() tren Optional rong nem: " + e.getClass().getSimpleName());
        }
    }
}
```

Kết quả thật (JDK 17):

```
--- orElse: LUON danh gia tham so ---
  -> expensiveDefault(orElse, Optional CO gia tri) DUOC GOI!
--- orElseGet: CHI danh gia (lazy) khi RONG ---
  (khong thay dong '-> expensiveDefault' o tren vi orElseGet LAZY)
Laptop -> Ma: SALE10
Mouse -> Khong co ma giam gia
get() tren Optional rong nem: NoSuchElementException
```

Bằng chứng rõ ràng: dù `present` đã có giá trị (không cần dùng tới default), `orElse()` **vẫn
gọi** `expensiveDefault()` (in ra dòng "DUOC GOI!"). `orElseGet()` với cùng điều kiện **không hề
gọi** hàm bên trong — vì tham số của nó là một `Supplier` (lambda), chỉ được gọi khi
`Optional` thực sự rỗng.

## Deep Dive

**Vì sao KHÔNG nên dùng `Optional` làm field của một Entity (ví dụ JPA `@Entity`)?** Có ba lý do
kỹ thuật cụ thể: (1) `Optional` **không implement `Serializable`** — nếu Entity cần serialize
(cache phân tán, session replication), field kiểu `Optional` sẽ gây lỗi runtime. (2) JPA/Hibernate
không có cách ánh xạ `Optional<String>` trực tiếp thành cột database theo kiểu tự nhiên — cần
converter tuỳ chỉnh, thêm phức tạp không cần thiết. (3) Bản thân tài liệu chính thức của
`Optional` (Javadoc từ Brian Goetz, kiến trúc sư trưởng của tính năng này) nói rõ nó được thiết
kế **chỉ** cho kiểu trả về của phương thức, để biểu diễn "có thể không có kết quả" ở API công
khai — không phải để làm kiểu lưu trữ trạng thái nội bộ của object. Field/property/tham số
phương thức nên dùng `null` (khi thực sự cần "không có giá trị") kèm kiểm tra tường minh hoặc
annotation (`@Nullable`), không phải `Optional`.

## Engineering Insight

**`Optional` giải quyết được vấn đề gì, và KHÔNG giải quyết được vấn đề gì?** Nó buộc lập trình
viên **đối mặt** với khả năng "không có giá trị" ngay tại điểm gọi (nhờ kiểu trả về tường minh),
nhưng nó **không ngăn được** việc lập trình viên gọi `.get()` một cách mù quáng mà không kiểm
tra `isPresent()` trước — khi đó vẫn nhận `NoSuchElementException`, về bản chất là cùng một loại
lỗi với `NullPointerException`, chỉ khác tên exception. Giá trị thực sự của `Optional` nằm ở
việc nó **thay đổi API contract** một cách tường minh (chữ ký phương thức tự nói lên ý định), và
khuyến khích phong cách xử lý khai báo (`map`/`orElse` thay vì `if-else` lồng nhau) — không phải
một "lá chắn" tuyệt đối chống lại mọi lỗi liên quan tới giá trị rỗng.

## Historical Note

`Optional` ra đời ở Java 8 (2014), lấy cảm hứng từ `Option`/`Maybe` trong các ngôn ngữ hàm
(Scala, Haskell). Brian Goetz (kiến trúc sư ngôn ngữ Java) đã công khai nhấn mạnh nhiều lần rằng
`Optional` được thiết kế **hạn chế** — chỉ dành cho kiểu trả về API công khai, không phải giải
pháp chung để thay thế mọi `null` trong toàn bộ codebase (khác với cách một số ngôn ngữ khác coi
kiểu "optional/maybe" là công dân hạng nhất ở khắp mọi nơi).

## Myth vs Reality

- **Myth:** "Nên dùng `Optional` cho mọi field có thể null, mọi tham số phương thức, để tránh
  `NullPointerException` hoàn toàn."
  **Reality:** Xem Deep Dive — chỉ nên dùng cho kiểu trả về của phương thức. Dùng làm field
  Entity hoặc tham số phương thức là chống chỉ định theo đúng thiết kế gốc của tính năng.

- **Myth:** "`orElse()` và `orElseGet()` hoàn toàn tương đương, chỉ khác cú pháp."
  **Reality:** Đã chứng minh bằng thực nghiệm — `orElse()` luôn đánh giá tham số (eager),
  `orElseGet()` chỉ đánh giá khi cần (lazy). Với giá trị mặc định tốn kém tính toán (gọi API,
  query DB), dùng nhầm `orElse()` gây lãng phí không cần thiết.

## Common Mistakes

- **Dùng `Optional` làm field của Entity/class thông thường** — xem Deep Dive, gây vấn đề
  serialization và không cần thiết theo đúng thiết kế gốc.
- **Dùng `orElse(tinhToanTonKem())` thay vì `orElseGet(() -> tinhToanTonKem())`** — gây lãng phí
  hiệu năng không cần thiết khi Optional đã có giá trị, đã chứng minh bằng thực nghiệm ở Example.
- **Gọi `.get()` mà không kiểm tra `isPresent()`** — mất hết lợi ích của `Optional`, quay lại
  đúng vấn đề `NullPointerException` ban đầu (chỉ đổi tên thành `NoSuchElementException`).
- **Dùng `Optional` làm tham số phương thức** — làm phức tạp API không cần thiết; nếu tham số có
  thể thiếu, thường tốt hơn nên dùng method overloading hoặc giá trị mặc định rõ ràng.

## Best Practices

- Chỉ dùng `Optional` làm kiểu trả về của phương thức public, khi kết quả thực sự có thể không
  tồn tại theo logic nghiệp vụ.
- Dùng `orElseGet()` (không phải `orElse()`) khi giá trị mặc định tốn kém để tính toán/tạo ra.
- Ưu tiên `map`/`flatMap`/`filter` theo phong cách khai báo thay vì gọi `isPresent()` rồi
  `if-else` thủ công.
- Không dùng `Optional` cho field, tham số phương thức, hoặc kiểu dữ liệu trong collection
  (`List<Optional<T>>` thường là dấu hiệu thiết kế có vấn đề).

## Debug Checklist

- [ ] `NoSuchElementException` từ `Optional.get()`? → kiểm tra đã gọi `isPresent()` hoặc dùng
      `orElse`/`orElseGet`/`ifPresent` thay vì `get()` trực tiếp chưa.
- [ ] Hiệu năng giảm bất thường ở nơi dùng `Optional.orElse()` với logic tính toán phức tạp bên
      trong? → đổi sang `orElseGet()` để tránh đánh giá không cần thiết.
- [ ] Lỗi serialization liên quan tới field kiểu `Optional`? → loại bỏ `Optional` khỏi field,
      chuyển về `null` truyền thống hoặc tách logic "optional" ra khỏi tầng lưu trữ.

## Summary

`Optional<T>` là kiểu trả về tường minh cho "có thể không có giá trị", buộc lập trình viên đối
mặt với trường hợp rỗng ngay tại chữ ký phương thức thay vì quy ước ngầm dễ quên. Đã chứng minh
bằng thực nghiệm: `orElse()` luôn đánh giá tham số (eager), `orElseGet()` chỉ đánh giá khi thực
sự cần (lazy) — quan trọng khi giá trị mặc định tốn kém. Chỉ nên dùng cho kiểu trả về của
phương thức public — **không** dùng làm field Entity (vấn đề serialization, không có ánh xạ ORM
tự nhiên) hay tham số phương thức, đúng theo thiết kế gốc mà Brian Goetz đã công khai nhấn
mạnh.

## Interview Questions

- Optional dùng khi nào, có nên dùng làm field trong Entity không?

**Mid**

- `orElse()` khác `orElseGet()` như thế nào? Khi nào sự khác biệt này thực sự quan trọng?

## Exercises

- [ ] Chạy lại `OptionalDemo` ở trên, xác nhận `orElse()` luôn gọi hàm còn `orElseGet()` thì
      không, đúng như mô tả.
- [ ] Viết một phương thức `findUserById(long id)` trả về `Optional<User>`, dùng `map`/`orElseThrow` để chuyển thành response API rõ ràng (200 OK hoặc 404 Not Found).
- [ ] Thử dùng `Optional` làm field trong một class implement `Serializable`, quan sát lỗi hoặc
      hành vi bất thường khi serialize.

## Cheat Sheet

| Phương thức | Hành vi |
| --- | --- |
| `orElse(giaTri)` | Trả về giá trị nếu có, ngược lại dùng giá trị mặc định — LUÔN đánh giá tham số |
| `orElseGet(supplier)` | Như trên nhưng LAZY — chỉ gọi supplier khi rỗng |
| `orElseThrow()` | Trả về giá trị hoặc ném `NoSuchElementException`/exception tuỳ chỉnh |
| `map(fn)` | Biến đổi giá trị nếu có, giữ nguyên rỗng nếu không |
| `ifPresent(consumer)` | Chạy consumer chỉ khi có giá trị |
| `.get()` | Lấy giá trị trực tiếp — TRÁNH dùng nếu chưa kiểm tra `isPresent()` |

**Không nên dùng `Optional` cho:** field của Entity/class, tham số phương thức, phần tử trong
collection.

## References

- Java SE API Documentation — `java.util.Optional`.
- Brian Goetz — "Optional: How to Use It" (bài viết chính thức về thiết kế của Optional).
