---
tags:
  - Java
  - Concurrency
  - Memory Visibility
  - JMM
---

# Memory Visibility

> Phase: Phase 3 — Concurrency
> Chapter slug: `memory-visibility`

## Metadata

```yaml
Chapter: Memory Visibility
Phase: Phase 3 — Concurrency
Difficulty: ★★★★★
Importance: ★★★★★
Interview Frequency: 70%
Prerequisites:
  - Chapter 01 — Thread
  - Phase 1, Chapter 06 — JIT Compiler
  - Phase 1, Chapter 04 — Runtime Data Areas (Heap)
Used Later:
  - volatile (Chapter 04) — giải pháp trực tiếp cho đúng vấn đề chapter này nêu ra
  - synchronized (Chapter 05) — cũng đảm bảo visibility, không chỉ mutual exclusion
Estimated Reading: 25 phút
Estimated Practice: 20 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

Chạy đoạn code sau — một thread `worker` chạy vòng lặp `while (!stopRequested)`, `main` chờ 1
giây rồi đặt `stopRequested = true`:

```java
static boolean stopRequested = false; // KHÔNG volatile

Thread worker = new Thread(() -> {
    while (!stopRequested) { /* làm việc */ }
    System.out.println("Worker dung lai");
});
worker.start();
Thread.sleep(1000);
stopRequested = true; // main dat gia tri moi
```

Trực giác nói: sau khi `main` đặt `stopRequested = true`, `worker` sẽ sớm thấy giá trị mới và
dừng lại. Thực tế đo được:

```
Sau 3s, worker VAN CHUA DUNG (van con isAlive=true) -- van de VISIBILITY that!
```

**Ba giây sau**, `worker` vẫn chạy như không có chuyện gì xảy ra — nó **không bao giờ** thấy
giá trị mới của `stopRequested`, dù `main` đã gán xong từ lâu, và cả hai đều truy cập đúng một
biến trên Heap. Đây không phải bug hiếm gặp — nó là hệ quả trực tiếp, có thể dự đoán được, của
chính điều bạn đã học ở [Phase 1, Chapter 06 — JIT Compiler](../phase-01-foundation/06-jit-compiler.md).

## Interview Question (Central)

> Vì sao một thread có thể không bao giờ thấy được giá trị mới mà một thread khác đã gán cho
> một biến, dù cả hai đều truy cập đúng biến đó trên Heap?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Tự tay tái hiện được vấn đề visibility bằng thực nghiệm — không chỉ đọc lý thuyết
- [ ] Giải thích được nguyên nhân: JIT Compiler (Phase 1, Chapter 06) tối ưu bằng cách cache
      giá trị biến vào thanh ghi CPU, không đọc lại từ bộ nhớ chính mỗi lần
- [ ] Hiểu **visibility** là một vấn đề tách biệt hoàn toàn với **race condition** (dữ liệu sai
      do nhiều thread cùng ghi) — dù cả hai đều thuộc nhóm "vấn đề concurrency"
- [ ] Biết mối liên hệ giữa visibility và Java Memory Model (JMM) ở mức khái niệm

## Prerequisites

- Chapter 01 — hiểu Thread cơ bản.
- Phase 1, Chapter 06 — hiểu JIT Compiler tối ưu code dựa trên hành vi runtime.
- Phase 1, Chapter 04 — hiểu Heap là vùng nhớ dùng chung giữa các thread.

## Used Later

- **volatile** (Chapter 04) — giải pháp trực tiếp, đơn giản nhất cho chính xác vấn đề chapter
  này minh hoạ.
- **synchronized** (Chapter 05) — ngoài mutual exclusion (loại trừ lẫn nhau), nó còn đảm bảo
  visibility — một điểm rất hay bị bỏ sót.

## Problem

CPU hiện đại không đọc/ghi trực tiếp vào RAM cho mỗi lệnh — quá chậm. Chúng dùng nhiều tầng
**cache** (bộ nhớ đệm cực nhanh, riêng cho từng lõi CPU) để tăng tốc. Khi một thread chạy trên
lõi CPU A ghi một giá trị, giá trị đó có thể chỉ nằm trong cache của lõi A một thời gian, **chưa
được đẩy xuống RAM** (bộ nhớ chính, nơi mọi lõi đều thấy được). Một thread khác chạy trên lõi B
đọc từ cache của **chính nó** — nếu cache B chưa được cập nhật, nó sẽ đọc **giá trị cũ**, dù
giá trị mới đã tồn tại "ở đâu đó" trong hệ thống.

## Concept

**Memory visibility** (khả năng nhìn thấy bộ nhớ) là câu hỏi: một thay đổi mà thread A thực
hiện trên một biến, thread B có **đảm bảo** thấy được nó hay không, và thấy **khi nào**. Trong
Java, nếu không có cơ chế đồng bộ hoá tường minh (`volatile`, `synchronized`, hoặc các lớp
`java.util.concurrent`), **không có đảm bảo nào cả** — thread B có thể thấy giá trị mới ngay
lập tức, sau một khoảng trễ, hoặc **không bao giờ**, tuỳ vào tối ưu hoá cụ thể của JIT Compiler
và kiến trúc CPU.

## Why?

Nếu JVM buộc mọi lần đọc/ghi biến đều phải đồng bộ trực tiếp với RAM (bỏ qua cache CPU hoàn
toàn): chương trình sẽ chạy **chậm hơn rất nhiều**, mất đi lợi ích của cache CPU — một trong
những cơ chế tăng tốc quan trọng nhất của phần cứng hiện đại. Java chọn để lập trình viên
**tường minh khai báo** biến nào cần đảm bảo visibility (`volatile`, Chapter 04) thay vì áp đặt
chi phí đó lên **mọi** biến — đúng triết lý "không bắt mọi người trả giá cho thứ họ không dùng"
đã thấy nhiều lần trước đó (ví dụ `HashMap` không tự đồng bộ hoá, Phase 2 Chapter 08).

## How?

Nguyên nhân kỹ thuật cụ thể gây ra hiện tượng ở Story: JIT Compiler (Phase 1, Chapter 06), khi
biên dịch vòng lặp `while (!stopRequested)` thành mã máy tối ưu, nhận thấy **trong phạm vi vòng
lặp**, không có gì (mà nó biết) làm thay đổi `stopRequested` — nó tối ưu bằng cách đọc giá trị
`stopRequested` **một lần**, lưu vào thanh ghi CPU, rồi lặp lại kiểm tra thanh ghi đó **mãi
mãi**, không bao giờ đọc lại từ bộ nhớ chính. Đây là tối ưu **hoàn toàn hợp lệ** theo góc nhìn
đơn luồng (single-threaded) của JIT — nó không "biết" (và theo mặc định, không cần biết) rằng
một thread khác có thể thay đổi biến đó bất cứ lúc nào.

## Visualization

```
Lõi CPU 1 (chạy "main")          Lõi CPU 2 (chạy "worker")
┌─────────────────┐              ┌─────────────────┐
│ Cache lõi 1       │              │ Cache lõi 2       │
│ stopRequested=true│              │ stopRequested=false│ ← JIT đã cache vào
└────────┬─────────┘              └────────┬─────────┘   thanh ghi, KHÔNG
         │                                  │              đọc lại RAM nữa
         ▼                                  ▼
    ┌──────────────── RAM (bộ nhớ chính) ────────────────┐
    │           stopRequested = true (đã ghi)              │
    └──────────────────────────────────────────────────────┘

Lõi 2 KHÔNG BAO GIỜ đọc lại RAM trong vòng lặp đã được JIT tối ưu
→ worker KHÔNG BAO GIỜ thấy giá trị true, dù RAM đã có giá trị đúng
```

## Example

Tái hiện đúng hiện tượng ở Story — một bug thật, tái lập được, không phải mô tả lý thuyết
suông:

```java
public class VisibilityDemo {
    static boolean stopRequested = false; // KHONG volatile

    public static void main(String[] args) throws InterruptedException {
        Thread worker = new Thread(() -> {
            int count = 0;
            while (!stopRequested) {
                count++;
            }
            System.out.println("Worker dung lai, count = " + count);
        });
        worker.start();

        Thread.sleep(1000);
        System.out.println("Main: dat stopRequested = true");
        stopRequested = true;

        worker.join(3000);
        if (worker.isAlive()) {
            System.out.println("Sau 3s, worker VAN CHUA DUNG (van con isAlive=true) -- van de VISIBILITY that!");
        } else {
            System.out.println("Worker da dung binh thuong");
        }
        System.exit(0);
    }
}
```

Kết quả thật (JDK 17):

```
Main: dat stopRequested = true
Sau 3s, worker VAN CHUA DUNG (van con isAlive=true) -- van de VISIBILITY that!
```

`worker` không bao giờ dừng trong 3 giây chờ — chương trình phải gọi `System.exit(0)` để thoát
cưỡng bức. Đây là bằng chứng thực nghiệm trực tiếp: **không phải mọi lần chạy code này đều tái
hiện đúng bug** (phụ thuộc JIT có quyết định tối ưu theo đúng cách đó hay không, tuỳ máy/JVM
cụ thể) — chính sự **không chắc chắn** này mới là điều đáng sợ nhất của loại bug này trong thực
tế.

## Deep Dive

**Vì sao "không phải lúc nào cũng tái hiện được" lại là điều tệ nhất, không phải điều đỡ tệ
hơn?** Vì nó khiến bug loại này gần như không thể phát hiện qua testing thông thường. Chương
trình có thể chạy **hoàn toàn đúng** hàng nghìn lần trên máy dev (JIT chưa kịp tối ưu tới mức
gây bug, hoặc kiến trúc CPU/JVM cụ thể không biểu hiện rõ), rồi bất ngờ "treo" trong production
sau khi chạy đủ lâu để JIT tối ưu sâu (Phase 1, Chapter 06: "JIT warmup" — càng chạy lâu, JIT
càng tối ưu tích cực). Đây là lý do memory visibility được xếp vào nhóm lỗi nguy hiểm bậc nhất
trong lập trình đa luồng: hoàn toàn hợp lệ về logic đơn luồng, chỉ lộ ra trong đúng điều kiện
đa luồng + tối ưu hoá cụ thể, thời điểm và cách lộ ra không đoán trước được.

## Engineering Insight

**Visibility khác Race Condition (đã gặp ở Phase 2, Chapter 11) như thế nào — dù cả hai đều là
"vấn đề concurrency"?** Đây là nhầm lẫn phổ biến, cần phân biệt rõ:

- **Race condition** (Phase 2, Chapter 11 — ví dụ `HashMap` mất dữ liệu dưới nhiều thread) là
  vấn đề về **thứ tự và tính nguyên tử** của nhiều thao tác — nhiều thread cùng đọc-sửa-ghi một
  giá trị mà không có sự phối hợp, dẫn tới ghi đè lẫn nhau.
- **Visibility** (chapter này) là vấn đề về **thời điểm một thread nhìn thấy** một giá trị đã
  được ghi — dù chỉ có **một** thread ghi (`main`, chỉ ghi một lần), thread khác vẫn có thể
  không bao giờ thấy được giá trị đó.

Hai vấn đề này **độc lập** với nhau — một biến có thể gặp vấn đề visibility mà không hề có
race condition (ví dụ ở Story, chỉ `main` ghi, `worker` chỉ đọc), và ngược lại. Nhầm lẫn hai
khái niệm này dẫn tới áp dụng sai giải pháp: `volatile` (Chapter 04) giải quyết visibility
nhưng **không** giải quyết race condition nếu có nhiều thread cùng ghi (sẽ minh hoạ rõ ở
Chapter 04); cần `synchronized`/`Atomic` (Chapter 05, 12) cho race condition.

## Historical Note

Vấn đề memory visibility không phải đặc thù riêng của Java — nó là hệ quả của kiến trúc CPU đa
lõi hiện đại (cache riêng từng lõi) kết hợp với các compiler tối ưu tích cực (không chỉ JIT —
compiler tĩnh như `gcc` cũng có vấn đề tương tự với biến chia sẻ giữa thread nếu không dùng
đúng cơ chế đồng bộ). Java Memory Model (JMM) — đặc tả chính thức quy định chính xác khi nào
một thread **đảm bảo** thấy được thay đổi của thread khác — được định nghĩa lại hoàn chỉnh và
chặt chẽ hơn nhiều trong JSR 133 (2004, đi cùng Java 5), sau khi phiên bản JMM gốc (Java 1.0)
bị phát hiện có nhiều lỗ hổng và mơ hồ nghiêm trọng, khiến nhiều tối ưu hoá compiler hợp lệ có
thể phá vỡ những giả định mà lập trình viên tưởng là hiển nhiên.

## Myth vs Reality

- **Myth:** "Miễn là biến nằm trên Heap (dùng chung giữa các thread), mọi thread đều thấy
  được giá trị mới nhất ngay lập tức."
  **Reality:** Hoàn toàn sai — đã chứng minh bằng thực nghiệm ở Example. Heap chỉ là nơi dữ
  liệu **tồn tại**, không đảm bảo gì về việc **khi nào** một thread khác thấy được thay đổi
  mới nhất.

- **Myth:** "Bug về memory visibility rất hiếm gặp trong thực tế, chỉ là lý thuyết sách vở."
  **Reality:** Nó là một trong những nguyên nhân phổ biến nhất của các bug "chỉ xảy ra trong
  production, không tái hiện được trên máy dev" liên quan tới đa luồng — cực kỳ thực tế, không
  phải lý thuyết suông.

## Common Mistakes

- **Dùng biến `boolean`/cờ hiệu (flag) thường để giao tiếp giữa các thread** (như
  `stopRequested` ở Story) mà không đánh dấu `volatile` (Chapter 04) — lỗi phổ biến bậc nhất
  trong lập trình đa luồng Java.
- **Tin rằng test pass nhiều lần trên máy dev nghĩa là code đa luồng đã đúng** — visibility bug
  có thể không lộ ra trong môi trường test (ít CPU, JIT chưa tối ưu đủ sâu) nhưng lộ ra trong
  production.
- **Nhầm lẫn visibility với race condition**, áp dụng sai công cụ sửa lỗi (xem Engineering
  Insight).

## Best Practices

- Bất kỳ biến nào được đọc bởi một thread và ghi bởi thread khác, không có cơ chế đồng bộ nào
  khác bảo vệ, phải được đánh dấu `volatile` (Chapter 04) tối thiểu.
- Không dựa vào việc "test pass nhiều lần" để kết luận code đa luồng đúng — bug visibility có
  thể ẩn mình rất lâu.
- Khi nghi ngờ một bug đa luồng "chập chờn, chỉ xảy ra dưới tải cao hoặc chạy lâu", luôn cân
  nhắc khả năng visibility là nguyên nhân gốc.

## Production Notes

**Vấn đề:** một service dùng một cờ hiệu `boolean` để yêu cầu một worker thread nền dừng lại
êm đẹp (graceful shutdown) khi ứng dụng tắt — hoạt động đúng lúc test, nhưng đôi khi worker
thread **không bao giờ dừng**, khiến ứng dụng "treo" khi cố tắt (hoặc container health check
timeout).

- **Triệu chứng:** shutdown hoạt động đúng đa số lần, nhưng thỉnh thoảng "treo" vô thời hạn,
  đặc biệt sau khi ứng dụng đã chạy một thời gian dài (JIT đã tối ưu sâu — đúng hiện tượng ở
  Story).
- **Root cause:** cờ hiệu dừng không được đánh dấu `volatile` — chính xác lỗi đã minh hoạ trong
  chapter này.
- **Debug:** lấy thread dump (Chapter 02) lúc "treo", xác nhận worker thread vẫn ở trạng thái
  `RUNNABLE` (đang chạy vòng lặp), không phải `BLOCKED`/`WAITING` — dấu hiệu đặc trưng của
  visibility bug (khác deadlock, nơi thread sẽ ở `BLOCKED`).
- **Solution:** đánh dấu cờ hiệu là `volatile` (Chapter 04).
- **Prevention:** coi mọi biến chia sẻ giữa các thread mà không có `synchronized`/`Lock` bảo vệ
  là một "nghi phạm visibility" mặc định — review kỹ trong code review, đặc biệt các cờ hiệu
  điều khiển vòng lặp/shutdown.

## Debug Checklist

- [ ] Thread nền không bao giờ dừng dù đã "yêu cầu dừng"? → kiểm tra cờ hiệu điều khiển có
      `volatile` không.
- [ ] Bug đa luồng "chập chờn", tệ hơn khi ứng dụng chạy lâu? → nghi ngờ visibility, liên hệ
      hiện tượng JIT warmup (Phase 1, Chapter 06).
- [ ] Thread dump cho thấy thread "treo" vẫn ở `RUNNABLE` (không `BLOCKED`)? → không phải
      deadlock, nghi ngờ vòng lặp bị "kẹt" do visibility.

## Source Code Walkthrough

Không có "source OpenJDK" cụ thể để đọc cho vấn đề này — nó là hệ quả của tương tác giữa JIT
Compiler (HotSpot C1/C2, Phase 1 Chapter 06) và kiến trúc bộ nhớ phần cứng (cache coherence
protocol của CPU), không phải một đoạn logic Java cụ thể. Công cụ thực tế để quan sát: chạy
cùng chương trình với `-Xint` (Phase 1, Chapter 06 — tắt hoàn toàn JIT, chỉ dùng Interpreter) —
bug visibility ở Example thường **biến mất** khi tắt JIT, vì Interpreter không thực hiện tối
ưu "cache biến vào thanh ghi" như JIT — một cách gián tiếp nhưng cụ thể để xác nhận đúng JIT là
nguyên nhân.

## Summary

**Memory visibility** là vấn đề về việc một thread có đảm bảo thấy được giá trị mới mà thread
khác vừa ghi hay không — không có đảm bảo nào nếu thiếu cơ chế đồng bộ tường minh
(`volatile`/`synchronized`). Nguyên nhân cụ thể: JIT Compiler (Phase 1, Chapter 06) có thể tối
ưu bằng cách cache giá trị biến vào thanh ghi CPU, không đọc lại từ bộ nhớ chính — hoàn toàn
hợp lệ theo góc nhìn đơn luồng, nhưng gây ra thread "không bao giờ" thấy thay đổi trong ngữ
cảnh đa luồng — đã chứng minh bằng thực nghiệm tái hiện được. Visibility là vấn đề **độc lập**
với race condition (Phase 2, Chapter 11) — nhầm lẫn hai khái niệm dẫn tới áp dụng sai giải
pháp.

## Interview Questions

**Junior**

- Memory visibility là gì?
- Một biến `boolean` dùng làm cờ hiệu giữa hai thread cần lưu ý điều gì?

**Mid**

- Giải thích nguyên nhân kỹ thuật cụ thể (liên quan JIT Compiler) gây ra vấn đề visibility.
- Visibility khác race condition như thế nào? Cho ví dụ minh hoạ sự khác biệt.

**Senior**

- Một service graceful shutdown "treo" ngẫu nhiên sau khi chạy lâu. Trình bày quy trình chẩn
  đoán, phân biệt với khả năng deadlock.
- Giải thích vì sao bug visibility đặc biệt nguy hiểm ở chỗ "không phải lúc nào cũng tái hiện
  được" — điều này ảnh hưởng thế nào tới chiến lược testing cho code đa luồng?

## Exercises

- [ ] Chạy lại đúng ví dụ `VisibilityDemo` ở trên, cố gắng tái hiện hiện tượng (có thể cần
      chạy vài lần hoặc điều chỉnh; kết quả có thể khác nhau tuỳ máy/JVM).
- [ ] Chạy lại với cờ `-Xint` (tắt JIT hoàn toàn, Phase 1 Chapter 06), so sánh hành vi.
- [ ] Sửa ví dụ bằng cách thêm `volatile` vào `stopRequested`, chạy lại, xác nhận `worker`
      dừng lại đúng lúc — xem trước [Chapter 04](04-volatile.md).

## Cheat Sheet

| Khái niệm | Visibility | Race Condition |
| --- | --- | --- |
| Vấn đề gì | Thread không thấy giá trị mới | Nhiều thao tác ghi đè lẫn nhau |
| Xảy ra khi | Dù chỉ 1 thread ghi | Nhiều thread cùng đọc-sửa-ghi |
| Giải pháp tối thiểu | `volatile` (Chapter 04) | `synchronized`/`Atomic` (Chapter 05, 12) |

**Dấu hiệu chẩn đoán:** bug đa luồng "chập chờn", tệ hơn khi chạy lâu (JIT đã tối ưu sâu),
thread dump cho thấy `RUNNABLE` (không `BLOCKED`) → nghi ngờ visibility trước tiên.

## References

- JSR 133: Java Memory Model and Thread Specification Revision.
- Java Concurrency in Practice (Brian Goetz) — Chapter 3: Sharing Objects.
- Java Language Specification (JLS) — Chapter 17: Threads and Locks.
