---
tags:
  - Java
  - Exception
  - Foundation
---

# Exception

> Phase: Phase 1 — Java Foundation
> Chapter slug: `exception`

## Metadata

```yaml
Chapter: Exception
Phase: Phase 1 — Java Foundation
Difficulty: ★★★
Importance: ★★★★★
Interview Frequency: 85%
Prerequisites:
  - Chapter 08 — OOP Fundamentals (Inheritance — Exception là một cây kế thừa)
  - Chapter 04 — Runtime Data Areas (StackOverflowError/OutOfMemoryError là Error, không phải Exception)
Used Later:
  - Exception Handler (Phase 5 — Spring) — @ExceptionHandler xử lý đúng cây kế thừa học ở đây
  - Transaction (Phase 5/6/7) — RuntimeException mới trigger rollback mặc định, chủ đề kinh điển
  - JVM Tuning, Performance Tuning (Phase 8) — đọc stack trace là kỹ năng debug production cốt lõi
Estimated Reading: 25 phút
Estimated Practice: 25 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Chapter 04](04-runtime-data-areas.md) — bạn đã tự tay bắt được
> `StackOverflowError` bằng `catch (StackOverflowError e)`.

Nếu `catch` bắt được cả `Error`, tại sao Java vẫn phân ra `Exception` và `Error` làm hai
nhánh khác nhau? Và trong `Exception`, tại sao có loại bạn **bắt buộc** phải viết `throws`
hoặc `try-catch` (không viết thì báo lỗi biên dịch — gọi là *checked*), còn loại khác thì
không (gọi là *unchecked*)? Đây là một trong những quyết định thiết kế gây tranh cãi nhiều
nhất trong lịch sử Java — nhiều ngôn ngữ ra đời sau (Kotlin, C#) đã cố tình **bỏ** checked
exception. Chapter này giải thích rõ sự phân chia đó, và một cạm bẫy kinh điển gần như ai học
Java cũng từng dính: `finally` có thể **âm thầm nuốt chửng** một exception đang bay.

## Interview Question (Central)

> Checked exception và unchecked exception khác nhau như thế nào? Java dùng tiêu chí gì để
> quyết định một exception nên là loại nào?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Vẽ được cây kế thừa đầy đủ: `Throwable` → `Error`/`Exception` →
      `RuntimeException`/checked exception
- [ ] Phân biệt rạch ròi checked và unchecked exception, biết tiêu chí Java dùng để phân loại
- [ ] Tự viết được một custom exception đúng chuẩn
- [ ] Tự tay tái hiện được cạm bẫy `finally` nuốt chửng exception và `finally` ghi đè giá trị
      `return`
- [ ] Đọc hiểu một stack trace thật để xác định đúng nguyên nhân gốc

## Prerequisites

- Chapter 08 — hiểu Inheritance, vì toàn bộ hệ thống Exception là một cây kế thừa nhiều tầng.
- Chapter 04 — đã gặp `StackOverflowError`/`OutOfMemoryError`, hai ví dụ cụ thể của `Error`
  (không phải `Exception`).

## Used Later

- **Exception Handler** (Phase 5 — Spring) — `@ExceptionHandler`/`@ControllerAdvice` của
  Spring bắt exception dựa đúng trên cây kế thừa học ở chapter này — biết `IllegalArgumentException`
  kế thừa `RuntimeException` mới hiểu vì sao một `@ExceptionHandler(RuntimeException.class)`
  bắt được nó.
- **Transaction** (Phase 5/6/7) — mặc định Spring chỉ tự động rollback transaction khi gặp
  `RuntimeException` (unchecked), **không** rollback với checked exception — một trong những
  câu hỏi phỏng vấn kinh điển nhất về Spring, nền tảng chính là phân loại ở chapter này.
- **JVM Tuning, Performance Tuning** (Phase 8) — đọc đúng một stack trace dài, tìm "Caused
  by" thật sự, là kỹ năng debug production hàng ngày.

## Problem

Chương trình có thể gặp sự cố ở nhiều "tầng" khác nhau về mức độ nghiêm trọng và khả năng
phục hồi: file không tồn tại (thường có thể xử lý, thử đường dẫn khác), tham số đầu vào sai
(lỗi lập trình, cần sửa code), hay hết bộ nhớ (gần như không thể phục hồi — đã gặp ở Chapter
04). Nếu xử lý tất cả theo cùng một cách, chương trình sẽ hoặc cố "bắt" cả những lỗi không thể
phục hồi được (lãng phí, có thể che giấu vấn đề nghiêm trọng hơn), hoặc bỏ sót không xử lý
những lỗi hoàn toàn có thể lường trước và nên xử lý.

## Concept

Toàn bộ lỗi trong Java kế thừa từ `Throwable`, chia làm hai nhánh chính:

```
Throwable
 ├── Error                         (nghiêm trọng, thường KHÔNG nên tự bắt để "xử lý tiếp")
 │    ├── StackOverflowError        [Chapter 04]
 │    ├── OutOfMemoryError          [Chapter 04]
 │    └── ExceptionInInitializerError  [Chapter 14]
 │
 └── Exception
      ├── RuntimeException          (UNCHECKED — không bắt buộc try-catch/throws)
      │    ├── NullPointerException
      │    ├── IllegalArgumentException
      │    ├── ArrayIndexOutOfBoundsException
      │    └── ClassCastException   [Chapter 03]
      │
      └── (các Exception khác)      (CHECKED — bắt buộc try-catch/throws)
           ├── IOException          [Chapter 20]
           └── SQLException
```

**Checked exception** — trình biên dịch **bắt buộc** bạn phải xử lý (`try-catch`) hoặc khai
báo lan truyền tiếp (`throws`), nếu không sẽ báo lỗi biên dịch. **Unchecked exception**
(`RuntimeException` và các lớp con của nó) — không bắt buộc, chương trình vẫn biên dịch dù
không xử lý gì cả.

## Why?

Tiêu chí phân loại: **checked exception dành cho những tình huống bên ngoài tầm kiểm soát của
code, có khả năng phục hồi hợp lý** (file không tồn tại, mất kết nối mạng — code gọi có thể
làm gì đó hữu ích, như thử lại hoặc báo người dùng). **Unchecked exception (RuntimeException)
dành cho lỗi lập trình** — truyền `null` sai chỗ, truy cập chỉ số mảng ngoài phạm vi, ép kiểu
sai — về nguyên tắc, những lỗi này **không nên xảy ra nếu code viết đúng**, và việc bắt buộc
`try-catch` cho mọi lời gọi method có khả năng ném `NullPointerException` (tức là gần như MỌI
method) sẽ khiến code ngập tràn `try-catch` vô nghĩa, không thực sự "xử lý" được gì.

## How?

Viết một custom exception đúng chuẩn — kế thừa đúng nhánh tuỳ theo bạn muốn checked hay
unchecked:

```java
// Unchecked — kế thừa RuntimeException — dùng cho lỗi nghiệp vụ do dữ liệu/logic sai
class InsufficientStockException extends RuntimeException {
    InsufficientStockException(String message) {
        super(message);
    }
}

// Checked — kế thừa Exception (không phải RuntimeException) — dùng cho tình huống
// bên ngoài, code gọi NÊN được ép phải xử lý
class PaymentGatewayException extends Exception {
    PaymentGatewayException(String message, Throwable cause) {
        super(message, cause); // "cause" giữ lại exception GỐC — xem Deep Dive
    }
}
```

`try-catch-finally` xử lý theo thứ tự: `try` chạy, nếu ném exception thì nhảy tới `catch` phù
hợp (so khớp theo cây kế thừa, `catch` đầu tiên khớp được chọn), và **`finally` luôn chạy**
— dù `try` thành công, dù `catch` xử lý xong, hay dù không `catch` nào khớp và exception đang
lan truyền ra ngoài.

## Visualization

```
try {
    // (1) chạy
} catch (SpecificException e) {
    // (2a) chạy NẾU (1) ném ra SpecificException hoặc lớp con của nó
} catch (Exception e) {
    // (2b) chạy NẾU (1) ném exception KHÁC, khớp với Exception (bắt rộng hơn)
} finally {
    // (3) LUÔN LUÔN chạy — bất kể (1) thành công, (2a)/(2b) chạy, hay
    //     exception không khớp catch nào và đang lan tiếp ra ngoài
}
```

## Example

**Custom exception + try-catch-finally cơ bản:**

```java
class InsufficientStockException extends RuntimeException {
    InsufficientStockException(String message) { super(message); }
}

public class ExceptionDemo {
    static void checkout(int stock, int quantity) {
        if (quantity > stock) {
            throw new InsufficientStockException(
                "Chỉ còn " + stock + " sản phẩm, không đủ cho " + quantity);
        }
        System.out.println("Đặt hàng thành công: " + quantity + " sản phẩm");
    }

    public static void main(String[] args) {
        try {
            checkout(5, 10);
        } catch (InsufficientStockException e) {
            System.out.println("Lỗi nghiệp vụ: " + e.getMessage());
        } finally {
            System.out.println("Đóng log giao dịch (luôn chạy)");
        }
    }
}
```

**Hai cạm bẫy kinh điển của `finally`** — chạy thật để tự chứng kiến:

```java
public class FinallyTrap {
    static int risky() {
        try {
            throw new RuntimeException("loi");
        } finally {
            return 42; // ⚠️
        }
    }

    static int withResource() {
        int x = 1;
        try {
            return x;
        } finally {
            x = 99; // ⚠️
        }
    }

    public static void main(String[] args) {
        System.out.println("risky() = " + risky());
        System.out.println("withResource() = " + withResource());
    }
}
```

Kết quả thật (JDK 17):

```
risky() = 42
withResource() = 1
```

Hai điều đáng kinh ngạc, cả hai đều đúng và đều là hành vi **có chủ đích** của Java, không phải
bug:

- `risky()` trả về `42` — dù `try` đã ném `RuntimeException("loi")`! Câu lệnh `return` trong
  `finally` **hoàn toàn nuốt chửng** exception đang bay, người gọi `risky()` **không bao giờ
  biết** đã từng có lỗi xảy ra.
- `withResource()` trả về `1`, không phải `99` — vì `return x` đã **chốt giá trị của `x` tại
  thời điểm đó** (là `1`) trước khi `finally` chạy; việc `finally` sửa `x = 99` sau đó không
  ảnh hưởng tới giá trị đã chốt để trả về.

## Deep Dive

**"Exception chaining" — vì sao constructor `PaymentGatewayException` ở phần How? có tham số
`Throwable cause`?** Khi bắt một exception ở tầng thấp (ví dụ `SQLException` khi gọi cổng
thanh toán) rồi ném lại một exception ở tầng cao hơn có ý nghĩa nghiệp vụ hơn
(`PaymentGatewayException`), nếu **không** giữ lại exception gốc, thông tin quý giá nhất để
debug (chính xác dòng code nào, nguyên nhân kỹ thuật nào) sẽ **mất vĩnh viễn**:

```java
try {
    callPaymentGateway();
} catch (SQLException e) {
    throw new PaymentGatewayException("Không kết nối được cổng thanh toán", e); // giữ "e"
    // SAI: throw new PaymentGatewayException("Không kết nối được cổng thanh toán");
    //      (mất hoàn toàn thông tin SQLException gốc)
}
```

Stack trace khi in ra sẽ hiện dòng `Caused by: java.sql.SQLException: ...` bên dưới exception
chính — đây chính là "Caused by" bạn thường thấy trong log production, và là lý do luôn phải
đọc **tới tận cùng** một stack trace, không chỉ đọc dòng đầu tiên (xem Debug Checklist).

## Engineering Insight

**Vì sao nhiều ngôn ngữ ra đời sau Java (Kotlin, C#, Python với cách tiếp cận khác) lại chọn
bỏ hẳn checked exception?** Ý tưởng ban đầu của checked exception (ép buộc xử lý lỗi có thể
phục hồi) hợp lý về lý thuyết, nhưng trong thực tế thường dẫn tới hai hệ quả không mong muốn:
(1) lập trình viên viết `catch (Exception e) { }` rỗng chỉ để code biên dịch được, **tệ hơn**
việc không xử lý gì cả — vì nó **âm thầm nuốt lỗi** mà không ai biết; (2) một checked exception
ở tầng rất sâu (ví dụ `SQLException`) buộc **mọi method ở mọi tầng gọi tới nó** phải khai báo
`throws` hoặc bọc lại, lan truyền một chi tiết hiện thực cấp thấp lên tận các tầng cao không
liên quan — vi phạm chính nguyên tắc Encapsulation đã học ở Chapter 08. Đây là lý do trong
thực hành Java hiện đại (đặc biệt trong Spring, sẽ gặp ở Phase 5), xu hướng phổ biến là **bọc
checked exception thành unchecked** ngay khi có thể — không né tránh checked exception hoàn
toàn (vẫn hữu ích ở ranh giới rõ ràng, như I/O — Chapter 20), nhưng không lan truyền nó khắp
toàn bộ codebase.

## Historical Note

Checked exception xuất hiện từ Java 1.0 (1996), lấy cảm hứng từ mong muốn tạo ra API "tự tài
liệu hoá" — nhìn `throws` là biết ngay method có thể thất bại thế nào. Qua gần 30 năm sử dụng
thực tế trên quy mô lớn, cộng đồng Java dần đồng thuận rằng thiết kế này có nhiều đánh đổi hơn
dự tính ban đầu (xem Engineering Insight) — bản thân JDK cũng có những API mới hơn né tránh
checked exception (ví dụ Stream API, Phase 4, không cho phép lambda ném checked exception một
cách tự nhiên, một nguồn gây khó chịu nổi tiếng khi kết hợp Stream với code ném checked
exception).

## Myth vs Reality

- **Myth:** "`finally` chỉ chạy khi `try`/`catch` không có `return`."
  **Reality:** `finally` **luôn luôn chạy**, kể cả khi `try`/`catch` có `return` — và tệ hơn,
  nếu `finally` cũng có `return`, nó sẽ **ghi đè** giá trị trả về của `try`/`catch` (xem
  `risky()` ở Example) — đây là lý do các linter/style guide hiện đại cấm tuyệt đối viết
  `return` bên trong `finally`.

- **Myth:** "Bắt `Exception` chung chung (`catch (Exception e)`) là cách an toàn để không bỏ
  sót lỗi nào."
  **Reality:** Bắt quá rộng che giấu luôn cả những lỗi lập trình nghiêm trọng
  (`NullPointerException`, `ClassCastException`...) mà lẽ ra nên được phát hiện và sửa ngay
  khi phát triển, không nên "nuốt" và tiếp tục chạy như thể không có gì xảy ra.

## Common Mistakes

- **`catch (Exception e) { }` rỗng** — "nuốt" mọi lỗi trong im lặng, một trong những
  anti-pattern tệ nhất trong toàn bộ Java, khiến hệ thống lỗi mà không ai biết cho tới khi hậu
  quả nghiệp vụ nghiêm trọng xảy ra.
- **`return` trong `finally`** — nuốt chửng exception đang bay, xem `risky()` ở Example.
- **Bọc lại exception nhưng không giữ `cause`** — mất thông tin gốc, xem Deep Dive.
- **Dùng exception để điều khiển luồng chương trình bình thường** (không phải tình huống lỗi
  thật sự) — tạo stack trace tốn hiệu năng không cần thiết, và làm code khó đọc hơn so với
  dùng `if`/return giá trị thông thường.

## Best Practices

- Không bao giờ để khối `catch` rỗng — tối thiểu phải log lại, kể cả khi thực sự không cần xử
  lý gì thêm.
- Không bao giờ viết `return`/`break`/`continue` bên trong `finally`.
- Khi bọc exception, luôn truyền `cause` (tham số `Throwable` thứ hai của constructor) để giữ
  nguyên vẹn thông tin gốc.
- Chỉ dùng checked exception cho những tình huống code gọi **thực sự có khả năng làm gì đó
  khác** khi gặp lỗi (thử lại, thông báo người dùng); nếu chỉ để log rồi ném tiếp, cân nhắc
  unchecked exception để tránh buộc mọi tầng gọi phải khai báo `throws`.

## Production Notes

**Vấn đề:** một lỗi nghiệp vụ nghiêm trọng (ví dụ đơn hàng bị mất) xảy ra trong production
nhưng **không có bất kỳ dòng log lỗi nào**, hệ thống trông như vẫn chạy bình thường.

- **Triệu chứng:** không alert, không exception trong log, nhưng dữ liệu/kết quả sai — giống
  hệt "sai lệch dữ liệu âm thầm" đã gặp ở [Chapter 13](13-hashcode.md), nhưng nguyên nhân khác
  hẳn.
- **Root cause:** một khối `catch (Exception e) { }` rỗng (hoặc chỉ log ở mức `debug` không ai
  xem) đã nuốt chửng chính xác exception lẽ ra phải dừng giao dịch lại — đúng anti-pattern đã
  nêu ở Common Mistakes.
- **Debug:** tìm kiếm toàn bộ codebase các khối `catch` rỗng hoặc chỉ có comment
  (`// TODO: handle later`) quanh khu vực nghiệp vụ liên quan — đây gần như luôn là thủ phạm
  trong loại sự cố "im lặng" này.
- **Solution:** thêm log đầy đủ (ở mức `error`, có stack trace) cho mọi `catch`, xem xét lại
  có nên để lỗi lan truyền tiếp (không nuốt) thay vì cố "xử lý" mà thực chất là bỏ qua.
- **Prevention:** cấm `catch` rỗng bằng static analysis (SpotBugs/SonarQube có sẵn rule cho
  việc này — sẽ gặp ở giai đoạn triển khai CI/CD thực tế); code review chú ý đặc biệt tới mọi
  khối `catch`.

## Debug Checklist

- [ ] Đọc stack trace: luôn tìm dòng `Caused by:` **cuối cùng** (sâu nhất) trước — đó thường
      mới là nguyên nhân gốc thật sự, không phải exception ngoài cùng.
- [ ] Nghi ngờ lỗi bị "nuốt" mất, không thấy log nào dù chắc chắn có sự cố? → tìm khối `catch`
      rỗng hoặc `return` trong `finally` ở khu vực nghi ngờ.
- [ ] `NullPointerException` xảy ra? → đây là unchecked, lỗi lập trình — tìm chính xác biến
      nào là `null` (từ Java 14+, thông báo NPE đã chi tiết hơn nhiều, tự nêu rõ biến nào).
- [ ] Method trả về giá trị "sai" dù logic có vẻ đúng? → kiểm tra có `finally` nào đang âm
      thầm ghi đè giá trị trả về không (xem `risky()`/`withResource()` ở Example).

## Source Code Walkthrough

Cơ chế `try-catch-finally` được hiện thực ở tầng bytecode bằng một **bảng xử lý exception**
(exception table) đi kèm mỗi method — mỗi mục trong bảng ghi: phạm vi bytecode nào (từ-đến)
được bảo vệ, loại exception nào khớp, và nhảy tới đâu nếu khớp. `javap -v` (verbose, chi tiết
hơn `-c`) hiện được bảng này rõ ràng — một cách quan sát cụ thể việc "bắt exception" không có
gì huyền bí, chỉ là một bảng tra cứu bytecode đặc biệt được JVM tham chiếu khi có exception
bay ra giữa chừng.

## Summary

`Throwable` chia hai nhánh: `Error` (nghiêm trọng, không nên tự bắt để "xử lý tiếp") và
`Exception` — trong đó `RuntimeException` (unchecked, dành cho lỗi lập trình) và các
`Exception` khác (checked, bắt buộc `try-catch`/`throws`, dành cho tình huống bên ngoài có thể
phục hồi). `finally` luôn chạy, kể cả khi `try`/`catch` đã `return` — và `return` bên trong
`finally` sẽ **ghi đè hoàn toàn**, kể cả nuốt chửng exception đang bay, một anti-pattern tuyệt
đối cần tránh. Khi bọc exception ở tầng cao hơn, luôn giữ lại `cause` để không mất thông tin
gốc trong stack trace — kỹ năng đọc đúng "Caused by:" là nền tảng debug production hàng ngày.

## Interview Questions

**Junior**

- What are the different types of Exceptions in Java?
- Checked exception và unchecked exception khác nhau ở điểm nào?

**Mid**

- Coding: Create a custom Exception class.
- Coding: Write a program to demonstrate Exception Handling (try-catch-finally with a custom
  scenario).
- `finally` có luôn chạy không? Có trường hợp nào nó không chạy không?

**Senior**

- Giải thích chi tiết tại sao `return` trong `finally` lại nguy hiểm — dùng đúng ví dụ
  `risky()` ở chapter này để minh hoạ, kể cả với người chưa từng thấy hiện tượng này.
- Vì sao nhiều ngôn ngữ hiện đại (Kotlin, C#) từ bỏ checked exception? Bạn có đồng ý với thiết
  kế đó không, và trong Java, bạn sẽ áp dụng nguyên tắc gì để giảm thiểu nhược điểm của checked
  exception mà không bỏ hẳn nó?

## Exercises

- [ ] Chạy lại đúng ví dụ `FinallyTrap` ở trên, xác nhận `risky() = 42` và
      `withResource() = 1`, tự giải thích lại bằng lời của chính bạn vì sao.
- [ ] Viết một custom checked exception (kế thừa `Exception`, không phải `RuntimeException`),
      thử gọi method ném nó **mà không** `try-catch`/`throws` — quan sát lỗi biên dịch.
- [ ] Viết một chuỗi ba tầng gọi nhau (A gọi B gọi C), C ném exception, B bắt và bọc lại thành
      exception khác **có giữ `cause`**, A bắt và in `printStackTrace()` — xác nhận thấy được
      cả hai dòng "Caused by" trong output.

## Cheat Sheet

| Loại | Ví dụ | Bắt buộc xử lý? |
| --- | --- | --- |
| `Error` | `StackOverflowError`, `OutOfMemoryError` | Không (và thường không nên cố bắt) |
| `RuntimeException` (unchecked) | `NullPointerException`, `IllegalArgumentException` | Không |
| `Exception` khác (checked) | `IOException`, `SQLException` | Có — `try-catch` hoặc `throws` |

| Quy tắc `finally` |
| --- |
| Luôn chạy, kể cả khi `try`/`catch` có `return` |
| `return` trong `finally` GHI ĐÈ mọi `return` trước đó, kể cả nuốt exception đang bay |
| KHÔNG BAO GIỜ viết `return`/`break`/`continue` trong `finally` |

## References

- Java Language Specification (JLS) — Chapter 11: Exceptions.
- `java.lang.Throwable` — Java SE API Documentation (Oracle).
- Effective Java (Joshua Bloch) — Item 69-77 (chương về Exception).
