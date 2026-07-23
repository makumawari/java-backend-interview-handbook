---
tags:
  - JPA
  - DirtyChecking
  - Hibernate
---

# Dirty Checking

> Phase: Phase 6 — Persistence
> Chapter slug: `dirty-checking`

## Metadata

```yaml
Chapter: Dirty Checking
Phase: Phase 6 — Persistence
Difficulty: ★★★★
Importance: ★★★★★
Interview Frequency: 75%
Prerequisites:
  - Chapter 05 — Persistence Context
Used Later:
  - Chapter 07 — Entity State
  - Chapter 12 — Transaction (JPA Transaction Boundary)
Estimated Reading: 15 phút
Estimated Practice: 12 phút
```

## Story

> Xem lại: [Chapter 05](05-persistence-context.md) — Persistence Context giữ tham chiếu tới mọi
> entity đã load. Thử sửa trực tiếp một field của entity đó — **không gọi bất kỳ phương thức
> `save()`/`update()` nào** — rồi commit transaction.

```java
Customer managed = em.find(Customer.class, id);
managed.setName("Ten Moi - KHONG goi save()");
// KHONG goi customerRepo.save(managed), KHONG goi em.merge(managed)
txManager.commit(tx);
```

Kết quả thật:

```
Da sua managed.setName(), CHUA goi save()/update() nao ca
Sap goi commit()...
Hibernate: update customer set name=? where id=?
```

Ngay khi `commit()` được gọi — không phải trước đó — một câu lệnh `UPDATE` **tự động** xuất
hiện, dù không có bất kỳ lời gọi `save()` nào trong toàn bộ đoạn code.

## Interview Question (Central)

> Dirty Checking trong JPA/Hibernate là gì? Vì sao sửa một entity đã load mà không gọi `save()`
> vẫn được cập nhật xuống database?

## Objectives

- [ ] Hiểu cơ chế Dirty Checking: Hibernate tự động phát hiện thay đổi trên entity Managed
- [ ] Tự tay chứng minh bằng thực nghiệm: `UPDATE` tự động xuất hiện khi commit, dù không gọi
      `save()`/`update()` tường minh
- [ ] Hiểu vì sao Dirty Checking chỉ hoạt động với entity **Managed** (nằm trong Persistence
      Context), không hoạt động với entity Detached

## Prerequisites

- Chapter 05 — bắt buộc phải hiểu Persistence Context trước, vì Dirty Checking chính là hành vi
  Hibernate thực hiện dựa trên dữ liệu Persistence Context lưu giữ.

## Used Later

- **Chapter 07 (Entity State)** — Dirty Checking chỉ áp dụng cho entity ở trạng thái Managed,
  chapter tiếp theo định nghĩa chính xác trạng thái này.
- **Chapter 12 (Transaction - JPA Transaction Boundary)** — thời điểm Dirty Checking thực sự
  chạy (flush) liên quan trực tiếp tới ranh giới transaction.

## Problem

Với JDBC thuần hoặc nhiều ORM đơn giản hơn, mọi thay đổi dữ liệu đều cần một lệnh ghi **tường
minh** (`UPDATE ... SET ...`) — lập trình viên phải tự nhớ gọi đúng lệnh ghi sau mỗi lần sửa đổi
dữ liệu trong bộ nhớ, dễ **quên** dẫn tới thay đổi "biến mất" (chỉ tồn tại trong object Java,
không bao giờ tới database).

## Concept

**Dirty Checking** ("kiểm tra bẩn" — phát hiện thay đổi) là cơ chế Hibernate **tự động so sánh**
trạng thái hiện tại của mọi entity **Managed** (đang nằm trong Persistence Context) với "ảnh
chụp" (snapshot) trạng thái của nó tại thời điểm mới được load — nếu phát hiện khác biệt, Hibernate
tự sinh câu lệnh `UPDATE` tương ứng, thường vào lúc **flush** (đồng bộ hoá Persistence Context
xuống database, xảy ra trước khi commit hoặc trước khi chạy một truy vấn liên quan).

## Why?

Dirty Checking chuyển trách nhiệm "nhớ gọi UPDATE" từ lập trình viên sang framework — vì
Persistence Context (Chapter 05) đã giữ **cả hai** phiên bản dữ liệu (snapshot ban đầu và trạng
thái hiện tại của entity trong bộ nhớ), Hibernate có đủ thông tin để **tự suy luận** chính xác
những gì đã thay đổi, không cần lập trình viên khai báo tường minh. Đây là một ứng dụng cụ thể
của nguyên lý "tận dụng thông tin đã có sẵn thay vì bắt người dùng khai báo lại" — cùng triết lý
với cách Spring tự sinh JPQL từ tên phương thức (Chapter 03).

## How?

```java
@PersistenceContext EntityManager em;

Customer managed = em.find(Customer.class, id); // entity o trang thai MANAGED
managed.setName("Ten Moi");                      // CHI sua field Java, KHONG goi gi them
// ... transaction tiep tuc, hoac ket thuc
// Luc FLUSH (thuong la luc commit), Hibernate TU DONG:
//   1. So sanh managed hien tai voi snapshot luc load
//   2. Phat hien "name" da doi
//   3. Tu sinh: UPDATE customer SET name = ? WHERE id = ?
```

## Visualization

```
em.find(Customer.class, id)
     │
     ▼ Hibernate luu LAI 2 thu vao Persistence Context:
        - snapshot: {name: "Ten Cu"}       <- "anh chup" luc vua load
        - entity that: managed (Java object) <- se bi sua doi sau nay

managed.setName("Ten Moi")
     │
     ▼ CHI sua entity THAT trong bo nho, snapshot VAN GIU nguyen "Ten Cu"

FLUSH (thuong o thoi diem commit):
     │
     ▼ Hibernate SO SANH: entity that ("Ten Moi") vs snapshot ("Ten Cu")
     │
     ▼ KHAC NHAU -> tu dong sinh: UPDATE customer SET name='Ten Moi' WHERE id=?
```

## Example

```java
Customer managed = em.find(Customer.class, id);
managed.setName("Ten Moi - KHONG goi save()");
System.out.println("Da sua managed.setName(), CHUA goi save()/update() nao ca");
System.out.println("Sap goi commit()...");
txManager.commit(tx);
```

Kết quả thật (Hibernate ORM 6.5.3.Final):

```
Hibernate: select c1_0.id,c1_0.name from customer c1_0 where c1_0.id=?
Da sua managed.setName(), CHUA goi save()/update() nao ca
Sap goi commit()...
Hibernate: update customer set name=? where id=?
```

Xác nhận lại từ một transaction hoàn toàn mới (đảm bảo đọc thật từ database, không phải cache):

```
=== Kiem tra lai tu mot transaction moi ===
Hibernate: select c1_0.id,c1_0.name from customer c1_0 where c1_0.id=?
Ten sau khi reload tu DB: Ten Moi - KHONG goi save()
```

Câu lệnh `UPDATE` chỉ xuất hiện **sau** dòng log `"Sap goi commit()..."` — xác nhận nó được sinh
ra chính xác tại thời điểm `commit()` (flush), không phải ngay khi `setName()` được gọi. Dữ liệu
mới thực sự đã được ghi xuống database — xác nhận qua việc đọc lại từ một transaction hoàn toàn
mới (Persistence Context khác, không thể "gian lận" bằng cách chỉ đọc từ bộ nhớ cùng transaction,
đã học ở Chapter 05).

## Deep Dive

**Hibernate so sánh "trạng thái hiện tại" và "snapshot" bằng cách nào — so sánh từng field một,
hay có cơ chế tối ưu hơn?** Mặc định, Hibernate thực hiện so sánh **từng field** giữa entity
hiện tại và snapshot đã lưu khi load — với entity có nhiều field, đây có thể là một chi phí đáng
kể nếu gọi thường xuyên. Hibernate cung cấp annotation `@DynamicUpdate` (đặc thù Hibernate, đã
nhắc tới ở Chapter 01 như một ví dụ về việc dự án khó "đổi implementation JPA" do phụ thuộc tính
năng riêng) để chỉ đưa **những cột thực sự thay đổi** vào câu `UPDATE` sinh ra (mặc định không
có `@DynamicUpdate`, Hibernate luôn `UPDATE` **toàn bộ** các cột, kể cả cột không đổi) — hữu ích
khi entity có nhiều field nhưng mỗi lần chỉ thường sửa một vài field, giảm kích thước câu lệnh
SQL thực thi.

## Engineering Insight

**Dirty Checking có thể gây ra vấn đề hiệu năng nào trong thực tế, và cách nhận biết?** Với một
transaction load **rất nhiều** entity (ví dụ một vòng lặp xử lý hàng nghìn bản ghi), Hibernate
phải so sánh **từng entity** với snapshot của nó tại mỗi lần flush — chi phí này tăng tuyến tính
theo số lượng entity đang Managed trong Persistence Context, có thể trở thành điểm nghẽn hiệu
năng thực sự với batch job xử lý khối lượng lớn dữ liệu. Đây là lý do các thao tác xử lý hàng
loạt (bulk update/delete) thường được khuyến nghị dùng JPQL/`@Modifying` tường minh (bỏ qua hoàn
toàn Dirty Checking, tác động trực tiếp xuống database) thay vì load từng entity rồi sửa từng
cái một trong vòng lặp — hoặc gọi `em.clear()` định kỳ trong batch job để giải phóng Persistence
Context, đánh đổi mất khả năng dirty checking cho các entity đã xử lý xong lấy hiệu năng tốt hơn.

## Historical Note

Dirty Checking là một trong những tính năng "gây ấn tượng mạnh" nhất khi Hibernate mới ra đời
(2001) — vào thời điểm phần lớn lập trình viên Java vẫn quen thuộc với JDBC thuần (viết tay mọi
câu `UPDATE`), khả năng "tự động phát hiện và lưu thay đổi" là một cải tiến năng suất rất lớn,
góp phần quan trọng vào sự phổ biến nhanh chóng của Hibernate trong cộng đồng Java giai đoạn
2001-2006, trước khi JPA (2006) chuẩn hoá hành vi này thành một phần của đặc tả chung.

## Myth vs Reality

- **Myth:** "Phải gọi `save()`/`update()` tường minh mỗi khi sửa dữ liệu, giống như JDBC thuần."
  **Reality:** Đã chứng minh bằng thực nghiệm — với entity Managed, chỉ cần sửa field Java trực
  tiếp, Hibernate tự động sinh `UPDATE` khi flush, không cần gọi bất kỳ phương thức lưu nào.

- **Myth:** "`UPDATE` được sinh ra ngay lập tức khi gọi setter, không phải đợi tới flush/commit."
  **Reality:** Đã chứng minh bằng thực nghiệm — `UPDATE` chỉ xuất hiện **sau** dòng log trước
  `commit()`, xác nhận nó chạy tại thời điểm flush, không phải ngay lúc `setName()`.

## Common Mistakes

- **Gọi `save()` tường minh một cách không cần thiết cho entity đã Managed** — không sai về mặt
  logic (Spring Data JPA xử lý đúng), nhưng thể hiện hiểu lầm về cơ chế Dirty Checking, có thể
  gây nhầm lẫn khi đọc code (tưởng cần thiết trong khi Hibernate tự lo).
- **Sửa đổi entity trong một vòng lặp xử lý hàng loạt mà không cân nhắc chi phí Dirty Checking**
  — gây vấn đề hiệu năng với batch job lớn, nên dùng bulk update JPQL thay thế.
- **Không hiểu vì sao `@DynamicUpdate` hữu ích** — bỏ lỡ cơ hội tối ưu SQL cho entity nhiều
  field, thường xuyên chỉ sửa một vài field.

## Debug Checklist

- [ ] Sửa entity nhưng thay đổi không xuất hiện trong database? → kiểm tra entity có đang ở
      trạng thái Managed không (Chapter 07) — Dirty Checking chỉ hoạt động với Managed entity.
- [ ] Không chắc `UPDATE` sinh ra ở đâu, khi nào? → bật `show-sql`, quan sát thời điểm chính xác
      câu lệnh xuất hiện (thường ngay trước commit, hoặc trước một truy vấn liên quan kích hoạt
      flush sớm).
- [ ] Hiệu năng batch job xử lý entity kém? → cân nhắc bulk update JPQL/`@Modifying` thay vì
      sửa từng entity trong vòng lặp, hoặc gọi `em.clear()` định kỳ.

## Summary

Dirty Checking là cơ chế Hibernate tự động phát hiện thay đổi trên entity Managed (so sánh với
snapshot lúc load) và tự sinh `UPDATE` tương ứng lúc flush — không cần gọi `save()`/`update()`
tường minh. Đã chứng minh bằng thực nghiệm: sửa trực tiếp field của entity Managed, không gọi
bất kỳ phương thức lưu nào, `UPDATE` vẫn tự động xuất hiện chính xác tại thời điểm `commit()`
(flush) — xác nhận dữ liệu thực sự đã lưu bằng cách đọc lại từ transaction mới. Cơ chế này có
chi phí tăng theo số lượng entity Managed, nên với batch job lớn, thường khuyến nghị bulk update
JPQL tường minh thay vì sửa từng entity trong vòng lặp.

## Interview Questions

**Mid**

- Dirty Checking trong JPA/Hibernate là gì?
- Vì sao sửa một entity đã load mà không gọi `save()` vẫn được cập nhật xuống database?

**Senior**

- Dirty Checking có thể gây vấn đề hiệu năng gì với batch job xử lý khối lượng lớn dữ liệu? Đề
  xuất giải pháp thay thế.
- `@DynamicUpdate` giải quyết vấn đề gì? Khi nào nên dùng nó?

## Exercises

- [ ] Chạy lại `DirtyCheckingTest` ở trên, xác nhận `UPDATE` xuất hiện đúng thời điểm và dữ liệu
      thực sự được lưu.
- [ ] Thêm `@DynamicUpdate` vào entity `Customer`, sửa chỉ một trong nhiều field, quan sát câu
      `UPDATE` sinh ra chỉ chứa đúng cột đã đổi (so với mặc định chứa mọi cột).
- [ ] Viết một batch job giả lập sửa 1000 entity trong một transaction, đo thời gian flush, so
      sánh với cách dùng bulk update JPQL cho cùng thao tác.

## Cheat Sheet

| | Sửa entity Managed | Bulk update JPQL |
| --- | --- | --- |
| Cần gọi `save()`? | Không (Dirty Checking tự lo) | N/A (`@Modifying` + `executeUpdate()`) |
| Thời điểm SQL chạy | Lúc flush (thường trước commit) | Ngay lập tức khi gọi |
| Phù hợp cho | Sửa từng entity, số lượng nhỏ-vừa | Sửa hàng loạt, số lượng lớn |
| Chi phí | Tăng theo số entity Managed | Cố định, không phụ thuộc Persistence Context |

## References

- Hibernate ORM Documentation — Dirty Checking, `@DynamicUpdate`.
- Jakarta Persistence Specification — Flush.
