---
tags:
  - Java
  - Immutable
  - Collections
---

# Immutable Collection

> Phase: Phase 2 — Collections
> Chapter slug: `immutable-collection`

## Metadata

```yaml
Chapter: Immutable Collection
Phase: Phase 2 — Collections
Difficulty: ★★
Importance: ★★★
Interview Frequency: 45%
Prerequisites:
  - Chapter 15 — Collections Utility
  - Phase 1, Chapter 11 — Immutable
Used Later:
  - CopyOnWrite (Chapter 17) — một dạng khác của "an toàn khi chia sẻ dữ liệu", nhưng cho phép sửa
Estimated Reading: 15 phút
Estimated Practice: 15 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Chapter 15](15-collections-utility.md) — `Collections.unmodifiableList(list)` chỉ
> là một **view** chặn ghi, list gốc vẫn hoàn toàn mutable, dễ gây cạm bẫy nếu quên điều đó.

Java 9 (2017) giới thiệu một cách khác để tạo collection bất biến, ngắn gọn hơn hẳn:

```java
List<String> old = Collections.unmodifiableList(new ArrayList<>(List.of("a", "b", "c")));
List<String> modern = List.of("a", "b", "c");
```

Không chỉ ngắn hơn — `List.of()` giải quyết triệt để đúng cạm bẫy của
`unmodifiableList()`: nó tạo ra một **bản sao thật sự độc lập**, không phải view. Và nó còn
làm nhiều hơn thế — nó **từ chối** những giá trị mà một `ArrayList` bình thường chấp nhận vô
tư: `null`. Chapter này đi qua toàn bộ họ `List.of()`/`Set.of()`/`Map.of()`, và giải thích rõ
vì sao chúng nghiêm khắc hơn hẳn so với vẻ ngoài đơn giản của chúng.

## Interview Question (Central)

> `List.of(...)` (Java 9+) khác `Collections.unmodifiableList()` (Java 1.2) như thế nào? Vì
> sao `List.of("a", null)` ném ra lỗi ngay lập tức?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Dùng thành thạo `List.of()`, `Set.of()`, `Map.of()`, `Map.ofEntries()`
- [ ] Phân biệt rõ: `List.of()` tạo **bản sao thật, bất biến thật**; `unmodifiableList()`
      (Chapter 15) chỉ là **view**
- [ ] Biết `List.of()` từ chối `null`, `Set.of()`/`Map.of()` từ chối trùng lặp — ngay khi tạo,
      không đợi tới lúc dùng mới lộ lỗi
- [ ] Áp dụng đúng nguyên tắc thiết kế immutable đã học ở Phase 1 vào ngữ cảnh collection

## Prerequisites

- Chapter 15 — đã biết `unmodifiableList()` chỉ là view, không phải bản sao.
- Phase 1, Chapter 11 — đã học 5 quy tắc thiết kế immutable object, áp dụng tương tự cho
  collection ở chapter này.

## Used Later

- **CopyOnWrite** (Chapter 17) — một hướng tiếp cận khác cho "an toàn khi chia sẻ dữ liệu",
  nhưng **cho phép sửa** (khác hoàn toàn bất biến tuyệt đối của chapter này) — so sánh trực
  tiếp hai triết lý ở chapter kế tiếp.

## Problem

`Collections.unmodifiableList()` (Chapter 15) chỉ chặn sửa **qua chính view** — không bảo vệ
khỏi việc list gốc bị sửa từ một tham chiếu khác, một cạm bẫy dễ mắc. Ngoài ra, nó cần **hai
bước** (tạo list gốc rồi bọc) để đạt một collection thực sự an toàn. Cần một cách tạo collection
bất biến **thật sự** (không phải view), ngắn gọn, khó dùng sai hơn.

## Concept

`List.of(...)`/`Set.of(...)`/`Map.of(...)` (Java 9+, JEP 269) là các static factory method tạo
ra một collection **hoàn toàn mới, độc lập, và tự thân bất biến** — không liên kết với bất kỳ
collection nào khác (khác hẳn `unmodifiableList()`, chỉ là lớp bọc quanh một list có sẵn).
Chúng áp dụng đúng nguyên tắc thiết kế immutable object đã học ở
[Phase 1, Chapter 11](../phase-01-foundation/11-immutable.md): dữ liệu được copy vào một cấu
trúc nội bộ riêng ngay khi tạo, không có method nào cho phép sửa sau đó.

## Why?

Nếu chỉ có `unmodifiableList()` (chỉ là view): mỗi lần cần một collection **thực sự** bất biến
phải nhớ làm đúng hai bước (tạo bản sao độc lập trước, rồi mới bọc unmodifiable) — dễ quên bước
copy, tái hiện đúng cạm bẫy đã học ở Chapter 15. `List.of()` loại bỏ hoàn toàn khả năng quên
bước này bằng cách gộp "copy" và "bất biến" vào **một** lời gọi duy nhất.

## How?

```java
List<String> list = List.of("a", "b", "c");        // bản sao bất biến, KHÔNG liên kết gì với đầu vào
Set<String> set = Set.of("a", "b", "c");             // tương tự cho Set
Map<String, Integer> map = Map.of("a", 1, "b", 2);   // tương tự cho Map (tối đa 10 cặp)
Map<String, Integer> mapBig = Map.ofEntries(          // dùng khi > 10 cặp
    Map.entry("a", 1), Map.entry("b", 2)
);
```

## Visualization

```
Collections.unmodifiableList(list)        List.of("a", "b", "c")
        │                                          │
        ▼                                          ▼
   VIEW quanh "list" GỐC                    BẢN SAO MỚI, ĐỘC LẬP HOÀN TOÀN
   (list gốc đổi → view đổi theo)            (không liên kết với bất kỳ
                                               list nào khác)
```

## Example

Xác nhận cả bốn đặc tính: chặn sửa, chặn `null`, chặn trùng lặp (`Set`/`Map`), và tương thích
với `List.copyOf()`:

```java
import java.util.*;

public class ImmutableCollDemo {
    public static void main(String[] args) {
        List<String> list = List.of("a", "b", "c");
        try {
            list.add("d");
        } catch (UnsupportedOperationException e) {
            System.out.println("List.of() chan add: " + e.getClass().getSimpleName());
        }

        try {
            List.of("a", "b", null);
        } catch (NullPointerException e) {
            System.out.println("List.of() chan null: " + e.getClass().getSimpleName());
        }

        try {
            Set.of("a", "a");
        } catch (IllegalArgumentException e) {
            System.out.println("Set.of() chan trung lap: " + e.getClass().getSimpleName());
        }
    }
}
```

Kết quả thật (JDK 17):

```
List.of() chan add: UnsupportedOperationException
List.of() chan null: NullPointerException
Set.of() chan trung lap: IllegalArgumentException
```

Ba loại lỗi khác nhau cho ba vi phạm khác nhau — không phải ngẫu nhiên, mỗi loại phản ánh đúng
bản chất vấn đề (xem Deep Dive).

## Deep Dive

**Vì sao `List.of()` từ chối `null` một cách "cực đoan" — trong khi `ArrayList` bình thường
chấp nhận `null` thoải mái?** Đây là quyết định thiết kế có chủ đích, không phải giới hạn kỹ
thuật. `null` trong một collection thường là nguồn gốc của rất nhiều `NullPointerException`
xảy ra **muộn** — khi ai đó duyệt qua collection và xử lý phần tử mà không kiểm tra `null`
trước. Bằng cách **chặn ngay lúc tạo** (`List.of("a", null)` ném lỗi ngay lập tức, không đợi
tới lúc dùng phần tử `null` mới lộ lỗi), API buộc lỗi phải lộ ra ở đúng nơi nó phát sinh — gần
với nguyên nhân gốc hơn nhiều so với việc để nó âm thầm lan truyền rồi gây lỗi ở một chỗ hoàn
toàn khác, khó truy vết. Đây là ứng dụng của nguyên tắc "fail fast" (thất bại sớm) — một triết
lý thiết kế API hiện đại rất phổ biến trong Java từ Java 8 trở đi.

## Engineering Insight

**`Set.of()`/`Map.of()` từ chối trùng lặp ngay lúc tạo — điều này khác gì so với `HashSet`
(Chapter 06) chấp nhận âm thầm loại bỏ trùng lặp?** Đây là điểm khác biệt tinh tế nhưng quan
trọng: `new HashSet<>(List.of("a", "a"))` sẽ tạo ra một `Set` có 1 phần tử **mà không báo lỗi
gì** — hành vi "âm thầm sửa lỗi" này có thể che giấu một bug thực sự (ví dụ dữ liệu đầu vào có
trùng lặp ngoài ý muốn, lẽ ra không nên xảy ra). `Set.of("a", "a")` **ném exception ngay lập
tức** — buộc người viết code phải đối mặt trực tiếp với việc "tôi có thực sự định đưa vào hai
giá trị giống hệt nhau không?" thay vì để `Set` âm thầm "sửa" giúp. Đây chính xác cùng triết lý
fail-fast đã phân tích ở Deep Dive, áp dụng cho một loại vi phạm khác.

## Historical Note

```
Java 1.2 (1998)
    ↓
Collections.unmodifiableList()/unmodifiableSet()/unmodifiableMap() — VIEW
chặn ghi, cần bọc quanh một collection gốc đã tồn tại (Chapter 15)
    ↓
Java 9 (2017) — JEP 269: Convenience Factory Methods for Collections
    ↓
List.of()/Set.of()/Map.of() ra đời — bản sao bất biến THẬT, tạo trực
tiếp trong MỘT lời gọi, từ chối null/trùng lặp NGAY LÚC TẠO thay vì
âm thầm chấp nhận như các cách cũ
    ↓
Java 10 (2018) — List.copyOf()/Set.copyOf()/Map.copyOf() bổ sung — tạo
bản sao bất biến từ một collection ĐÃ CÓ SẴN (đã dùng ở Phase 1 Chapter 11
cho ImmutableProduct)
```

## Myth vs Reality

- **Myth:** "`List.of()` chỉ là cách viết ngắn gọn hơn của
  `Collections.unmodifiableList(new ArrayList<>(...))`, hành vi giống hệt."
  **Reality:** Khác biệt quan trọng — `List.of()` là **bản sao độc lập thật**, không phải
  view; và nó còn **chặn `null`** ngay lúc tạo, điều `unmodifiableList()` không làm.

- **Myth:** "Đưa giá trị trùng lặp vào `Set.of(...)` sẽ tự động loại bỏ, giống `HashSet`."
  **Reality:** Xem Example — `Set.of("a", "a")` ném `IllegalArgumentException` ngay lập tức,
  không âm thầm loại bỏ như `HashSet`.

## Common Mistakes

- **Truyền `null` vào `List.of(...)` mà không lường trước** — với dữ liệu động (không phải
  hằng số viết tay), dễ gặp `NullPointerException` bất ngờ nếu nguồn dữ liệu có khả năng chứa
  `null`.
- **Nghĩ `Map.of()` phù hợp cho mọi kích thước** — `Map.of()` chỉ hỗ trợ tối đa **10 cặp** key-
  value (giới hạn overload cứng); nhiều hơn phải dùng `Map.ofEntries()`.
- **Cố sửa một `List.of()` rồi ngạc nhiên với `UnsupportedOperationException`** — đây không
  phải lỗi, là hành vi thiết kế đúng của một collection bất biến thật sự.

## Best Practices

- Dùng `List.of()`/`Set.of()`/`Map.of()` cho mọi collection hằng số hoặc dữ liệu không nên đổi
  sau khi tạo — thay thế `unmodifiableXxx()` (Chapter 15) trong code mới.
- Dùng `List.copyOf(existing)` khi cần một bản sao bất biến từ một collection **đã tồn tại**
  (thay vì viết tay các phần tử).
- Với dữ liệu động có khả năng chứa `null`, lọc bỏ `null` (hoặc xử lý riêng) trước khi truyền
  vào `List.of()`.

## Production Notes

**Vấn đề:** một service trả về danh sách cấu hình mặc định bằng `List.of(...)` chạy ổn định
qua nhiều lần deploy, đến một lần cập nhật thêm giá trị mới thì service crash ngay lúc khởi
động với `NullPointerException`.

- **Triệu chứng:** crash xảy ra ngay khi class chứa `List.of(...)` được nạp (static
  initializer — liên hệ [Phase 1, Chapter 14](../phase-01-foundation/14-object-lifecycle.md)),
  trước khi bất kỳ logic nghiệp vụ nào chạy.
- **Root cause:** giá trị mới được thêm vào danh sách cấu hình thực chất là một biến (không
  phải hằng số viết tay), và biến đó vô tình là `null` (ví dụ đọc từ file cấu hình thiếu một
  dòng) — `List.of()` phát hiện và chặn ngay, đúng đúng thiết kế fail-fast.
- **Debug:** đọc chính xác dòng `List.of(...)` gây lỗi, kiểm tra từng tham số truyền vào có
  khả năng là `null` không.
- **Solution:** sửa nguồn gốc gây ra giá trị `null` (file cấu hình thiếu, logic đọc dữ liệu
  sai); nếu `null` là tình huống hợp lệ cần xử lý, lọc bỏ trước khi truyền vào `List.of()`.
- **Prevention:** coi việc `List.of()` chặn `null` là một **lợi ích**, không phải phiền toái —
  nó đã giúp phát hiện lỗi cấu hình ngay lúc khởi động thay vì để nó âm thầm gây lỗi khó truy
  vết ở một tầng xử lý xa hơn nhiều.

## Debug Checklist

- [ ] `NullPointerException` ngay tại dòng `List.of(...)`/`Set.of(...)`/`Map.of(...)`? →
      kiểm tra từng tham số, xác định chính xác cái nào là `null` và tại sao.
- [ ] `IllegalArgumentException` tại `Set.of(...)`/`Map.of(...)`? → có giá trị/key trùng lặp
      trong tham số truyền vào.
- [ ] Cần hơn 10 cặp key-value bất biến? → dùng `Map.ofEntries(Map.entry(...), ...)` thay vì
      `Map.of(...)`.

## Source Code Walkthrough

`List.of(...)` (với số lượng tham số nhỏ, ≤10) sử dụng các overload chuyên biệt
(`List.of(E e1)`, `List.of(E e1, E e2)`...) trả về các implementation nội bộ tối ưu riêng cho
từng kích thước nhỏ (`List12` cho 1-2 phần tử, tránh tạo mảng thừa) — một tối ưu nhỏ nhưng thể
hiện sự chăm chút hiệu năng cho trường hợp phổ biến nhất (collection nhỏ). Với nhiều hơn 10
phần tử, dùng overload varargs chung (`List.of(E... elements)`), trả về một implementation dựa
trên mảng nội bộ, tương tự tinh thần `List.copyOf()` đã dùng ở
[Phase 1, Chapter 11](../phase-01-foundation/11-immutable.md).

## Summary

`List.of()`/`Set.of()`/`Map.of()` (Java 9+) tạo collection **bất biến thật sự** — bản sao độc
lập hoàn toàn, không phải view như `unmodifiableXxx()` (Chapter 15). Chúng áp dụng triết lý
fail-fast: từ chối `null` (`List.of`) và giá trị/key trùng lặp (`Set.of`/`Map.of`) **ngay lúc
tạo**, thay vì âm thầm chấp nhận như `ArrayList`/`HashSet`/`HashMap` thông thường — buộc lỗi lộ
ra sớm, gần nguyên nhân gốc, dễ debug hơn nhiều so với để nó lan truyền âm thầm. `Map.of()` giới
hạn tối đa 10 cặp; nhiều hơn cần `Map.ofEntries()`.

## Interview Questions

**Junior**

- `List.of(...)` khác `Collections.unmodifiableList()` như thế nào?
- Điều gì xảy ra nếu gọi `List.of("a", null)`?

**Mid**

- Vì sao `Set.of()`/`Map.of()` ném exception khi có trùng lặp, trong khi `HashSet`/`HashMap`
  âm thầm loại bỏ?
- `Map.of()` có giới hạn gì? Dùng gì khi cần nhiều hơn giới hạn đó?

**Senior**

- Giải thích triết lý "fail-fast" đứng sau việc `List.of()` chặn `null` ngay lúc tạo — so sánh
  với cách `ArrayList`/`unmodifiableList()` xử lý (hoặc không xử lý) cùng vấn đề.
- Một service crash ngay lúc khởi động sau khi cập nhật cấu hình, lỗi tại dòng `List.of(...)`.
  Đây là dấu hiệu tốt hay xấu về mặt thiết kế hệ thống? Giải thích quan điểm của bạn.

## Exercises

- [ ] Chạy lại đúng ví dụ `ImmutableCollDemo` ở trên, xác nhận cả ba loại lỗi.
- [ ] So sánh trực tiếp: tạo cùng một danh sách bằng
      `Collections.unmodifiableList(new ArrayList<>(List.of(...)))` và bằng `List.of(...)`
      trực tiếp — xác nhận cả hai đều chặn `add()`, nhưng chỉ cách thứ hai chặn được cả `null`
      ngay lúc tạo.
- [ ] Viết một `Map` bất biến với hơn 10 cặp key-value bằng `Map.ofEntries()`.

## Cheat Sheet

| API | Bất biến thật? | Chặn `null`? | Chặn trùng lặp? |
| --- | --- | --- | --- |
| `Collections.unmodifiableList(list)` | Không (view) | Không | — |
| `List.of(...)` | Có | Có | — |
| `Set.of(...)` | Có | Có | Có |
| `Map.of(...)` (≤10 cặp) | Có | Có | Có (key) |
| `Map.ofEntries(...)` (không giới hạn) | Có | Có | Có (key) |
| `List.copyOf(existing)` | Có | Có | — |

## References

- JEP 269: Convenience Factory Methods for Collections.
- Java SE API Documentation — `java.util.List#of`, `java.util.Set#of`, `java.util.Map#of`.
