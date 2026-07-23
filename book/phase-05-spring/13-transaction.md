---
tags:
  - Spring
  - Transaction
---

# Spring Transaction Management (@Transactional)

> Phase: Phase 5 — Spring
> Chapter slug: `transaction`

## Metadata

```yaml
Chapter: Transaction (Spring Transaction Management)
Phase: Phase 5 — Spring
Difficulty: ★★★★★
Importance: ★★★★★
Interview Frequency: 90%
Prerequisites:
  - Chapter 12 — AOP
Used Later:
  - Phase 6 — Persistence (JPA/Hibernate)
Estimated Reading: 20 phút
Estimated Practice: 18 phút
```

## Story

> Xem lại: [Chapter 12](12-aop.md) — self-invocation (`this.method()`) khiến AOP proxy bị bỏ
> qua hoàn toàn. `@Transactional` **chính là một dạng AOP advice** — nghĩa là nó chịu **đúng
> giới hạn tương tự**. Điều này có thể gây ra một trong những lỗi khó phát hiện nhất trong lập
> trình Spring: API trả về `200 OK` thành công, nhưng dữ liệu cập nhật lại **không bao giờ**
> được ghi xuống database.

Chuyển 100 từ tài khoản Alice sang Bob, cố tình ném lỗi **sau khi** đã ghi cả hai tài khoản, so
sánh 4 kịch bản:

```
=== Goi TRUC TIEP internalTransferTx() qua proxy (co @Transactional that su) ===
Alice balance (ky vong ROLLBACK ve 1000): 1000.0

=== Unchecked exception (RuntimeException) ===
Alice balance SAU khi loi (ky vong ROLLBACK ve 1000): 1000.0

=== Goi placeOrderNoTx() -> TU GOI internalTransferTx() qua 'this' (SELF-INVOCATION) ===
Alice balance (self-invocation, @Transactional bi BO QUA, KHONG rollback?): 900.0

=== Checked exception (Exception thuong) ===
Alice balance SAU khi loi (MAC DINH KHONG rollback): 900.0
```

Hai kịch bản đầu rollback đúng (balance quay về `1000.0`). Nhưng **self-invocation** và
**checked exception** đều khiến balance dừng lại ở `900.0` — tiền đã "biến mất" khỏi tài khoản
Alice mà **không hề rollback**, dù cả hai đều ném ra exception.

## Interview Question (Central)

> Your `@Transactional` method is executing, but the transaction is not rolling back. Why? Your
> REST API returns 200 OK, but the database update never happens. How is that possible?
> (self-invocation / proxy)

## Objectives

- [ ] Giải thích `@Transactional` được hiện thực hoá qua AOP proxy, chịu chung giới hạn
      self-invocation đã học ở Chapter 12
- [ ] Tự tay chứng minh bằng thực nghiệm: self-invocation khiến `@Transactional` bị bỏ qua hoàn
      toàn — dữ liệu không rollback dù có exception
- [ ] Tự tay chứng minh quy tắc rollback mặc định: unchecked exception rollback, checked
      exception thì KHÔNG (trừ khi khai báo tường minh)

## Prerequisites

- Chapter 12 — bắt buộc phải hiểu AOP proxy và self-invocation limitation trước chapter này.

## Used Later

- **Phase 6 (Persistence)** — transaction management là nền tảng bắt buộc khi làm việc với
  JPA/Hibernate.

## Problem

Một thao tác nghiệp vụ thường gồm **nhiều bước ghi dữ liệu liên quan tới nhau** (trừ tiền tài
khoản A, cộng tiền tài khoản B) — nếu chỉ một bước thất bại giữa chừng, để lại các bước đã thành
công trước đó **không được hoàn tác**, dữ liệu rơi vào trạng thái không nhất quán (tiền "biến
mất" hoặc "tự nhân đôi"). Quản lý điều này thủ công (tự viết `try-catch` gọi rollback từng bước)
cực kỳ dễ sai sót và lặp lại nhiều nơi.

## Concept

**`@Transactional`** đánh dấu một method cần chạy trong **một transaction database duy nhất** —
Spring tự động `begin transaction` trước khi method chạy, `commit` nếu method hoàn tất bình
thường, hoặc **`rollback`** (hoàn tác mọi thay đổi đã thực hiện trong method đó) nếu có exception
xảy ra. Cơ chế này hiện thực hoá qua **AOP proxy** (chính xác đúng cơ chế đã học ở Chapter 12) —
một `TransactionInterceptor` bọc quanh method, mở transaction trước khi gọi method thật, và
quyết định commit/rollback sau khi method đó chạy xong (thành công hoặc ném exception).

## Why?

Vì `@Transactional` dùng chung cơ chế AOP proxy với mọi advice khác (Chapter 12), nó **kế thừa
chính xác** giới hạn self-invocation: proxy chỉ chặn được lời gọi **từ bên ngoài** class, không
chặn được khi method tự gọi chính nó (hoặc method khác trong cùng class) qua `this`. Về quy tắc
rollback: Spring mặc định chỉ rollback khi gặp **unchecked exception** (`RuntimeException`,
`Error`) — phản ánh triết lý thiết kế của Spring: checked exception (Phase 1, Chapter 15) thường
đại diện cho một điều kiện **có thể phục hồi/xử lý được** trong logic nghiệp vụ (ví dụ hiển thị
thông báo cho người dùng), trong khi unchecked exception thường đại diện cho **lỗi không lường
trước**, cần huỷ bỏ toàn bộ thao tác đang dở dang.

## How?

```java
@Transactional
public void transfer(Long fromId, Long toId, double amount) {
    // moi thay doi trong method nay CUNG mot transaction
    from.setBalance(from.getBalance() - amount);
    repo.save(from);
    to.setBalance(to.getBalance() + amount);
    repo.save(to);
    // neu co RuntimeException nem ra o day -> TU DONG rollback CA HAI thay doi tren
}

@Transactional(rollbackFor = Exception.class) // BAT rollback ca checked exception
public void transferStrict(...) throws Exception { ... }
```

## Visualization

```
Truong hop 1: Goi TU BEN NGOAI qua proxy (dung):

  Client ──► Proxy.internalTransferTx() ──► [BEGIN TX] ──► method that chay, nem RuntimeException
                                                 │
                                            [ROLLBACK] (proxy bat duoc exception, huy giao dich)
  => Alice balance: 1000.0 (dung nhu ban dau)

Truong hop 2: SELF-INVOCATION (this.internalTransferTx() tu ben trong):

  Client ──► Proxy.placeOrderNoTx() ──► [KHONG CO @Transactional tren placeOrderNoTx]
                                            │
                                       instance GOC.placeOrderNoTx() dang chay
                                            │
                                       this.internalTransferTx()  <- KHONG QUA PROXY!
                                            │
                                       chay TRUC TIEP, KHONG CO transaction nao duoc mo
                                       repo.save() COMMIT NGAY (auto-commit tung cau lenh)
                                            │
                                       nem RuntimeException -> KHONG AI rollback duoc nua!
  => Alice balance: 900.0 (DA MAT TIEN, khong rollback duoc)

Truong hop 3: Checked exception (KHONG khai bao rollbackFor):

  Proxy.transferCheckedFails() ──► [BEGIN TX] ──► method nem Exception (checked)
                                        │
                                   Proxy THAY: day la checked exception
                                        │
                                   MAC DINH: COMMIT (khong rollback!)
  => Alice balance: 900.0 (DA MAT TIEN, vi checked exception KHONG kich hoat rollback mac dinh)
```

## Example

```java
@Service
public class TransferService {
    @Transactional
    public void internalTransferTx(Long fromId, Long toId, double amount) {
        doTransfer(fromId, toId, amount);
        throw new RuntimeException("Loi trong internalTransferTx!");
    }

    // KHONG co @Transactional, tu goi internalTransferTx() qua 'this'
    public void placeOrderNoTx(Long fromId, Long toId, double amount) {
        this.internalTransferTx(fromId, toId, amount); // SELF-INVOCATION!
    }

    @Transactional
    public void transferCheckedFails(Long fromId, Long toId, double amount) throws Exception {
        doTransfer(fromId, toId, amount);
        throw new Exception("Loi nghiep vu (checked)!"); // checked exception
    }
}
```

Kết quả thật (Spring Boot 3.3.4, H2 in-memory database):

```
=== Goi TRUC TIEP internalTransferTx() qua proxy (co @Transactional that su) ===
Alice balance (ky vong ROLLBACK ve 1000): 1000.0

=== Goi placeOrderNoTx() -> TU GOI internalTransferTx() qua 'this' (SELF-INVOCATION) ===
Alice balance (self-invocation, @Transactional bi BO QUA, KHONG rollback?): 900.0

=== Checked exception (Exception thuong) ===
Alice balance SAU khi loi (MAC DINH KHONG rollback): 900.0
```

Gọi trực tiếp `internalTransferTx()` từ bên ngoài (qua proxy thật): rollback đúng, balance quay
về `1000.0`. Gọi qua `placeOrderNoTx()` (self-invocation): balance dừng ở `900.0` — tiền đã bị
trừ khỏi Alice **vĩnh viễn**, dù `RuntimeException` đã được ném ra — vì `@Transactional` trên
`internalTransferTx()` **hoàn toàn không được kích hoạt** khi gọi qua `this`. Với checked
exception (`transferCheckedFails`, không khai báo `rollbackFor`): balance cũng dừng ở `900.0` —
transaction **commit bình thường** dù có exception, vì mặc định Spring chỉ rollback cho unchecked
exception.

## Deep Dive

**Vì sao self-invocation trong ngữ cảnh `@Transactional` nguy hiểm hơn hẳn so với AOP logging
đơn thuần (Chapter 12)?** Với AOP logging, self-invocation chỉ khiến **mất một vài dòng log** —
gây bất tiện nhưng vô hại về mặt dữ liệu. Với `@Transactional`, self-invocation khiến method
**hoàn toàn không chạy trong transaction nào cả** — mỗi câu lệnh `repo.save()` bên trong đó chạy
theo chế độ **auto-commit** của database (commit ngay lập tức, độc lập với nhau). Khi exception
xảy ra **sau khi** một số câu lệnh đã auto-commit, không có transaction nào để rollback — dữ liệu
đã ghi vĩnh viễn ở trạng thái **không nhất quán** (Alice mất tiền nhưng Bob không nhận được, tuỳ
thứ tự thao tác), và ứng dụng **không hề biết** điều này đã xảy ra — không có exception nào báo
hiệu "transaction không hoạt động", vì về mặt cú pháp, code hoàn toàn hợp lệ, chỉ là annotation
đã bị bỏ qua một cách âm thầm.

## Engineering Insight

**Vì sao Spring mặc định chỉ rollback cho unchecked exception — quyết định này có phải một cạm
bẫy thiết kế, hay có lý do chính đáng?** Đây thực chất là điểm khác biệt lớn với hành vi mặc
định của EJB (chuẩn Java EE cũ) — EJB mặc định **rollback cho mọi exception**. Spring cố tình
chọn ngược lại: checked exception, theo triết lý thiết kế Java (Phase 1, Chapter 15), thường
biểu thị một điều kiện nghiệp vụ có thể **dự đoán trước và xử lý được** (ví dụ
`InsufficientFundsException` — số dư không đủ, một tình huống hợp lệ về nghiệp vụ, không phải
lỗi hệ thống) — code gọi có thể muốn **giữ lại** một phần thay đổi đã thực hiện (ví dụ ghi log
audit về lần thử chuyển tiền thất bại) thay vì rollback toàn bộ. Dù có lý do thiết kế hợp lý,
đây vẫn là một trong những "cạm bẫy" gây nhầm lẫn nhiều nhất cho lập trình viên mới chuyển từ
nền tảng khác — khuyến nghị thực tế phổ biến nhất là **luôn khai báo tường minh**
`@Transactional(rollbackFor = Exception.class)` khi có bất kỳ khả năng nào ném checked exception
bên trong, thay vì dựa vào hành vi mặc định dễ gây hiểu lầm.

## Historical Note

`@Transactional` (annotation-based) xuất hiện từ Spring 2.0 (2006) — trước đó, transaction phải
khai báo qua XML (`<tx:advice>`) hoặc lập trình thủ công với `TransactionTemplate`. Việc kế thừa
cùng cơ chế AOP proxy (thay vì một cơ chế hoàn toàn riêng biệt) là quyết định thiết kế nhất quán
của Spring — cùng một nền tảng kỹ thuật (`BeanPostProcessor` tạo proxy, Chapter 05-06) phục vụ
cho cả logging, security, caching (Chapter 16), và transaction — giúp toàn bộ hệ sinh thái Spring
AOP-based dễ học và dễ đoán hành vi hơn khi đã hiểu rõ nguyên lý gốc.

## Myth vs Reality

- **Myth:** "`@Transactional` đảm bảo rollback cho MỌI loại exception."
  **Reality:** Đã chứng minh bằng thực nghiệm — mặc định chỉ rollback cho unchecked exception
  (`RuntimeException`/`Error`); checked exception yêu cầu khai báo tường minh `rollbackFor`.

- **Myth:** "Chỉ cần đánh dấu `@Transactional` trên method là chắc chắn transaction hoạt động,
  bất kể method đó được gọi từ đâu."
  **Reality:** Đã chứng minh bằng thực nghiệm — self-invocation khiến `@Transactional` bị bỏ
  qua hoàn toàn, không có bất kỳ cảnh báo hay exception nào báo hiệu điều này.

## Common Mistakes

- **Gọi một method `@Transactional` qua `this` từ bên trong cùng class** — lỗi self-invocation
  phổ biến nhất, gây mất dữ liệu âm thầm như đã chứng minh.
- **Ném checked exception bên trong method `@Transactional` mà không khai báo `rollbackFor`** —
  dữ liệu commit dù có lỗi, một trong những nguồn gốc phổ biến của bug "dữ liệu không nhất quán"
  khó tái hiện trong production.
- **Đặt `@Transactional` ở tầng Controller thay vì tầng Service** — làm transaction kéo dài
  không cần thiết qua cả logic xử lý HTTP request/response, tăng nguy cơ giữ kết nối database
  lâu hơn mức cần thiết.

## Best Practices

- Luôn đặt `@Transactional` ở tầng Service, trên method public (self-invocation không phải vấn
  đề khi gọi từ Controller — đó là lời gọi từ bên ngoài, đi qua proxy đúng cách).
- Khai báo tường minh `@Transactional(rollbackFor = Exception.class)` khi method có khả năng
  ném checked exception mà bạn muốn rollback.
- Tránh gọi các method `@Transactional` khác qua `this` trong cùng class — tách ra bean riêng
  nếu cần (đúng giải pháp đã học ở Chapter 12).
- Viết test tích hợp (integration test) xác nhận rollback hoạt động đúng cho các luồng nghiệp vụ
  quan trọng, thay vì chỉ tin tưởng annotation "chắc chắn đúng".

## Production Notes

**Vấn đề:** một API chuyển khoản trả về `200 OK` thành công, nhưng thỉnh thoảng người dùng báo
cáo bị trừ tiền mà bên nhận không nhận được.

- **Triệu chứng:** không có exception nào xuất hiện trong log ứng dụng (vì self-invocation không
  gây lỗi biên dịch/runtime rõ ràng), chỉ phát hiện qua đối chiếu số liệu kế toán định kỳ.
- **Root cause:** method xử lý chuyển khoản chính (`@Transactional`) được một method khác trong
  cùng Service gọi qua `this` (self-invocation), khiến transaction không bao giờ thực sự mở —
  mỗi bước ghi dữ liệu auto-commit độc lập.
- **Debug:** viết test tích hợp cố tình ném exception giữa chừng, xác nhận dữ liệu có rollback
  đúng không; kiểm tra code có lời gọi `this.methodTransactional()` nào trong cùng class không.
- **Solution:** tách method `@Transactional` ra một bean riêng, gọi qua dependency injection
  thay vì `this`.
- **Prevention:** thiết lập quy tắc code review bắt buộc kiểm tra self-invocation cho mọi method
  `@Transactional`; viết test tích hợp cho mọi luồng nghiệp vụ có thao tác ghi dữ liệu nhiều
  bước.

## Debug Checklist

- [ ] API trả về thành công nhưng dữ liệu không nhất quán/không được ghi đúng? → kiểm tra
      method `@Transactional` có bị gọi qua self-invocation không.
- [ ] Transaction không rollback dù có exception rõ ràng? → kiểm tra exception đó là checked hay
      unchecked; nếu checked, cần khai báo `rollbackFor` tường minh.
- [ ] Cần xác nhận transaction có thực sự đang hoạt động? → kiểm tra
      `TransactionSynchronizationManager.isActualTransactionActive()` hoặc bật log
      `logging.level.org.springframework.transaction=DEBUG` để theo dõi begin/commit/rollback.

## Summary

`@Transactional` hiện thực hoá qua AOP proxy (Chapter 12) — mở transaction trước khi method
chạy, commit nếu thành công, rollback nếu có unchecked exception. Đã chứng minh bằng thực
nghiệm bốn kịch bản: gọi trực tiếp qua proxy rollback đúng; **self-invocation** (`this.method()`
từ bên trong cùng class) khiến `@Transactional` bị bỏ qua hoàn toàn, dữ liệu commit từng phần
không thể rollback; **checked exception** không khai báo `rollbackFor` cũng khiến transaction
commit thay vì rollback theo đúng mặc định của Spring. Cả hai đều là nguồn gốc phổ biến của lỗi
"API trả về thành công nhưng dữ liệu sai" — cực kỳ khó phát hiện vì không có exception/cảnh báo
nào lộ ra ngoài.

## Interview Questions

- Your "@Transactional" method is executing, but the transaction is not rolling back. Why?
- Your REST API returns 200 OK, but the database update never happens. How is that possible?
  (self-invocation / proxy)

**Senior**

- Giải thích chi tiết vì sao self-invocation khiến `@Transactional` bị bỏ qua, dựa trên cơ chế
  AOP proxy đã học.
- Vì sao Spring mặc định không rollback cho checked exception? Đây có phải một quyết định thiết
  kế hợp lý không?

## Exercises

- [ ] Chạy lại `TransactionRollbackTest` ở trên, xác nhận cả 4 kịch bản cho kết quả đúng như mô
      tả.
- [ ] Sửa `transferCheckedFails` thêm `@Transactional(rollbackFor = Exception.class)`, chạy lại,
      xác nhận balance rollback đúng về giá trị ban đầu.
- [ ] Tách `internalTransferTx()` ra một bean `TransactionalTransferExecutor` riêng, tiêm vào
      `TransferService`, gọi qua bean đó thay vì `this` trong `placeOrderNoTx()`, xác nhận
      rollback hoạt động đúng.

## Cheat Sheet

| Kịch bản | Rollback? |
| --- | --- |
| Gọi từ bên ngoài (qua proxy), unchecked exception | Có |
| Gọi từ bên ngoài (qua proxy), checked exception, không khai báo `rollbackFor` | **Không** |
| Gọi từ bên ngoài (qua proxy), checked exception, có `rollbackFor = Exception.class` | Có |
| Self-invocation (`this.method()`), bất kỳ loại exception nào | **Không** (proxy bị bỏ qua hoàn toàn) |

## References

- Spring Framework Documentation — Transaction Management: https://docs.spring.io/spring-framework/reference/data-access/transaction.html
- Spring Framework Documentation — "Understanding AOP Proxies" (self-invocation limitation, cùng
  áp dụng cho Transaction).
