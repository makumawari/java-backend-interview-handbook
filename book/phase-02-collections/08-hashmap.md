---
tags:
  - Java
  - HashMap
  - Collections
  - Interview
---

# HashMap

> Phase: Phase 2 — Collections
> Chapter slug: `hashmap`

## Metadata

```yaml
Chapter: HashMap
Phase: Phase 2 — Collections
Difficulty: ★★★★★
Importance: ★★★★★
Interview Frequency: 95%
Prerequisites:
  - Chapter 07 — Map
  - Phase 1, Chapter 12-13 — equals()/hashCode()
  - Phase 1, Chapter 04 — Runtime Data Areas (Heap)
Used Later:
  - LinkedHashMap (Chapter 09), TreeMap (Chapter 10), ConcurrentHashMap (Chapter 11)
  - HashSet (Chapter 06) — đã học trước, nay hiểu trọn vẹn cơ chế thật đứng sau nó
  - Spring Cache, Hibernate First-Level Cache (Phase 5-6) — đều dựa trên HashMap
Estimated Reading: 35 phút
Estimated Practice: 40 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

Giả sử bạn có 10 triệu `User`. Bạn muốn tìm `User` theo `id`. Bạn sẽ làm thế nào?

Dùng `List<User>` và duyệt tuyến tính? Với 10 triệu phần tử, tìm một `User` có thể phải xem
qua **5 triệu phần tử** (trung bình) trước khi tìm thấy — với một web server xử lý hàng nghìn
request/giây, đây là thảm hoạ hiệu năng.

`HashMap` giải quyết đúng bài toán này: tra cứu gần như **tức thì**, bất kể có 10, 10 nghìn,
hay 10 triệu phần tử. Bí mật đằng sau tốc độ đó không phải phép màu — nó là một chuỗi quyết
định thiết kế rất cụ thể, mỗi quyết định giải quyết đúng một vấn đề. Chapter này mở nắp
`HashMap` ra, không chỉ đọc tài liệu mà **quan sát trực tiếp** cấu trúc bucket, va chạm hash, và
khoảnh khắc nó tự chuyển từ danh sách liên kết sang cây đỏ-đen — bằng chính Reflection bạn đã
học ở Phase 1.

## Interview Question (Central)

> Giải thích cơ chế hoạt động bên trong của `HashMap`. Vì sao nó nhanh, và khi nào nó không còn
> nhanh nữa?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Giải thích được cấu trúc bucket array, hàm băm (hash spreading), và cách tính chỉ số
      bucket từ `hashCode()`
- [ ] Tự tay quan sát bằng Reflection: bucket thật, va chạm hash thật, resize thật
- [ ] Giải thích cơ chế **treeify** — khi nào và tại sao một bucket chuyển từ danh sách liên
      kết sang cây đỏ-đen, tự tay tái hiện bằng chứng
- [ ] Trả lời chính xác: có thể sửa `HashMap` trong lúc duyệt không, và fail-fast hoạt động ra
      sao ở đây
- [ ] Debug được các lỗi `HashMap` phổ biến nhất trong thực tế

## Prerequisites

- Chapter 07 — hiểu `Map` là ánh xạ key-value, key phải duy nhất.
- Phase 1, Chapter 12-13 — hợp đồng `equals()`/`hashCode()` — nền tảng bắt buộc, `HashMap`
  không hoạt động đúng nếu hợp đồng này bị vi phạm (đã minh hoạ ở Phase 1).
- Phase 1, Chapter 04 — biết object nằm trên Heap.

## Used Later

- **LinkedHashMap, TreeMap, ConcurrentHashMap** (Chapter 09-11) — cả ba đều là biến thể/mở
  rộng của chính cơ chế học ở đây.
- **HashSet** (Chapter 06, đã học) — giờ bạn hiểu trọn vẹn cơ chế thật sự đứng sau nó.
- **Spring Cache, Hibernate First-Level Cache** (Phase 5-6) — hầu hết cơ chế cache trong Java
  đều xây trên nền `HashMap`/`ConcurrentHashMap`.

## Problem

Tra cứu tuyến tính (`List`, duyệt từng phần tử so sánh `equals()`) là O(n) — không chấp nhận
được cho dữ liệu lớn, tra cứu thường xuyên. Cần một cấu trúc cho phép "nhảy thẳng" gần như tới
đúng vị trí phần tử cần tìm, không phải so sánh lần lượt từng phần tử.

## Concept

`HashMap` dùng một **mảng các bucket** (`table`, kiểu `Node[]`). Vị trí (chỉ số) một key rơi
vào được tính trực tiếp từ `hashCode()` của nó — không cần so sánh với các key khác trước. Tra
cứu trở thành: tính chỉ số (O(1)) → nhảy thẳng tới đúng bucket (O(1)) → so sánh `equals()` chỉ
với (thường rất ít) phần tử **trong cùng bucket đó** — trung bình O(1) thay vì O(n).

## Why?

Nếu dùng mảng phẳng và tính chỉ số trực tiếp bằng chính giá trị `hashCode()` (một số `int` rất
lớn, hàng tỷ giá trị khả dĩ): sẽ cần một mảng khổng lồ để không bao giờ trùng chỉ số — không
khả thi. `HashMap` giải quyết bằng phép **modulo** (thực chất là AND bit, xem How?) để "ép"
`hashCode()` (có thể là số bất kỳ) vào đúng phạm vi kích thước mảng thực tế (16, 32, 64...) —
đánh đổi: nhiều key khác nhau có thể bị ép về **cùng một chỉ số** (hash collision, không thể
tránh khỏi tuyệt đối — đã giải thích bằng nguyên lý Dirichlet ở
[Phase 1, Chapter 13](../phase-01-foundation/13-hashcode.md)).

## How?

Ba bước khi `put(key, value)`:

1. **Hash spreading**: `hash(key) = key.hashCode() ^ (key.hashCode() >>> 16)` — XOR 16 bit cao
   với 16 bit thấp của `hashCode()` gốc.
2. **Tính chỉ số bucket**: `index = (table.length - 1) & hash` — phép AND bit, tương đương
   modulo khi `table.length` luôn là luỹ thừa của 2 (xem Engineering Insight).
3. **Xử lý va chạm (collision)**: nếu bucket tại `index` đã có phần tử khác (`hashCode()` khác
   nhau nhưng "ép" về cùng chỉ số, hoặc `hashCode()` giống hệt) — nối thêm vào một **danh sách
   liên kết** tại đúng bucket đó. Nếu danh sách liên kết trong một bucket dài **quá 8 phần tử**
   VÀ `table.length >= 64`, nó tự động chuyển thành **cây đỏ-đen** (Red-Black Tree) — gọi là
   **treeify**.

Khi số phần tử vượt quá `capacity × 0.75` (load factor mặc định), toàn bộ `table` được
**resize**: tạo mảng mới gấp đôi kích thước, tính lại chỉ số và phân bổ lại **mọi** phần tử.

## Visualization

```
key.hashCode()
     │
     ▼
hash spreading: h ^ (h >>> 16)
     │
     ▼
index = (table.length - 1) & spread(hash)
     │
     ▼
┌─────────────────────────────── table (bucket array) ────────────────────────────┐
│ [0]  [1]           [2]  [3]  [4]              ...                                │
│      banana→date        egg                                                      │
│      (2 phần tử,         (1 phần tử)                                             │
│       danh sách liên kết,                                                        │
│       collision)                                                                  │
└────────────────────────────────────────────────────────────────────────────────┘
       nếu 1 bucket > 8 phần tử VÀ table.length >= 64
                    │
                    ▼
       bucket đó chuyển thành CÂY ĐỎ-ĐEN (TreeNode), không còn là danh sách liên kết
```

## Example

**Xác nhận hash spreading và collision chain thật** bằng Reflection — đúng những gì hình vẽ ở
trên mô tả:

```java
import java.lang.reflect.*;
import java.util.HashMap;

public class HashSpreadDemo3 {
    public static void main(String[] args) throws Exception {
        HashMap<String, String> map = new HashMap<>();
        String[] keys = {"apple", "banana", "cherry", "date", "egg"};
        for (String k : keys) map.put(k, "v-" + k);

        Field tableField = HashMap.class.getDeclaredField("table");
        tableField.setAccessible(true);
        Object[] table = (Object[]) tableField.get(map);

        Class<?> nodeClass = Class.forName("java.util.HashMap$Node");
        Field keyField = nodeClass.getDeclaredField("key");
        Field nextField = nodeClass.getDeclaredField("next");
        keyField.setAccessible(true);
        nextField.setAccessible(true);

        for (int i = 0; i < table.length; i++) {
            if (table[i] == null) continue;
            StringBuilder sb = new StringBuilder("bucket[" + i + "]: ");
            Object node = table[i];
            while (node != null) {
                sb.append(keyField.get(node)).append(" -> ");
                node = nextField.get(node);
            }
            sb.append("null");
            System.out.println(sb);
        }
    }
}
```

Chạy với `--add-opens java.base/java.util=ALL-UNNAMED`, kết quả thật (JDK 17, capacity mặc
định 16):

```
bucket[0]: banana -> date -> null
bucket[1]: apple -> cherry -> null
bucket[4]: egg -> null
```

`banana` và `date` **va chạm** vào cùng bucket 0, `apple` và `cherry` va chạm vào cùng bucket
1 — cả hai cặp thực sự nằm chung một danh sách liên kết (`next` trỏ tới nhau), đúng cơ chế đã
mô tả ở How?.

**Xác nhận resize** — theo dõi `table.length` khi thêm phần tử liên tục:

```java
HashMap<Integer, String> map = new HashMap<>();
// ... thêm dần, theo dõi table.length qua Reflection
```

Kết quả thật:

```
size=1 -> table.length=16
size=13 -> table.length=32   (13 > 16 × 0.75 = 12)
size=25 -> table.length=64   (25 > 32 × 0.75 = 24)
```

Đúng khớp `load factor` 0.75: resize xảy ra ngay khi kích thước vượt quá 75% capacity hiện tại.

**Xác nhận treeify** — cố ý ép 9 key va chạm cùng một bucket:

```java
HashMap<BadKey, Integer> map = new HashMap<>();
for (int i = 0; i < 100; i++) map.put(new BadKey(1000 + i), i); // đẩy table lên >= 64
map.clear();
for (int i = 0; i < 9; i++) map.put(new BadKey(i), i); // 9 key CÙNG hashCode() = 42

// class BadKey { hashCode() { return 42; } ... }
```

Kết quả thật:

```
table.length = 256
Bucket #42 node class = TreeNode
```

Bucket chứa 9 phần tử va chạm (> ngưỡng 8) đã **thực sự** chuyển thành `TreeNode` (cây đỏ-đen)
— không phải suy luận từ tài liệu, mà quan sát trực tiếp kiểu dữ liệu của node trong bộ nhớ.

## Deep Dive

**Vì sao phải "spread" hash (`h ^ (h >>> 16)`) thay vì dùng thẳng `hashCode()`?** Vì
`index = (table.length - 1) & hash` chỉ dùng **các bit thấp** của `hash` (với capacity nhỏ như
16, chỉ 4 bit thấp nhất được dùng). Nếu nhiều key có `hashCode()` khác nhau ở các bit **cao**
nhưng giống nhau ở bit **thấp** (khá phổ biến với một số thuật toán hash tồi hoặc dữ liệu có
quy luật), chúng sẽ va chạm hàng loạt dù `hashCode()` gốc trông "khác nhau đủ". Việc XOR 16 bit
cao vào 16 bit thấp trước khi tính chỉ số giúp "trộn" thông tin từ toàn bộ 32 bit vào đúng
phần được dùng để định vị bucket — giảm đáng kể tần suất collision cho các bộ dữ liệu khoá có
tính quy luật.

**Vì sao `table.length` luôn là luỹ thừa của 2, không phải một số nguyên tố (thường gặp ở các
hash table khác)?** Để `(table.length - 1) & hash` **tương đương chính xác** với
`hash % table.length`, nhưng nhanh hơn nhiều — phép AND bit là một trong những lệnh CPU rẻ
nhất, còn modulo (`%`) là phép chia, tốn hơn nhiều lần chu kỳ CPU. Đây chỉ đúng khi
`table.length` là luỹ thừa của 2 (`- 1` cho ra một dãy bit toàn số 1, ví dụ 16-1=15=`01111`) —
lý do `HashMap` luôn ép capacity về luỹ thừa của 2 gần nhất, kể cả khi bạn tự chỉ định một số
khác trong constructor.

## Engineering Insight

**Tại sao `HashMap` không dùng `synchronized` toàn bộ để thread-safe?** Đây là câu hỏi cho
chính `HashMap` — và câu trả lời ngắn gọn: **nó không thread-safe, có chủ đích.** Đồng bộ hoá
(khoá) mọi thao tác sẽ khiến hiệu năng đơn luồng (single-thread, trường hợp phổ biến nhất) chậm
đi đáng kể vì chi phí khoá không cần thiết khi chỉ có một thread dùng. JDK cung cấp
`ConcurrentHashMap` (Chapter 11) như một lựa chọn **riêng biệt** cho ngữ cảnh đa luồng, dùng kỹ
thuật khoá tinh vi hơn nhiều (khoá từng bucket, không khoá toàn bộ) — để những ai **không cần**
thread-safety không phải trả giá cho nó. Đây là nguyên tắc thiết kế xuyên suốt Java: không bắt
mọi người trả chi phí cho một tính năng họ không dùng tới.

**Tại sao load factor mặc định là 0.75, không phải 0.5 hay 0.9?** Đây là điểm cân bằng giữa
hai chi phí đối nghịch. Load factor thấp (ví dụ 0.5): ít collision hơn (tra cứu nhanh hơn),
nhưng lãng phí bộ nhớ nhiều hơn (một nửa bucket luôn trống) và resize thường xuyên hơn. Load
factor cao (ví dụ 0.9): tiết kiệm bộ nhớ hơn, nhưng nhiều collision hơn, tra cứu chậm dần. 0.75
được các nhà thiết kế Java (dựa trên phân tích thống kê phân bố Poisson của số phần tử mỗi
bucket) chọn làm điểm cân bằng tốt cho phần lớn trường hợp sử dụng thông thường — đủ thấp để
giữ số lượng va chạm trung bình nhỏ, đủ cao để không lãng phí bộ nhớ quá mức.

## Historical Note

```
Java 2 (1998) — Java 7
    ↓
Mỗi bucket va chạm là một danh sách liên kết ĐƠN THUẦN — worst case (mọi
key va chạm cùng 1 bucket, ví dụ do tấn công cố ý chọn hashCode() trùng
nhau — "HashDoS" attack) khiến tra cứu suy biến thành O(n), toàn bộ HashMap
chậm như một List
    ↓
Java 8 (2014)
    ↓
Treeify ra đời — bucket va chạm quá 8 phần tử (VÀ table đủ lớn, >= 64)
tự động chuyển sang cây đỏ-đen, đảm bảo worst case chỉ còn O(log n) thay
vì O(n) — giải pháp trực tiếp cho lỗ hổng HashDoS và các trường hợp
collision xấu tình cờ
```

Đây chính xác là lý do sâu xa cho câu hỏi phỏng vấn kinh điển "HashMap xử lý collision như thế
nào, khi nào chuyển từ LinkedList sang Red-Black Tree" — không phải một chi tiết ngẫu nhiên, mà
là phản ứng thiết kế trực tiếp cho một lớp lỗ hổng bảo mật/hiệu năng có thật.

## Myth vs Reality

- **Myth:** "`HashMap` luôn tra cứu O(1)."
  **Reality:** O(1) chỉ là **trung bình** (amortized), giả định hash phân bố đều. Worst case
  (trước Java 8: mọi key va chạm cùng bucket) là O(n); từ Java 8, nhờ treeify, worst case được
  cải thiện xuống O(log n) — không bao giờ là "luôn luôn O(1)" một cách tuyệt đối.

- **Myth:** "Có thể xoá phần tử trong `HashMap` khi đang duyệt bằng for-each, miễn là dùng
  `map.remove(key)` trực tiếp."
  **Reality:** Giống hệt `ArrayList`/`Collection` nói chung (Chapter 01), `HashMap` có iterator
  fail-fast — sửa cấu trúc map trực tiếp trong for-each (không qua `Iterator.remove()`) ném
  `ConcurrentModificationException`. Muốn xoá trong lúc duyệt, phải dùng
  `iterator.remove()` hoặc `map.entrySet().removeIf(...)`.

## Common Mistakes

- **Dùng key kiểu mutable** (một object có field có thể đổi sau khi đã dùng làm key) — nếu
  field tham gia `hashCode()` bị sửa sau khi thêm vào `HashMap`, entry đó "biến mất" khỏi
  đúng bucket cũ (đã cảnh báo chi tiết ở
  [Phase 1, Chapter 13](../phase-01-foundation/13-hashcode.md)) — `HashMap` là nơi hậu quả của
  lỗi này thể hiện rõ ràng nhất.
- **Không chỉ định capacity ban đầu** khi biết trước quy mô dữ liệu — gây nhiều lần resize
  không cần thiết, mỗi lần resize là O(n) để phân bổ lại toàn bộ phần tử.
- **Sửa map trực tiếp trong for-each** thay vì dùng `Iterator.remove()`.

## Production Notes

**Vấn đề:** một service tra cứu dữ liệu qua `HashMap` cache nội bộ đột nhiên chậm hẳn dưới tải
cao, dù số lượng entry không tăng đột biến.

- **Triệu chứng:** độ trễ tra cứu tăng đáng kể, CPU cao bất thường cho một thao tác "chỉ là tra
  cứu map".
- **Root cause phổ biến:** key được dùng có `hashCode()` phân bố kém (nhiều key khác nhau nhưng
  băm về rất ít giá trị) — gây collision hàng loạt vào một số ít bucket, biến tra cứu O(1) kỳ
  vọng thành gần O(n)/O(log n) trên các bucket bị quá tải. Trường hợp nghiêm trọng hơn: dữ liệu
  đến từ input người dùng, kẻ tấn công cố ý gửi các giá trị được tính toán để có cùng
  `hashCode()` (HashDoS, xem Historical Note) — dù Java 8+ đã giảm nhẹ tác động bằng treeify,
  vẫn có ảnh hưởng hiệu năng đáng kể so với trường hợp bình thường.
- **Debug:** kiểm tra phân bố `hashCode()` của các key thực tế đang dùng; nếu nghi ngờ tấn
  công, kiểm tra nguồn gốc dữ liệu key (có đến từ input không tin cậy không).
- **Solution:** cải thiện `hashCode()` của key (đảm bảo phân bố đều — dùng `Objects.hash()`
  đúng cách như đã học ở Phase 1); với dữ liệu từ nguồn không tin cậy, cân nhắc giới hạn số
  lượng entry hoặc dùng cấu trúc dữ liệu ít nhạy cảm hơn với kiểu tấn công này.
- **Prevention:** không dùng trực tiếp dữ liệu người dùng chưa qua xử lý làm key của
  `HashMap` công khai (public-facing) nếu có thể; luôn đảm bảo `hashCode()` của các class tự
  viết dùng làm key phân bố tốt.

## Debug Checklist

- [ ] Tra cứu `HashMap` chậm bất thường dưới tải cao? → nghi ngờ collision hàng loạt, kiểm tra
      phân bố `hashCode()` của key thực tế.
- [ ] Entry "biến mất" khỏi `HashMap` dù chắc chắn đã `put()`? → kiểm tra key có phải object
      mutable bị sửa field tham gia `hashCode()` sau khi thêm vào không (xem lại Phase 1
      Chapter 13).
- [ ] `ConcurrentModificationException` khi duyệt `HashMap`? → tìm chỗ sửa map trực tiếp trong
      for-each, đổi sang `Iterator.remove()`.
- [ ] Muốn xác nhận trực tiếp bucket/collision/treeify của một `HashMap` cụ thể? → dùng kỹ
      thuật Reflection y hệt Example.

## Source Code Walkthrough

```
HashMap.put(key, value)
    ↓
hash(key) = key.hashCode() ^ (key.hashCode() >>> 16)
    ↓
putVal(hash, key, value, ...)
    ↓
index = (table.length - 1) & hash
    ↓
table[index] rỗng?
    ├── Có → tạo Node mới, gán thẳng vào table[index]
    └── Không (va chạm) →
         Node hiện tại đã là TreeNode? → treeify path (chèn vào cây đỏ-đen)
         Chưa → duyệt danh sách liên kết, nối Node mới vào cuối
                đếm số phần tử trong danh sách, nếu > TREEIFY_THRESHOLD (8)
                VÀ table.length >= MIN_TREEIFY_CAPACITY (64)
                → treeifyBin() — chuyển cả bucket thành cây đỏ-đen
    ↓
++size > threshold (capacity × loadFactor)? → resize()
```

## Summary

`HashMap` tra cứu nhanh nhờ tính trực tiếp chỉ số bucket từ `hashCode()` (qua bước "spread" để
trộn đều bit cao/thấp, rồi AND với `table.length - 1`), thay vì so sánh tuần tự. Va chạm hash
(nhiều key rơi cùng bucket) được xử lý bằng danh sách liên kết, và từ Java 8, tự động
**treeify** thành cây đỏ-đen nếu một bucket có hơn 8 phần tử (và bảng đủ lớn ≥64) — đã xác nhận
toàn bộ cơ chế này bằng Reflection thực nghiệm, không chỉ lý thuyết. Resize xảy ra khi vượt quá
`capacity × 0.75` (load factor mặc định), nhân đôi kích thước và phân bổ lại toàn bộ phần tử —
tốn kém, nên luôn chỉ định capacity ban đầu khi biết trước quy mô dữ liệu. `HashMap` không
thread-safe có chủ đích — cần `ConcurrentHashMap` (Chapter 11) cho ngữ cảnh đa luồng.

## Interview Questions

**Junior**

- Explain the internal working of HashMap.
- Can we add/modify elements in a HashMap while iterating over it?

**Mid**

- What is Fail-Fast behavior in HashMap? What exception is thrown if we modify a HashMap
  during iteration?
- HashMap xử lý collision như thế nào? Khi nào chuyển từ LinkedList sang Red-Black Tree?
- Vì sao `table.length` của HashMap luôn là luỹ thừa của 2?

**Senior**

- Giải thích chi tiết bước "hash spreading" (`h ^ (h >>> 16)`). Vì sao nó cần thiết, và nó
  giải quyết vấn đề gì mà dùng thẳng `hashCode()` không giải quyết được?
- Một service dùng HashMap cache tra cứu chậm hẳn dưới tải cao dù số entry không tăng đột
  biến. Trình bày quy trình điều tra, liên hệ trực tiếp cơ chế collision/treeify đã học.
- So sánh đánh đổi giữa load factor thấp và cao — vì sao 0.75 được chọn làm mặc định?

## Exercises

- [ ] Chạy lại đúng ba ví dụ Reflection ở trên (hash spreading/collision, resize, treeify) —
      xác nhận kết quả giống hệt trên máy bạn.
- [ ] Viết một class `BadKey` với `hashCode()` luôn trả về hằng số cố định — thêm 20 phần tử
      vào `HashMap`, đo thời gian tra cứu, so sánh với 20 phần tử có `hashCode()` phân bố tốt
      — quan sát chênh lệch hiệu năng do collision hàng loạt.
- [ ] Viết một class key có field mutable tham gia `hashCode()`, thêm vào `HashMap`, sửa field
      đó, rồi thử `map.get()` với một object mới có cùng giá trị field — xác nhận không tìm
      thấy entry cũ, liên hệ lại Phase 1 Chapter 13.

## Cheat Sheet

| Khái niệm | Giá trị mặc định |
| --- | --- |
| Capacity ban đầu | 16 |
| Load factor | 0.75 |
| Ngưỡng treeify | > 8 phần tử trong 1 bucket |
| Điều kiện treeify | Bucket > 8 phần tử **VÀ** `table.length >= 64` |
| Resize | Nhân đôi kích thước, khi `size > capacity × loadFactor` |

```
index = (table.length - 1) & (hash ^ (hash >>> 16))
```

| Thao tác | Độ phức tạp trung bình | Worst case |
| --- | --- | --- |
| `get()`/`put()` | O(1) | O(log n) (từ Java 8, nhờ treeify) |

## References

- Java SE API Documentation — `java.util.HashMap`.
- OpenJDK source — `java.base/java/util/HashMap.java`, method `putVal()`, `resize()`,
  `treeifyBin()`.
- Effective Java (Joshua Bloch) — Item 11 (hashCode), liên quan trực tiếp hiệu năng HashMap.
