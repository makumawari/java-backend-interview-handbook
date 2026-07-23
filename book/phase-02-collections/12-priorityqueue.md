---
tags:
  - Java
  - PriorityQueue
  - Collections
---

# PriorityQueue

> Phase: Phase 2 — Collections
> Chapter slug: `priorityqueue`

## Metadata

```yaml
Chapter: PriorityQueue
Phase: Phase 2 — Collections
Difficulty: ★★★
Importance: ★★★
Interview Frequency: 50%
Prerequisites:
  - Chapter 01 — Collection Framework (Queue)
  - Chapter 14 — Comparable vs Comparator (xem trước nếu chưa học)
Used Later:
  - System Design (Phase 9) — hàng đợi ưu tiên xử lý task/event theo mức độ khẩn cấp
Estimated Reading: 20 phút
Estimated Practice: 20 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

Bạn xử lý đơn hàng theo độ ưu tiên — đơn VIP xử lý trước đơn thường. Bạn dùng `PriorityQueue`,
thêm vào theo thứ tự bất kỳ, rồi tò mò in thử toàn bộ hàng đợi ra xem:

```java
PriorityQueue<Integer> pq = new PriorityQueue<>();
int[] values = {50, 20, 80, 10, 30, 5, 90};
for (int v : values) pq.add(v);
System.out.println(pq); // ⚠️
```

Kết quả:

```
[5, 20, 10, 50, 30, 80, 90]
```

Không phải `[5, 10, 20, 30, 50, 80, 90]` như bạn kỳ vọng từ một "hàng đợi ưu tiên"! Thứ tự
trông gần như ngẫu nhiên. Đây không phải bug — nó là một chi tiết dễ hiểu sai nhất về
`PriorityQueue`: nó **đảm bảo** phần tử lấy ra bằng `poll()` luôn đúng thứ tự ưu tiên, nhưng
**không đảm bảo** thứ tự khi bạn duyệt/in nó ra trực tiếp (`toString()`, `iterator()`). Chapter
này giải thích chính xác vì sao — bằng cách nhìn vào cấu trúc **heap** đứng bên dưới.

## Interview Question (Central)

> `PriorityQueue` đảm bảo điều gì? Vì sao duyệt trực tiếp qua nó (for-each/`toString()`) không
> cho ra đúng thứ tự sắp xếp, trong khi gọi `poll()` liên tục thì luôn đúng?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Giải thích cấu trúc **binary heap** — mảng biểu diễn cây nhị phân, chỉ đảm bảo quan hệ
      cha-con, không đảm bảo thứ tự toàn cục
- [ ] Tự tay tái hiện được sự khác biệt giữa duyệt trực tiếp và `poll()` liên tục
- [ ] Dùng đúng `PriorityQueue` với `Comparator` tuỳ chỉnh (không chỉ thứ tự tự nhiên)
- [ ] Biết độ phức tạp: `offer()`/`poll()` O(log n), `peek()` O(1)

## Prerequisites

- Chapter 01 — biết `Queue` là một nhánh của `Collection`.
- Chapter 14 (xem trước nếu chưa học) — `Comparable`/`Comparator` quyết định thứ tự ưu tiên.

## Used Later

- **System Design** (Phase 9) — hàng đợi ưu tiên là thành phần phổ biến trong hệ thống xử lý
  task/event theo độ khẩn cấp (job scheduler, event processing).

## Problem

`Queue` thường (FIFO — vào trước ra trước) không đủ cho bài toán cần xử lý theo **độ ưu
tiên** thay vì theo thứ tự đến. Cần một cấu trúc luôn cho ra phần tử "nhỏ nhất" (hoặc "lớn
nhất", tuỳ định nghĩa) hiện có, hiệu quả hơn nhiều so với việc `sort()` lại toàn bộ mỗi lần có
phần tử mới.

## Concept

`PriorityQueue` cài đặt bằng **binary heap** (cụ thể: min-heap theo mặc định) — một cây nhị
phân **hoàn chỉnh** (complete binary tree, các tầng luôn lấp đầy từ trái sang phải) được biểu
diễn bằng **một mảng phẳng**, không dùng con trỏ như `LinkedList` (Chapter 04). Heap chỉ đảm
bảo đúng **một tính chất**: giá trị tại mỗi node **nhỏ hơn hoặc bằng** giá trị của hai node con
của nó (heap property) — **không** đảm bảo bất kỳ quan hệ thứ tự nào giữa các node **anh em**
hay giữa các tầng khác nhau.

## Why?

Nếu `PriorityQueue` giữ luôn một mảng **hoàn toàn sắp xếp**: mỗi lần `offer()` một phần tử mới
sẽ cần chèn đúng vị trí, tốn O(n) để dịch chuyển (giống `ArrayList.add(index, e)`, Chapter 03).
Heap chỉ yêu cầu **một tính chất cục bộ, yếu hơn nhiều** (cha ≤ con) thay vì thứ tự toàn cục —
đủ để đảm bảo phần tử **nhỏ nhất luôn ở gốc** (root, chỉ số 0 của mảng), lấy ra tức thì O(1)
(`peek()`), và chèn/xoá chỉ cần điều chỉnh cục bộ dọc theo một đường đi từ gốc tới lá, O(log n)
— rẻ hơn nhiều so với giữ toàn bộ mảng luôn sắp xếp.

## How?

```
offer(v): thêm v vào cuối mảng (heap), rồi "nổi lên" (sift up) —
          so sánh với node cha, hoán đổi nếu vi phạm heap property,
          lặp lại cho tới khi đúng vị trí — O(log n)

poll():   lấy phần tử tại gốc (LUÔN là nhỏ nhất), thay gốc bằng phần tử
          CUỐI mảng, rồi "chìm xuống" (sift down) — so sánh với 2 con,
          hoán đổi với con nhỏ hơn nếu vi phạm, lặp lại — O(log n)

peek():   chỉ đọc phần tử tại gốc (chỉ số 0) — O(1), không thay đổi cấu trúc
```

## Visualization

```
Mảng: [5, 20, 10, 50, 30, 80, 90]     (đây CHÍNH XÁC là output toString() ở Story!)

Biểu diễn dạng cây nhị phân (chỉ số i có con tại 2i+1, 2i+2):

                    5 (index 0)
                 /              \
           20 (idx1)          10 (idx2)
           /      \            /      \
     50(idx3)  30(idx4)   80(idx5)  90(idx6)

Heap property: mỗi node ≤ hai node con — ĐÚNG cho mọi node ở đây
   (5 ≤ 20, 5 ≤ 10; 20 ≤ 50, 20 ≤ 30; 10 ≤ 80, 10 ≤ 90)

NHƯNG: 20 và 10 (hai node anh em) KHÔNG có quan hệ thứ tự bắt buộc nào
       giữa chúng — đây là lý do duyệt mảng trực tiếp KHÔNG cho ra thứ tự
       sắp xếp toàn cục, dù root luôn là nhỏ nhất.
```

## Example

Xác nhận trực tiếp: duyệt (`toString()`/for-each) khác hoàn toàn với lấy ra liên tục bằng
`poll()`:

```java
import java.util.PriorityQueue;

public class PriorityQueueDemo {
    public static void main(String[] args) {
        PriorityQueue<Integer> pq = new PriorityQueue<>();
        int[] values = {50, 20, 80, 10, 30, 5, 90};
        for (int v : values) pq.add(v);

        System.out.println("Duyet bang for-each (iterator, KHONG dam bao thu tu sap xep): " + pq);

        System.out.print("Lay ra bang poll() lien tuc (LUON dung thu tu tang dan): ");
        StringBuilder sb = new StringBuilder();
        while (!pq.isEmpty()) {
            sb.append(pq.poll()).append(" ");
        }
        System.out.println(sb.toString().trim());
    }
}
```

Kết quả thật (JDK 17):

```
Duyet bang for-each (iterator, KHONG dam bao thu tu sap xep): [5, 20, 10, 50, 30, 80, 90]
Lay ra bang poll() lien tuc (LUON dung thu tu tang dan): 5 10 20 30 50 80 90
```

Cùng một `PriorityQueue`, hai cách "xem" cho hai kết quả khác hẳn nhau: `toString()` in đúng
thứ tự **vật lý trong mảng heap** (chỉ đảm bảo cha ≤ con); `poll()` liên tục luôn cho ra đúng
thứ tự tăng dần tuyệt đối, vì mỗi lần `poll()` đều lấy đúng phần tử nhỏ nhất hiện có tại thời
điểm đó.

## Deep Dive

**Vì sao biểu diễn bằng mảng (không dùng Node/con trỏ như `TreeMap`, Chapter 10) lại khả thi
cho một cấu trúc cây?** Vì heap luôn là cây **hoàn chỉnh** (complete — mọi tầng lấp đầy từ trái
sang phải, chỉ tầng cuối có thể chưa đầy) — tính chất này cho phép tính trực tiếp vị trí cha/con
chỉ bằng phép toán số học trên chỉ số (`cha = (i-1)/2`, `con trái = 2i+1`, `con phải = 2i+2`),
không cần lưu con trỏ tường minh. Đây là lý do heap tiết kiệm bộ nhớ hơn hẳn cấu trúc cây dùng
Node (như `TreeMap`) — không tốn thêm bộ nhớ cho con trỏ, đổi lại **chỉ dùng được** cho cây
hoàn chỉnh, không tổng quát được cho cây bất kỳ như Red-Black Tree.

## Engineering Insight

**Vì sao `PriorityQueue` không implement `NavigableSet`-kiểu-API (như `TreeMap`, Chapter 10) —
tại sao không thể `floorKey()` trên một heap?** Vì heap **cố tình hy sinh** thông tin thứ tự
toàn cục để đổi lấy chèn/xoá phần tử ưu tiên nhất cực rẻ (O(log n), so với O(log n) của
`TreeMap` cho **mọi** thao tác điều hướng, nhưng heap rẻ hơn về hằng số thực tế và đơn giản hơn
nhiều để cài đặt/tối ưu). Đây là một đánh đổi thiết kế có chủ đích: nếu bài toán chỉ cần "luôn
biết và lấy ra phần tử ưu tiên nhất", heap là lựa chọn tối ưu; nếu cần đầy đủ các truy vấn
"lân cận theo thứ tự" như `TreeMap`, phải trả giá bằng cấu trúc phức tạp hơn (cây cân bằng đầy
đủ, không chỉ heap). Không có cấu trúc nào "tốt hơn tuyệt đối" — mỗi cấu trúc tối ưu cho đúng
tập câu hỏi nó được thiết kế để trả lời.

## Historical Note

`PriorityQueue` xuất hiện từ Java 5 (2004) — khá muộn so với các cấu trúc dữ liệu khác của
Collection Framework (Java 2, 1998). Trước đó, lập trình viên Java cần hàng đợi ưu tiên phải tự
cài đặt heap thủ công hoặc dùng thư viện bên thứ ba — một khoảng trống đáng chú ý trong thư
viện chuẩn suốt 6 năm.

## Myth vs Reality

- **Myth:** "`PriorityQueue` là một danh sách luôn được sắp xếp, chỉ khác tên gọi."
  **Reality:** Xem Example — duyệt trực tiếp **không** cho ra thứ tự sắp xếp. Chỉ `poll()`
  liên tục mới đảm bảo đúng thứ tự.

- **Myth:** "In một `PriorityQueue` ra màn hình để debug là đủ để biết thứ tự xử lý thực tế."
  **Reality:** `toString()` chỉ phản ánh thứ tự **vật lý trong mảng heap**, hoàn toàn khác thứ
  tự xử lý thực tế (qua `poll()`) — dễ gây hiểu lầm nghiêm trọng khi debug nếu không biết rõ
  điều này.

## Common Mistakes

- **Giả định `for (int v : pq)` duyệt theo đúng thứ tự ưu tiên** — sai, xem Example. Muốn xử lý
  theo đúng thứ tự ưu tiên, phải dùng `poll()` liên tục (thường trong vòng lặp
  `while (!pq.isEmpty())`), không phải for-each.
- **Sửa phần tử đang nằm trong `PriorityQueue` (nếu phần tử là object mutable) mà không
  remove/re-add** — heap property có thể bị vi phạm âm thầm, `poll()` sau đó không còn đảm bảo
  đúng thứ tự.

## Best Practices

- Muốn lấy ra theo đúng thứ tự ưu tiên, luôn dùng `poll()` (hoặc `peek()` để chỉ xem không lấy
  ra), không dùng for-each/`toString()`.
- Nếu cần thứ tự ưu tiên tuỳ chỉnh (không phải thứ tự tự nhiên), truyền `Comparator` vào
  constructor (Chapter 14) thay vì dựa vào `Comparable` mặc định của phần tử.
- Không sửa trực tiếp field của phần tử đang nằm trong `PriorityQueue` nếu field đó ảnh hưởng
  thứ tự — remove rồi add lại nếu cần thay đổi độ ưu tiên.

## Production Notes

**Vấn đề:** một job scheduler dùng `PriorityQueue` để xử lý task theo độ ưu tiên; log debug in
ra trạng thái hàng đợi bằng cách duyệt trực tiếp (`for task : queue`) cho thấy thứ tự "có vẻ
sai", khiến đội vận hành nghi ngờ có bug trong logic ưu tiên.

- **Triệu chứng:** log debug hiển thị thứ tự "lộn xộn", nhưng task **thực tế được xử lý** vẫn
  đúng thứ tự ưu tiên (vì code xử lý dùng `poll()`, không phải for-each).
- **Root cause:** hiểu lầm giữa mục đích debug (dùng for-each để log) và mục đích xử lý (dùng
  `poll()`) — đây không phải bug, mà là hiểu sai bản chất `toString()`/iterator của
  `PriorityQueue` như đã học ở chapter này.
- **Debug:** xác nhận code xử lý task **thực sự** dùng `poll()`, không phải duyệt trực tiếp;
  nếu chỉ log debug dùng for-each, đó chỉ là vấn đề hiển thị, không phải vấn đề logic.
- **Solution:** nếu cần log ra đúng thứ tự xử lý dự kiến (không làm ảnh hưởng hàng đợi thật),
  tạo một bản sao (`new PriorityQueue<>(original)`) rồi `poll()` liên tục trên bản sao đó chỉ
  để log.
- **Prevention:** tài liệu hoá rõ ràng trong code/runbook rằng thứ tự hiển thị của
  `PriorityQueue` khác thứ tự xử lý thực tế, tránh gây hoang mang không cần thiết khi debug.

## Debug Checklist

- [ ] Log debug `PriorityQueue` "trông sai thứ tự" nhưng xử lý thực tế vẫn đúng? → xác nhận
      code xử lý dùng `poll()`, chỉ log debug đang duyệt trực tiếp — không phải bug thật.
- [ ] Thứ tự xử lý thực tế (`poll()`) sai? → kiểm tra `Comparable`/`Comparator` (Chapter 14)
      định nghĩa đúng thứ tự ưu tiên mong muốn chưa.
- [ ] Nghi ngờ heap bị hỏng cấu trúc? → kiểm tra có đang sửa trực tiếp field ảnh hưởng thứ tự
      của phần tử đang nằm trong queue không.

## Source Code Walkthrough

```
PriorityQueue.offer(e)
    ↓
queue[size] = e  (thêm vào cuối mảng)
    ↓
siftUp(size, e): so sánh e với cha tại (i-1)/2,
    nếu e < cha → hoán đổi, lặp lại với vị trí mới, cho tới gốc hoặc hết vi phạm

PriorityQueue.poll()
    ↓
result = queue[0]  (luôn là nhỏ nhất)
    ↓
lấy phần tử CUỐI mảng, đặt vào vị trí 0
    ↓
siftDown(0): so sánh với 2 con, hoán đổi với con NHỎ HƠN nếu vi phạm,
    lặp lại cho tới lá hoặc hết vi phạm
    ↓
return result
```

## Summary

`PriorityQueue` cài đặt bằng **binary heap**, biểu diễn dạng mảng phẳng, chỉ đảm bảo một tính
chất cục bộ: mỗi node ≤ hai node con. Điều này đủ để `peek()`/`poll()` luôn trả về đúng phần tử
nhỏ nhất O(1)/O(log n), nhưng **không** đảm bảo thứ tự toàn cục khi duyệt trực tiếp
(`toString()`/for-each) — đã chứng minh bằng thực nghiệm: duyệt cho `[5, 20, 10, 50, 30, 80,
90]`, trong khi `poll()` liên tục luôn cho đúng `5 10 20 30 50 80 90`. Muốn xử lý theo đúng thứ
tự ưu tiên, luôn dùng `poll()`, không bao giờ dựa vào thứ tự duyệt trực tiếp.

## Interview Questions

**Junior**

- `PriorityQueue` đảm bảo điều gì?
- Duyệt `PriorityQueue` bằng for-each có cho ra đúng thứ tự sắp xếp không?

**Mid**

- Giải thích cấu trúc binary heap và heap property.
- `offer()`/`poll()` trên `PriorityQueue` có độ phức tạp gì? Vì sao `peek()` lại O(1)?

**Senior**

- Vì sao heap biểu diễn được bằng mảng phẳng mà không cần con trỏ, trong khi `TreeMap`
  (Chapter 10) cần Node với con trỏ tường minh?
- So sánh đánh đổi giữa dùng `PriorityQueue` (heap) và `TreeMap` (Red-Black Tree) cho bài toán
  cần "luôn lấy ra phần tử nhỏ nhất" — khi nào chọn cái nào?

## Exercises

- [ ] Chạy lại đúng ví dụ `PriorityQueueDemo` ở trên, xác nhận kết quả giống hệt.
- [ ] Vẽ tay cây nhị phân từ mảng `[5, 20, 10, 50, 30, 80, 90]`, xác nhận heap property đúng ở
      mọi node theo công thức chỉ số đã học ở phần Deep Dive.
- [ ] Tạo `PriorityQueue<Product>` với `Comparator` sắp theo giá giảm dần (max-heap thay vì
      min-heap mặc định) — xác nhận `poll()` liên tục cho ra đúng thứ tự giá giảm dần.

## Cheat Sheet

| Thao tác | Độ phức tạp | Đảm bảo thứ tự? |
| --- | --- | --- |
| `offer(e)` | O(log n) | — |
| `poll()` | O(log n) | Luôn đúng thứ tự ưu tiên |
| `peek()` | O(1) | Luôn đúng (phần tử ưu tiên nhất) |
| `toString()` / for-each / iterator | O(n) để duyệt | **KHÔNG** đảm bảo thứ tự sắp xếp |

## References

- Java SE API Documentation — `java.util.PriorityQueue`.
- Oracle: "The Java Tutorials" — phần Queue interface.
- Thomas H. Cormen et al., *Introduction to Algorithms* — chương Heapsort/Priority Queue.
