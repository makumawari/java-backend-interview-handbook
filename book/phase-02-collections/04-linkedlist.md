---
tags:
  - Java
  - LinkedList
  - Collections
---

# LinkedList

> Phase: Phase 2 — Collections
> Chapter slug: `linkedlist`

## Metadata

```yaml
Chapter: LinkedList
Phase: Phase 2 — Collections
Difficulty: ★★★
Importance: ★★★
Interview Frequency: 65%
Prerequisites:
  - Chapter 03 — ArrayList
Used Later:
  - Deque (Chapter 13) — LinkedList cũng implement Deque, dùng được như stack/queue
Estimated Reading: 20 phút
Estimated Practice: 20 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Chapter 03](03-arraylist.md) — `add(index, e)`/`remove(index)` ở giữa/đầu
> `ArrayList` tốn O(n) vì phải dịch chuyển phần tử trong mảng.

Nghe có vẻ `LinkedList` sẽ luôn thắng `ArrayList` cho những thao tác này — "không cần dịch
chuyển gì cả, chỉ cần nối lại vài con trỏ". Nhiều lập trình viên vì lý do này chọn `LinkedList`
"để an toàn" cho mọi trường hợp cần chèn/xoá nhiều. Nhưng thử đo thật:

```
get(giua) x20000  -- ArrayList: 0ms,   LinkedList: 883ms
add(0, x) x20000  -- ArrayList: 88ms,  LinkedList: 0ms
```

`LinkedList` thắng tuyệt đối ở `add(0, x)` — đúng như kỳ vọng. Nhưng thua **thảm hại** ở
`get()` ngay giữa danh sách — chậm hơn `ArrayList` tới hàng trăm lần. Đây không phải một chi
tiết vụn vặt: nó là bằng chứng cho thấy "LinkedList nhanh hơn cho chèn/xoá" chỉ đúng **một nửa
sự thật**. Chapter này giải thích đầy đủ cả hai nửa.

## Interview Question (Central)

> `ArrayList` và `LinkedList` khác nhau như thế nào về cấu trúc bên trong? Khi nào thực sự nên
> chọn `LinkedList` thay vì mặc định dùng `ArrayList`?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Giải thích cấu trúc **doubly-linked list** (danh sách liên kết đôi) đứng sau `LinkedList`
- [ ] Tự tay đo được bằng thực nghiệm: `LinkedList` chậm hơn hẳn `ArrayList` khi truy cập ngẫu
      nhiên, nhanh hơn hẳn khi chèn/xoá ở hai đầu
- [ ] Biết `LinkedList` cũng implement `Deque` (Chapter 13), dùng được như stack/queue
- [ ] Đưa ra được quyết định đúng: chọn `ArrayList` hay `LinkedList` dựa trên **loại thao tác
      chiếm ưu thế**, không dựa trên trực giác chung chung

## Prerequisites

- Chapter 03 — đã hiểu cơ chế mảng nội bộ và độ phức tạp từng thao tác của `ArrayList`.

## Used Later

- **Deque** (Chapter 13) — `LinkedList` implement cả `List` lẫn `Deque`, dùng được như hàng
  đợi hai đầu (stack/queue) — chapter đó sẽ so sánh `LinkedList` với `ArrayDeque` cho đúng vai
  trò này.

## Problem

`ArrayList` (Chapter 03) trả giá O(n) cho `add(index, e)`/`remove(index)` ở giữa/đầu vì dữ liệu
phải nằm **liền kề vật lý** trong một mảng. Cần một cấu trúc dữ liệu **không** yêu cầu tính
liền kề — cho phép chèn/xoá ở bất kỳ vị trí nào mà không phải dịch chuyển hàng loạt phần tử
khác.

## Concept

`LinkedList` là một **doubly-linked list**: mỗi phần tử được bọc trong một **Node** riêng, chứa
dữ liệu và hai con trỏ — `next` (trỏ tới Node kế tiếp) và `prev` (trỏ tới Node trước đó). Các
Node **không cần nằm liền kề trong bộ nhớ** — mỗi Node có thể nằm rải rác bất kỳ đâu trên Heap
(Phase 1, Chapter 04), liên kết với nhau hoàn toàn qua con trỏ.

```java
class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;
}
```

## Why?

Chèn một phần tử mới vào giữa danh sách chỉ cần tạo một Node mới và **nối lại 4 con trỏ** (2
con trỏ của Node mới, và 2 con trỏ của hai Node lân cận) — O(1), không đụng tới bất kỳ Node nào
khác. Đây chính là điều `ArrayList` không làm được (phải dịch chuyển toàn bộ phần tử phía sau).
Đổi lại: vì các Node rải rác trong bộ nhớ, không có cách nào "nhảy thẳng" tới phần tử thứ k như
tính địa chỉ trực tiếp trên mảng — phải **đi bộ** từng Node một từ đầu (hoặc cuối, nếu gần cuối
hơn) cho tới khi tới đúng vị trí, đây chính là nguồn gốc chi phí O(n) cho truy cập ngẫu nhiên.

## How?

```
add(index, e) trên LinkedList:
1. Đi bộ (traverse) từ đầu HOẶC cuối (tuỳ index gần đầu hay gần cuối hơn)
   tới đúng vị trí — O(n)
2. Tạo Node mới
3. Nối lại 4 con trỏ (next/prev của Node mới VÀ của 2 Node lân cận) — O(1)

add(0, e) hoặc add(size, e) (HAI ĐẦU):
   Bỏ qua bước đi bộ (đã biết chính xác đầu/cuối) → CHỈ O(1)
```

## Visualization

```
ArrayList (mảng liền kề):
  [A][B][C][D][E]     get(2) = tính địa chỉ trực tiếp → O(1)
   0  1  2  3  4        chèn giữa → dịch chuyển C,D,E → O(n)

LinkedList (Node rải rác, nối bằng con trỏ):
  head ⇄ [A] ⇄ [B] ⇄ [C] ⇄ [D] ⇄ [E] ⇄ tail
                                          get(2) = ĐI BỘ từ head qua A,B rồi tới C → O(n)
         chèn giữa B-C: tạo Node mới,
         nối 4 con trỏ, KHÔNG đụng A,D,E → O(1)
```

## Example

Đo thật chênh lệch hiệu năng cho hai loại thao tác — truy cập ngẫu nhiên vs. chèn ở đầu:

```java
import java.util.*;

public class ListPerfDemo {
    public static void main(String[] args) {
        int n = 50_000;
        List<Integer> arrayList = new ArrayList<>();
        List<Integer> linkedList = new LinkedList<>();
        for (int i = 0; i < n; i++) { arrayList.add(i); linkedList.add(i); }

        long t1 = time(() -> { for (int i = 0; i < 20000; i++) arrayList.get(arrayList.size() / 2); });
        long t2 = time(() -> { for (int i = 0; i < 20000; i++) linkedList.get(linkedList.size() / 2); });
        System.out.println("get(giua) x20000 -- ArrayList: " + t1 + "ms, LinkedList: " + t2 + "ms");

        long t3 = time(() -> { for (int i = 0; i < 20000; i++) arrayList.add(0, i); });
        long t4 = time(() -> { for (int i = 0; i < 20000; i++) linkedList.add(0, i); });
        System.out.println("add(0, x) x20000 -- ArrayList: " + t3 + "ms, LinkedList: " + t4 + "ms");
    }

    static long time(Runnable r) {
        long start = System.nanoTime();
        r.run();
        return (System.nanoTime() - start) / 1_000_000;
    }
}
```

Kết quả thật (JDK 17):

```
get(giua) x20000 -- ArrayList: 0ms, LinkedList: 883ms
add(0, x) x20000 -- ArrayList: 88ms, LinkedList: 0ms
```

Hai thái cực đối nghịch hoàn toàn: `ArrayList` gần như tức thì cho `get()` nhưng tốn 88ms cho
`add(0, ...)`; `LinkedList` gần như tức thì cho `add(0, ...)` nhưng tốn tới 883ms cho `get()`
giữa danh sách — chênh lệch hơn 10 lần so với cả `add(0)` chậm nhất của `ArrayList`. Không có
cấu trúc nào "nhanh hơn" một cách tuyệt đối — chỉ có "nhanh hơn cho đúng loại thao tác".

## Deep Dive

**`LinkedList.get(index)` tối ưu bằng cách chọn đi bộ từ đầu hay từ cuối** — không phải lúc nào
cũng đi từ `head`. Nếu `index < size / 2`, nó đi bộ từ `head`; nếu `index >= size / 2`, nó đi
bộ từ `tail` (ngược lại) — tối ưu này giảm một nửa số bước đi bộ trung bình, nhưng **không đổi
độ phức tạp tiệm cận**: vẫn là O(n), chỉ là hằng số ẩn phía trước nhỏ hơn 2 lần. Đây là lý do dù
có tối ưu này, `LinkedList.get()` vẫn chậm hơn `ArrayList.get()` hàng trăm lần trong Example —
O(n) dù được tối ưu vẫn luôn thua xa O(1) khi n đủ lớn.

## Engineering Insight

**Cái giá bộ nhớ ẩn của `LinkedList` mà benchmark thời gian không thể hiện.** Mỗi phần tử
trong `ArrayList` chỉ tốn đúng một ô trong mảng (8 byte cho một tham chiếu trên JVM 64-bit
thông thường). Mỗi phần tử trong `LinkedList` cần một object `Node` **riêng** — ngoài dữ liệu
thật, còn tốn thêm bộ nhớ cho **header object** (mọi object Java đều có overhead vài byte cho
metadata, liên hệ [Phase 1 Chapter 04](../phase-01-foundation/04-runtime-data-areas.md)) cộng
với **hai con trỏ** `next`/`prev`. Với danh sách rất lớn, `LinkedList` có thể tốn bộ nhớ gấp
3-4 lần so với `ArrayList` chứa cùng dữ liệu — một chi phí thường bị bỏ qua khi chỉ so sánh tốc
độ thao tác, nhưng rất thật trong production khi làm việc với dữ liệu lớn (liên hệ trực tiếp
[Phase 1 Chapter 07 — Garbage Collection](../phase-01-foundation/07-garbage-collection.md):
nhiều object nhỏ rải rác cũng tạo áp lực lớn hơn lên GC so với một mảng liền khối).

## Historical Note

`LinkedList` tồn tại từ Java 2 (1998). Trong nhiều năm, nó cũng implement `Queue` — Java 6
(2006) bổ sung để implement thêm `Deque` (Chapter 13), chính thức biến nó thành một lựa chọn
cho cả vai trò stack/queue hai đầu, không chỉ đơn thuần là một `List`.

## Myth vs Reality

- **Myth:** "`LinkedList` luôn nhanh hơn `ArrayList` cho thao tác chèn/xoá."
  **Reality:** Chỉ đúng cho chèn/xoá ở **hai đầu** (O(1)). Chèn/xoá ở **giữa** vẫn cần đi bộ
  O(n) để tìm tới vị trí trước — không nhanh hơn `ArrayList` một cách rõ rệt như trực giác
  thường nghĩ.

- **Myth:** "`LinkedList` là lựa chọn 'an toàn hơn' khi không chắc thao tác nào sẽ chiếm ưu
  thế."
  **Reality:** Trong thực tế, phần lớn ứng dụng đọc (truy cập ngẫu nhiên) nhiều hơn hẳn ghi ở
  giữa danh sách — `ArrayList` gần như luôn là lựa chọn mặc định tốt hơn trừ khi có bằng chứng
  cụ thể (đo đạc thật) cho thấy ngược lại.

## Common Mistakes

- **Chọn `LinkedList` "cho chắc" mà không đo đạc thực tế loại thao tác nào chiếm ưu thế trong
  ứng dụng** — dẫn tới hiệu năng tệ hơn nếu ứng dụng thực chất đọc nhiều hơn ghi.
- **Dùng vòng lặp `for (int i = 0; i < list.size(); i++) list.get(i)`** trên `LinkedList` —
  mỗi lần `get(i)` đều đi bộ lại từ đầu/cuối, biến một vòng lặp O(n) tưởng như đơn giản thành
  O(n²) thực tế. Nên dùng `Iterator`/for-each (đi bộ tuần tự một lần duy nhất) thay vì
  `get(index)` lặp lại trên `LinkedList`.

## Best Practices

- Mặc định chọn `ArrayList` trừ khi có bằng chứng đo đạc cụ thể cho thấy `LinkedList` (hoặc
  `ArrayDeque`, Chapter 13) phù hợp hơn.
- Nếu dùng `LinkedList`, luôn duyệt bằng `Iterator`/for-each, không dùng `get(index)` lặp lại
  trong vòng lặp.
- Khi cần vai trò stack/queue thuần tuý (không cần truy cập ngẫu nhiên), cân nhắc
  `ArrayDeque` (Chapter 13) — thường nhanh hơn `LinkedList` cho vai trò này nhờ dùng mảng vòng
  (circular array) thay vì Node rời rạc.

## Production Notes

**Vấn đề:** một đoạn code xử lý danh sách bằng vòng lặp `for (int i = 0; i < list.size(); i++)`
chạy nhanh trên dữ liệu test nhỏ, nhưng chậm bất thường (gần như "đứng hình") khi dữ liệu
production lớn hơn nhiều.

- **Triệu chứng:** thời gian xử lý tăng **phi tuyến** (nhanh hơn nhiều so với tuyến tính) theo
  kích thước dữ liệu — dấu hiệu đặc trưng của việc vô tình biến một thuật toán O(n) thành
  O(n²).
- **Root cause:** biến `list` thực chất là một `LinkedList` (có thể do code cũ, hoặc do một
  API trả về `LinkedList` thay vì `ArrayList`), và vòng lặp dùng `list.get(i)` — mỗi lần gọi
  đều tốn O(n) đi bộ, nhân với n lần lặp thành O(n²).
- **Debug:** kiểm tra kiểu thực sự của `list` (không chỉ nhìn khai báo biến — có thể khai báo
  là `List` nhưng object thật là `LinkedList`); profiling sẽ cho thấy phần lớn thời gian nằm
  trong việc đi bộ Node.
- **Solution:** đổi vòng lặp sang dùng `Iterator`/for-each (không dùng `get(index)`), hoặc đổi
  cấu trúc dữ liệu sang `ArrayList` nếu truy cập ngẫu nhiên là thao tác chính.
- **Prevention:** luôn ưu tiên for-each thay vì vòng lặp `get(index)` theo chỉ số cho `List`
  nói chung, trừ khi chắc chắn implementation là `ArrayList` và cần chỉ số vì lý do khác.

## Debug Checklist

- [ ] Vòng lặp duyệt `List` chậm bất thường, tăng phi tuyến theo kích thước dữ liệu? → kiểm
      tra implementation thật có phải `LinkedList` không, và có đang dùng `get(index)` lặp lại
      không.
- [ ] Cần chọn giữa `ArrayList`/`LinkedList` mà không chắc? → đo đạc thật loại thao tác chiếm
      ưu thế (đọc ngẫu nhiên hay chèn/xoá hai đầu), đừng dựa vào trực giác.

## Source Code Walkthrough

```
LinkedList.get(index)
    ↓
checkElementIndex(index) — kiểm tra hợp lệ
    ↓
node(index):
    if (index < (size >> 1)):
        đi bộ từ FIRST, theo next, index bước
    else:
        đi bộ từ LAST, theo prev, (size - 1 - index) bước
    ↓
trả về node.item
```

## Summary

`LinkedList` là doubly-linked list — mỗi phần tử một Node riêng, liên kết qua con trỏ
`next`/`prev`, không yêu cầu liền kề vật lý trong bộ nhớ. Chèn/xoá ở **hai đầu** là O(1) (không
cần dịch chuyển gì); nhưng truy cập ngẫu nhiên (`get(index)`) là O(n) vì phải **đi bộ** từng
Node — đã đo thực nghiệm chênh lệch hơn 800 lần so với `ArrayList` cho thao tác này. Chi phí bộ
nhớ mỗi phần tử cũng cao hơn hẳn `ArrayList` (mỗi Node cần thêm object header + 2 con trỏ).
Không có lựa chọn "tốt hơn tuyệt đối" — quyết định phải dựa trên loại thao tác thực sự chiếm ưu
thế trong ứng dụng, lý tưởng nhất là đo đạc thật thay vì dựa vào trực giác chung chung.

## Interview Questions

**Junior**

- `LinkedList` dùng cấu trúc dữ liệu gì bên dưới? Khác `ArrayList` ở điểm cơ bản nào?
- `add(0, e)` trên `LinkedList` có độ phức tạp gì?

**Mid**

- Giải thích vì sao `get(index)` trên `LinkedList` là O(n), dù nó có tối ưu đi bộ từ đầu hoặc
  cuối tuỳ vị trí gần bên nào hơn.
- Cho một ví dụ cụ thể code vô tình biến O(n) thành O(n²) khi dùng sai cách với `LinkedList`.

**Senior**

- ArrayList vs LinkedList: khi nào dùng mỗi loại? Trả lời dựa trên phân tích độ phức tạp từng
  thao tác, không chỉ dựa trên "cảm giác chung chung".
- Phân tích chi phí bộ nhớ ẩn của `LinkedList` so với `ArrayList` — trong bối cảnh nào chi phí
  này thực sự đáng kể trong production?

## Exercises

- [ ] Chạy lại đúng ví dụ `ListPerfDemo` ở trên, xác nhận chênh lệch tương tự trên máy bạn.
- [ ] Viết vòng lặp `for (int i = 0; i < list.size(); i++) list.get(i)` trên một `LinkedList`
      50.000 phần tử, đo thời gian, so sánh với duyệt bằng for-each — xác nhận chênh lệch rất
      lớn, tự giải thích vì sao (gợi ý: O(n) so với O(n²)).
- [ ] Đo thêm thao tác `remove(size / 2)` (xoá giữa danh sách) cho cả hai cấu trúc — dự đoán
      trước kết quả dựa trên lý thuyết đã học, rồi xác nhận bằng thực nghiệm.

## Cheat Sheet

| Thao tác | ArrayList | LinkedList |
| --- | --- | --- |
| `get(index)` | O(1) | O(n) |
| `add(e)` (cuối) | O(1) amortized | O(1) |
| `add(0, e)` / `remove(0)` (đầu) | O(n) | O(1) |
| `add(index, e)` (giữa) | O(n) | O(n) (đi bộ) + O(1) (nối con trỏ) |
| Bộ nhớ mỗi phần tử | Thấp (chỉ 1 ô mảng) | Cao hơn (Node + 2 con trỏ) |

**Chọn ArrayList khi:** đọc/truy cập ngẫu nhiên nhiều (mặc định nên chọn).
**Chọn LinkedList/ArrayDeque khi:** chèn/xoá chủ yếu ở hai đầu, hiếm khi truy cập ngẫu nhiên.

## References

- Java SE API Documentation — `java.util.LinkedList`.
- OpenJDK source — `java.base/java/util/LinkedList.java`, method `node(int)`.
