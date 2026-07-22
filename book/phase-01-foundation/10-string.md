---
tags:
  - Java
  - String
  - Immutable
  - Foundation
---

# String

> Phase: Phase 1 — Java Foundation
> Chapter slug: `string`

## Metadata

```yaml
Chapter: String
Phase: Phase 1 — Java Foundation
Difficulty: ★★★
Importance: ★★★★★
Interview Frequency: 90%
Prerequisites:
  - Chapter 04 — Runtime Data Areas (Heap, Metaspace)
  - Chapter 09 — Object (toString/equals/hashCode)
Used Later:
  - Immutable (Chapter 11) — String là ví dụ mở đầu cho nguyên tắc thiết kế immutable object
  - equals()/hashCode() (Chapter 12-13) — String là ví dụ override chuẩn mực
  - HashMap (Phase 2) — String làm key phổ biến nhất, dựa trên hashCode cached ở đây
  - Toàn bộ handbook — String xuất hiện trong hầu hết mọi ví dụ code
Estimated Reading: 25 phút
Estimated Practice: 25 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

Thử tưởng tượng — chỉ là giả định — `String` trong Java là **mutable** (có thể sửa nội dung
sau khi tạo), giống như một mảng ký tự bình thường. Chuyện gì sẽ xảy ra?

```
"admin" được dùng làm key trong HashMap lưu quyền hạn người dùng
        ↓
Ai đó (một đoạn code ở rất xa, không liên quan) sửa nội dung chuỗi "admin" thành "guest"
        ↓
HashMap tính lại hashCode dựa trên nội dung mới → tra cứu key "admin" giờ TRẢ VỀ SAI CHỖ,
hoặc không tìm thấy nữa — toàn bộ cấu trúc dữ liệu bên trong HashMap bị hỏng
        ↓
Nếu "admin" từng được dùng làm tên class truyền vào ClassLoader (Chapter 03)
        ↓
Việc sửa được chuỗi SAU KHI đã truyền cho hệ thống bảo mật là một lỗ hổng nghiêm trọng —
kẻ tấn công có thể "đánh tráo" giá trị sau khi đã qua bước kiểm tra
```

Đây không phải chuyện viễn tưởng — nó là chính xác lý do các nhà thiết kế Java quyết định
`String` phải **immutable** (bất biến) tuyệt đối ngay từ Java 1.0. Chapter này giải thích rõ
ba điều: `String` bất biến nghĩa là gì, vì sao bất biến lại quan trọng đến mức đó, và bên
trong nó thực sự hoạt động ra sao.

## Interview Question (Central)

> Vì sao String trong Java là immutable? Nếu String mutable thì điều gì sẽ thực sự xảy ra?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Giải thích được `String` immutable nghĩa là gì (khác với biến tham chiếu tới `String`
      có thể đổi)
- [ ] Nêu được 3 lý do cụ thể String phải bất biến: bảo mật, thread-safety, String Pool
- [ ] Hiểu **String Pool** hoạt động ra sao, phân biệt `"a" == "a"` với
      `new String("a") == "a"`
- [ ] Biết `hashCode()` của `String` được **cache** — hệ quả trực tiếp của tính bất biến
- [ ] Dùng đúng `StringBuilder` khi cần nối chuỗi nhiều lần trong vòng lặp

## Prerequisites

- Chapter 04 — biết Heap/Metaspace là gì, để hiểu String Pool nằm ở đâu.
- Chapter 09 — biết `equals()`/`hashCode()`/`toString()` mặc định của `Object`, để so sánh
  với cách `String` override chúng.

## Used Later

- **Immutable** (Chapter 11) — chapter đó tổng quát hoá nguyên tắc thiết kế immutable object
  cho **class của riêng bạn**; `String` ở chapter này là ví dụ nền tảng, sống động nhất.
- **equals()/hashCode()** (Chapter 12-13) — `String` là ví dụ kinh điển của một class override
  đúng cặp `equals()`/`hashCode()` theo nội dung, không theo tham chiếu (khác `Object` mặc
  định — Chapter 09).
- **HashMap** (Phase 2) — `String` là loại key phổ biến nhất trong thực tế; hiểu
  `hashCode()` được cache ở đây giải thích một phần vì sao dùng `String` làm key hiệu quả.

## Problem

`String` được dùng ở khắp mọi nơi trong một chương trình Java: tên class truyền cho
`ClassLoader` (Chapter 03), key trong `HashMap`, nội dung SQL query, đường dẫn file, dữ liệu
người dùng nhập vào. Rất nhiều trong số này là **dữ liệu nhạy cảm về mặt an toàn** — nếu giá
trị chuỗi có thể bị thay đổi bởi bất kỳ đoạn code nào đang giữ tham chiếu tới nó, sau khi đã
được một hệ thống khác (bảo mật, classloader, hash-based collection) kiểm tra/xử lý, hậu quả
có thể rất nghiêm trọng (xem Story).

## Concept

**Immutable** nghĩa là: một khi object `String` đã được tạo ra, **nội dung bên trong nó không
bao giờ thay đổi được nữa** trong suốt vòng đời object đó. Mọi method trông như "sửa" chuỗi
(`concat()`, `replace()`, `toUpperCase()`...) thực chất **luôn tạo ra một object `String` mới**,
không đụng vào object cũ.

```java
String a = "hello";
a = a.concat(" world");
```

Dòng 2 **không** sửa object `"hello"` — nó tạo một object `String` hoàn toàn mới chứa
`"hello world"`, rồi gán **tham chiếu** `a` sang trỏ tới object mới đó. Object `"hello"` cũ
vẫn còn nguyên vẹn, không đổi (chỉ là không còn ai giữ tham chiếu tới nó nữa — đủ điều kiện
để GC dọn, xem lại [Chapter 07](07-garbage-collection.md)).

## Why?

Ba lý do cụ thể `String` phải bất biến:

1. **Bảo mật.** Tham số kiểu `String` (tên file, tên class, URL, câu lệnh SQL...) thường được
   kiểm tra/validate **trước** khi dùng. Nếu `String` mutable, một đoạn code độc hại giữ tham
   chiếu tới cùng object có thể sửa nội dung **sau khi** đã qua bước kiểm tra nhưng
   **trước khi** thực sự được dùng — một dạng tấn công TOCTOU (Time-Of-Check to
   Time-Of-Use).
2. **Thread-safety.** Một object bất biến có thể được nhiều thread cùng đọc **an toàn tuyệt
   đối** mà không cần `synchronized` (sẽ gặp ở Phase 3) — vì không có ai sửa được nó, không
   có khái niệm "race condition" khi đọc dữ liệu bất biến.
3. **String Pool an toàn.** Xem phần How? — nếu `String` mutable, việc nhiều biến cùng trỏ
   tới **một object dùng chung** trong String Pool sẽ cực kỳ nguy hiểm: sửa một biến sẽ vô
   tình làm thay đổi giá trị của mọi biến khác đang "vô tình" trỏ chung object đó.

## How?

**String Pool** (còn gọi String Intern Pool) là một vùng nhớ đặc biệt (nằm trong Heap từ Java
7 trở đi — xem Historical Note) nơi JVM lưu các chuỗi literal (viết trực tiếp bằng dấu
`"..."`) để **dùng chung**, tránh tạo nhiều object trùng nội dung một cách lãng phí:

```java
String a = "hello";       // literal → JVM tìm/tạo trong String Pool
String b = "hello";       // literal giống hệt → TÁI SỬ DỤNG object trong Pool, không tạo mới
String c = new String("hello");  // "new" → LUÔN tạo object MỚI ngoài Pool, dù nội dung giống
String d = c.intern();    // ép lấy (hoặc thêm vào) bản trong Pool ứng với nội dung "hello"
```

Việc dùng chung này **chỉ an toàn** vì `String` bất biến — nếu `a` sửa được nội dung, `b`
(đang trỏ chung một object) sẽ bị ảnh hưởng theo dù code không hề đụng tới biến `b`.

## Visualization

```
String Pool (trong Heap, dùng chung)          Heap (object thường)
┌─────────────────────┐
│   "hello"  ◀─────────┼──── a (biến, chỉ là tham chiếu)
│      ▲                │
└──────┼───────────────┘
       └──────────────────── b (biến khác, TRỎ CHUNG object với a)

                                               ┌─────────────────┐
                                    c ─────────▶│  "hello" (bản   │
                                               │   RIÊNG, do new) │
                                               └─────────────────┘
```

`a` và `b` trỏ chung **một** object trong Pool (`a == b` là `true`). `c` là một object hoàn
toàn khác, nằm ngoài Pool (`a == c` là `false`), dù nội dung ký tự giống hệt nhau
(`a.equals(c)` vẫn là `true`).

## Example

Xác nhận toàn bộ lý thuyết trên bằng thực nghiệm thật:

```java
public class StringPoolDemo {
    public static void main(String[] args) {
        String a = "hello";
        String b = "hello";
        String c = new String("hello");
        String d = c.intern();

        System.out.println("a == b: " + (a == b));
        System.out.println("a == c: " + (a == c));
        System.out.println("a == d: " + (a == d));
        System.out.println("a.equals(c): " + a.equals(c));
    }
}
```

Kết quả thật (JDK 17):

```
a == b: true
a == c: false
a == d: true
a.equals(c): true
```

Khớp chính xác với sơ đồ ở phần Visualization: `a`/`b` cùng một object trong Pool (`==` true);
`c` là object riêng do `new` (`==` false với `a`); `d` là kết quả `intern()` — ép lấy bản
trong Pool, nên lại `==` với `a`; nhưng `equals()` luôn `true` vì so sánh **nội dung**, không
phụ thuộc object nào đang giữ nội dung đó.

## Deep Dive

**`hashCode()` của `String` được cache — hệ quả trực tiếp của bất biến.** Vì nội dung
`String` không bao giờ đổi, `hashCode()` của nó cũng **không bao giờ đổi** trong suốt vòng đời
object — JVM tận dụng điều này bằng cách chỉ tính `hashCode()` **một lần duy nhất** (lần đầu
được gọi), lưu lại kết quả trong một field nội bộ, các lần gọi sau chỉ trả về giá trị đã lưu,
không tính lại. Đây là tối ưu **chỉ có thể thực hiện được vì bất biến** — nếu `String` mutable,
việc cache `hashCode()` sẽ sai ngay khi nội dung bị sửa mà không tính lại. Tối ưu này có ảnh
hưởng thực tế: dùng `String` làm key trong `HashMap` (Phase 2) với cùng nội dung được tra cứu
lặp lại nhiều lần sẽ hưởng lợi trực tiếp từ việc không phải tính lại hash mỗi lần.

**Compact Strings (Java 9, JEP 254).** Trước Java 9, `String` lưu nội dung bằng `char[]`
(mỗi ký tự chiếm 2 byte, kể cả ký tự ASCII thuần chỉ cần 1 byte). Từ Java 9, `String` chuyển
sang lưu bằng `byte[]` kèm một cờ `coder` đánh dấu encoding (Latin-1 một byte/ký tự, hoặc
UTF-16 hai byte/ký tự khi cần) — với chuỗi chủ yếu là ký tự ASCII/Latin-1 (rất phổ biến trong
thực tế), thay đổi này giảm gần một nửa bộ nhớ dùng cho `String` trên toàn hệ thống mà không
cần đổi bất kỳ dòng code nào ở tầng ứng dụng.

## Engineering Insight

**Vì sao nối chuỗi bằng `+` trong vòng lặp bị coi là anti-pattern, trong khi Chapter 01 lại
cho thấy `+` được biên dịch thành `invokedynamic` tối ưu?** Hai điều này không mâu thuẫn — mấu
chốt nằm ở **số lượng phép nối** và **phạm vi biên dịch**. `invokedynamic`/
`StringConcatFactory` (Chapter 01) tối ưu tốt cho một biểu thức nối chuỗi **cố định trong một
dòng code** (như `"a" + b + "c"`), vì `javac` biết chính xác tại thời điểm biên dịch cần nối
bao nhiêu phần. Nhưng trong vòng lặp:

```java
String result = "";
for (int i = 0; i < 1000; i++) {
    result += i;  // MỖI VÒNG LẶP tạo một String MỚI (vì bất biến!), object cũ thành rác
}
```

— mỗi lần `result += i` chạy là một lần gọi nối chuỗi **độc lập** (một `invokedynamic` riêng
mỗi vòng lặp), tạo ra một `String` mới mỗi lần, bỏ object cũ làm rác cho GC dọn (Chapter 07).
Với 1000 vòng lặp, chương trình tạo ra **tới 1000 object `String` trung gian** chỉ để cuối
cùng dùng đúng một object cuối. Đây chính là hệ quả trực tiếp của tính bất biến: không thể
"sửa tại chỗ", bắt buộc phải tạo mới liên tục. Giải pháp là `StringBuilder` — một class
**mutable** riêng biệt, dùng một buffer nội bộ có thể sửa tại chỗ, chỉ tạo `String` bất biến
**một lần duy nhất** ở bước cuối (`.toString()`).

## Historical Note

```
Trước Java 7
    ↓
String Pool nằm trong PermGen (xem Chapter 03/04) — kích thước cố định,
dễ gây OutOfMemoryError: PermGen space nếu chương trình intern() quá nhiều
chuỗi khác nhau
    ↓
Java 7 (2011)
    ↓
String Pool chuyển sang nằm trong HEAP thường — tận dụng khả năng mở rộng
linh hoạt của Heap, cũng được GC dọn như object thường nếu không còn ai giữ
tham chiếu (kể cả từ trong Pool)
    ↓
Java 9 (2017)
    ↓
JEP 254: Compact Strings — đổi cách lưu trữ nội bộ char[] → byte[] + coder
(xem Deep Dive), giảm đáng kể bộ nhớ dùng cho String trên toàn hệ thống
```

## Myth vs Reality

- **Myth:** "`String` immutable nghĩa là biến kiểu `String` không đổi được giá trị."
  **Reality:** Biến `String` (tham chiếu) đổi được bình thường
  (`a = a.concat(" world")` ở phần Concept hoàn toàn hợp lệ). Cái **không đổi được** là
  **nội dung của chính object** `String` đã tạo ra — object cũ luôn giữ nguyên, biến chỉ trỏ
  sang object mới.

- **Myth:** "Dùng `+` để nối chuỗi luôn kém hiệu quả, phải luôn dùng `StringBuilder`."
  **Reality:** Với một biểu thức nối chuỗi cố định trong một dòng, `javac` (từ Java 9) đã tự
  tối ưu bằng `invokedynamic`/`StringConcatFactory` (Chapter 01) — hiệu quả tương đương hoặc
  tốt hơn tự viết `StringBuilder` thủ công. Vấn đề hiệu năng chỉ thật sự xảy ra khi nối chuỗi
  **lặp lại nhiều lần trong vòng lặp** (xem Engineering Insight).

## Common Mistakes

- **Dùng `==` để so sánh nội dung `String` thay vì `.equals()`.** Lỗi cực kỳ phổ biến với
  người mới, đặc biệt nguy hiểm vì `==` **có thể** cho kết quả "đúng" tình cờ (khi cả hai đều
  là literal, cùng nằm trong Pool — như `a == b` ở Example) khiến bug không lộ ra ngay, chỉ
  xuất hiện khi một trong hai chuỗi đến từ `new String(...)` hoặc được ghép/đọc từ nguồn khác
  (file, network, user input — luôn tạo object mới, không tự động vào Pool).
- **Nối chuỗi bằng `+` trong vòng lặp lớn** — xem Engineering Insight, nên dùng
  `StringBuilder` thay thế.
- **Gọi `intern()` bừa bãi** nghĩ rằng sẽ luôn tiết kiệm bộ nhớ — `intern()` cũng có chi phí
  (tra cứu trong Pool), và nếu chương trình intern quá nhiều chuỗi **khác nhau**, Pool có thể
  phình to không kiểm soát, phản tác dụng.

## Best Practices

- Luôn dùng `.equals()` (hoặc `Objects.equals()` nếu có thể `null`) để so sánh nội dung
  `String`, không bao giờ dùng `==` trừ khi **cố ý** muốn so sánh định danh object.
- Dùng `StringBuilder` khi nối chuỗi trong vòng lặp; dùng `+` bình thường cho biểu thức nối
  cố định, ngắn gọn trong một dòng — không cần lo lắng thái quá về hiệu năng ở trường hợp
  này (xem Myth vs Reality).
- Cẩn trọng khi nối `String` chứa dữ liệu nhạy cảm (mật khẩu) — vì bất biến, các bản `String`
  trung gian có thể tồn tại trong bộ nhớ lâu hơn dự kiến cho tới khi GC dọn (không có cách
  nào "xoá sạch" nội dung `String` ngay lập tức như có thể làm với `char[]`) — đây là lý do
  API xử lý mật khẩu (như JDBC, một số thư viện bảo mật) thường dùng `char[]` thay vì
  `String`.

## Production Notes

**Vấn đề:** dịch vụ xử lý dữ liệu dạng text (log processing, báo cáo, export CSV...) tiêu tốn
Heap và thời gian GC (Chapter 07) tăng bất thường khi lượng dữ liệu tăng.

- **Triệu chứng:** GC (đặc biệt Minor GC — Chapter 07) chạy thường xuyên hơn hẳn so với dự
  kiến dựa trên lượng dữ liệu thực tế đang xử lý.
- **Root cause phổ biến:** nối chuỗi lớn bằng `+` bên trong vòng lặp xử lý dữ liệu (xem
  Engineering Insight) — mỗi vòng lặp tạo một `String` trung gian mới, hàng nghìn/triệu object
  ngắn hạn đổ vào Young Generation, buộc Minor GC phải làm việc nhiều hơn hẳn mức cần thiết.
- **Debug:** dùng `jcmd <pid> GC.class_histogram` (Chapter 07) trong lúc xử lý — nếu thấy số
  lượng object `java.lang.String`/`char[]`/`byte[]` áp đảo bất thường, đây là dấu hiệu rõ
  ràng.
- **Solution:** thay `+` bằng `StringBuilder` ở đúng những vòng lặp nối chuỗi lớn/nhiều lần.
- **Prevention:** đưa việc review "nối chuỗi trong vòng lặp" vào code review checklist cho
  các đoạn code xử lý dữ liệu khối lượng lớn; hầu hết công cụ static analysis
  (SpotBugs/SonarQube — sẽ gặp ở giai đoạn sau) cũng có sẵn rule cảnh báo mẫu hình này.

## Debug Checklist

- [ ] So sánh hai chuỗi cho kết quả sai/không nhất quán? → kiểm tra có đang dùng `==` thay vì
      `.equals()` không.
- [ ] Nghi ngờ tốn nhiều bộ nhớ/GC vì xử lý chuỗi? → tìm vòng lặp có `+=` hoặc `+` nối
      `String`, cân nhắc đổi sang `StringBuilder`.
- [ ] Cần biết một `String` cụ thể có đang trong Pool hay không? → so sánh bằng `==` với một
      literal cùng nội dung, hoặc gọi `.intern()` rồi so `==`.

## Source Code Walkthrough

```
String.hashCode()
    ↓
Kiểm tra field nội bộ "hash" đã tính chưa (mặc định 0, nghĩa là chưa)
    ↓
Chưa tính → duyệt qua từng ký tự, áp dụng công thức
             s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]
             LƯU kết quả vào field "hash"
    ↓
Đã tính rồi (lần gọi sau) → TRẢ VỀ NGAY giá trị đã lưu, không tính lại
```

Công thức nhân với 31 (một số nguyên tố) là lựa chọn có chủ đích của các nhà thiết kế Java —
giảm khả năng đụng độ hash (hash collision) cho các chuỗi thường gặp, đồng thời `31 * i` có
thể được JIT Compiler (Chapter 06) tối ưu thành phép dịch bit `(i << 5) - i`, nhanh hơn phép
nhân thông thường trên nhiều kiến trúc CPU.

## Summary

`String` bất biến (immutable): mọi method "sửa" chuỗi thực chất luôn tạo object mới, không
đụng vào object cũ. Bất biến tồn tại vì ba lý do cụ thể — bảo mật (chống sửa dữ liệu sau khi
đã kiểm tra), thread-safety (đọc an toàn không cần khoá), và **String Pool** (nhiều biến dùng
chung một object literal một cách an toàn). Bất biến còn cho phép JVM cache sẵn `hashCode()`,
tối ưu hiệu năng khi dùng `String` làm key `HashMap`. Đánh đổi: mỗi phép "sửa" đều tạo rác mới
— với vòng lặp nối chuỗi nhiều lần, dùng `StringBuilder` (mutable, có chủ đích) thay vì `+`.

## Interview Questions

**Junior**

- Why is String immutable in Java?
- `==` và `.equals()` khi so sánh `String` khác nhau như thế nào?

**Mid**

- String Pool là gì? Giải thích sự khác nhau giữa `"a" == "a"` và `new String("a") == "a"`.
- Vì sao nối chuỗi bằng `+` trong vòng lặp lớn bị coi là không hiệu quả? Giải pháp là gì?

**Senior**

- Nếu `String` là mutable, hãy mô tả cụ thể một kịch bản tấn công bảo mật có thể xảy ra
  (liên hệ ClassLoader ở Chapter 03 hoặc HashMap ở Phase 2).
- Giải thích vì sao `hashCode()` của `String` có thể được cache an toàn, trong khi
  `hashCode()` mặc định của một object mutable bất kỳ (Chapter 09) thì không nên làm vậy.

## Exercises

- [ ] Chạy lại đúng ví dụ `StringPoolDemo` ở trên, xác nhận kết quả giống hệt trên máy bạn.
- [ ] Viết hai đoạn code nối 100.000 số nguyên thành một chuỗi — một bằng `+` trong vòng lặp,
      một bằng `StringBuilder`. Đo thời gian chạy bằng `System.nanoTime()` (nhớ bài học
      warmup ở [Chapter 06](06-jit-compiler.md)), so sánh chênh lệch.
- [ ] Dùng `javap -c` xem bytecode của cả hai đoạn code trên — tìm điểm khác biệt rõ nhất
      (gợi ý: đếm số lần `invokedynamic`/`StringConcatFactory` xuất hiện ở mỗi bên).

## Cheat Sheet

| So sánh | `==` | `.equals()` |
| --- | --- | --- |
| So sánh gì | Tham chiếu (địa chỉ object) | Nội dung ký tự |
| `"a" == "a"` | `true` (cùng object trong Pool) | `true` |
| `new String("a") == "a"` | `false` (object khác nhau) | `true` |

| Muốn | Dùng |
| --- | --- |
| So sánh nội dung String | `.equals()`, không bao giờ `==` |
| Nối chuỗi cố định, ít lần | `+` (đã được `javac` tối ưu — Chapter 01) |
| Nối chuỗi nhiều lần trong vòng lặp | `StringBuilder` |
| Ép một String vào String Pool | `.intern()` |

## References

- Java Language Specification (JLS) — Chapter 3.10.5: String Literals.
- `java.lang.String` — Java SE API Documentation (Oracle).
- JEP 254: Compact Strings.
- Effective Java (Joshua Bloch) — Item 17: "Minimize mutability".
