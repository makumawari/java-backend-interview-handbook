---
tags:
  - Java
  - JVM
  - Memory
  - Heap
  - Stack
  - Foundation
---

# Runtime Data Areas

> Phase: Phase 1 — Java Foundation
> Chapter slug: `runtime-data-areas`

## Metadata

```yaml
Chapter: Runtime Data Areas
Phase: Phase 1 — Java Foundation
Difficulty: ★★★
Importance: ★★★★★
Interview Frequency: 90%
Prerequisites:
  - Chapter 01 — Một chương trình Java chạy như thế nào?
  - Chapter 03 — Class Loader
Used Later:
  - Execution Engine (Chapter 05) — Stack là nơi Interpreter thao tác operand stack
  - Garbage Collection (Chapter 07) — Heap là "sân chơi" chính của GC
  - HashMap, mọi Collection (Phase 2) — object luôn nằm trên Heap
  - JVM Tuning (Phase 8) — -Xmx, -Xss chỉnh trực tiếp các vùng nhớ ở đây
Estimated Reading: 25 phút
Estimated Practice: 30 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Chapter 01](01-mot-chuong-trinh-java-chay-nhu-the-nao.md), ví dụ
> `Product product = new Product("Áo thun", 199_000);`

Câu lệnh đó tạo ra **hai** thứ, không phải một: một object `Product` thật (chứa tên, giá), và
một biến `product` giữ *tham chiếu* tới object đó. Hai thứ này không nằm cùng một chỗ trong bộ
nhớ.

Bằng chứng: viết một method đệ quy gọi chính nó vô hạn lần, chương trình sẽ crash với
`StackOverflowError`. Còn viết một vòng lặp tạo object mới liên tục không bao giờ dừng,
chương trình sẽ crash với `OutOfMemoryError`. **Hai lỗi khác nhau, hai vùng nhớ khác nhau.**
Chapter này trả lời chính xác: JVM chia bộ nhớ runtime thành những vùng nào, vùng nào chứa
cái gì, và vì sao phải chia như vậy.

## Interview Question (Central)

> JVM lưu trữ dữ liệu ở đâu khi chương trình đang chạy? Heap và Stack khác nhau như thế nào,
> và vì sao sự khác biệt đó lại quan trọng?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Kể tên 5 vùng nhớ runtime chính: PC Register, JVM Stack, Native Method Stack, Heap,
      Method Area/Metaspace
- [ ] Giải thích được vùng nào **per-thread** (mỗi thread một bản riêng), vùng nào
      **shared** (dùng chung toàn JVM)
- [ ] Tự tay tái hiện được `StackOverflowError` và `OutOfMemoryError`, hiểu vì sao mỗi lỗi
      xảy ra ở một vùng nhớ khác nhau
- [ ] Vẽ được sơ đồ biến cục bộ (Stack) trỏ tới object (Heap) qua tham chiếu

## Prerequisites

- Chapter 01 — biết object được tạo ra khi chạy `new`.
- Chapter 03 — biết Class Loader nạp thông tin class vào bộ nhớ trước khi tạo được object.

## Used Later

- **Execution Engine** (Chapter 05) — Interpreter thao tác trực tiếp trên *operand stack*,
  một phần của JVM Stack sẽ mô tả ở đây.
- **Garbage Collection** (Chapter 07) — GC chỉ dọn Heap; Stack và PC Register được giải phóng
  tự động khi method kết thúc, không cần GC.
- **Mọi cấu trúc dữ liệu ở Phase 2** (HashMap, ArrayList...) — mọi object của các cấu trúc
  này luôn nằm trên Heap, dù biến tham chiếu tới chúng có thể là biến cục bộ (trên Stack).
- **JVM Tuning** (Phase 8) — cờ `-Xmx`/`-Xms` chỉnh kích thước Heap, `-Xss` chỉnh kích thước
  Stack mỗi thread, đều tác động trực tiếp lên các vùng nhớ ở chapter này.

## Problem

Một chương trình đang chạy cần lưu nhiều loại dữ liệu có vòng đời và đặc tính rất khác nhau:

- Biến cục bộ (`int i`, tham chiếu `product`) — chỉ tồn tại trong lúc method đang chạy, biến
  mất ngay khi method return.
- Object thật (`new Product(...)`) — có thể tồn tại lâu hơn method tạo ra nó (ví dụ được lưu
  vào một List ở method khác), vòng đời không gắn với một method cụ thể.
- Thông tin về chính class (tên method, tên field, bytecode...) — cần dùng chung cho **mọi**
  object cùng loại, không nên tạo lại mỗi lần `new`.

Nếu lưu tất cả vào một vùng nhớ duy nhất, không thể tối ưu quản lý (biến cục bộ có thể cấp
phát/giải phóng cực nhanh theo kiểu ngăn xếp, trong khi object thì không đoán trước được khi
nào hết dùng) và không thể cô lập lỗi (một thread đệ quy vô hạn sẽ làm crash toàn bộ chương
trình thay vì chỉ chính thread đó).

## Concept

JVM chia bộ nhớ runtime thành nhiều vùng (**Runtime Data Areas**), mỗi vùng phục vụ một loại
dữ liệu và có chiến lược quản lý riêng. Chia theo tiêu chí quan trọng nhất: **per-thread** hay
**shared**:

**Per-thread** (mỗi thread có một bản riêng, thread khác không đụng vào được):

- **PC Register (Program Counter)** — con trỏ giữ địa chỉ bytecode đang thực thi của thread.
- **JVM Stack** — chồng các "frame", mỗi lần gọi method tạo một frame mới chứa biến cục bộ và
  operand stack (chi tiết ở Chapter 05) của method đó.
- **Native Method Stack** — tương tự JVM Stack nhưng dành cho code native (JNI, không phải
  bytecode Java).

**Shared** (toàn bộ JVM, mọi thread, dùng chung):

- **Heap** — nơi mọi object (`new ...`) thực sự được cấp phát.
- **Method Area** (hiện thực trong HotSpot gọi là **Metaspace** từ Java 8) — thông tin class:
  tên, cấu trúc, bytecode method, static field.

## Why?

Nếu Stack là shared thay vì per-thread: một thread đệ quy vô hạn sẽ làm crash *toàn bộ*
chương trình (mọi thread khác cũng mất bộ nhớ Stack), thay vì chỉ riêng thread đó nhận
`StackOverflowError`. Per-thread Stack cô lập lỗi hiệu quả.

Nếu Heap không tách khỏi Stack: object không thể "sống lâu hơn" method tạo ra nó — mất khả
năng trả về object từ method, lưu object vào field của object khác, hay chia sẻ object giữa
nhiều thread — tức là mất gần như toàn bộ lập trình hướng đối tượng thực dụng.

## How?

```
new Product("Áo thun", 199_000)
        │
        ├── (1) Đọc thông tin class Product (đã có sẵn trong Method Area/Metaspace
        │        từ lúc Class Loader nạp — xem Chapter 03) để biết cần cấp phát bao
        │        nhiêu byte, field nào.
        │
        ├── (2) Cấp phát vùng nhớ cho object thật trên HEAP.
        │
        └── (3) Biến `product` — chỉ là MỘT THAM CHIẾU (giống địa chỉ) — được lưu
                 trong frame hiện tại trên JVM STACK của thread đang chạy.
```

## Visualization

```
Thread-1 (JVM Stack)              HEAP (shared, mọi thread)
┌───────────────────────┐         ┌──────────────────────────┐
│ frame: main()          │         │  Product object          │
│  product ──────────────┼────────▶│   name = "Áo thun"       │
│  (chỉ là tham chiếu)   │         │   price = 199000          │
└───────────────────────┘         └──────────────────────────┘

Method Area / Metaspace (shared)
┌─────────────────────────────────────────┐
│ class Product { String name; int price; │
│   Product(String,int) {...}             │
│   String getName() {...} ...  }         │
└─────────────────────────────────────────┘
```

Biến `product` trên Stack không chứa "Áo thun" hay `199000` — nó chỉ chứa một tham chiếu
(mũi tên) trỏ sang object thật nằm trên Heap. Khi `main()` kết thúc, frame trên Stack bị huỷ
ngay lập tức — nhưng object trên Heap **không nhất thiết bị huỷ theo**, nó chỉ mất đi một
tham chiếu trỏ tới (xem Chapter 07 — Garbage Collection).

## Example

**Tái hiện `StackOverflowError`** (mỗi lần gọi method tạo một frame mới trên JVM Stack; đệ
quy vô hạn = Stack đầy):

```java
public class StackDemo {
    static int depth = 0;
    static void recurse() {
        depth++;
        recurse();
    }
    public static void main(String[] args) {
        try {
            recurse();
        } catch (StackOverflowError e) {
            System.out.println("StackOverflowError sau khoảng " + depth + " lần gọi đệ quy");
        }
    }
}
```

Chạy thật (JDK 17, cấu hình Stack mặc định):

```
StackOverflowError sau khoảng 45892 lần gọi đệ quy
```

**Tái hiện `OutOfMemoryError`** (mỗi vòng lặp tạo object mới trên Heap; Heap giới hạn nhỏ để
lỗi xảy ra nhanh, dễ quan sát):

```java
import java.util.ArrayList;
import java.util.List;

public class OomDemo {
    public static void main(String[] args) {
        List<int[]> blocks = new ArrayList<>();
        try {
            while (true) {
                blocks.add(new int[1_000_000]);
            }
        } catch (OutOfMemoryError e) {
            System.out.println("OutOfMemoryError sau khi cấp phát " + blocks.size()
                + " block (mỗi block ~4MB)");
        }
    }
}
```

Chạy với `java -Xmx64m OomDemo` (giới hạn Heap còn 64MB để lỗi xảy ra nhanh), kết quả thật:

```
OutOfMemoryError sau khi cấp phát 15 block (mỗi block ~4MB)
```

15 × 4MB ≈ 60MB — khớp với giới hạn 64MB đã đặt. Chú ý: `blocks` (biến `List`) nằm trên Stack,
nhưng **nội dung** của `blocks` (từng mảng `int[]`) nằm trên Heap — đúng lý thuyết ở phần
Concept.

## Deep Dive

**Vì sao hai lỗi trên đều là `Error`, không phải `Exception`?** Cả hai kế thừa từ
`java.lang.Error`, nhóm dành cho các vấn đề nghiêm trọng mà ứng dụng thường **không nên** cố
bắt và tiếp tục chạy bình thường (khác với `Exception` — sẽ bàn kỹ ở Chapter 14). Việc bắt
được `StackOverflowError`/`OutOfMemoryError` trong ví dụ trên chỉ nhằm mục đích minh hoạ —
trong code thật, gặp các lỗi này thường là dấu hiệu cần sửa thiết kế (đệ quy sai, rò rỉ bộ
nhớ), không phải thứ để `catch` rồi tiếp tục.

**Heap không phải một khối đồng nhất.** HotSpot chia Heap thành các "generation" dựa trên giả
thuyết: phần lớn object chết rất trẻ (tạo ra rồi bị bỏ gần như ngay lập tức — ví dụ biến tạm
trong vòng lặp). Việc chia vùng cho object mới (Young Generation) tách khỏi vùng cho object
sống lâu (Old Generation) giúp GC quét vùng object trẻ thường xuyên mà không phải quét toàn bộ
Heap mỗi lần — cơ chế đầy đủ thuộc về Chapter 07, ở đây chỉ cần biết Heap **không phẳng**.

## Engineering Insight

**Vì sao JVM Stack là per-thread nhưng Heap là shared — đây có phải điều hiển nhiên không?**
Không hẳn. Đó là một lựa chọn thiết kế đánh đổi giữa **tốc độ** và **khả năng chia sẻ dữ
liệu**. Biến cục bộ chỉ cần thiết trong phạm vi một lời gọi method của một thread cụ thể — cấp
phát/giải phóng theo kiểu ngăn xếp (push khi vào method, pop khi return) là O(1) tuyệt đối,
không cần GC can thiệp. Object thì ngược lại: bản chất của lập trình hướng đối tượng là chia
sẻ object giữa nhiều method, nhiều thread — bắt buộc phải có một vùng nhớ chung, và vùng chung
đó buộc phải trả giá bằng việc cần một cơ chế dọn dẹp phức tạp hơn (GC) vì không thể biết
trước "khi nào hết dùng" như Stack.

## Historical Note

```
Trước Java 8
    ↓
Method Area hiện thực bằng PermGen (Permanent Generation) — nằm TRONG Heap,
kích thước cố định (dễ gặp OutOfMemoryError: PermGen space khi nạp quá nhiều class,
phổ biến với các framework sinh class động lúc runtime)
    ↓
Java 8 (2014)
    ↓
PermGen bị loại bỏ, thay bằng Metaspace — nằm ở NATIVE MEMORY (ngoài Heap),
tự động mở rộng theo nhu cầu (mặc định không giới hạn, có thể set qua -XX:MaxMetaspaceSize)
```

Đây là lý do nếu bạn tra cứu tài liệu Java cũ, sẽ thấy nhắc tới lỗi
`OutOfMemoryError: PermGen space` — lỗi này gần như không còn gặp ở Java 8 trở lên, thay vào
đó (hiếm hơn nhiều) là `OutOfMemoryError: Metaspace`.

## Myth vs Reality

- **Myth:** "Biến cục bộ và object nó tham chiếu tới nằm cùng một chỗ trong bộ nhớ."
  **Reality:** Biến cục bộ (tham chiếu) nằm trên Stack; object thật nằm trên Heap — hai vùng
  nhớ tách biệt hoàn toàn (xem Visualization).

- **Myth:** "Object lớn tạo `StackOverflowError` nếu tạo quá nhiều."
  **Reality:** Tạo quá nhiều object gây `OutOfMemoryError` (Heap đầy).
  `StackOverflowError` chỉ liên quan tới **độ sâu lời gọi method** (đệ quy quá sâu), không
  liên quan tới kích thước object.

## Common Mistakes

- **Đệ quy không có điều kiện dừng, hoặc điều kiện dừng sai** → `StackOverflowError`. Rất phổ
  biến với người mới học đệ quy.
- **Giữ tham chiếu tới object không còn cần dùng** (ví dụ: thêm vào `static List` rồi quên
  không bao giờ xoá) → object không bao giờ đủ điều kiện để GC dọn, dẫn tới
  `OutOfMemoryError` dần dần theo thời gian — đây chính là "memory leak kiểu Java", sẽ bàn kỹ
  ở Chapter 07.
- **Nhầm lẫn kích thước biến tham chiếu với kích thước object.** Một biến `Product product`
  luôn chiếm cùng một lượng bộ nhớ nhỏ trên Stack (kích thước một con trỏ) bất kể object
  `Product` nó trỏ tới lớn hay nhỏ.

## Best Practices

- Tránh đệ quy không có giới hạn rõ ràng; với dữ liệu có thể rất sâu/lớn, cân nhắc chuyển
  sang vòng lặp hoặc dùng cấu trúc dữ liệu tường minh thay vì dựa vào Stack của ngôn ngữ.
- Khi debug nghi ngờ rò rỉ bộ nhớ, đặt câu hỏi "vẫn còn ai giữ tham chiếu tới object này?"
  trước khi nghĩ tới việc tăng `-Xmx` — tăng Heap chỉ trì hoãn triệu chứng, không sửa
  nguyên nhân.

## Production Notes

**Vấn đề:** ứng dụng chạy ổn định nhiều giờ/ngày, rồi đột ngột crash với
`java.lang.OutOfMemoryError: Java heap space`.

- **Triệu chứng:** thời gian phản hồi chậm dần trước khi crash (GC phải chạy ngày càng
  thường xuyên để cố giải phóng chỗ trống — xem Chapter 07), rồi crash hẳn.
- **Root cause thường gặp:** memory leak kiểu Java — object không còn dùng tới nhưng vẫn còn
  ai đó giữ tham chiếu (ví dụ: `static Map` dùng làm cache nhưng không bao giờ evict,
  `ThreadLocal` không được `remove()`, listener/callback đăng ký nhưng không hủy đăng ký).
- **Debug:** lấy heap dump (`jmap -dump` hoặc `-XX:+HeapDumpOnOutOfMemoryError` để tự động
  dump lúc crash), phân tích bằng công cụ (Eclipse MAT, VisualVM) để tìm object nào đang
  chiếm nhiều bộ nhớ nhất và **ai đang giữ tham chiếu tới chúng** (GC root path).
- **Solution:** loại bỏ tham chiếu không cần thiết (đặt `null`, gọi `remove()`, dùng
  `WeakHashMap`/cache có TTL thay vì `Map` thường cho mục đích cache).
- **Prevention:** review kỹ mọi nơi dùng biến `static` chứa collection; thiết lập giám sát
  xu hướng sử dụng Heap theo thời gian (không chỉ giá trị tức thời) để phát hiện leak sớm,
  trước khi crash thật xảy ra.

## Debug Checklist

- [ ] `StackOverflowError`? → tìm đệ quy thiếu điều kiện dừng, không phải vấn đề bộ nhớ Heap.
- [ ] `OutOfMemoryError: Java heap space`? → nghi ngờ leak, lấy heap dump, tìm GC root đang
      giữ tham chiếu không cần thiết — đừng vội tăng `-Xmx`.
- [ ] `OutOfMemoryError: Metaspace`? → thường do nạp quá nhiều class động (proxy, class sinh
      lúc runtime bởi framework) — kiểm tra có đang tạo class động không cần thiết không.
- [ ] Hiệu năng chậm dần theo thời gian trước khi crash? → dấu hiệu GC đang phải làm việc
      ngày càng nhiều — dấu hiệu sớm của memory leak, đừng đợi tới lúc crash mới điều tra.

## Source Code Walkthrough

Ở mức chapter này, không có "source Java" để đọc — các vùng nhớ này là cấu trúc nội bộ của
JVM (hiện thực bằng C++ trong HotSpot), không phải API Java bạn gọi trực tiếp. Công cụ thực tế
để "nhìn thấy" chúng là `jmap`/`jcmd` (xem Cheat Sheet) và heap dump — cách quan sát gián tiếp
qua công cụ, thay vì đọc source.

## Summary

JVM chia bộ nhớ runtime thành các vùng có đặc tính khác nhau: **per-thread** (PC Register,
JVM Stack, Native Method Stack — cô lập theo từng thread, cấp phát/giải phóng cực nhanh theo
kiểu ngăn xếp) và **shared** (Heap chứa object thật, Method Area/Metaspace chứa thông tin
class). Biến cục bộ chỉ là tham chiếu nằm trên Stack; object thật luôn nằm trên Heap — đây là
lý do đệ quy sai gây `StackOverflowError` (Stack đầy) còn tạo object không kiểm soát gây
`OutOfMemoryError` (Heap đầy), hai lỗi hoàn toàn khác nhau về bản chất dù đều là "hết bộ nhớ".

## Interview Questions

**Junior**

- Heap và Stack khác nhau như thế nào? Cái nào chứa object, cái nào chứa biến cục bộ?
- `StackOverflowError` và `OutOfMemoryError` khác nhau ở đâu?

**Mid**

- Vì sao JVM Stack là per-thread nhưng Heap là shared? Đánh đổi nằm ở đâu?
- Metaspace khác PermGen (Java 7 trở về trước) như thế nào? Vì sao lại đổi?

**Senior**

- Một ứng dụng chạy ổn định rồi dần chậm lại và crash `OutOfMemoryError` sau vài ngày chạy
  liên tục. Bạn sẽ điều tra theo quy trình nào, dùng công cụ gì?
- Giải thích tại sao "tăng `-Xmx`" thường chỉ là giải pháp tạm thời cho vấn đề memory leak,
  không phải giải pháp triệt để.

## Exercises

- [ ] Chạy lại đúng hai ví dụ `StackDemo` và `OomDemo` ở trên. Thử đổi `-Xmx64m` thành các
      giá trị khác nhau, quan sát số block thay đổi tương ứng thế nào.
- [ ] Thêm cờ `-Xss256k` (giảm kích thước Stack) khi chạy `StackDemo`, so sánh số lần đệ quy
      trước khi crash với lúc chạy mặc định — giải thích vì sao lại đổi.
- [ ] Viết một chương trình tạo một `static List` rồi liên tục thêm object vào nhưng không
      bao giờ xoá — mô phỏng một memory leak thật, quan sát bằng `jmap -histo:live <pid>`
      xem loại object nào đang tăng nhanh nhất.

## Cheat Sheet

| Vùng nhớ | Phạm vi | Chứa gì | Lỗi hết bộ nhớ tương ứng |
| --- | --- | --- | --- |
| PC Register | Per-thread | Địa chỉ bytecode đang chạy | — |
| JVM Stack | Per-thread | Frame method, biến cục bộ, operand stack | `StackOverflowError` |
| Native Method Stack | Per-thread | Lời gọi native/JNI | `StackOverflowError` |
| Heap | Shared | Object thật (`new ...`) | `OutOfMemoryError: Java heap space` |
| Method Area / Metaspace | Shared | Thông tin class, static field | `OutOfMemoryError: Metaspace` |

| Cờ JVM | Chỉnh gì |
| --- | --- |
| `-Xmx` | Kích thước Heap tối đa |
| `-Xms` | Kích thước Heap ban đầu |
| `-Xss` | Kích thước JVM Stack mỗi thread |
| `-XX:MaxMetaspaceSize` | Giới hạn Metaspace (mặc định gần như không giới hạn) |
| `-XX:+HeapDumpOnOutOfMemoryError` | Tự động dump Heap khi crash OOM, phục vụ debug |

## References

- Java Virtual Machine Specification (Oracle) — Chapter 2.5: Run-Time Data Areas.
- Oracle: "Java Platform, Standard Edition HotSpot Virtual Machine Garbage Collection Tuning
  Guide" — phần mô tả các vùng Heap.
- JEP liên quan tới Metaspace — ghi chú phát hành Java 8 (loại bỏ PermGen).
