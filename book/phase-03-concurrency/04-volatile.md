---
tags:
  - Java
  - volatile
  - Concurrency
---

# volatile

> Phase: Phase 3 — Concurrency
> Chapter slug: `volatile`

## Metadata

```yaml
Chapter: volatile
Phase: Phase 3 — Concurrency
Difficulty: ★★★★
Importance: ★★★★★
Interview Frequency: 80%
Prerequisites:
  - Chapter 03 — Memory Visibility
Used Later:
  - synchronized (Chapter 05) — đảm bảo cả visibility LẪN atomicity, mạnh hơn volatile
  - Atomic (Chapter 12) — giải pháp đúng đắn cho bài toán "đếm" mà volatile không giải quyết được
Estimated Reading: 20 phút
Estimated Practice: 20 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Chapter 03](03-memory-visibility.md) — bạn đã tự tay tái hiện một `worker` thread
> không bao giờ thấy được `stopRequested = true` mà `main` đã gán.

Sửa đúng một từ khoá:

```java
static volatile boolean stopRequested = false; // thêm "volatile"
```

Chạy lại chính xác chương trình đó:

```
Worker dung lai, count = -811462510
Worker da dung ngay lap tuc nho volatile!
```

`worker` dừng lại **ngay lập tức**. Một từ khoá duy nhất giải quyết trọn vẹn vấn đề đã ám ảnh
suốt chapter trước. Nhưng đừng vội coi `volatile` là "cây đũa thần" cho mọi vấn đề đa luồng —
chapter này sẽ chỉ ra chính xác nó giải quyết được gì, và một giới hạn quan trọng mà rất nhiều
người học sai: `volatile` **không** làm cho phép toán `counter++` trở nên an toàn.

## Interview Question (Central)

> `volatile` giải quyết vấn đề gì? Nó có làm cho `counter++` trở thành thao tác an toàn với
> nhiều thread không?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Giải thích chính xác `volatile` đảm bảo điều gì: mọi lần đọc luôn lấy giá trị mới nhất từ
      bộ nhớ chính, mọi lần ghi được đẩy xuống bộ nhớ chính ngay lập tức
- [ ] Tự tay xác nhận `volatile` sửa được vấn đề visibility (Chapter 03)
- [ ] Tự tay chứng minh `volatile` **không** giải quyết được race condition cho thao tác không
      nguyên tử như `counter++`
- [ ] Biết khi nào `volatile` là đủ (cờ hiệu boolean, publish một tham chiếu) và khi nào cần
      công cụ mạnh hơn (`synchronized`, `Atomic`)

## Prerequisites

- Chapter 03 — đã hiểu rõ vấn đề visibility và nguyên nhân (JIT cache biến vào thanh ghi).

## Used Later

- **synchronized** (Chapter 05) — đảm bảo cả visibility lẫn tính nguyên tử (atomicity) của một
  khối code, mạnh hơn `volatile` nhưng có chi phí khác (mutual exclusion).
- **Atomic** (Chapter 12) — giải pháp đúng đắn, hiệu quả cho chính bài toán "đếm an toàn với
  nhiều thread" mà chapter này chỉ ra `volatile` không giải quyết được.

## Problem

Vấn đề visibility (Chapter 03) cần một cách để **buộc** JIT Compiler không được tối ưu "cache
vào thanh ghi" cho một biến cụ thể — cần một tín hiệu tường minh, ở cấp ngôn ngữ, nói rằng
"biến này có thể bị thay đổi bởi thread khác, luôn đọc/ghi trực tiếp với bộ nhớ chính".

## Concept

`volatile` là một từ khoá đánh dấu một field, cam kết hai điều:

1. **Mọi lần đọc** biến `volatile` luôn lấy giá trị **mới nhất** từ bộ nhớ chính — JIT Compiler
   bị cấm tối ưu "cache vào thanh ghi" cho biến này.
2. **Mọi lần ghi** biến `volatile` được đẩy xuống bộ nhớ chính **ngay lập tức**, không trì hoãn
   trong cache riêng của CPU.

`volatile` **chỉ** đảm bảo visibility — nó **không** đảm bảo tính nguyên tử (atomicity) cho các
thao tác gồm nhiều bước (đọc, sửa, ghi).

## Why?

Nếu không có `volatile`: như đã thấy ở Chapter 03, JIT có toàn quyền tối ưu theo góc nhìn đơn
luồng, có thể khiến một thread không bao giờ thấy thay đổi từ thread khác. `volatile` cho phép
lập trình viên **tường minh** đánh dấu đúng những biến cần đảm bảo này, để JIT biết **không**
được áp dụng tối ưu "cache vào thanh ghi" cho riêng chúng — trong khi vẫn giữ được tối ưu đó
cho mọi biến khác không cần chia sẻ giữa các thread.

## How?

```java
static volatile boolean stopRequested = false;

// Thread khác:
stopRequested = true; // GHI: đẩy xuống bộ nhớ chính NGAY, mọi thread khác thấy được
```

Nhưng với thao tác gồm nhiều bước:

```java
static volatile int counter = 0;
counter++; // THỰC CHẤT là 3 bước: (1) ĐỌC counter, (2) CỘNG 1, (3) GHI counter
           // volatile chỉ đảm bảo MỖI bước đọc/ghi riêng lẻ là visible,
           // KHÔNG đảm bảo cả 3 bước diễn ra "liền mạch", không bị chen ngang
```

## Visualization

```
counter++ (counter là volatile) thực chất là 3 bước KHÔNG NGUYÊN TỬ:

Thread A: (1) ĐỌC counter = 5
Thread B:                        (1) ĐỌC counter = 5   ← ĐỌC CÙNG GIÁ TRỊ 5!
Thread A: (2) tính 5 + 1 = 6
Thread A: (3) GHI counter = 6   (visible ngay nhờ volatile)
Thread B:                        (2) tính 5 + 1 = 6
Thread B:                        (3) GHI counter = 6   ← GHI ĐÈ, MẤT 1 LẦN TĂNG!

Kết quả: counter = 6, dù đáng lẽ phải là 7 sau 2 lần "++"
```

## Example

**`volatile` sửa đúng vấn đề Chapter 03:**

```java
public class VisibilityFixed {
    static volatile boolean stopRequested = false; // CO volatile

    public static void main(String[] args) throws InterruptedException {
        Thread worker = new Thread(() -> {
            int count = 0;
            while (!stopRequested) { count++; }
            System.out.println("Worker dung lai, count = " + count);
        });
        worker.start();

        Thread.sleep(1000);
        System.out.println("Main: dat stopRequested = true");
        stopRequested = true;

        worker.join(3000);
        System.out.println(worker.isAlive() ? "Worker VAN CHUA DUNG" : "Worker da dung ngay lap tuc nho volatile!");
    }
}
```

Kết quả thật (JDK 17):

```
Main: dat stopRequested = true
Worker dung lai, count = -811462510
Worker da dung ngay lap tuc nho volatile!
```

**Nhưng `volatile` KHÔNG sửa được bài toán đếm** — chạy 10 thread, mỗi thread tăng
`volatile int counter` 100.000 lần:

```java
static volatile int counter = 0;
// 10 thread, mỗi thread: for (...) counter++;
```

Kết quả thật:

```
volatile counter++ : 240104 (mong doi 1000000)
```

Mất tới **76%** số lần tăng — gần như không khá hơn `int` thường (Chapter 05 sẽ so sánh trực
tiếp). `volatile` chỉ đảm bảo mỗi bước đọc/ghi riêng lẻ visible ngay, nhưng không ngăn được hai
thread cùng đọc **cùng một giá trị cũ** trước khi bất kỳ ai ghi giá trị mới — đúng minh hoạ ở
Visualization.

## Deep Dive

**Vì sao ví dụ `stopRequested` hoạt động đúng với `volatile`, nhưng `counter++` thì không —
khác biệt cốt lõi là gì?** `stopRequested = true` là **một thao tác ghi duy nhất, nguyên tử tự
nhiên** (gán một giá trị `boolean` là một lệnh CPU đơn, không thể bị "chen ngang" giữa chừng).
`counter++` là **ba thao tác riêng biệt** (đọc, cộng, ghi) — `volatile` đảm bảo từng thao tác
trong ba thao tác đó là visible ngay khi nó xảy ra, nhưng **không** đảm bảo không có thao tác
nào của thread khác "chen vào giữa" ba bước đó. Quy tắc ghi nhớ: `volatile` phù hợp cho biến mà
mọi lần cập nhật là **gán trực tiếp một giá trị mới, độc lập với giá trị cũ** (cờ hiệu, publish
một tham chiếu tới object bất biến — Phase 1 Chapter 11); **không** phù hợp cho biến mà lần cập
nhật **phụ thuộc vào giá trị hiện tại** của chính nó (đếm, cộng dồn, so sánh-rồi-cập-nhật).

## Engineering Insight

**`volatile` liên quan gì tới `happens-before` — khái niệm cốt lõi của Java Memory Model
(JMM)?** JMM định nghĩa quan hệ `happens-before` giữa các thao tác của các thread khác nhau —
nếu thao tác X `happens-before` thao tác Y, mọi hiệu ứng của X (bao gồm ghi bộ nhớ) được đảm
bảo **visible** với Y. Ghi vào một biến `volatile` và đọc lại **chính biến đó** từ thread khác
tạo ra một quan hệ `happens-before` tường minh — đây chính là cơ chế chính thức (không chỉ là
"quy ước") đứng sau việc `stopRequested = true` (Story) được đảm bảo visible cho `worker`.
`synchronized` (Chapter 05) cũng tạo `happens-before` (giữa unlock của một thread và lock tiếp
theo của thread khác trên **cùng** một lock) — đây là lý do cả hai đều giải quyết được vấn đề
visibility, dù cơ chế cụ thể (thanh ghi CPU vs. hàng đợi lock) khác nhau hoàn toàn.

## Historical Note

```
Trước Java 5 (2004)
    ↓
Đặc tả volatile GỐC (Java 1.0) có nhiều lỗ hổng — ví dụ KHÔNG đảm bảo
"reordering" (thứ tự thực thi lệnh) bị chặn đúng cách, khiến volatile
kém tin cậy hơn nhiều so với ngày nay trong một số trường hợp phức tạp
    ↓
Java 5 (2004) — JSR 133 định nghĩa lại Java Memory Model HOÀN CHỈNH,
volatile được TĂNG CƯỜNG ngữ nghĩa đáng kể — thêm đảm bảo chống reordering
xung quanh thao tác volatile, làm cho nó đáng tin cậy như mô tả trong
chapter này
```

Đây là lý do khi đọc tài liệu/sách xuất bản trước 2004, ngữ nghĩa `volatile` mô tả có thể khác
(yếu hơn) so với Java hiện đại — một chi tiết lịch sử quan trọng để tránh nhầm lẫn khi tham
khảo tài liệu cũ.

## Myth vs Reality

- **Myth:** "`volatile` làm cho mọi thao tác trên biến đó trở nên thread-safe hoàn toàn."
  **Reality:** Xem Example — `volatile int counter` vẫn mất dữ liệu nghiêm trọng dưới nhiều
  thread cùng `counter++`. `volatile` chỉ đảm bảo visibility, không đảm bảo atomicity cho thao
  tác nhiều bước.

- **Myth:** "`volatile` và `synchronized` về cơ bản làm cùng một việc, chỉ khác cú pháp."
  **Reality:** `synchronized` (Chapter 05) đảm bảo cả visibility **lẫn** mutual exclusion
  (loại trừ lẫn nhau — chỉ một thread thực thi khối code tại một thời điểm); `volatile` chỉ
  đảm bảo visibility, không có mutual exclusion.

## Common Mistakes

- **Dùng `volatile` cho biến counter/cộng dồn** — lỗi trọng tâm của chapter, đúng minh hoạ ở
  Example.
- **Nghĩ rằng thêm `volatile` là "đã xử lý xong" mọi vấn đề đa luồng cho một biến** mà không
  phân tích kỹ bản chất thao tác trên biến đó (gán trực tiếp hay phụ thuộc giá trị cũ).
- **Dùng `volatile` thay cho `synchronized` khi cần bảo vệ nhiều biến liên quan tới nhau cùng
  lúc** — `volatile` chỉ hoạt động đúng cho từng biến độc lập, không đảm bảo tính nhất quán
  giữa nhiều biến `volatile` khác nhau được cập nhật "cùng lúc" về mặt logic.

## Best Practices

- Dùng `volatile` cho: cờ hiệu boolean điều khiển vòng lặp/shutdown, publish một tham chiếu
  tới object bất biến (Phase 1, Chapter 11) sau khi khởi tạo xong.
- Không dùng `volatile` cho: biến đếm, tổng dồn, hoặc bất kỳ cập nhật nào phụ thuộc giá trị
  hiện tại — dùng `Atomic` (Chapter 12) hoặc `synchronized` (Chapter 05) thay thế.
- Khi không chắc chắn liệu `volatile` có đủ hay không, mặc định chọn giải pháp mạnh hơn
  (`synchronized`/`Atomic`) — cái giá hiệu năng thường nhỏ hơn nhiều so với cái giá của một bug
  race condition khó phát hiện.

## Production Notes

**Vấn đề:** sau khi "sửa" một bug visibility bằng cách thêm `volatile` vào một biến đếm request
đang xử lý, hệ thống hết bug "treo" nhưng số liệu đếm vẫn thỉnh thoảng sai lệch nhẹ so với thực
tế.

- **Triệu chứng:** không còn hiện tượng "treo"/không phản hồi (đúng là visibility đã được sửa),
  nhưng số liệu thống kê vẫn không chính xác tuyệt đối dưới tải cao.
- **Root cause:** nhầm lẫn giữa hai vấn đề (visibility và race condition, xem Deep Dive) —
  `volatile` sửa đúng phần visibility (tại sao hệ thống từng "treo"), nhưng biến đó là bộ đếm
  (`counter++`), vẫn còn race condition chưa được giải quyết.
- **Debug:** xác nhận biến bị sai lệch số liệu có phải đang dùng `volatile` cho thao tác
  `++`/`+=` không — nếu có, đây chính là nguyên nhân còn sót lại.
- **Solution:** đổi từ `volatile int` sang `AtomicInteger` (Chapter 12) hoặc bọc
  `synchronized` quanh thao tác đếm.
- **Prevention:** khi review code sửa lỗi đa luồng, luôn tự hỏi "biến này có thao tác nào phụ
  thuộc giá trị hiện tại của chính nó không?" — nếu có, `volatile` đơn thuần không bao giờ đủ.

## Debug Checklist

- [ ] Đã thêm `volatile` nhưng vẫn còn dữ liệu sai lệch/mất mát dưới tải nhiều thread? →
      kiểm tra biến đó có đang bị `++`/`+=`/cập nhật phụ thuộc giá trị cũ không — nếu có,
      `volatile` không đủ, cần `Atomic`/`synchronized`.
- [ ] Cần xác định `volatile` có đủ cho một biến cụ thể không? → tự hỏi: "mọi lần ghi có phải
      là gán trực tiếp một giá trị mới, độc lập với giá trị cũ không?" Nếu có, đủ; nếu không,
      chưa đủ.

## Source Code Walkthrough

Ở tầng bytecode, field `volatile` được đánh dấu bằng cờ `ACC_VOLATILE` trong class file — JIT
Compiler (HotSpot) khi biên dịch bytecode truy cập field này sẽ chèn thêm các **memory barrier**
(hàng rào bộ nhớ — lệnh CPU đặc biệt ngăn compiler/CPU sắp xếp lại thứ tự đọc/ghi qua ranh giới
đó, và buộc đồng bộ với bộ nhớ chính) xung quanh mỗi lần đọc/ghi — đây chính là cơ chế cấp thấp
hiện thực đúng ngữ nghĩa `happens-before` đã bàn ở Engineering Insight, không phải một "phép
màu" trừu tượng mà là các lệnh CPU cụ thể được chèn thêm vào bytecode đã biên dịch.

## Summary

`volatile` đảm bảo **visibility**: mọi lần đọc lấy giá trị mới nhất từ bộ nhớ chính, mọi lần
ghi đẩy xuống bộ nhớ chính ngay lập tức — đã chứng minh sửa đúng vấn đề Chapter 03. Nhưng
`volatile` **không** đảm bảo **atomicity** cho thao tác nhiều bước như `counter++` — đã chứng
minh bằng thực nghiệm: 10 thread tăng biến `volatile int` 100.000 lần mỗi thread chỉ đạt 24%
kết quả đúng. Quy tắc: dùng `volatile` cho gán trực tiếp giá trị mới (cờ hiệu, publish tham
chiếu bất biến); dùng `synchronized`/`Atomic` (Chapter 05, 12) cho cập nhật phụ thuộc giá trị
hiện tại.

## Interview Questions

**Junior**

- `volatile` dùng để làm gì?
- `volatile` có làm cho `counter++` an toàn với nhiều thread không?

**Mid**

- Giải thích vì sao `volatile boolean flag = true` hoạt động đúng nhưng
  `volatile int counter; counter++` thì không.
- volatile khác synchronized như thế nào?

**Senior**

- Giải thích khái niệm `happens-before` và vai trò của `volatile` trong việc tạo ra quan hệ
  đó — liên hệ Java Memory Model (JMM).
- Một hệ thống sau khi thêm `volatile` hết bug "treo" nhưng vẫn còn sai lệch số liệu nhẹ dưới
  tải cao. Giải thích chính xác nguyên nhân còn sót lại và cách sửa triệt để.

## Exercises

- [ ] Chạy lại đúng ví dụ `VisibilityFixed` ở trên, xác nhận `worker` dừng ngay lập tức.
- [ ] Chạy lại đúng ví dụ đếm với `volatile int counter`, xác nhận kết quả sai lệch đáng kể so
      với kỳ vọng.
- [ ] Thử publish một tham chiếu tới một object bất biến (Phase 1, Chapter 11) qua một field
      `volatile` từ một thread, đọc nó từ thread khác — xác nhận luôn thấy đúng object đã publish
      (không phải `null` hay trạng thái dở dang), minh hoạ use case chính đáng của `volatile`.

## Cheat Sheet

| Đảm bảo | `volatile` |
| --- | --- |
| Visibility (đọc/ghi đồng bộ với bộ nhớ chính) | Có |
| Atomicity (thao tác nhiều bước là nguyên tử) | Không |
| Mutual exclusion (loại trừ lẫn nhau) | Không |

| Dùng `volatile` cho | Không dùng `volatile` cho |
| --- | --- |
| Cờ hiệu boolean (`stopRequested`) | Biến đếm (`counter++`) |
| Publish tham chiếu bất biến sau khi khởi tạo | Cập nhật phụ thuộc giá trị hiện tại |

## References

- Java Language Specification (JLS) — Chapter 17.4: Memory Model.
- JSR 133: Java Memory Model and Thread Specification Revision.
- Java Concurrency in Practice (Brian Goetz) — Chapter 3.1: Visibility.
