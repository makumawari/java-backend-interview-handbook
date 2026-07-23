---
tags:
  - JPA
  - OptimisticLock
  - Hibernate
---

# Optimistic Lock (@Version)

> Phase: Phase 6 — Persistence
> Chapter slug: `optimistic-lock`

## Metadata

```yaml
Chapter: Optimistic Lock
Phase: Phase 6 — Persistence
Difficulty: ★★★★★
Importance: ★★★★★
Interview Frequency: 80%
Prerequisites:
  - Chapter 06 — Dirty Checking
  - Chapter 12 — Transaction (JPA Transaction Boundary)
  - Phase 3, Chapter 12-13 — Atomic, CAS
Used Later:
  - Chapter 14 — Pessimistic Lock
Estimated Reading: 20 phút
Estimated Practice: 15 phút
```

## Story

> Xem lại: Phase 3, Chapter 12-13 (Atomic, CAS) — `compareAndSet(expected, new)` chỉ thành công
> khi giá trị **chưa bị ai thay đổi** kể từ lần đọc gần nhất. `@Version` trong JPA áp dụng
> **chính xác cùng nguyên lý CAS đó**, nhưng ở tầng database thay vì tầng bộ nhớ JVM.

Mô phỏng "User1" và "User2" cùng đọc một `Order` **cùng lúc** (cùng version), rồi cả hai đều thử
sửa:

```
User1 va User2 cung doc version = 0 / 0
Hibernate: update orders set customer_id=?,status=?,version=? where id=? and version=?
Sau khi User2 commit, version trong DB = 1, status = CONFIRMED_BY_USER2

=== User1 GIO MOI sua (dang giu ban sao CU, version=0), thu luu ===
Loi khi User1 co update voi version CU (0) trong khi DB da la version 1: OptimisticLockException
Trang thai CUOI CUNG trong DB: status=CONFIRMED_BY_USER2, version=1
```

User2 ghi trước, thành công, `version` tăng lên `1`. User1 (vẫn giữ bản sao cũ, `version=0`) cố
ghi sau — bị **từ chối thẳng** với `OptimisticLockException`, không hề "ghi đè" âm thầm lên thay
đổi của User2.

## Interview Question (Central)

> Your application starts creating duplicate records even though you're checking if they already
> exist. How would you fix it?

## Objectives

- [ ] Hiểu `@Version` hiện thực hoá optimistic locking dựa trên nguyên lý CAS (Phase 3, Chapter
      13) ở tầng database
- [ ] Tự tay chứng minh bằng thực nghiệm: hai "người dùng" cùng đọc, một ghi trước thành công,
      người ghi sau với dữ liệu cũ bị `OptimisticLockException`
- [ ] Giải thích chính xác vì sao "check rồi tạo mới nếu chưa tồn tại" (check-then-act) không đủ
      để tránh bản ghi trùng lặp dưới tải đồng thời, và optimistic lock (hoặc các giải pháp liên
      quan) giải quyết vấn đề này ra sao

## Prerequisites

- Chapter 06 — hiểu Dirty Checking, cơ chế phát hiện entity đã thay đổi.
- Chapter 12 — hiểu flush/commit, thời điểm optimistic lock thực sự được kiểm tra.
- Phase 3, Chapter 12-13 — hiểu CAS, nguyên lý nền tảng của optimistic locking.

## Used Later

- **Chapter 14 (Pessimistic Lock)** — giải pháp thay thế, khoá trước thay vì kiểm tra sau, cho
  các tình huống optimistic lock không phù hợp.

## Problem

Khi nhiều giao dịch (request, thread, hoặc nhiều instance ứng dụng — Phase 5, Chapter 15) cùng
đọc **và** sửa cùng một bản ghi gần như đồng thời, mà không có cơ chế phối hợp nào, giao dịch ghi
**sau cùng** sẽ **ghi đè âm thầm** lên thay đổi của giao dịch ghi trước đó — hiện tượng gọi là
"lost update" (mất cập nhật). Tệ hơn, với logic "kiểm tra tồn tại rồi mới tạo mới nếu chưa có"
(check-then-act — chính là nguồn gốc câu hỏi phỏng vấn của chapter này), hai giao dịch cùng
"kiểm tra" gần như đồng thời (cả hai đều thấy "chưa tồn tại") rồi cả hai **đều tạo mới** — dẫn
tới bản ghi trùng lặp, dù code có vẻ đã "kiểm tra cẩn thận".

## Concept

**Optimistic Locking** (`@Version`) thêm một cột phiên bản (`version`, tự động tăng mỗi lần
`UPDATE`) vào entity — mọi câu `UPDATE`/`DELETE` Hibernate sinh ra tự động thêm điều kiện
`WHERE ... AND version = ?` (khớp với giá trị `version` **tại thời điểm entity được đọc**). Nếu
điều kiện đó không khớp (vì một giao dịch khác đã `UPDATE` bản ghi đó, làm `version` tăng lên
kể từ lần đọc), câu lệnh **ảnh hưởng 0 dòng** — Hibernate phát hiện điều này và ném
`OptimisticLockException`.

## Why?

Đây chính xác là nguyên lý CAS (Compare-And-Swap, Phase 3 Chapter 13) áp dụng ở tầng database:
"so sánh giá trị hiện tại (`version`) với giá trị kỳ vọng, chỉ ghi nếu khớp" — hoàn toàn không
cần khoá độc quyền (blocking lock) trong suốt thời gian giao dịch xử lý (khác với Pessimistic
Lock, Chapter 14) — chỉ kiểm tra **tại thời điểm ghi**. Đây là lựa chọn tối ưu khi xung đột ghi
đồng thời **hiếm khi thực sự xảy ra** (đúng tên gọi "optimistic" — lạc quan rằng phần lớn thời
gian không có ai khác cùng sửa bản ghi đó cùng lúc) — chi phí chỉ phát sinh khi xung đột thực sự
xảy ra (phải xử lý exception, thường là retry), không phải trả trước cho mọi giao dịch như
pessimistic lock.

## How?

```java
@Entity
class Order {
    @Version
    private Long version; // Hibernate TU DONG quan ly, khong tu tay gan gia tri
}

// Cap nhat binh thuong nhu moi entity khac
Order o = em.find(Order.class, id);
o.setStatus("CONFIRMED");
// UPDATE orders SET status=?, version=? WHERE id=? AND version=<gia tri luc doc>
// Neu 0 dong bi anh huong -> OptimisticLockException
```

## Visualization

```
User1 doc (version=0)  ──┐
User2 doc (version=0)  ──┤  CA HAI cung thay version=0
                          │
User2 sua, COMMIT TRUOC:
  UPDATE orders SET version=1 WHERE id=? AND version=0  <- KHOP (DB dang la 0) -> THANH CONG
  DB gio la version=1

User1 sua, COMMIT SAU (van tuong version=0):
  UPDATE orders SET version=1 WHERE id=? AND version=0  <- KHONG KHOP (DB da la 1)
       │
       ▼  0 dong bi anh huong
       │
       ▼  Hibernate PHAT HIEN, nem OptimisticLockException
```

## Example

```java
Order forUser1 = em.find(Order.class, orderId);
em.detach(forUser1); // gia lap User1 giu ban sao nay o tang service

Order forUser2 = em.find(Order.class, orderId);
forUser2.setStatus("CONFIRMED_BY_USER2");
txManager.commit(txUser2); // User2 GHI TRUOC, thanh cong

// User1 gio moi sua, van dang giu ban sao CU (version=0)
forUser1.setStatus("CONFIRMED_BY_USER1");
try {
    em.merge(forUser1);
    txManager.commit(txUser1);
} catch (OptimisticLockException e) {
    System.out.println("Loi: " + e.getClass().getSimpleName());
}
```

Kết quả thật (Hibernate ORM 6.5.3.Final):

```
User1 va User2 cung doc version = 0 / 0
Hibernate: update orders set customer_id=?,status=?,version=? where id=? and version=?
Sau khi User2 commit, version trong DB = 1, status = CONFIRMED_BY_USER2

Loi khi User1 co update voi version CU (0) trong khi DB da la version 1: OptimisticLockException
Trang thai CUOI CUNG trong DB: status=CONFIRMED_BY_USER2, version=1
```

Trạng thái cuối cùng xác nhận: chỉ thay đổi của User2 (ghi trước) tồn tại trong database, thay
đổi của User1 (ghi sau, dữ liệu cũ) bị từ chối hoàn toàn — không có "lost update" nào xảy ra,
đúng mục đích thiết kế của optimistic locking.

## Deep Dive

**Trả lời trực tiếp câu hỏi phỏng vấn trung tâm: vì sao "kiểm tra tồn tại rồi mới tạo" vẫn tạo ra
bản ghi trùng lặp, và optimistic lock (hoặc giải pháp liên quan) giải quyết ra sao?** Logic
"if (!exists) create()" có một **khoảng hở thời gian** (race window) giữa bước kiểm tra
(`SELECT ... WHERE ...`) và bước tạo mới (`INSERT`) — nếu hai giao dịch chạy đủ gần nhau, **cả
hai** đều thực hiện `SELECT` **trước khi** bất kỳ giao dịch nào kịp `INSERT`, nên cả hai đều thấy
"chưa tồn tại", và cả hai đều `INSERT` — tạo ra hai bản ghi trùng lặp. `@Version` (optimistic
lock) **không trực tiếp** giải quyết được vấn đề này (nó chỉ bảo vệ **UPDATE** trên bản ghi **đã
tồn tại**, không ngăn được hai `INSERT` độc lập) — giải pháp đúng đắn hơn cho chính xác tình
huống "race condition khi tạo mới" là **Unique Index** ở tầng database (bắt buộc một cột/tổ hợp
cột phải duy nhất, database tự từ chối `INSERT` thứ hai) kết hợp bắt `DataIntegrityViolationException`, hoặc dùng **Pessimistic Lock** (Chapter 14) để khoá độc quyền toàn bộ "khoảng hở" kiểm
tra-rồi-tạo. Đây chính là lý do câu hỏi gốc gợi ý xem thêm "Isolation, Lock, Unique Index,
Pessimistic Lock" — optimistic lock chỉ là **một phần** của bức tranh đầy đủ.

## Engineering Insight

**Trong thực tế, xử lý `OptimisticLockException` như thế nào — chỉ đơn giản báo lỗi cho người
dùng, hay có chiến lược tốt hơn?** Chiến lược phổ biến nhất là **retry**: bắt
`OptimisticLockException`, đọc lại entity mới nhất từ database (lấy đúng `version` hiện tại),
áp dụng lại thay đổi nghiệp vụ trên dữ liệu mới đó, rồi thử lưu lại — lặp lại một số lần giới hạn
trước khi thực sự báo lỗi cho người dùng. Chiến lược này phù hợp khi thay đổi nghiệp vụ **độc
lập** với giá trị cụ thể đọc được (ví dụ "tăng số lượng tồn kho thêm 5" có thể áp dụng lại an
toàn trên dữ liệu mới) — nhưng **không** phù hợp khi người dùng đã nhập dữ liệu dựa trên phiên
bản cũ (ví dụ form chỉnh sửa văn bản) — trong trường hợp đó, cách xử lý đúng thường là báo cho
người dùng "dữ liệu đã bị người khác thay đổi, vui lòng tải lại" thay vì tự động ghi đè.

## Historical Note

Optimistic locking như một mẫu hình thiết kế (không riêng JPA) đã tồn tại rộng rãi trong các hệ
thống database từ lâu — JPA/Hibernate chuẩn hoá nó thành annotation `@Version` đơn giản, tự động
sinh và kiểm tra điều kiện `WHERE version = ?` mà không cần lập trình viên tự viết SQL thủ công.
Về mặt khái niệm, nó cùng họ với CAS (Phase 3, Chapter 13) và ETag trong HTTP (cơ chế tương tự
cho việc kiểm tra "dữ liệu chưa bị thay đổi" ở tầng giao thức web) — cùng một nguyên lý "lạc
quan, kiểm tra tại thời điểm ghi" xuất hiện lặp lại ở nhiều tầng khác nhau của hệ thống phần mềm.

## Myth vs Reality

- **Myth:** "`@Version` (optimistic lock) đủ để ngăn mọi loại race condition, bao gồm cả tạo bản
  ghi trùng lặp."
  **Reality:** Đã giải thích ở Deep Dive — nó chỉ bảo vệ `UPDATE` trên bản ghi đã tồn tại,
  **không** ngăn được race condition khi hai giao dịch cùng `INSERT` bản ghi mới; cần Unique
  Index hoặc Pessimistic Lock cho đúng tình huống đó.

- **Myth:** "`OptimisticLockException` luôn là lỗi nghiêm trọng cần dừng xử lý ngay."
  **Reality:** Xem Engineering Insight — với nhiều tình huống, retry với dữ liệu mới nhất là
  chiến lược xử lý hợp lý, không cần dừng hẳn.

## Common Mistakes

- **Chỉ dùng "check tồn tại rồi tạo mới" mà không có Unique Index bảo vệ ở tầng database** —
  nguồn gốc trực tiếp của câu hỏi phỏng vấn trung tâm chapter này, dễ tạo bản ghi trùng lặp dưới
  tải đồng thời.
- **Không xử lý `OptimisticLockException`, để nó lan ra thành lỗi 500 chung chung** — trải
  nghiệm người dùng kém; nên có chiến lược retry hoặc thông báo rõ ràng.
- **Nhầm lẫn optimistic lock bảo vệ được cả tình huống tạo mới trùng lặp** — xem Deep Dive, đây
  là ngộ nhận phổ biến khi trả lời phỏng vấn.

## Best Practices

- Luôn kết hợp Unique Index ở tầng database cho các trường cần đảm bảo duy nhất (ví dụ email,
  mã đơn hàng) — đây là lớp bảo vệ cuối cùng, không thể bị "lách qua" bất kể logic tầng ứng dụng.
- Dùng `@Version` cho các entity có khả năng bị nhiều giao dịch cùng sửa đổi đồng thời, xung đột
  hiếm khi thực sự xảy ra.
- Có chiến lược retry rõ ràng khi bắt `OptimisticLockException`, phù hợp với ngữ cảnh nghiệp vụ
  cụ thể (tự động retry vs báo người dùng tải lại).

## Debug Checklist

- [ ] Ứng dụng tạo bản ghi trùng lặp dù có logic "kiểm tra tồn tại trước khi tạo"? → kiểm tra có
      Unique Index bảo vệ ở tầng database không — đây là nguyên nhân/giải pháp chính xác, không
      phải `@Version`.
- [ ] `OptimisticLockException` xuất hiện thường xuyên, ảnh hưởng trải nghiệm người dùng? → đánh
      giá lại mức độ xung đột thực tế — nếu quá thường xuyên, có thể Pessimistic Lock (Chapter
      14) phù hợp hơn optimistic.
- [ ] Không chắc `@Version` có đang hoạt động đúng không? → bật `show-sql`, xác nhận điều kiện
      `AND version = ?` xuất hiện trong câu `UPDATE`/`DELETE`.

## Summary

Optimistic Lock (`@Version`) áp dụng nguyên lý CAS (Phase 3, Chapter 13) ở tầng database: mọi
`UPDATE`/`DELETE` tự động thêm điều kiện `WHERE version = ?`, thất bại (0 dòng ảnh hưởng) nếu
bản ghi đã bị giao dịch khác sửa trước đó. Đã chứng minh bằng thực nghiệm: hai "người dùng" cùng
đọc `version=0`, người ghi trước thành công (version tăng lên 1), người ghi sau với dữ liệu cũ
nhận `OptimisticLockException` — không có "lost update" nào xảy ra. Tuy nhiên, optimistic lock
**không** giải quyết được vấn đề tạo bản ghi trùng lặp từ logic "kiểm tra rồi tạo mới" (chỉ bảo
vệ UPDATE trên bản ghi đã tồn tại) — giải pháp đúng cho vấn đề đó là Unique Index ở tầng database
kết hợp xử lý `DataIntegrityViolationException`, hoặc Pessimistic Lock (Chapter 14).

## Interview Questions

- Your application starts creating duplicate records even though you're checking if they already
  exist. How would you fix it? (see also Isolation, Lock, Unique Index, Pessimistic Lock)

**Senior**

- Giải thích vì sao `@Version` (optimistic lock) không đủ để ngăn bản ghi trùng lặp từ logic
  "kiểm tra tồn tại rồi tạo mới". Đề xuất giải pháp đúng đắn.
- Nêu chiến lược xử lý `OptimisticLockException` phù hợp cho hai ngữ cảnh khác nhau: (1) tăng số
  lượng tồn kho, (2) người dùng chỉnh sửa văn bản qua form.

## Exercises

- [ ] Chạy lại `OptimisticLockTest` ở trên, xác nhận `OptimisticLockException` xuất hiện đúng
      như mô tả, và trạng thái cuối cùng chỉ phản ánh thay đổi của giao dịch ghi trước.
- [ ] Thêm Unique Index cho một trường của `Customer` (ví dụ `name`), viết test mô phỏng hai
      giao dịch cùng "kiểm tra rồi tạo mới" gần như đồng thời, xác nhận
      `DataIntegrityViolationException` ngăn được bản ghi trùng lặp mà `@Version` không ngăn
      được.
- [ ] Viết một chiến lược retry đơn giản cho `OptimisticLockException`: bắt exception, đọc lại
      entity mới nhất, áp dụng lại thay đổi, thử lưu lại tối đa 3 lần.

## Cheat Sheet

| Cơ chế | Bảo vệ được gì | Không bảo vệ được gì |
| --- | --- | --- |
| `@Version` (Optimistic Lock) | Lost update khi UPDATE bản ghi đã tồn tại | Race condition khi INSERT bản ghi mới |
| Unique Index (database) | Trùng lặp giá trị ở cột cần duy nhất, kể cả khi INSERT đồng thời | Không tự xử lý logic retry |
| Pessimistic Lock (Chapter 14) | Toàn bộ khoảng hở "kiểm tra rồi hành động" | Hiệu năng thấp hơn dưới tải cao |

## References

- Jakarta Persistence Specification — Optimistic Locking.
- Hibernate ORM Documentation — `@Version`, Optimistic and Pessimistic Locking.
