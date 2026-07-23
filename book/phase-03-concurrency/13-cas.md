---
tags:
  - Java
  - CAS
  - Concurrency
---

# CAS (Compare-And-Swap) và bài toán ABA

> Phase: Phase 3 — Concurrency
> Chapter slug: `cas`

## Metadata

```yaml
Chapter: CAS
Phase: Phase 3 — Concurrency
Difficulty: ★★★★
Importance: ★★★
Interview Frequency: 40%
Prerequisites:
  - Chapter 12 — Atomic
Used Later:
  - Cấu trúc dữ liệu lock-free nâng cao (ngoài phạm vi handbook, nhưng CAS là nền tảng bắt buộc để hiểu)
Estimated Reading: 15 phút
Estimated Practice: 10 phút
```

## Story

> Xem lại: [Chapter 12](12-atomic.md) — `AtomicInteger.compareAndSet(expected, newValue)` chỉ
> thành công khi giá trị hiện tại **đúng bằng** `expected`. Nhưng "giá trị hiện tại đúng bằng
> expected" có thực sự chứng minh **không có gì thay đổi** trong lúc đó không?

Giả sử một thread đọc giá trị `A`, chuẩn bị CAS đổi thành `C`. Trong lúc nó "chuẩn bị", một
thread khác đổi `A` → `B` → rồi lại đổi ngược về `A`. Khi thread đầu tiên thực hiện
`compareAndSet(A, C)`, giá trị hiện tại **lại đúng là `A`** — CAS thành công, dù giá trị **đã
thực sự bị thay đổi (hai lần)** ở giữa. Đây gọi là **bài toán ABA**.

## Interview Question (Central)

> Bài toán ABA trong CAS là gì? Nó có thể gây ra hậu quả gì, và làm sao để phát hiện/ngăn chặn
> nó?

## Objectives

- [ ] Giải thích chính xác cơ chế CAS (Compare-And-Swap) hoạt động ở cấp CPU
- [ ] Tự tay chứng minh được bài toán ABA bằng thực nghiệm: `AtomicReference` không phát hiện
      được A→B→A
- [ ] Biết `AtomicStampedReference` giải quyết ABA bằng cách thêm "stamp" (số phiên bản)

## Prerequisites

- Chapter 12 — hiểu `compareAndSet` cơ bản trên `AtomicInteger`.

## Used Later

- Nền tảng để hiểu mọi cấu trúc dữ liệu lock-free (hàng đợi lock-free, stack lock-free) — nằm
  ngoài phạm vi handbook nhưng CAS + ABA là kiến thức bắt buộc phải có trước khi tìm hiểu sâu
  hơn.

## Problem

`compareAndSet(expected, newValue)` chỉ kiểm tra "giá trị hiện tại có bằng `expected` không" —
nó **không** biết được liệu giá trị đó có bị thay đổi rồi đổi ngược lại hay không trong khoảng
thời gian giữa lúc đọc và lúc CAS. Với kiểu dữ liệu nguyên thủy (int, long) thường không sao vì
bản thân "quay lại giá trị cũ" không gây hại logic. Nhưng với **tham chiếu object** (ví dụ node
trong một cấu trúc dữ liệu lock-free như stack/queue), việc object bị thay bằng một object khác
rồi thay lại bằng object gốc (dù giá trị "nhìn giống nhau") có thể để lại cấu trúc dữ liệu ở
trạng thái hỏng, vì các con trỏ/liên kết nội bộ đã bị thay đổi trong lúc đó.

## Concept

**CAS (Compare-And-Swap)** là lệnh CPU nguyên tử: so sánh giá trị hiện tại với một giá trị kỳ
vọng (`expected`), nếu khớp thì đổi thành giá trị mới, tất cả trong một bước không thể bị ngắt.
**Bài toán ABA** là hạn chế cố hữu của CAS thuần: nó chỉ so sánh **giá trị cuối cùng**, không
biết giá trị đó có bị thay đổi trung gian hay không.

## Why?

CAS được thiết kế tối giản có chủ đích — nó chỉ cần so sánh giá trị, không cần theo dõi lịch
sử. Điều này giúp nó cực nhanh (một lệnh CPU). Nhưng chính sự tối giản đó là nguồn gốc của ABA:
để phát hiện "đã có thay đổi trung gian", cần thêm thông tin ngoài giá trị đơn thuần — đó là lý
do JDK cung cấp thêm `AtomicStampedReference`, gắn kèm một "stamp" (số nguyên tăng dần, đại
diện cho phiên bản) đi cùng giá trị.

## How?

```java
AtomicStampedReference<String> ref = new AtomicStampedReference<>("A", 0);
int[] stampHolder = new int[1];
String value = ref.get(stampHolder); // lay ca gia tri LAN stamp hien tai
int stamp = stampHolder[0];

// CAS phai khop CA gia tri LAN stamp moi thanh cong
ref.compareAndSet("A", "C", stamp, stamp + 1);
```

## Visualization

```
ABA voi AtomicReference thuong (KHONG phat hien duoc):

  Thread 1:  doc A ─────────────────────────► CAS(A, C) → THANH CONG (SAI LAM!)
  Thread 2:         set(B) ──► set(A)  ← gia tri quay ve "A" nhung DA CO 2 LAN THAY DOI

ABA voi AtomicStampedReference (PHAT HIEN duoc qua stamp):

  Thread 1:  doc (A, stamp=0) ────────────────► CAS(A,C, stamp 0→1) → THAT BAI (dung!)
  Thread 2:         set(B, stamp=1) ──► set(A, stamp=2)  ← stamp thuc te la 2, khong phai 0
```

## Example

```java
import java.util.concurrent.atomic.AtomicReference;
import java.util.concurrent.atomic.AtomicStampedReference;

public class ABADemo {
    public static void main(String[] args) {
        AtomicReference<String> ref = new AtomicReference<>("A");
        String original = ref.get();

        ref.set("B");
        ref.set("A"); // "am tham" doi A->B->A

        boolean success = ref.compareAndSet(original, "C");
        System.out.println("AtomicReference CAS(A->C) sau khi bi doi A->B->A ngam: " + success
            + " (gia tri hien tai: " + ref.get() + ")");

        AtomicStampedReference<String> stamped = new AtomicStampedReference<>("A", 0);
        int[] stampHolder = new int[1];
        stamped.get(stampHolder);
        int originalStamp = stampHolder[0];

        stamped.set("B", 1);
        stamped.set("A", 2);

        boolean stampedSuccess = stamped.compareAndSet("A", "C", originalStamp, originalStamp + 1);
        System.out.println("AtomicStampedReference CAS sau khi bi doi ngam: " + stampedSuccess
            + " (gia tri: " + stamped.getReference() + ", stamp: " + stamped.getStamp() + ")");
    }
}
```

Kết quả thật (JDK 17):

```
AtomicReference CAS(A->C) sau khi bi doi A->B->A ngam: true (gia tri hien tai: C)
AtomicStampedReference CAS(A->C, stamp 0->1) sau khi bi doi ngam (stamp thuc te la 2): false (gia tri hien tai: A, stamp: 2)
```

Bằng chứng rõ ràng: `AtomicReference` CAS **thành công** (`true`) dù giá trị đã bị đổi hai lần
ở giữa — ABA thực sự xảy ra. `AtomicStampedReference` CAS **thất bại** (`false`) đúng như mong
đợi, vì stamp hiện tại là `2`, không khớp với stamp kỳ vọng là `0` — nó phát hiện được đã có
thay đổi trung gian dù giá trị cuối cùng "trông giống" giá trị gốc.

## Deep Dive

**Vì sao ABA thường vô hại với kiểu nguyên thủy nhưng nguy hiểm với con trỏ/object trong cấu
trúc dữ liệu lock-free?** Với `int`/`long`, "giá trị quay lại như cũ" thực sự đồng nghĩa "trạng
thái quay lại như cũ" — không có tác dụng phụ ẩn. Nhưng với node trong một lock-free stack
(ví dụ), "con trỏ head quay lại trỏ đúng node gốc" **không đảm bảo** các liên kết `next` bên
trong node đó vẫn còn nguyên vẹn — node có thể đã bị pop ra, node khác bị push vào, rồi chính
node gốc bị push trở lại (nhưng giờ `next` của nó trỏ tới một node khác đã bị giải phóng/thay
đổi). CAS chỉ nhìn thấy "địa chỉ con trỏ giống nhau", không nhìn thấy toàn bộ cấu trúc đã bị xáo
trộn.

## Engineering Insight

**Trong thực tế viết ứng dụng backend, có cần tự tay lo về ABA không?** Hiếm khi. ABA chủ yếu
là mối lo khi **tự viết** cấu trúc dữ liệu lock-free cấp thấp (thứ mà tuyệt đại đa số backend
engineer không cần tự làm — `ConcurrentHashMap`, `ConcurrentLinkedQueue` trong JDK đã tự xử lý
đúng đắn bên trong). Giá trị thực sự của việc hiểu ABA nằm ở việc **hiểu đúng giới hạn của
CAS** — để không lầm tưởng CAS là "phát hiện mọi thay đổi", và để đọc hiểu được code JDK/thư
viện lock-free khi cần debug sâu.

## Historical Note

`AtomicStampedReference` và `AtomicMarkableReference` (biến thể chỉ cần một cờ boolean thay vì
số nguyên đầy đủ) ra đời cùng đợt JSR 166 (Java 5, 2004), ngay từ đầu đã được thiết kế kèm với
`AtomicReference` chính vì nhóm tác giả (Doug Lea) đã lường trước bài toán ABA từ các nghiên
cứu cấu trúc dữ liệu lock-free trước đó.

## Myth vs Reality

- **Myth:** "CAS thành công nghĩa là chắc chắn không có thread nào khác đụng vào giá trị đó."
  **Reality:** CAS chỉ đảm bảo giá trị **tại thời điểm CAS** khớp với `expected` — không đảm
  bảo không có thay đổi trung gian (chính là bài toán ABA, đã chứng minh ở Example).

- **Myth:** "ABA là lỗi hiếm gặp, không cần quan tâm trong thực tế."
  **Reality:** Với kiểu dữ liệu nguyên thủy, đúng là hiếm gây hại. Nhưng với con trỏ/object
  trong cấu trúc dữ liệu tự viết, nó là nguồn lỗi rất khó tái hiện và debug (chỉ xảy ra dưới
  điều kiện race cụ thể) — đây là lý do Deep Dive nhấn mạnh nó nguy hiểm hơn nhiều so với vẻ
  ngoài "hiếm gặp".

## Common Mistakes

- **Tự viết cấu trúc dữ liệu lock-free dùng `AtomicReference` mà không cân nhắc ABA** — dễ để
  lại lỗi cực khó tái hiện, chỉ xảy ra dưới race condition cụ thể.
- **Dùng `AtomicStampedReference` cho mọi trường hợp "cho chắc"** — overhead cao hơn
  `AtomicReference` thường (phải đóng gói thêm stamp), chỉ nên dùng khi thực sự cần phát hiện
  ABA (khi tự viết cấu trúc dữ liệu lock-free với con trỏ).
- **Nhầm lẫn ABA với data race thông thường** — ABA là vấn đề logic đặc thù của CAS, khác với
  race condition đơn giản đã học ở Chapter 03-05.

## Best Practices

- Chỉ cần lo về ABA khi tự viết cấu trúc dữ liệu lock-free thao tác trực tiếp trên con trỏ
  (node, reference) — không cần khi dùng các collection có sẵn của JDK.
- Dùng `AtomicStampedReference`/`AtomicMarkableReference` khi thực sự cần phát hiện thay đổi
  trung gian, chấp nhận overhead cao hơn `AtomicReference` thường.
- Ưu tiên dùng cấu trúc dữ liệu concurrent có sẵn của JDK (Phase 2, Chapter 11) thay vì tự viết
  lock-free từ đầu, trừ khi có lý do hiệu năng thực sự bắt buộc.

## Debug Checklist

- [ ] Cấu trúc dữ liệu lock-free tự viết thỉnh thoảng bị hỏng dưới tải cao, khó tái hiện? → cân
      nhắc bài toán ABA là nghi phạm hàng đầu.
- [ ] Dùng `AtomicReference` cho con trỏ trong cấu trúc dữ liệu tự viết? → đánh giá xem có cần
      chuyển sang `AtomicStampedReference` không.

## Summary

CAS chỉ so sánh **giá trị cuối cùng**, không biết giá trị đó có bị thay đổi trung gian (A→B→A)
hay không — đây là bài toán ABA, đã chứng minh bằng thực nghiệm: `AtomicReference.compareAndSet`
thành công dù giá trị đã bị đổi hai lần ở giữa. `AtomicStampedReference` giải quyết bằng cách
gắn kèm một "stamp" (số phiên bản) — CAS chỉ thành công khi cả giá trị lẫn stamp đều khớp, giúp
phát hiện đúng đã có thay đổi trung gian. Trong thực tế backend, ABA chủ yếu liên quan tới việc
tự viết cấu trúc dữ liệu lock-free — hiếm khi cần tự xử lý khi dùng collection có sẵn của JDK.

## Interview Questions

**Mid**

- Bài toán ABA trong CAS là gì? Cho ví dụ cụ thể.

**Senior**

- `AtomicStampedReference` giải quyết bài toán ABA như thế nào? Vì sao không dùng
  `AtomicReference` cho mọi trường hợp cần CAS trên object?
- Vì sao ABA nguy hiểm hơn với cấu trúc dữ liệu lock-free thao tác trên con trỏ so với kiểu
  nguyên thủy?

## Exercises

- [ ] Chạy lại `ABADemo` ở trên, xác nhận `AtomicReference` CAS thành công còn
      `AtomicStampedReference` CAS thất bại đúng như mô tả.
- [ ] Thử viết một `AtomicMarkableReference` demo tương tự (dùng cờ boolean thay vì stamp số
      nguyên), so sánh use case phù hợp giữa hai lớp này.

## Cheat Sheet

| | `AtomicReference` | `AtomicStampedReference` |
| --- | --- | --- |
| So sánh khi CAS | Chỉ giá trị | Giá trị + stamp (phiên bản) |
| Phát hiện ABA | Không | Có |
| Overhead | Thấp | Cao hơn (đóng gói thêm stamp) |
| Dùng khi | Không thao tác trực tiếp trên con trỏ cấu trúc dữ liệu | Tự viết cấu trúc dữ liệu lock-free |

## References

- Java SE API Documentation — `java.util.concurrent.atomic.AtomicStampedReference`,
  `AtomicMarkableReference`.
- Java Concurrency in Practice (Brian Goetz) — Chapter 15.4: The ABA Problem.
