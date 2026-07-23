---
tags:
  - Java
  - ArrayList
  - Collections
---

# ArrayList

> Phase: Phase 2 — Collections
> Chapter slug: `arraylist`

## Metadata

```yaml
Chapter: ArrayList
Phase: Phase 2 — Collections
Difficulty: ★★★
Importance: ★★★★
Interview Frequency: 75%
Prerequisites:
  - Chapter 02 — List
  - Phase 1, Chapter 04 — Runtime Data Areas (Heap, mảng)
Used Later:
  - LinkedList (Chapter 04) — so sánh trực tiếp đối trọng
  - Mọi chapter dùng List làm ví dụ trong suốt handbook
Estimated Reading: 20 phút
Estimated Practice: 20 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

`ArrayList` được giới thiệu là "mảng có thể tự động lớn lên". Nghe hợp lý — nhưng mảng trong
Java (Phase 1, Chapter 04) có kích thước **cố định tuyệt đối** ngay khi tạo ra, không có cách
nào "tự lớn lên" ở tầng ngôn ngữ. Vậy `list.add()` lần thứ 11 vào một `ArrayList` mới tạo (mặc
định chứa được 10 phần tử) thực sự làm gì bên dưới?

Chapter này mở nắp `ArrayList` ra xem trực tiếp — bằng Reflection (Phase 1, Chapter 19) để
quan sát chính xác mảng nội bộ lớn lên như thế nào, tại sao lại lớn theo đúng tỉ lệ 1.5 lần chứ
không phải một con số tuỳ tiện, và cái giá thực sự phải trả cho việc "co giãn" đó.

## Interview Question (Central)

> `ArrayList` được cài đặt dựa trên mảng, nhưng mảng có kích thước cố định. Vậy `ArrayList`
> "tự lớn lên" bằng cách nào? Cái giá phải trả là gì?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Giải thích cơ chế **resize**: tạo mảng mới lớn hơn, copy toàn bộ phần tử cũ sang
- [ ] Tự tay quan sát được mảng nội bộ lớn lên theo đúng tỉ lệ ~1.5 lần bằng Reflection
- [ ] Giải thích độ phức tạp từng thao tác: `get()` O(1), `add()` cuối O(1) amortized,
      `add()`/`remove()` giữa O(n)
- [ ] Biết vì sao nên dùng constructor `new ArrayList<>(capacity)` khi biết trước kích thước

## Prerequisites

- Chapter 02 — hợp đồng `List` (thứ tự, trùng lặp, index).
- Phase 1, Chapter 04 — biết mảng nằm trên Heap, kích thước cố định lúc tạo.

## Used Later

- **LinkedList** (Chapter 04) — chapter kế tiếp là đối trọng trực tiếp, so sánh đánh đổi hiệu
  năng cho từng loại thao tác.
- Gần như mọi ví dụ code từ đây về sau trong handbook dùng `ArrayList` làm lựa chọn `List` mặc
  định — cần hiểu rõ chi phí ẩn đằng sau trước khi dùng "theo quán tính".

## Problem

Mảng Java có kích thước cố định — không có cách "thêm một ô" vào một mảng đã tạo. Nhưng nhu
cầu thực tế gần như luôn là "một danh sách có thể lớn dần theo thời gian, không biết trước kích
thước cuối cùng". Cần một cấu trúc **dùng mảng làm nền** nhưng **che giấu** việc phải tạo mảng
mới mỗi khi hết chỗ.

## Concept

`ArrayList` giữ bên trong một mảng `Object[]` (gọi là `elementData`). Khi `add()` mà mảng đã
đầy, nó thực hiện **resize**: tạo một mảng **mới, lớn hơn**, copy toàn bộ phần tử từ mảng cũ
sang, rồi mảng cũ trở thành rác (đủ điều kiện GC — Phase 1 Chapter 07). Việc "co giãn" mà người
dùng thấy chỉ là ảo giác — thực chất là thay hẳn một mảng khác mỗi lần cần lớn hơn.

## Why?

Nếu mỗi lần `add()` đều tạo mảng mới lớn hơn đúng 1 phần tử: `add()` sẽ luôn tốn O(n) (copy
toàn bộ n phần tử cũ) — thêm n phần tử liên tiếp sẽ tốn tổng cộng O(n²), quá chậm cho danh sách
lớn. Thay vào đó, `ArrayList` **cấp dư** mỗi lần resize (lớn hơn hẳn, không chỉ vừa đủ 1 chỗ) —
đổi lại, phần lớn lời gọi `add()` không cần resize gì cả (còn chỗ trống sẵn), chỉ occasionally
mới tốn O(n) để resize — tính trung bình trên nhiều lần gọi (**amortized**), `add()` cuối danh
sách có chi phí trung bình O(1).

## How?

Khi resize, `ArrayList` tính kích thước mới theo công thức: **`newCapacity = oldCapacity +
(oldCapacity >> 1)`** — tức là tăng thêm **50%** (dịch phải 1 bit tương đương chia 2), không
phải nhân đôi như nhiều người lầm tưởng.

```
Bắt đầu: capacity = 10 (mặc định)
Đầy (11 phần tử cần thêm) → resize: 10 + 10/2 = 15
Đầy tiếp (16 phần tử) → resize: 15 + 15/2 = 22 (làm tròn xuống)
```

## Visualization

```
add() thứ 11 vào ArrayList đang capacity=10 (đã đầy)
        │
        ▼
1. Tính capacity mới: 10 + (10 >> 1) = 15
        │
        ▼
2. Tạo mảng MỚI Object[15]
        │
        ▼
3. Arrays.copyOf() — COPY TOÀN BỘ 10 phần tử cũ sang mảng mới  [O(n)]
        │
        ▼
4. Mảng CŨ (Object[10]) không còn ai tham chiếu → RÁC, chờ GC [Ch.07 Phase 1]
        │
        ▼
5. Thêm phần tử thứ 11 vào mảng mới
```

## Example

Quan sát trực tiếp mảng nội bộ `elementData` lớn lên bằng Reflection (Phase 1, Chapter 19) —
cần cờ `--add-opens` vì `java.util` không tự `opens` cho unnamed module (Phase 1, Chapter 22):

```java
import java.lang.reflect.Field;
import java.util.ArrayList;

public class ArrayListGrowth {
    public static void main(String[] args) throws Exception {
        ArrayList<Integer> list = new ArrayList<>();
        Field elementDataField = ArrayList.class.getDeclaredField("elementData");
        elementDataField.setAccessible(true);

        int lastCap = -1;
        for (int i = 0; i < 20; i++) {
            list.add(i);
            Object[] arr = (Object[]) elementDataField.get(list);
            if (arr.length != lastCap) {
                System.out.println("size=" + list.size() + " -> capacity=" + arr.length);
                lastCap = arr.length;
            }
        }
    }
}
```

Chạy `java --add-opens java.base/java.util=ALL-UNNAMED ArrayListGrowth`, kết quả thật
(JDK 17):

```
size=1 -> capacity=10
size=11 -> capacity=15
size=16 -> capacity=22
```

Khớp chính xác công thức ở phần How?: 10 → 15 (10 + 5) → 22 (15 + 7, làm tròn xuống từ 15+7.5).

## Deep Dive

**Độ phức tạp từng thao tác, và vì sao khác nhau:**

| Thao tác | Độ phức tạp | Vì sao |
| --- | --- | --- |
| `get(index)` | O(1) | Mảng cho phép tính địa chỉ trực tiếp từ index |
| `add(e)` (cuối danh sách) | O(1) amortized | Xem phần Why? — hầu hết không cần resize |
| `add(index, e)` (giữa) | O(n) | Phải dịch chuyển mọi phần tử phía sau sang phải 1 vị trí |
| `remove(index)` (giữa) | O(n) | Phải dịch chuyển mọi phần tử phía sau sang trái 1 vị trí |

`add(index, e)`/`remove(index)` ở giữa danh sách tốn O(n) vì mảng yêu cầu các phần tử phải
**liền kề về mặt vật lý** trong bộ nhớ (Phase 1, Chapter 04) — chèn/xoá ở giữa bắt buộc phải
dịch chuyển mọi phần tử phía sau để lấp/tạo khoảng trống, dùng `System.arraycopy()` (một
method native cực nhanh cho việc copy khối bộ nhớ liên tục, nhưng vẫn là O(n) về độ phức tạp
thuật toán).

## Engineering Insight

**Vì sao chọn tỉ lệ tăng 1.5 lần, không phải 2 lần (nhân đôi, như nhiều cấu trúc dữ liệu
dynamic array khác, ví dụ Python `list` hay C++ `std::vector` một số hiện thực)?** Đây là đánh
đổi giữa **lãng phí bộ nhớ** và **số lần resize**. Tăng gấp đôi (2x) giảm số lần resize hơn nữa
nhưng có thể lãng phí tới 50% bộ nhớ đã cấp phát mà không dùng tới (nếu kích thước cuối cùng
dừng lại ngay sau một lần resize). Tăng 1.5x lãng phí ít hơn (tối đa ~33%) nhưng resize thường
xuyên hơn một chút. Có một lý do kỹ thuật sâu hơn ủng hộ 1.5x: với tỉ lệ tăng trưởng **nhỏ hơn
tỉ lệ vàng** (golden ratio, ≈1.618), bộ nhớ của các mảng cũ đã giải phóng (do resize) có thể
được **tái sử dụng** bởi các lần cấp phát sau đó trong cùng vùng nhớ liền kề — một chi tiết tối
ưu ở tầng cấp phát bộ nhớ (memory allocator) mà tỉ lệ 2x không đảm bảo được. Đây chính là lý do
nhiều cấu trúc dynamic array hiện đại (bao gồm `ArrayList` của Java) chọn hệ số tăng trưởng nhỏ
hơn 2, không phải một con số tuỳ tiện.

## Historical Note

`ArrayList` xuất hiện từ Java 2 (1998, cùng đợt Collection Framework). Công thức resize
`oldCapacity + (oldCapacity >> 1)` không đổi qua các phiên bản — một trong những phần ổn định
nhất của JDK. Thay đổi đáng chú ý duy nhất: Java 7 thêm tối ưu **lazy initialization** — một
`ArrayList` tạo bằng constructor không tham số (`new ArrayList<>()`) không cấp phát mảng
`Object[10]` ngay lập tức, mà dùng một mảng rỗng dùng chung (`DEFAULTCAPACITY_EMPTY_ELEMENTDATA`)
cho tới khi phần tử **đầu tiên** thực sự được thêm vào — tránh lãng phí bộ nhớ cho những
`ArrayList` được tạo ra nhưng không bao giờ dùng tới (khá phổ biến trong thực tế, ví dụ field
khởi tạo sẵn nhưng nhiều object không bao giờ thêm phần tử nào).

## Myth vs Reality

- **Myth:** "`ArrayList` tăng gấp đôi kích thước mỗi lần resize."
  **Reality:** Tăng **1.5 lần** (`oldCapacity + oldCapacity/2`), không phải gấp đôi — xem bằng
  chứng thực nghiệm ở Example.

- **Myth:** "`add()` luôn là thao tác O(1) trên `ArrayList`."
  **Reality:** `add()` **ở cuối** danh sách là O(1) amortized. `add(index, e)` ở giữa/đầu danh
  sách là O(n) — một nhầm lẫn phổ biến dẫn thẳng tới lựa chọn sai giữa `ArrayList` và
  `LinkedList` (Chapter 04).

## Common Mistakes

- **Tạo `ArrayList` không chỉ định capacity ban đầu khi đã biết trước kích thước gần đúng** —
  bỏ lỡ cơ hội tránh nhiều lần resize không cần thiết.
- **Chèn/xoá liên tục ở đầu danh sách** (`add(0, e)`, `remove(0)`) trong vòng lặp lớn — mỗi lần
  đều là O(n), tổng cộng O(n²) cho n thao tác — một anti-pattern hiệu năng phổ biến, nên cân
  nhắc `LinkedList`/`ArrayDeque` (Chapter 04, 13) cho use case này.

## Best Practices

- Nếu biết trước (hoặc ước lượng được) kích thước cuối cùng, dùng
  `new ArrayList<>(expectedSize)` để tránh resize nhiều lần.
- Với thao tác chèn/xoá thường xuyên ở đầu hoặc giữa danh sách, cân nhắc `LinkedList` (Chapter
  04) hoặc `ArrayDeque` (Chapter 13) thay vì `ArrayList`.
- Mặc định chọn `ArrayList` cho phần lớn use case thông thường (đọc nhiều, ghi chủ yếu ở cuối)
  — đây là lựa chọn hiệu quả nhất trong đa số trường hợp thực tế.

## Production Notes

**Vấn đề:** một job xử lý dữ liệu hàng loạt (batch), xây dựng dần một `ArrayList` lớn (hàng
triệu phần tử) bằng `add()` liên tục, chạy chậm hơn nhiều so với ước tính, đặc biệt ở giai đoạn
đầu.

- **Triệu chứng:** thời gian xử lý không tuyến tính với số lượng phần tử — chậm hơn hẳn dự kiến
  khi profiling.
- **Root cause:** `ArrayList` khởi tạo mặc định (capacity 10), phải resize rất nhiều lần khi
  lớn dần tới hàng triệu phần tử — mỗi lần resize là một lần copy O(n) toàn bộ phần tử đã có,
  tích luỹ thành chi phí đáng kể (dù vẫn amortized O(1) về mặt lý thuyết, hằng số ẩn phía sau
  không hề nhỏ với dữ liệu lớn).
- **Debug:** profiling cho thấy thời gian đáng kể trong `Arrays.copyOf()`/`grow()` nội bộ của
  `ArrayList`.
- **Solution:** nếu biết trước (hoặc ước lượng được từ nguồn dữ liệu, ví dụ `COUNT(*)` từ
  database trước khi fetch) kích thước cuối cùng, khởi tạo `new ArrayList<>(estimatedSize)`
  ngay từ đầu.
- **Prevention:** với các job xử lý dữ liệu lớn đã biết trước quy mô, luôn ước lượng capacity
  ban đầu thay vì dùng constructor mặc định.

## Debug Checklist

- [ ] Xây dựng `ArrayList` lớn dần chậm bất thường? → kiểm tra có đang dùng constructor mặc
      định (capacity 10) cho danh sách cuối cùng rất lớn không, cân nhắc chỉ định capacity ban
      đầu.
- [ ] Thao tác `add()`/`remove()` ở đầu/giữa danh sách chậm? → đây là O(n) theo đúng bản chất
      cấu trúc mảng, cân nhắc `LinkedList`/`ArrayDeque` nếu thao tác này diễn ra thường xuyên.

## Source Code Walkthrough

```
ArrayList.add(e)
    ↓
ensureCapacityInternal() — kiểm tra còn chỗ trống không
    ↓
Nếu đầy: grow() — tính newCapacity = oldCapacity + (oldCapacity >> 1)
    ↓
Arrays.copyOf(elementData, newCapacity) — tạo mảng mới, copy dữ liệu cũ
    ↓
elementData[size++] = e — gán phần tử mới vào cuối mảng (đã chắc chắn còn chỗ)
```

## Summary

`ArrayList` dùng một mảng `Object[]` nội bộ; "co giãn" thực chất là tạo mảng **mới** lớn hơn
theo tỉ lệ **1.5 lần** (không phải 2 lần) rồi copy toàn bộ phần tử cũ sang — đã xác nhận bằng
Reflection thực nghiệm. Nhờ cấp dư mỗi lần resize, `add()` ở cuối danh sách có chi phí trung
bình O(1) (amortized), nhưng `add()`/`remove()` ở giữa hoặc đầu danh sách luôn O(n) vì phải
dịch chuyển phần tử. Khi biết trước kích thước, luôn khởi tạo `ArrayList` với capacity ban đầu
để tránh resize nhiều lần không cần thiết.

## Interview Questions

**Junior**

- `ArrayList` dùng cấu trúc dữ liệu gì bên dưới?
- `get(index)` trên `ArrayList` có độ phức tạp là gì?

**Mid**

- Giải thích cơ chế resize của `ArrayList`. Tỉ lệ tăng trưởng là bao nhiêu?
- Vì sao `add(0, e)` chậm hơn nhiều so với `add(e)` (thêm ở cuối)?

**Senior**

- Giải thích vì sao `ArrayList` chọn hệ số tăng trưởng 1.5 thay vì 2 — phân tích đánh đổi giữa
  lãng phí bộ nhớ và tần suất resize.
- Một job xử lý batch xây dựng `ArrayList` hàng triệu phần tử chạy chậm hơn dự kiến. Trình bày
  cách bạn sẽ chẩn đoán và tối ưu.

## Exercises

- [ ] Chạy lại đúng ví dụ `ArrayListGrowth` ở trên (nhớ thêm cờ `--add-opens`), xác nhận dãy
      capacity giống hệt.
- [ ] Viết đoạn code đo thời gian `add(0, e)` lặp lại 20.000 lần trên một `ArrayList` đã có sẵn
      50.000 phần tử — so sánh với `add(e)` (cuối danh sách) cùng số lần, xác nhận chênh lệch
      lớn.
- [ ] Thử tạo `ArrayList` bằng `new ArrayList<>(1000)`, dùng lại kỹ thuật Reflection ở Example
      để xác nhận capacity ban đầu đúng bằng 1000, không phải 10.

## Cheat Sheet

| Thao tác | Độ phức tạp |
| --- | --- |
| `get(index)` | O(1) |
| `add(e)` (cuối) | O(1) amortized |
| `add(index, e)` / `remove(index)` (giữa/đầu) | O(n) |

| Công thức resize |
| --- |
| `newCapacity = oldCapacity + (oldCapacity >> 1)` (tăng ~50%) |

## References

- Java SE API Documentation — `java.util.ArrayList`.
- OpenJDK source — `java.base/java/util/ArrayList.java`, method `grow()`.
