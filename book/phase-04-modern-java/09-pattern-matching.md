---
tags:
  - Java
  - PatternMatching
  - ModernJava
---

# Pattern Matching

> Phase: Phase 4 — Modern Java
> Chapter slug: `pattern-matching`

## Metadata

```yaml
Chapter: Pattern Matching
Phase: Phase 4 — Modern Java
Difficulty: ★★★★
Importance: ★★★★
Interview Frequency: 55%
Prerequisites:
  - Chapter 07 — Record
  - Chapter 08 — Sealed Class
Used Later:
  - Xử lý phản hồi API đa dạng kiểu dữ liệu (Phase 5+)
Estimated Reading: 18 phút
Estimated Practice: 15 phút
```

## Story

> Xem lại: [Chapter 08](08-sealed-class.md) — sealed hierarchy cho phép `switch` biết trước
> toàn bộ tập hợp subtype. Nhưng viết `case CreditCard c -> c.cardNumber()` mới chỉ "bắt được
> loại", còn phải tự gọi `c.cardNumber()` để lấy ra field bên trong — nếu `record` có nhiều
> field lồng nhau, việc "lấy ra" từng field vẫn cần nhiều dòng.

So sánh cách viết kiểm tra kiểu **trước và sau** pattern matching:

```java
// Truoc Java 16: kiem tra kieu roi TU CAST thu cong
if (obj instanceof String) {
    String s = (String) obj;
    return "Do dai: " + s.length();
}

// Tu Java 16: pattern matching, bien duoc gan TU DONG neu kiem tra dung
if (obj instanceof String s) {
    return "Do dai: " + s.length();
}
```

Cả hai cho cùng kết quả, nhưng pattern matching loại bỏ hoàn toàn bước cast thủ công — và biến
`s` còn có một hành vi bất ngờ hơn nữa: nó vẫn "sống" được **ngoài** khối `if` trong một số
trường hợp.

## Interview Question (Central)

> Pattern matching trong Java (`instanceof`, `switch`, record pattern) là gì? Nó khác kiểm tra
> kiểu + cast thủ công truyền thống như thế nào?

## Objectives

- [ ] Dùng thành thạo `instanceof` pattern matching, hiểu khái niệm "flow scoping"
- [ ] Dùng thành thạo `switch` pattern matching kết hợp record pattern để "destructure" trực
      tiếp field của record
- [ ] Dùng guarded pattern (`when`) để thêm điều kiện lọc trong từng nhánh `case`

## Prerequisites

- Chapter 07 — hiểu record, vì record pattern destructure trực tiếp field của record.
- Chapter 08 — hiểu sealed hierarchy, vì `switch` pattern matching thường kết hợp với nó để đạt
  tính exhaustive.

## Used Later

- **Xử lý phản hồi API đa dạng kiểu dữ liệu** (Phase 5+) — pattern matching kết hợp sealed
  interface là công cụ mạnh để xử lý các loại response/event khác nhau một cách an toàn kiểu.

## Problem

Kiểm tra kiểu truyền thống (`instanceof` + cast thủ công) tách rời hai bước lẽ ra luôn đi cùng
nhau: "kiểm tra kiểu có đúng không" và "nếu đúng, lấy ra giá trị đã cast". Việc phải viết cast
thủ công không chỉ dài dòng mà còn có thể gây lỗi nếu quên hoặc cast sai kiểu. Với dữ liệu lồng
nhau (record chứa record khác), việc "lấy ra" từng field lồng sâu bên trong còn phức tạp hơn
nữa nếu chỉ dùng kiểm tra kiểu + cast + gọi accessor thủ công từng bước.

## Concept

**Pattern matching** hợp nhất "kiểm tra kiểu/cấu trúc" và "trích xuất giá trị" thành một bước
duy nhất. Có 3 dạng chính: **`instanceof` pattern** (Java 16) — kiểm tra kiểu và gán biến cùng
lúc; **`switch` pattern matching** (Java 21) — mở rộng `switch` để so khớp theo kiểu, không chỉ
giá trị hằng số; **record pattern** (Java 21) — "destructure" trực tiếp các field của record
ngay trong pattern, không cần gọi accessor riêng.

## Why?

Hợp nhất kiểm tra kiểu và trích xuất giá trị vào một cú pháp duy nhất loại bỏ toàn bộ khả năng
lỗi "cast sai kiểu" (vì trình biên dịch tự đảm bảo biến pattern luôn đúng kiểu đã kiểm tra) và
giảm đáng kể boilerplate khi xử lý dữ liệu lồng nhau. Kết hợp với sealed hierarchy (Chapter 08),
`switch` pattern matching cho phép biểu đạt logic "xử lý khác nhau tuỳ loại dữ liệu" theo cách
**khai báo, an toàn kiểu, và được compiler xác nhận đầy đủ** — một phong cách lập trình gần với
"xử lý theo cấu trúc dữ liệu" (structural processing) phổ biến ở các ngôn ngữ hàm.

## How?

```java
// instanceof pattern (Java 16+)
if (obj instanceof String s && s.length() > 5) { ... }

// switch pattern matching + record pattern (Java 21+)
double area = switch (shape) {
    case Circle(double r) -> Math.PI * r * r;           // destructure truc tiep field r
    case Rectangle(double w, double h) -> w * h;         // destructure w, h cung luc
};

// guarded pattern (them dieu kien loc bang 'when')
String result = switch (shape) {
    case Circle c when c.radius() > 10 -> "Hinh tron lon";
    case Circle c -> "Hinh tron nho";
    default -> "Khac";
};
```

## Visualization

```
instanceof truyen thong:              instanceof pattern matching:

  if (obj instanceof String) {          if (obj instanceof String s) {
      String s = (String) obj;              // s TU DONG co san, DA duoc cast
      ...                                   ...
  }                                      }

record pattern (destructure):

  case Circle(double r) -> ...
         │        │
         │        └── lay TRUC TIEP field "radius" cua Circle, dat ten la r
         └── kiem tra dung la Circle

Flow scoping (bien pattern "song sot" ngoai khoi if):

  if (!(obj instanceof String s)) {
      return "khong phai String";  // nhanh nay return/throw -> "cat" luong
  }
  // s VAN CON HIEU LUC o day! vi nhanh khong-match da return truoc do
  return "do dai: " + s.length();
```

## Example

**`instanceof` pattern matching:**

```java
static String describeOld(Object obj) {
    if (obj instanceof String s && s.length() > 5) {
        return "Chuoi dai: " + s;
    }
    return "Khac";
}
```

**`switch` pattern matching + record pattern:**

```java
sealed interface Shape permits Circle, Rectangle {}
record Circle(double radius) implements Shape {}
record Rectangle(double width, double height) implements Shape {}

static double area(Shape shape) {
    return switch (shape) {
        case Circle(double r) -> Math.PI * r * r;
        case Rectangle(double w, double h) -> w * h;
    };
}

static String classify(Shape shape) {
    return switch (shape) {
        case Circle c when c.radius() > 10 -> "Hinh tron LON (r=" + c.radius() + ")";
        case Circle c -> "Hinh tron nho (r=" + c.radius() + ")";
        case Rectangle r when r.width() == r.height() -> "Hinh vuong (canh=" + r.width() + ")";
        case Rectangle r -> "Hinh chu nhat (" + r.width() + "x" + r.height() + ")";
    };
}
```

Kết quả thật (Temurin 26, `switch` pattern matching + record pattern ổn định từ Java 21):

```
Chuoi dai: Hello World
Khac
area(Circle(5)) = 78.53981633974483
area(Rectangle(4,4)) = 16.0
Hinh tron LON (r=15.0)
Hinh tron nho (r=3.0)
Hinh vuong (canh=5.0)
Hinh chu nhat (3.0x7.0)
```

`case Circle(double r) -> Math.PI * r * r` lấy trực tiếp field `radius` của `Circle` (đặt tên
`r`) mà **không cần** gọi `circle.radius()` — trình biên dịch tự "mở" record ngay trong pattern.
Guarded pattern (`when`) cho phép phân biệt "hình tròn lớn" và "hình tròn nhỏ" **cùng kiểu**
`Circle` nhưng theo điều kiện giá trị field — điều mà `switch` truyền thống (chỉ so khớp theo
kiểu hoặc hằng số) không làm được.

**Flow scoping — biến pattern "sống" ngoài khối `if`:**

```java
static String newStyle(Object obj) {
    if (!(obj instanceof String s)) {
        return "Khong phai String";
    }
    return "Do dai: " + s.length(); // s VAN hieu luc o day
}
```

Kết quả thật:

```
Do dai: 5
Do dai: 5
Khong phai String
```

`s` được dùng ở dòng `return "Do dai: " + s.length()` — nằm **ngoài** khối `if` ban đầu — vẫn
biên dịch và chạy đúng, vì trình biên dịch chứng minh được: nếu luồng thực thi tới được dòng đó,
điều kiện `!(obj instanceof String s)` chắc chắn đã là `false` (vì nhánh `true` đã `return`
trước đó), nên `s` chắc chắn đã được gán và hợp lệ.

## Deep Dive

**"Flow scoping" hoạt động dựa trên nguyên lý gì để trình biên dịch xác nhận biến pattern an
toàn ngoài khối kiểm tra ban đầu?** Trình biên dịch phân tích **luồng điều khiển** (control flow
analysis): một biến pattern được coi là "definitely matched" (chắc chắn đã khớp) tại một điểm
trong code nếu **mọi đường đi có thể tới điểm đó** đều đã đi qua nhánh mà pattern khớp thành
công. Với `if (!(obj instanceof String s)) { return ...; }`, chỉ có **một đường đi duy nhất** để
tới dòng code sau khối `if` — đó là khi điều kiện `!(...)` là `false`, tức `obj instanceof String
s` là `true`, tức `s` chắc chắn đã được gán. Đây là lý do flow scoping hoạt động với `return`/`throw`/`continue`/`break` (các lệnh "cắt luồng" một cách chắc chắn) nhưng không hoạt động nếu
nhánh không-khớp không cắt luồng rõ ràng — trình biên dịch sẽ báo lỗi biến "có thể chưa được gán"
nếu không chứng minh được tính an toàn này.

## Engineering Insight

**Vì sao "record pattern" + "sealed interface" (Chapter 08) được coi là bước tiến quan trọng
nhất của Java hướng tới phong cách lập trình hàm, so với các tính năng riêng lẻ trước đó (lambda,
Stream)?** Lambda và Stream (Chapter 01, 04) mang phong cách hàm vào việc **xử lý tập hợp dữ
liệu** (collection). Sealed + record + pattern matching mang phong cách hàm vào việc **định
nghĩa và xử lý cấu trúc dữ liệu** — cho phép biểu diễn "một giá trị chỉ có thể là một trong số
các dạng cố định, mỗi dạng có cấu trúc riêng" (sum type, Chapter 08) và xử lý nó bằng cách "so
khớp theo cấu trúc" (pattern matching) thay vì polymorphism truyền thống (mỗi subtype tự override
một phương thức xử lý bản thân nó). Đây là hai triết lý thiết kế khác nhau tồn tại song song
trong Java hiện đại: OOP cổ điển (polymorphism, đóng gói hành vi vào object) và structural pattern
matching (tách dữ liệu khỏi hành vi, xử lý tập trung theo cấu trúc) — mỗi cách phù hợp với ngữ
cảnh khác nhau.

## Historical Note

`instanceof` pattern matching ổn định ở Java 16 (JEP 394, 2021). `switch` pattern matching và
record pattern trải qua nhiều vòng preview (Java 17, 19, 20) trước khi cùng ổn định ở **Java 21**
(JEP 441, JEP 440, tháng 9/2023) — cùng đợt với Virtual Thread (Phase 3, Chapter 19), khiến Java
21 trở thành một bản LTS đặc biệt quan trọng với nhiều tính năng nền tảng hội tụ cùng lúc.

## Myth vs Reality

- **Myth:** "Pattern matching chỉ là cú pháp gọn hơn cho `instanceof` + cast, không có gì khác
  biệt về mặt an toàn."
  **Reality:** Ngoài việc gọn hơn, nó loại bỏ hoàn toàn khả năng cast sai kiểu (compiler tự đảm
  bảo), và flow scoping (xem Deep Dive) mang lại hành vi biến pattern thông minh hơn nhiều so
  với biến cast thủ công thông thường.

- **Myth:** "Record pattern chỉ hoạt động với record đơn giản, không destructure được dữ liệu
  lồng nhau."
  **Reality:** Record pattern hỗ trợ **lồng nhau** — có thể viết
  `case Order(Customer(String name), List<Item> items) -> ...` để destructure trực tiếp cả
  field bên trong một record lồng trong record khác, dù ví dụ trong chapter này chỉ dùng trường
  hợp đơn giản để minh hoạ nguyên lý cốt lõi.

## Common Mistakes

- **Vẫn viết cast thủ công sau khi đã dùng `instanceof` pattern** — mất hết lợi ích của pattern
  matching; biến pattern đã có sẵn, đúng kiểu, không cần cast lại.
- **Dùng quá nhiều điều kiện phức tạp trong guarded pattern (`when`)** — làm giảm khả năng đọc;
  nếu điều kiện quá dài, cân nhắc tách thành một phương thức boolean riêng.
- **Nhầm lẫn thứ tự `case` trong guarded pattern** — như Example, `case Circle c when
  c.radius() > 10` phải đứng **trước** `case Circle c` (không điều kiện) — nếu đảo ngược, nhánh
  không điều kiện sẽ luôn khớp trước, khiến nhánh có điều kiện không bao giờ được chạy tới.

## Best Practices

- Ưu tiên `instanceof` pattern matching thay vì kiểm tra kiểu + cast thủ công cho mọi code mới.
- Kết hợp `switch` pattern matching với sealed hierarchy (Chapter 08) để tận dụng tính
  exhaustive, để trình biên dịch tự bảo vệ khi có subtype mới.
- Sắp xếp `case` có `when` (điều kiện cụ thể hơn) trước `case` không điều kiện (tổng quát hơn)
  của cùng một kiểu.

## Debug Checklist

- [ ] Nhánh `case` có `when` không bao giờ được chạy tới? → kiểm tra thứ tự `case`, nhánh cụ thể
      hơn phải đứng trước nhánh tổng quát của cùng kiểu.
- [ ] Lỗi biên dịch "variable might not have been initialized" khi dùng biến pattern ngoài khối
      `if`? → xác nhận nhánh không-khớp có thực sự "cắt luồng" (return/throw/continue/break)
      một cách chắc chắn không.

## Summary

Pattern matching hợp nhất kiểm tra kiểu/cấu trúc và trích xuất giá trị thành một bước:
`instanceof` pattern (Java 16) loại bỏ cast thủ công, `switch` pattern matching + record pattern
(Java 21) cho phép "destructure" trực tiếp field của record ngay trong `case`. Đã chứng minh
bằng thực nghiệm: `case Circle(double r)` lấy trực tiếp field `radius` không cần gọi accessor;
guarded pattern (`when`) phân biệt theo điều kiện giá trị trong cùng một kiểu; flow scoping cho
phép biến pattern "sống" ngoài khối kiểm tra ban đầu khi trình biên dịch chứng minh được tính an
toàn qua phân tích luồng điều khiển. Kết hợp với sealed interface (Chapter 08), đây là bước tiến
quan trọng của Java hướng tới phong cách xử lý dữ liệu theo cấu trúc (structural processing).

## Interview Questions

**Mid**

- Pattern matching trong Java là gì? Nó khác `instanceof` + cast truyền thống như thế nào?
- Record pattern là gì? Cho ví dụ destructure một record trong `switch`.

**Senior**

- Giải thích "flow scoping" — vì sao một biến pattern có thể "sống" ngoài khối `if` ban đầu
  trong một số trường hợp nhưng không phải mọi trường hợp?
- Kết hợp sealed interface + record + pattern matching mang lại lợi ích gì so với thiết kế OOP
  truyền thống dùng polymorphism?

## Exercises

- [ ] Chạy lại `PatternMatchingDemo` và `FlowScopingDemo` ở trên (cần JDK 21+), xác nhận kết
      quả đúng như mô tả.
- [ ] Viết một sealed interface `Event` với các loại con `LoginEvent(String user)`,
      `LogoutEvent(String user, long durationSeconds)`, dùng record pattern trong `switch` để xử
      lý từng loại.
- [ ] Thử đảo ngược thứ tự hai nhánh `case Circle c when ...` và `case Circle c`, quan sát lỗi
      biên dịch "this case label is dominated by a preceding case label".

## Cheat Sheet

| Tính năng | Từ Java | Cú pháp |
| --- | --- | --- |
| `instanceof` pattern | 16 | `if (obj instanceof String s)` |
| `switch` pattern matching | 21 | `case String s -> ...` |
| Record pattern | 21 | `case Circle(double r) -> ...` |
| Guarded pattern | 21 | `case Circle c when c.radius() > 10 -> ...` |
| Flow scoping | 16 | Biến pattern hợp lệ ngoài khối kiểm tra nếu compiler chứng minh được luồng an toàn |

## References

- JEP 394: Pattern Matching for instanceof — https://openjdk.org/jeps/394
- JEP 441: Pattern Matching for switch — https://openjdk.org/jeps/441
- JEP 440: Record Patterns — https://openjdk.org/jeps/440
