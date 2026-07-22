---
tags:
  - Java
  - JVM
  - GC
  - Memory
  - Foundation
---

# Garbage Collection

> Phase: Phase 1 — Java Foundation
> Chapter slug: `garbage-collection`

## Metadata

```yaml
Chapter: Garbage Collection
Phase: Phase 1 — Java Foundation
Difficulty: ★★★★
Importance: ★★★★★
Interview Frequency: 85%
Prerequisites:
  - Chapter 04 — Runtime Data Areas
  - Chapter 06 — JIT Compiler
Used Later:
  - WeakHashMap, IdentityHashMap (Phase 2) — dựa trực tiếp trên khái niệm reachability ở đây
  - equals()/hashCode(), mọi Collection (Phase 2) — vòng đời object ảnh hưởng thiết kế cache
  - JVM Tuning, Performance Tuning (Phase 8) — chọn/chỉnh GC algorithm cho production
Estimated Reading: 25 phút
Estimated Practice: 25 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Chapter 04](04-runtime-data-areas.md) — bạn đã tự tay tạo ra
> `OutOfMemoryError` bằng cách liên tục `new int[1_000_000]` mà không bao giờ dừng.

Nhưng phần lớn thời gian, chương trình Java **không** hết bộ nhớ, dù liên tục tạo object mới
suốt vòng đời chạy — có khi hàng triệu object mỗi giây trong một hệ thống xử lý request. Không
có dòng code nào bạn viết gọi `free(product)` hay `delete product` như C/C++. Vậy ai đang dọn
dẹp, và dọn dẹp **cái gì**, **khi nào**?

Câu trả lời tưởng như đơn giản ("Garbage Collector tự động dọn rác") thực ra ẩn chứa một câu
hỏi khó hơn: làm sao JVM biết một object đã "hết dùng" khi bản thân chương trình còn đang
chạy, không ai báo trước? Đây chính là bài toán mà Garbage Collection giải quyết.

## Interview Question (Central)

> Vì sao Java không có `free()`/`delete()` như C/C++ mà vẫn quản lý được bộ nhớ? Garbage
> Collector xác định một object "đã chết" bằng cách nào?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Hiểu khái niệm **reachability** — GC không đếm tham chiếu, nó truy vết từ GC Roots
- [ ] Giải thích được **Generational Hypothesis** và vì sao Heap được chia thành Young/Old
      Generation
- [ ] Phân biệt Minor GC (Young Gen) và Major/Full GC (Old Gen) về tần suất và chi phí
- [ ] Tự tay quan sát được GC hoạt động thật qua `-Xlog:gc`
- [ ] Nhận ra rằng "GC tự động" không đồng nghĩa "không thể memory leak trong Java"

## Prerequisites

- Chapter 04 — đã biết Heap là nơi object thật sự nằm, và `OutOfMemoryError` là gì.
- Chapter 06 — đã biết JVM có các cơ chế chạy nền (JIT) tối ưu song song với thực thi chính;
  GC là cơ chế chạy nền thứ hai theo tinh thần tương tự.

## Used Later

- **WeakHashMap, IdentityHashMap** (Phase 2) — các cấu trúc dữ liệu này chỉ có ý nghĩa khi đã
  hiểu reachability: `WeakHashMap` cố tình giữ tham chiếu "yếu" để không cản GC dọn key.
- **equals()/hashCode(), mọi Collection** (Phase 2) — thiết kế cache đúng cách phụ thuộc vào
  hiểu vòng đời object mà GC quản lý.
- **JVM Tuning, Performance Tuning** (Phase 8) — chọn GC algorithm (G1, ZGC...) và tinh chỉnh
  tham số là công việc tối ưu production thực tế, xây trên nền chapter này.

## Problem

Chương trình tạo object liên tục (Chapter 04), nhưng Heap có giới hạn (`-Xmx`). Nếu không ai
dọn dẹp object không còn dùng, chương trình sẽ luôn kết thúc bằng `OutOfMemoryError`, bất kể
chạy nhanh hay chậm. Nhưng "dọn dẹp" cần trả lời được câu hỏi khó: **làm sao biết một object
không còn ai dùng nữa**, khi chương trình có thể lưu tham chiếu tới nó ở bất kỳ đâu — biến cục
bộ, field của object khác, mảng, cấu trúc dữ liệu lồng nhau nhiều tầng?

## Concept

JVM giải quyết bằng **reachability** (khả năng truy cập được): một object được coi là "còn
sống" nếu có thể truy tới nó bằng một chuỗi tham chiếu bắt đầu từ một tập điểm xuất phát đặc
biệt gọi là **GC Roots** (biến cục bộ đang có trên Stack — Chapter 04, static field, tham
chiếu JNI...). Object nào **không** thể truy tới từ bất kỳ GC Root nào — dù bộ nhớ vẫn còn
chứa nó — bị coi là rác, đủ điều kiện để GC thu hồi.

Cách hiện thực dựa trên **Generational Hypothesis** (giả thuyết thế hệ): quan sát thực nghiệm
cho thấy phần lớn object "chết rất trẻ" (tạo ra, dùng ngay, bỏ đi gần như lập tức — ví dụ biến
tạm trong một vòng lặp xử lý request). Dựa trên giả thuyết này, Heap được chia thành:

- **Young Generation** — nơi object mới sinh ra. Được GC quét rất thường xuyên (**Minor GC**),
  nhanh, vì phần lớn object ở đây đã chết ngay khi quét tới.
- **Old Generation** — nơi chứa object đã sống đủ lâu qua nhiều lần Minor GC (được "thăng
  hạng" — promote). Được quét ít thường xuyên hơn nhưng tốn kém hơn mỗi lần (**Major/Full
  GC**), vì object ở đây thường lớn và nhiều hơn.

## Why?

Nếu GC quét toàn bộ Heap mỗi lần (không chia generation): với ứng dụng có Heap lớn (hàng chục
GB, phổ biến trong hệ thống production thật), mỗi lần GC sẽ dừng chương trình (**GC pause**)
rất lâu — không chấp nhận được với hệ thống cần độ trễ thấp. Chia generation cho phép GC tập
trung quét vùng nhỏ (Young Gen) thường xuyên với chi phí thấp, chỉ đụng tới vùng lớn (Old Gen)
khi thực sự cần — giảm đáng kể tổng thời gian dừng chương trình.

## How?

```
Object mới → sinh ra trong Young Generation (vùng Eden)
        │
        │  Minor GC quét Eden: object nào không còn reachable → thu hồi ngay
        │                       object nào còn sống → chuyển sang vùng Survivor
        │
        │  Object sống sót qua NHIỀU lần Minor GC liên tiếp
        ▼
   Được "thăng hạng" (promote) sang Old Generation
        │
        │  Old Generation đầy dần theo thời gian
        ▼
   Major/Full GC quét Old Generation (chậm hơn, ít xảy ra hơn nhiều)
```

Việc xác định "reachable hay không" được thực hiện bằng cách xuất phát từ GC Roots, đi theo
mọi tham chiếu có thể tới (giống duyệt đồ thị): object nào **không** bị chạm tới trong lần
duyệt này là rác.

## Visualization

```
GC Roots                          HEAP
┌──────────────┐
│ Stack thread1 │──┐
│  (biến cục bộ)│  │        Young Generation          Old Generation
├──────────────┤  │       ┌──────────────────┐      ┌─────────────┐
│ Static fields │──┼──────▶│ [A] [B]    [C]    │─────▶│ [D] (sống lâu)│
└──────────────┘  │       │  ●    ●      ×     │      └─────────────┘
                    │       └──────────────────┘
                    │        [C] không có mũi tên nào trỏ tới từ GC Roots
                    │        → KHÔNG reachable → RÁC, Minor GC thu hồi
                    └──────▶ [A], [B] còn reachable → sống sót
```

`[C]` vẫn còn nằm vật lý trong bộ nhớ Heap cho tới khi GC quét tới — nhưng về mặt logic, nó đã
"chết" ngay từ thời điểm không còn tham chiếu nào từ GC Roots trỏ tới nó, không phải tại thời
điểm GC thực sự chạy.

## Example

Quan sát GC hoạt động thật, dùng cờ logging chuẩn (tương tự tinh thần `-Xlog:class+load` ở
Chapter 01, nhưng cho GC):

```java
import java.util.ArrayList;
import java.util.List;

public class GcDemo {
    public static void main(String[] args) {
        List<byte[]> garbage = new ArrayList<>();
        for (int i = 0; i < 200; i++) {
            garbage.add(new byte[100_000]);
            if (garbage.size() > 20) {
                garbage.remove(0); // bỏ tham chiếu tới object cũ nhất
            }
        }
        System.out.println("Xong, giữ lại " + garbage.size() + " block");
    }
}
```

Chạy với `java -Xmx20m -Xlog:gc GcDemo` (giới hạn Heap nhỏ để GC xảy ra và log lại được),
kết quả thật (JDK 17, GC mặc định là G1):

```
[0.003s][info][gc] Using G1
[0.014s][info][gc] GC(0) Pause Young (Normal) (G1 Evacuation Pause) 10M->3M(24M) 0.308ms
[0.015s][info][gc] GC(1) Pause Young (Normal) (G1 Evacuation Pause) 13M->3M(24M) 0.188ms
Xong, giữ lại 20 block
```

Đọc log: `10M->3M(24M)` nghĩa là trước lần GC này Heap đang dùng 10MB, sau khi GC xong chỉ còn
3MB đang dùng (tổng Heap cấp phát 24MB) — đúng bằng chứng cho `garbage.remove(0)` liên tục làm
object cũ mất reachability, GC thu hồi thành công, chỉ mất **dưới 1 mili-giây** mỗi lần (
`0.308ms`, `0.188ms`) — Minor GC trên Young Gen thường rất nhanh, đúng như phần Why? đã giải
thích.

## Deep Dive

**Vì sao log ghi "Using G1"?** Từ Java 9, **G1 (Garbage-First)** là GC mặc định của HotSpot,
thay thế Parallel GC (mặc định trước đó). G1 chia Heap thành nhiều vùng nhỏ (region) thay vì
hai khối Young/Old liền mạch truyền thống, ưu tiên dọn trước những vùng có nhiều rác nhất
("Garbage-First" — đúng tên gọi) để tối ưu tỉ lệ dọn rác trên mỗi mili-giây dừng chương trình.
Đây không phải chapter đi sâu vào từng thuật toán GC (dành cho Phase 8 — Performance Tuning) —
ở mức nền tảng, chỉ cần biết: **có nhiều GC algorithm khác nhau, đánh đổi khác nhau giữa
throughput và độ trễ**, và G1 là lựa chọn cân bằng mặc định hiện nay.

## Engineering Insight

**Vì sao GC lại dựa trên "reachability" thay vì cách đơn giản hơn: đếm số tham chiếu tới mỗi
object (reference counting), giống một số ngôn ngữ khác?** Reference counting có một lỗ hổng
nghiêm trọng: **chu trình tham chiếu** (A trỏ tới B, B trỏ ngược lại A, nhưng không ai từ bên
ngoài còn trỏ tới cả A lẫn B) — đếm tham chiếu sẽ không bao giờ về 0 cho cả hai, dù chúng thực
chất đã "chết" theo nghĩa reachability. GC dựa trên truy vết từ GC Roots (giống duyệt đồ thị)
xử lý đúng trường hợp này một cách tự nhiên: nếu không ai từ GC Roots truy được tới A hay B,
cả hai đều bị coi là rác — bất kể chúng có trỏ vòng vào nhau hay không. Đây là lý do thiết kế
"truy vết từ gốc" được hầu hết GC hiện đại (bao gồm JVM) chọn thay vì đếm tham chiếu đơn thuần.

## Historical Note

```
Trước Java 9
    ↓
Parallel GC là mặc định — tối ưu throughput (tổng công việc xử lý được),
chấp nhận GC pause có thể khá dài
    ↓
Java 9 (2017)
    ↓
G1 trở thành GC mặc định — cân bằng throughput và độ trễ tốt hơn
    ↓
Java 14 (2020) — CMS (Concurrent Mark Sweep, một GC độ trễ thấp từng phổ biến,
    đã bị đánh dấu deprecated từ Java 9) chính thức bị GỠ BỎ khỏi JDK
    ↓
Từ Java 15 trở đi — ZGC và Shenandoah (GC độ trễ cực thấp, pause dưới
    mili-giây dù Heap hàng trăm GB) dần trưởng thành, sẵn sàng cho production
```

## Myth vs Reality

- **Myth:** "Java có GC nên không bao giờ bị memory leak."
  **Reality:** GC chỉ thu hồi object **không còn reachable**. Nếu code vô tình giữ tham
  chiếu mãi mãi tới object không dùng nữa (ví dụ: thêm vào `static List` rồi quên xoá — xem
  lại [Chapter 04 — Production Notes](04-runtime-data-areas.md)), object đó **vẫn
  reachable** theo đúng định nghĩa, GC sẽ không bao giờ đụng tới nó — đây chính là "memory
  leak kiểu Java", vẫn hoàn toàn có thể xảy ra dù có GC.

- **Myth:** "GC chạy thì chương trình luôn bị dừng (stop-the-world), luôn gây giật lag."
  **Reality:** Đúng là nhiều pha GC cần dừng toàn bộ thread ứng dụng (stop-the-world) để đảm
  bảo an toàn khi truy vết reachability, nhưng như ví dụ ở trên, Minor GC hiện đại (G1) chỉ
  mất **dưới 1 mili-giây** trong tình huống thông thường — không phải lúc nào "GC chạy" cũng
  đồng nghĩa "chương trình giật lag đáng kể".

## Common Mistakes

- **Coi `System.gc()` là cách "dọn rác ngay lập tức".** `System.gc()` chỉ là một **gợi ý**
  cho JVM nên chạy GC — JVM có quyền bỏ qua hoàn toàn. Gọi `System.gc()` thường xuyên trong
  code nghiệp vụ là một anti-pattern, không đảm bảo hiệu quả và có thể gây pause không cần
  thiết.
- **Đặt biến `= null` một cách máy móc ngay sau khi dùng xong**, nghĩ rằng bắt buộc phải làm
  vậy để "giúp" GC. Với biến cục bộ, JVM đã đủ thông minh để nhận ra biến không còn được dùng
  tới nữa trong phần code còn lại của method (dựa trên phân tích luồng dữ liệu) — việc gán
  `null` thủ công thường không cần thiết, chỉ thực sự có ý nghĩa với field sống lâu (như
  field của object tồn tại lâu, hoặc phần tử trong collection — đúng tình huống
  `garbage.remove(0)` ở ví dụ trên).
- **Nhầm "Full GC chạy nhiều" là bình thường.** Full GC chạy thường xuyên là dấu hiệu cảnh
  báo (Old Generation liên tục đầy) — không phải hiện tượng bình thường như Minor GC.

## Best Practices

- Không gọi `System.gc()` trong code nghiệp vụ; nếu cần ép GC chạy khi debug/benchmark, thực
  hiện thủ công ngoài code (qua công cụ như `jcmd <pid> GC.run`), không nhúng vào logic ứng
  dụng.
- Khi thiết kế cache hoặc cấu trúc dữ liệu sống lâu, luôn tự hỏi: "ai giữ tham chiếu tới các
  entry này, và tham chiếu đó có bao giờ được gỡ bỏ không?" — trả lời rõ câu này ngay từ đầu
  thiết kế, thay vì xử lý sự cố memory leak sau này.
- Theo dõi tần suất và thời lượng GC (Minor vs Full) như một chỉ số sức khoẻ ứng dụng thường
  trực trong production, không chỉ khi có sự cố.

## Production Notes

**Vấn đề:** ứng dụng ngày càng chậm, log cho thấy Full GC chạy ngày càng thường xuyên, cuối
cùng crash `OutOfMemoryError`.

- **Triệu chứng:** độ trễ tăng dần trong nhiều giờ/ngày (không đột ngột), tỉ lệ thời gian
  dành cho GC trên tổng thời gian chạy tăng dần — chỉ số kinh điển là **GC overhead**: nếu
  JVM dành hơn 98% thời gian cho GC mà chỉ thu hồi được dưới 2% Heap mỗi lần, HotSpot tự ném
  `OutOfMemoryError: GC overhead limit exceeded` — một biến thể khác của OOM, khác với
  `Java heap space` đã gặp ở Chapter 04.
- **Root cause:** memory leak thật sự (object vẫn reachable dù logic nghiệp vụ coi như "đã
  dùng xong") — Old Generation liên tục đầy dần vì ngày càng nhiều object bị kẹt lại đó mãi
  mãi, không phải do tải tăng đột biến.
- **Debug:** theo dõi biểu đồ Heap usage theo thời gian (không chỉ một thời điểm) — leak thật
  cho thấy đường xu hướng đi lên đều đặn dù có GC, khác với dùng bộ nhớ bình thường (dao động
  lên xuống theo tải nhưng có đáy ổn định). Kết hợp heap dump như đã nêu ở
  [Chapter 04 — Production Notes](04-runtime-data-areas.md) để tìm chính xác GC Root nào
  đang giữ tham chiếu không cần thiết.
- **Solution:** loại bỏ tham chiếu gây leak (xem lại Common Mistakes và Chapter 04).
- **Prevention:** thiết lập cảnh báo (alert) dựa trên xu hướng GC overhead / tần suất Full
  GC, không chỉ dựa trên mức sử dụng CPU/RAM tức thời — phát hiện leak sớm trước khi ảnh
  hưởng người dùng thật.

## Debug Checklist

- [ ] Full GC chạy ngày càng thường xuyên, độ trễ tăng dần theo thời gian (không đột ngột)?
      → nghi ngờ memory leak, không phải tải tăng.
- [ ] `OutOfMemoryError: GC overhead limit exceeded`? → phân biệt với `Java heap space`
      (Chapter 04) — đây là dấu hiệu GC đang "cố gắng tuyệt vọng" mà không hiệu quả, thường
      cũng dẫn tới cùng một hướng điều tra: leak.
- [ ] Nghi ngờ có nơi gọi `System.gc()` trong code nghiệp vụ? → tìm và loại bỏ, đây thường là
      anti-pattern không giải quyết vấn đề gốc.
- [ ] Muốn xác nhận GC nào đang chạy, tần suất ra sao? → chạy với `-Xlog:gc` như ví dụ trong
      chapter, quan sát trực tiếp.

## Source Code Walkthrough

Ở mức chapter nền tảng, không cần đọc source C++ của G1/thuật toán GC cụ thể trong HotSpot —
đây là một trong những phần phức tạp và thay đổi nhiều nhất giữa các phiên bản JDK, để dành
cho Phase 8 (Performance Tuning) khi cần tối ưu production thật với một GC algorithm cụ thể.
Công cụ thực tế để quan sát hành vi GC ở mức chapter này là `-Xlog:gc` (đã dùng ở Example) và
`jcmd <pid> GC.class_histogram` để xem loại object nào đang chiếm Heap nhiều nhất.

## Summary

Garbage Collector không đếm tham chiếu — nó truy vết **reachability** từ một tập gốc gọi là
GC Roots; object nào không thể truy tới được coi là rác. Dựa trên **Generational Hypothesis**
(hầu hết object chết trẻ), Heap chia thành Young Generation (quét thường xuyên bằng Minor GC,
rất nhanh) và Old Generation (quét ít hơn nhưng tốn kém hơn bằng Major/Full GC). "GC tự động"
không có nghĩa là Java miễn nhiễm với memory leak — nếu code vẫn giữ tham chiếu (dù vô tình)
tới object không dùng nữa, object đó vẫn reachable và GC sẽ không bao giờ thu hồi nó.

## Interview Questions

**Junior**

- Garbage Collector xác định một object là "rác" dựa trên tiêu chí gì?
- Young Generation và Old Generation khác nhau như thế nào?

**Mid**

- Vì sao Heap được chia thành nhiều generation thay vì quét toàn bộ mỗi lần GC?
- Java có GC, vậy memory leak có còn xảy ra được không? Giải thích bằng một ví dụ cụ thể.

**Senior**

- Vì sao GC dựa trên reachability (truy vết từ GC Roots) thay vì reference counting? Chu
  trình tham chiếu (circular reference) gây vấn đề gì cho reference counting mà
  reachability xử lý được?
- Một service sản xuất cho thấy tần suất Full GC tăng dần trong 3 ngày liên tục rồi crash
  OOM. Trình bày quy trình điều tra bạn sẽ làm, từ log tới heap dump tới xác định root cause.

## Exercises

- [ ] Chạy lại đúng ví dụ `GcDemo` ở trên với `-Xlog:gc`, thử tăng/giảm `-Xmx`, quan sát tần
      suất và kích thước các lần GC thay đổi ra sao.
- [ ] Sửa `GcDemo` để **không** gọi `garbage.remove(0)` (giữ lại toàn bộ 200 block thay vì
      chỉ giữ 20) — chạy lại với cùng `-Xmx20m`, dự đoán trước điều gì sẽ xảy ra, sau đó xác
      nhận bằng cách chạy thật.
- [ ] Dùng `jcmd <pid> GC.class_histogram` trên một chương trình Java đang chạy (có thể chính
      là `GcDemo` sửa để chạy vòng lặp dài hơn) để xem loại object nào đang chiếm Heap nhiều
      nhất tại một thời điểm.

## Cheat Sheet

| Khái niệm | Ý nghĩa |
| --- | --- |
| GC Roots | Điểm xuất phát truy vết: biến cục bộ trên Stack, static field, tham chiếu JNI |
| Reachability | Object truy được từ GC Roots → còn sống; không truy được → rác |
| Young Generation | Nơi object mới sinh; quét bằng Minor GC, thường xuyên, nhanh |
| Old Generation | Nơi object sống lâu (đã promote); quét bằng Major/Full GC, ít hơn, chậm hơn |
| Minor GC | Quét Young Gen |
| Major/Full GC | Quét Old Gen (hoặc toàn bộ Heap) |
| G1 | GC mặc định từ Java 9 trở đi |

| Cờ JVM | Việc gì |
| --- | --- |
| `-Xlog:gc` | Log cơ bản mỗi lần GC chạy |
| `-XX:+HeapDumpOnOutOfMemoryError` | Tự dump Heap khi OOM (xem Chapter 04) |
| `jcmd <pid> GC.run` | Gợi ý JVM chạy GC ngay (không đảm bảo, dùng khi debug) |
| `jcmd <pid> GC.class_histogram` | Xem loại object nào chiếm Heap nhiều nhất |

## References

- Oracle: "Java Platform, Standard Edition HotSpot Virtual Machine Garbage Collection Tuning
  Guide".
- JEP 248: Make G1 the Default Garbage Collector.
- JEP 363: Remove the Concurrent Mark Sweep (CMS) Garbage Collector.
