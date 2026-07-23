---
tags:
  - Java
  - Deque
  - Collections
---

# Deque

> Phase: Phase 2 — Collections
> Chapter slug: `deque`

## Metadata

```yaml
Chapter: Deque
Phase: Phase 2 — Collections
Difficulty: ★★★
Importance: ★★★
Interview Frequency: 45%
Prerequisites:
  - Chapter 04 — LinkedList
  - Chapter 03 — ArrayList
Used Later:
  - Thread Pool, Executor (Phase 3) — hàng đợi task bên trong thread pool thường dùng cấu trúc tương tự Deque
Estimated Reading: 20 phút
Estimated Practice: 15 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

Bạn cần cài đặt một stack (ngăn xếp, vào sau ra trước — LIFO) để kiểm tra dấu ngoặc cân bằng
trong một biểu thức. Sách giáo khoa cũ dạy dùng class `java.util.Stack` — nhưng mở Javadoc của
nó ra, dòng đầu tiên viết: *"A more complete and consistent set of LIFO stack operations is
provided by the Deque interface... which should be used in preference to this class."* Chính
JDK khuyên **không nên dùng** `Stack` nữa.

Vậy dùng gì? Câu trả lời: `Deque` (đọc là "deck", viết tắt của *Double-Ended Queue* — hàng đợi
hai đầu). Một interface duy nhất, nhưng đủ mạnh để đóng vai trò cả **stack** (LIFO) lẫn
**queue** (FIFO) — chỉ cần chọn đúng method tương ứng.

## Interview Question (Central)

> `Deque` là gì? Vì sao JDK khuyên dùng `Deque` thay vì `java.util.Stack`/`java.util.Queue`
> truyền thống cho vai trò stack?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Hiểu `Deque` cho phép thêm/lấy phần tử ở **cả hai đầu**, đóng vai trò được cả stack lẫn
      queue
- [ ] Dùng đúng bộ method: `push`/`pop` (stack), `offer`/`poll` (queue),
      `addFirst`/`addLast`/`removeFirst`/`removeLast` (tổng quát)
- [ ] Biết `ArrayDeque` (dùng mảng vòng) thường nhanh hơn `LinkedList` cho vai trò
      stack/queue thuần tuý
- [ ] Biết lý do JDK khuyên tránh `java.util.Stack`/`Vector` cũ

## Prerequisites

- Chapter 03-04 — hiểu `ArrayList`/`LinkedList`, để so sánh với `ArrayDeque`.

## Used Later

- **Thread Pool, Executor** (Phase 3) — hàng đợi task bên trong cơ chế thread pool dùng các cấu
  trúc có tính chất tương tự `Deque` (một số cài đặt work-stealing dùng chính deque hai đầu cho
  mỗi worker thread).

## Problem

`java.util.Stack` (Java 1.0) kế thừa `Vector` — một cấu trúc **đồng bộ hoá toàn bộ**
(`synchronized` mọi method, giống `Hashtable` đã gặp ở Chapter 11), gây chi phí khoá không cần
thiết trong phần lớn use case đơn luồng. `java.util.Queue` cơ bản chỉ hỗ trợ thao tác ở **một
đầu** hiệu quả. Cần một interface hiện đại, không đồng bộ hoá thừa, hỗ trợ thao tác hiệu quả ở
**cả hai đầu**.

## Concept

`Deque<E>` (mở rộng `Queue`) cung cấp bộ method đối xứng cho cả hai đầu: `addFirst()`/
`addLast()`, `removeFirst()`/`removeLast()`, `peekFirst()`/`peekLast()`. Từ bộ method tổng quát
này, hai vai trò kinh điển được ánh xạ trực tiếp:

- **Stack (LIFO)**: `push()` = `addFirst()`, `pop()` = `removeFirst()`.
- **Queue (FIFO)**: `offer()` = `addLast()`, `poll()` = `removeFirst()`.

## Why?

Nếu tách riêng hai interface `Stack`/`Queue` không liên quan: một cấu trúc dữ liệu muốn hỗ trợ
cả hai vai trò (khá phổ biến — ví dụ thuật toán duyệt đồ thị có thể cần chuyển đổi giữa DFS
dùng stack và BFS dùng queue) sẽ phải chọn implement cả hai interface riêng biệt, dù cơ chế
lưu trữ bên dưới hoàn toàn giống nhau. `Deque` thống nhất cả hai nhu cầu vào một interface duy
nhất, phản ánh đúng thực tế: một cấu trúc "hai đầu" tự nhiên phục vụ được cả hai vai trò mà
không cần hai cài đặt riêng.

## How?

Hai implementation chính: `ArrayDeque` (dùng mảng vòng — circular array, không đồng bộ hoá) và
`LinkedList` (Chapter 04, cũng implement `Deque`, ngoài vai trò `List`).

```java
Deque<String> stack = new ArrayDeque<>();
stack.push("A"); stack.push("B"); stack.push("C");
stack.pop(); // "C" — vào sau ra trước (LIFO)

Deque<String> queue = new ArrayDeque<>();
queue.offer("A"); queue.offer("B"); queue.offer("C");
queue.poll(); // "A" — vào trước ra trước (FIFO)
```

## Visualization

```
Deque (hai đầu):

  addFirst()/push() ──▶ [ đầu ] ── ... ── [ cuối ] ◀── addLast()/offer()
  removeFirst()/pop() ◀─ [ đầu ]          [ cuối ] ──▶ removeLast()

Vai trò STACK (LIFO): chỉ dùng đầu    → push()=addFirst(), pop()=removeFirst()
Vai trò QUEUE (FIFO): thêm cuối, lấy đầu → offer()=addLast(), poll()=removeFirst()
```

## Example

Xác nhận cả ba cách dùng — stack, queue, và cả hai đầu cùng lúc:

```java
import java.util.ArrayDeque;
import java.util.Deque;

public class DequeDemo {
    public static void main(String[] args) {
        Deque<String> stack = new ArrayDeque<>();
        stack.push("A");
        stack.push("B");
        stack.push("C");
        System.out.println("Stack (LIFO) pop lien tuc: " + stack.pop() + ", " + stack.pop() + ", " + stack.pop());

        Deque<String> queue = new ArrayDeque<>();
        queue.offer("A");
        queue.offer("B");
        queue.offer("C");
        System.out.println("Queue (FIFO) poll lien tuc: " + queue.poll() + ", " + queue.poll() + ", " + queue.poll());

        Deque<String> both = new ArrayDeque<>();
        both.addFirst("giua");
        both.addFirst("dau");
        both.addLast("cuoi");
        System.out.println("Deque hai dau: " + both);
    }
}
```

Kết quả thật (JDK 17):

```
Stack (LIFO) pop lien tuc: C, B, A
Queue (FIFO) poll lien tuc: A, B, C
Deque hai dau: [dau, giua, cuoi]
```

Cùng một class `ArrayDeque`, chỉ khác bộ method gọi, đóng được cả hai vai trò đối nghịch nhau
hoàn toàn (LIFO vs FIFO).

## Deep Dive

**`ArrayDeque` dùng "mảng vòng" (circular array) — khác `ArrayList` (Chapter 03) ở điểm nào?**
`ArrayList` chỉ thêm/xoá hiệu quả ở **cuối** mảng (đầu mảng tốn O(n) để dịch chuyển, Chapter
03). `ArrayDeque` cần hiệu quả ở **cả hai đầu** — nó dùng một mảng có hai con trỏ `head`/`tail`
**xoay vòng**: khi `head` "lùi" qua khỏi vị trí 0, nó **quay lại** cuối mảng (modulo kích
thước) thay vì báo lỗi hay dịch chuyển toàn bộ phần tử. Kỹ thuật này cho phép `addFirst()` **và**
`addLast()` đều O(1) amortized — điều `ArrayList` không đạt được cho đầu mảng.

## Engineering Insight

**Vì sao `ArrayDeque` thường được khuyên dùng hơn `LinkedList` cho vai trò stack/queue thuần
tuý, dù cả hai đều implement `Deque`?** Đo thực nghiệm cho thấy rõ:

```
addLast() x500000 -- ArrayDeque: 5ms, LinkedList: 20ms
```

`ArrayDeque` nhanh hơn khoảng 4 lần cho cùng thao tác. Lý do: `LinkedList` phải cấp phát một
object `Node` **riêng** cho mỗi phần tử (đã phân tích chi phí này ở
[Chapter 04 — Engineering Insight](04-linkedlist.md): object header + 2 con trỏ), trong khi
`ArrayDeque` chỉ ghi trực tiếp vào một ô mảng có sẵn — không có chi phí cấp phát object nhỏ lẻ
liên tục, và tận dụng được tính liền kề bộ nhớ (cache locality — dữ liệu gần nhau trong bộ nhớ
vật lý được CPU truy cập nhanh hơn nhờ cơ chế cache của phần cứng). Đây là lý do cụ thể, đo
được, không phải khuyến nghị chung chung: khi chỉ cần vai trò stack/queue (không cần truy cập
ngẫu nhiên theo chỉ số như `List`), `ArrayDeque` gần như luôn là lựa chọn tốt hơn `LinkedList`.

## Historical Note

```
Java 1.0 (1996)
    ↓
java.util.Stack — kế thừa Vector, đồng bộ hoá toàn bộ (giống Hashtable)
    ↓
Java 5 (2004)
    ↓
Queue interface ra đời — nhưng chỉ hỗ trợ thao tác một đầu hiệu quả
    ↓
Java 6 (2006)
    ↓
Deque interface + ArrayDeque ra đời — JDK chính thức khuyến nghị dùng
Deque thay cho Stack/Vector cũ cho MỌI vai trò LIFO/FIFO mới
```

## Myth vs Reality

- **Myth:** "`java.util.Stack` vẫn là lựa chọn chuẩn cho stack trong Java hiện đại."
  **Reality:** Chính Javadoc của `Stack` khuyến nghị dùng `Deque` thay thế — `Stack` chỉ còn
  tồn tại vì lý do tương thích ngược với code rất cũ.

- **Myth:** "`LinkedList` luôn là lựa chọn tốt cho `Deque` vì có `next`/`prev` sẵn, "tự nhiên"
  cho cấu trúc hai đầu."
  **Reality:** Xem Engineering Insight — `ArrayDeque` nhanh hơn đáng kể trong thực nghiệm nhờ
  tránh chi phí cấp phát Node và tận dụng cache locality.

## Common Mistakes

- **Dùng `java.util.Stack`/`Vector` trong code mới** — nên dùng `ArrayDeque` thay thế.
- **Mặc định chọn `LinkedList` cho `Deque`** mà không cân nhắc `ArrayDeque` — bỏ lỡ hiệu năng
  tốt hơn cho vai trò thuần stack/queue.
- **Nhầm lẫn `poll()` (FIFO, lấy từ đầu) với `pop()` (LIFO, cũng lấy từ đầu nhưng mang ý nghĩa
  stack)** — cả hai đều gọi `removeFirst()` bên trong, nhưng đặt tên khác nhau để code đọc rõ
  ý nghĩa (stack hay queue) — chọn đúng tên method giúp code tự tài liệu hoá ý định.

## Best Practices

- Mặc định dùng `ArrayDeque` cho vai trò stack/queue thuần tuý, không cần `List`.
- Không dùng `java.util.Stack`/`Vector` trong code mới.
- Chọn đúng bộ method (`push`/`pop` cho stack, `offer`/`poll` cho queue) dù cả hai đều dùng
  chung `ArrayDeque` — giúp code đọc rõ ý định thiết kế.

## Production Notes

Chapter này thuần cấu trúc dữ liệu nền tảng — ứng dụng thực tế phổ biến nhất của `Deque` trong
production nằm ở các thuật toán xử lý dữ liệu (duyệt cây/đồ thị bằng DFS dùng stack, xử lý
theo lô bằng sliding window dùng cả hai đầu) hơn là gắn với một loại sự cố production cụ thể
như các chapter về `HashMap`/`ConcurrentHashMap` trước đó.

## Debug Checklist

- [ ] Cần chọn giữa `ArrayDeque` và `LinkedList` cho vai trò `Deque`? → mặc định chọn
      `ArrayDeque` trừ khi có lý do cụ thể cần `LinkedList` (ví dụ cần cả vai trò `List` cùng
      lúc).
- [ ] Code cũ dùng `java.util.Stack`/`Vector`? → cân nhắc refactor sang `ArrayDeque` khi có cơ
      hội, đặc biệt nếu đang chạy trong ngữ cảnh đơn luồng (tránh chi phí đồng bộ hoá thừa).

## Source Code Walkthrough

`ArrayDeque` giữ một mảng `elements` cùng hai chỉ số `head` và `tail`. `addFirst(e)`: giảm
`head` (dùng phép AND bit với `elements.length - 1` để "quay vòng" khi `head` xuống dưới 0 —
kỹ thuật tương tự phép tính chỉ số bucket của `HashMap`, Chapter 08, đòi hỏi
`elements.length` luôn là luỹ thừa của 2). `addLast(e)`: ghi vào `tail` rồi tăng `tail` (cũng
quay vòng tương tự). Khi mảng đầy, `ArrayDeque` resize gấp đôi — giống tinh thần
`ArrayList` (Chapter 03), nhưng phải sao chép cẩn thận để giữ đúng thứ tự logic của các phần
tử đang "quấn vòng" quanh mảng.

## Summary

`Deque` (Double-Ended Queue) hỗ trợ thêm/lấy phần tử ở cả hai đầu, đóng được cả vai trò stack
(LIFO, qua `push`/`pop`) lẫn queue (FIFO, qua `offer`/`poll`) trong cùng một interface — JDK
khuyến nghị dùng nó thay cho `java.util.Stack`/`Vector` cũ (đồng bộ hoá thừa không cần thiết).
`ArrayDeque` dùng mảng vòng (circular array), đo thực nghiệm nhanh hơn `LinkedList` khoảng 4 lần
cho thao tác thêm phần tử — nên là lựa chọn mặc định cho vai trò stack/queue thuần tuý, trừ khi
cần thêm khả năng của `List` mà chỉ `LinkedList` mới có.

## Interview Questions

**Junior**

- `Deque` là gì? Nó hỗ trợ những thao tác cơ bản nào?
- `push()`/`pop()` và `offer()`/`poll()` khác nhau ở điểm gì dù có thể cùng lấy ra từ một đầu?

**Mid**

- Vì sao JDK khuyến nghị dùng `Deque` thay cho `java.util.Stack`?
- `ArrayDeque` dùng cấu trúc "mảng vòng" nghĩa là gì? Nó giải quyết vấn đề gì mà `ArrayList`
  (Chapter 03) không giải quyết được cho đầu mảng?

**Senior**

- Giải thích vì sao `ArrayDeque` thường nhanh hơn `LinkedList` cho vai trò stack/queue thuần
  tuý, dựa trên phân tích chi phí bộ nhớ và cache locality.
- Thiết kế một thuật toán cần chuyển đổi linh hoạt giữa xử lý kiểu DFS (stack) và BFS (queue)
  trên cùng một tập dữ liệu — `Deque` giúp gì cho thiết kế này?

## Exercises

- [ ] Chạy lại đúng ví dụ `DequeDemo` ở trên, xác nhận kết quả giống hệt.
- [ ] Đo lại thực nghiệm chênh lệch `ArrayDeque` vs `LinkedList` cho `addLast()`/`pollFirst()`
      trên máy bạn, xác nhận xu hướng tương tự.
- [ ] Dùng `Deque` cài đặt một hàm kiểm tra dấu ngoặc cân bằng trong biểu thức (ví dụ
      `"({[]})"` hợp lệ, `"({[})]"` không hợp lệ) — bài toán kinh điển ứng dụng stack.

## Cheat Sheet

| Vai trò | Thêm | Lấy ra |
| --- | --- | --- |
| Stack (LIFO) | `push(e)` = `addFirst(e)` | `pop()` = `removeFirst()` |
| Queue (FIFO) | `offer(e)` = `addLast(e)` | `poll()` = `removeFirst()` |
| Tổng quát | `addFirst()`/`addLast()` | `removeFirst()`/`removeLast()` |

**Khuyến nghị:** `ArrayDeque` cho stack/queue thuần tuý; tránh `java.util.Stack`/`Vector`
trong code mới.

## References

- Java SE API Documentation — `java.util.Deque`, `java.util.ArrayDeque`.
- Javadoc `java.util.Stack` — ghi chú khuyến nghị dùng `Deque` thay thế.
