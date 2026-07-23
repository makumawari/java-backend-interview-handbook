---
tags:
  - Java
  - IdentityHashMap
  - Collections
---

# IdentityHashMap

> Phase: Phase 2 — Collections
> Chapter slug: `identityhashmap`

## Metadata

```yaml
Chapter: IdentityHashMap
Phase: Phase 2 — Collections
Difficulty: ★★
Importance: ★★
Interview Frequency: 20%
Prerequisites:
  - Chapter 08 — HashMap
  - Phase 1, Chapter 12 — equals()
Used Later:
  - (Chapter cuối Phase 2 — không có chapter Phase 2 nào phụ thuộc trực tiếp)
Estimated Reading: 15 phút
Estimated Practice: 10 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

Đây là chapter cuối cùng của Phase 2 — và nó khép lại đúng bằng cách **đảo ngược** toàn bộ tinh
thần của 18 chapter trước. Từ Chapter 05 (`Set`), mọi cấu trúc dữ liệu "dựa trên hash" đều dùng
`equals()`/`hashCode()` (Phase 1, Chapter 12-13) để quyết định "hai phần tử có là một không".
`IdentityHashMap` phá vỡ quy tắc đó một cách triệt để:

```java
String a = new String("key");
String b = new String("key"); // NỘI DUNG giống hệt "a", nhưng là OBJECT KHÁC

Map<String, Integer> normal = new HashMap<>();
normal.put(a, 1);
normal.put(b, 2); // equals() đúng -> GHI ĐÈ lên entry của a
System.out.println(normal.size()); // 1

Map<String, Integer> identity = new IdentityHashMap<>();
identity.put(a, 1);
identity.put(b, 2); // a != b về THAM CHIẾU -> COI LÀ 2 KEY KHÁC NHAU
System.out.println(identity.size()); // 2
```

`HashMap` nói "giống nhau" (dựa trên `equals()`). `IdentityHashMap` nói "khác nhau" (dựa trên
`==`) — cho **cùng một cặp key**. Đây không phải một biến thể nhỏ của `HashMap` — nó là một
công cụ dùng cho một lớp bài toán hoàn toàn khác: khi bạn cần biết "đây có đúng là **cùng một
object trong bộ nhớ**" hay không, không quan tâm nội dung.

## Interview Question (Central)

> `IdentityHashMap` khác `HashMap` ở điểm nào? Khi nào cần dùng `==` thay vì `equals()` để so
> sánh key?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Biết `IdentityHashMap` dùng `==` (so sánh tham chiếu) thay vì `equals()`/`hashCode()`
      để xác định key
- [ ] Tự tay tái hiện được sự khác biệt giữa `HashMap` và `IdentityHashMap` cho cùng một cặp
      object
- [ ] Biết các use case chính đáng: phát hiện chu trình khi duyệt đồ thị object (ví dụ
      serialization), theo dõi object theo đúng định danh vật lý

## Prerequisites

- Chapter 08 — hiểu cơ chế bucket/hash của `HashMap`.
- Phase 1, Chapter 12 — hiểu `equals()` so sánh nội dung, `==` so sánh tham chiếu.

## Used Later

Đây là chapter cuối của Phase 2 — không có chapter Phase 2 nào phụ thuộc trực tiếp vào
`IdentityHashMap`. Kiến thức này sẽ hữu ích khi đọc source code JDK/framework (nhiều nơi dùng
`IdentityHashMap` cho bài toán phát hiện chu trình, ví dụ trong Jackson khi serialize object có
tham chiếu vòng).

## Problem

`equals()` (Phase 1, Chapter 12) so sánh **nội dung** — hai object khác nhau nhưng "trông giống
hệt nhau" được coi là một. Nhưng một số bài toán cần chính xác điều ngược lại: theo dõi từng
**object vật lý cụ thể**, bất kể nội dung của chúng giống hay khác nhau — ví dụ khi duyệt một
cấu trúc dữ liệu lồng nhau để phát hiện **chu trình** (object A tham chiếu tới B, B tham chiếu
ngược lại A): cần biết "object này tôi đã **thăm qua đúng nó** chưa", không phải "tôi đã thăm
một object có nội dung giống nó chưa".

## Concept

`IdentityHashMap` là một implementation `Map` **cố tình vi phạm** hợp đồng chung của interface
`Map` (Javadoc của chính nó nói rõ điều này) bằng cách dùng `==` (so sánh tham chiếu, định danh
object trong bộ nhớ) thay vì `equals()` để xác định hai key có "là một" hay không.

## Why?

Nếu dùng `HashMap` thường cho bài toán phát hiện chu trình: hai object khác nhau nhưng
"tình cờ" có nội dung giống hệt nhau (`equals()` trả về `true`) sẽ bị coi là "đã thăm", dù thực
tế chương trình chưa từng đi qua đúng object đó — dẫn tới bỏ sót việc duyệt một nhánh dữ liệu
hợp lệ. `IdentityHashMap` đảm bảo chỉ đúng **object đã thực sự đi qua** (cùng địa chỉ tham
chiếu) mới được coi là "đã thăm", bất kể có bao nhiêu object khác "trông giống hệt" nó.

## How?

```java
Map<Object, Boolean> visited = new IdentityHashMap<>();

void traverse(Object node) {
    if (visited.containsKey(node)) return; // đã thăm ĐÚNG OBJECT NÀY chưa, không phải "giống nó"
    visited.put(node, true);
    // ... duyệt tiếp các node liên quan
}
```

## Visualization

```
HashMap:           key được coi là "trùng" nếu   a.equals(b) == true
IdentityHashMap:    key được coi là "trùng" nếu   a == b  (CÙNG một object trong bộ nhớ)

new String("x")  ≠  new String("x")   theo == (hai object khác nhau)
new String("x")  =  new String("x")   theo equals() (nội dung giống nhau)

     HashMap:            coi 2 String này là "TRÙNG"     → size giảm còn 1
     IdentityHashMap:    coi 2 String này là "KHÁC NHAU"  → size vẫn là 2
```

## Example

Xác nhận trực tiếp — cùng một cặp object, hai kết luận trái ngược nhau tuỳ implementation:

```java
import java.util.HashMap;
import java.util.IdentityHashMap;
import java.util.Map;

public class IdentityDemo {
    public static void main(String[] args) {
        String a = new String("key");
        String b = new String("key"); // noi dung giong het "a" nhung khac object

        Map<String, Integer> normal = new HashMap<>();
        normal.put(a, 1);
        normal.put(b, 2); // equals() dung -> GHI DE len entry cua a
        System.out.println("HashMap thuong, size = " + normal.size());

        Map<String, Integer> identity = new IdentityHashMap<>();
        identity.put(a, 1);
        identity.put(b, 2); // a != b ve tham chieu -> COI LA 2 KEY KHAC NHAU
        System.out.println("IdentityHashMap, size = " + identity.size());
    }
}
```

Kết quả thật (JDK 17):

```
HashMap thuong, size = 1
IdentityHashMap, size = 2
```

Cùng thao tác `put(a, ...)` rồi `put(b, ...)` với `a`/`b` có nội dung y hệt nhau, hai
implementation kết luận hoàn toàn trái ngược — đúng bản chất cốt lõi của chapter này.

## Deep Dive

**Vì sao Javadoc chính thức của `IdentityHashMap` tự nhận là "vi phạm hợp đồng chung của
`Map`"?** Vì interface `Map` (Chapter 07) **ngầm định** rằng việc xác định key dựa trên
`equals()` — đây là một giả định mà hầu hết code làm việc với `Map` đều dựa vào một cách vô
thức. `IdentityHashMap` phá vỡ giả định đó một cách có chủ đích, và tự documentation rõ ràng
việc này để không ai dùng nhầm nó như một `HashMap` thông thường. Đây là một trường hợp hiếm
hoi trong JDK mà một class công khai thừa nhận "tôi cố tình không tuân theo hợp đồng chuẩn của
interface tôi implement" — một minh chứng cho việc đôi khi phá vỡ quy ước chung là lựa chọn
đúng đắn, miễn là được tài liệu hoá rõ ràng, tường minh.

## Engineering Insight

**`IdentityHashMap` được cài đặt khác `HashMap` như thế nào để đạt hiệu năng tốt cho việc so
sánh `==`?** Nó **không** dùng danh sách liên kết cho các bucket va chạm (khác `HashMap`,
Chapter 08) — thay vào đó dùng kỹ thuật **open addressing** (địa chỉ mở): key và value được
lưu **xen kẽ trực tiếp** trong cùng một mảng phẳng (`key1, value1, key2, value2, ...`), và khi
va chạm xảy ra, nó tìm ô trống **tiếp theo** trong chính mảng đó (linear probing) thay vì tạo
node phụ. Cách này tối ưu hơn cho trường hợp hash dựa trên `System.identityHashCode()` (định
danh bộ nhớ, thường phân bố khá đều) và tránh chi phí cấp phát object `Node` cho mỗi phần tử —
một thiết kế chuyên biệt, khác hẳn `HashMap`, phù hợp với đặc thù sử dụng hẹp của
`IdentityHashMap`.

## Historical Note

`IdentityHashMap` xuất hiện từ Java 1.4 (2002) — ra đời chủ yếu để phục vụ chính nhu cầu nội
bộ của JDK và các framework serialization/introspection (ví dụ phát hiện chu trình tham chiếu
khi serialize một object graph phức tạp) — không phải một cấu trúc dữ liệu hướng tới đại đa số
lập trình viên ứng dụng thông thường, phản ánh đúng qua Interview Frequency thấp nhất trong
toàn bộ Phase 2.

## Myth vs Reality

- **Myth:** "`IdentityHashMap` chỉ là một biến thể tối ưu hiệu năng của `HashMap`."
  **Reality:** Nó thay đổi hoàn toàn **ngữ nghĩa** so sánh key (`==` thay vì `equals()`), không
  phải chỉ là tối ưu hiệu năng — dùng nhầm nó cho mục đích của `HashMap` thông thường sẽ cho
  kết quả hoàn toàn sai (như Example đã chứng minh).

- **Myth:** "Tên gọi 'Identity' chỉ là chi tiết đặt tên, không ảnh hưởng nhiều tới hành vi."
  **Reality:** "Identity" ở đây có nghĩa chính xác là "định danh vật lý" (cùng một object trong
  bộ nhớ) — đây là toàn bộ bản chất của class, không phải chi tiết phụ.

## Common Mistakes

- **Dùng `IdentityHashMap` như một `HashMap` thông thường** mà không nhận ra khác biệt ngữ
  nghĩa — dẫn tới lỗi khó hiểu (hai key "giống hệt nhau về nội dung" bị coi là khác nhau).
- **Dùng `HashMap` thường cho bài toán cần phát hiện chu trình theo định danh object** — dẫn
  tới bỏ sót nhánh dữ liệu hợp lệ nếu tình cờ có hai object nội dung giống nhau (xem Why?).

## Best Practices

- Chỉ dùng `IdentityHashMap` khi bài toán thực sự cần so sánh theo **định danh object**
  (thường gặp nhất: phát hiện chu trình khi duyệt cấu trúc dữ liệu lồng nhau, hoặc theo dõi
  object theo đúng thực thể vật lý bất kể nội dung).
- Không dùng `IdentityHashMap` như một "HashMap nhanh hơn" cho mục đích chung — ngữ nghĩa khác
  hoàn toàn, không phải vấn đề hiệu năng.
- Luôn tài liệu hoá rõ ràng khi dùng `IdentityHashMap` trong code, vì hành vi của nó đủ khác
  thường để gây bối rối cho người đọc code sau này nếu không có ghi chú.

## Production Notes

Chapter này thuần công cụ chuyên biệt, hiếm khi tự nó là nguồn gốc trực tiếp của sự cố
production trong code ứng dụng thông thường — ứng dụng thực tế phổ biến nhất nằm sâu bên trong
các framework (serialization, dependency injection container theo dõi bean theo định danh vật
lý) hơn là trong code nghiệp vụ hàng ngày. Vấn đề thường gặp nhất (nếu có) là dùng **nhầm**
`IdentityHashMap` khi ý định thực sự là dùng `HashMap` — một lỗi cấu hình/thiết kế hơn là một
loại sự cố runtime điển hình.

## Debug Checklist

- [ ] `Map` cho kết quả "sai" — hai key trông giống hệt nhau nhưng bị coi là khác nhau (hoặc
      ngược lại)? → kiểm tra có đang nhầm dùng `IdentityHashMap` thay vì `HashMap`, hoặc ngược
      lại, so với ý định thiết kế ban đầu.

## Source Code Walkthrough

`IdentityHashMap.hash(key, length)` dùng `System.identityHashCode(key)` — một `native` method
(giống các method `native` của `Object` đã gặp ở
[Phase 1, Chapter 09](../phase-01-foundation/09-object.md)) trả về một mã định danh dựa trên
chính object (không thể override, không liên quan tới `hashCode()` mà class có thể tự định
nghĩa) — đây là bằng chứng cụ thể cho việc `IdentityHashMap` hoàn toàn tách biệt khỏi cơ chế
`equals()`/`hashCode()` mà lập trình viên có thể tự viết, dùng thẳng định danh nội tại của JVM.

## Summary

`IdentityHashMap` dùng `==` (so sánh tham chiếu, định danh object trong bộ nhớ) thay vì
`equals()`/`hashCode()` để xác định key — cố tình vi phạm hợp đồng chung của `Map`, được chính
Javadoc thừa nhận rõ ràng. Nó không phải biến thể hiệu năng của `HashMap`, mà là công cụ cho
một lớp bài toán khác hẳn: theo dõi object theo đúng định danh vật lý (phát hiện chu trình khi
duyệt cấu trúc dữ liệu lồng nhau là use case phổ biến nhất). Cài đặt bên trong dùng open
addressing (không phải danh sách liên kết như `HashMap`), lưu key/value xen kẽ trong một mảng
phẳng. Đây là công cụ chuyên biệt, hẹp — không nên dùng thay thế `HashMap` cho mục đích chung.

## Interview Questions

**Junior**

- `IdentityHashMap` khác `HashMap` ở điểm nào?
- `IdentityHashMap` so sánh key bằng gì?

**Mid**

- Cho một ví dụ cụ thể `IdentityHashMap` cho kết quả khác `HashMap` cho cùng một cặp key.
- Khi nào nên dùng `IdentityHashMap` thay vì `HashMap`?

**Senior**

- Giải thích vì sao Javadoc của `IdentityHashMap` tự nhận "vi phạm hợp đồng chung của Map" —
  đây có phải một thiết kế API tốt không? Phân tích quan điểm của bạn.
- Mô tả một bài toán thực tế (ví dụ serialization một object graph có chu trình) mà
  `IdentityHashMap` là công cụ đúng đắn, và giải thích vì sao `HashMap` thường không giải quyết
  được đúng bài toán đó.

## Exercises

- [ ] Chạy lại đúng ví dụ `IdentityDemo` ở trên, xác nhận kết quả giống hệt.
- [ ] Viết một hàm duyệt cấu trúc dữ liệu lồng nhau có khả năng chứa chu trình (ví dụ một danh
      sách các node tự tham chiếu vòng lại chính nó), dùng `IdentityHashMap` để phát hiện và
      tránh vòng lặp vô hạn khi duyệt.
- [ ] Thử thay `IdentityHashMap` bằng `HashMap` thường trong bài tập trên (giả sử các node có
      `equals()` so sánh nội dung) — tạo tình huống hai node khác nhau nhưng nội dung giống hệt
      nhau, quan sát hành vi duyệt bị sai như thế nào.

## Cheat Sheet

| | HashMap | IdentityHashMap |
| --- | --- | --- |
| So sánh key bằng | `equals()`/`hashCode()` | `==` / `System.identityHashCode()` |
| Cấu trúc bucket va chạm | Danh sách liên kết / cây | Open addressing (linear probing) |
| Tuân theo hợp đồng `Map` chuẩn? | Có | Không (tự nhận vi phạm) |
| Use case chính | Mục đích chung | Phát hiện chu trình, theo dõi định danh object |

## References

- Java SE API Documentation — `java.util.IdentityHashMap`.
- `java.lang.System#identityHashCode` — Java SE API Documentation.
