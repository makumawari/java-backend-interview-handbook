---
tags:
  - Java
  - Module
  - JPMS
  - Foundation
---

# Module System

> Phase: Phase 1 — Java Foundation
> Chapter slug: `module-system`

## Metadata

```yaml
Chapter: Module System
Phase: Phase 1 — Java Foundation
Difficulty: ★★★
Importance: ★★★
Interview Frequency: 30%
Prerequisites:
  - Chapter 02 — JDK, JRE, JVM (jlink)
  - Chapter 03 — Class Loader (Platform Class Loader nạp module JDK)
  - Chapter 19 — Reflection (setAccessible bị giới hạn bởi module)
Used Later:
  - Docker, JVM Tuning (Phase 8) — jlink dựng runtime image tối giản cho container, dựa trực tiếp trên module
Estimated Reading: 20 phút
Estimated Practice: 15 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

Đây là chapter cuối cùng của Phase 1 — và nó khép lại đúng vòng tròn đã mở ra từ
[Chapter 02](02-jdk-jre-jvm.md): "Từ Java 11, JRE không còn được phân phối độc lập nữa" — lúc
đó chapter đã hẹn "chi tiết đầy đủ ở Chapter 20 (Module System)". Giờ là lúc trả lời trọn vẹn.

Và [Chapter 19](19-reflection.md) cũng để lại một câu hỏi bỏ ngỏ: từ Java 9,
`setAccessible(true)` — thứ tưởng như "vượt qua được mọi rào cản" — bắt đầu bị JVM **chặn lại**
trong một số trường hợp. Thử trực tiếp:

```
Exception in thread "main" java.lang.reflect.InaccessibleObjectException:
Unable to make field private java.lang.String com.example.secret.Secret.value
accessible: module modA does not "opens com.example.secret" to unnamed module
```

`private` bị Reflection phá vỡ dễ dàng ở Chapter 19. Giờ có một rào cản **mới**, đứng trên cả
`private`, mà ngay cả Reflection cũng không tự động vượt qua được nữa. Rào cản đó chính là chủ
đề chapter này: **Module System** (JPMS — Java Platform Module System, ra đời Java 9).

## Interview Question (Central)

> Module System giải quyết vấn đề gì mà package/JAR truyền thống (trước Java 9) không giải
> quyết được? Vì sao nó còn giới hạn được cả sức mạnh của Reflection?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Giải thích được "JAR hell" / "classpath hell" — vấn đề cụ thể mà Module System giải
      quyết
- [ ] Đọc hiểu và tự viết được `module-info.java` cơ bản (`exports`, `requires`)
- [ ] Phân biệt `exports` (mở API public thông thường) và `opens` (cho phép Reflection sâu,
      kể cả `private`) — hai mức truy cập khác nhau
- [ ] Tự tay tái hiện đúng lỗi ở Story, rồi tự sửa bằng `opens`
- [ ] Biết `jlink` dùng thông tin module để dựng runtime image tối giản

## Prerequisites

- Chapter 02 — đã biết `jlink` được nhắc tới như lời giải cho "JRE không còn độc lập".
- Chapter 03 — đã biết Platform Class Loader nạp "module JDK khác `java.base`".
- Chapter 19 — đã thấy `setAccessible()` mạnh tới mức bỏ qua `private`/`final`.

## Used Later

- **Docker, JVM Tuning** (Phase 8) — `jlink` dùng thông tin `module-info` để dựng một runtime
  image chỉ chứa đúng những module ứng dụng cần, giảm đáng kể kích thước container image so
  với đóng gói cả một JDK/JRE đầy đủ.

## Problem

Trước Java 9, đơn vị đóng gói code Java là **JAR** — về bản chất chỉ là một file zip chứa
`.class`, không có khai báo tường minh "JAR này phụ thuộc JAR nào", "JAR này công khai package
nào cho bên ngoài dùng". Hậu quả trong thực tế, gọi chung là **"JAR hell"**:

- Hai JAR khác nhau có thể chứa **cùng một package** với nội dung khác nhau, đưa cả hai vào
  classpath — không có cách nào biết chắc lớp nào thực sự được nạp (liên hệ trực tiếp tới bài
  toán "hai bản của cùng một class" đã gặp ở [Chapter 03](03-class-loader.md)).
- Không có gì ngăn code của bạn "thò tay" vào các package **nội bộ** của JDK không nhằm cho
  ứng dụng dùng (ví dụ `sun.misc.*`) — nhiều thư viện phổ biến từng làm vậy, khiến JDK không
  thể tự do thay đổi chi tiết nội bộ vì sợ "phá" hàng loạt ứng dụng đang dựa vào những API
  không chính thức đó.
- Không thể xây một runtime tối giản — package JRE cũ (trước Chapter 02 đã nhắc) buộc phải là
  "trọn gói", dù ứng dụng chỉ dùng một phần nhỏ.

## Concept

**Module** là một đơn vị đóng gói bậc cao hơn JAR, có khai báo tường minh trong một file
`module-info.java` ở gốc: module này **tên gì**, **phụ thuộc** (requires) module nào,
**công khai** (exports) package nào cho module khác dùng.

```java
module com.example.greet {
    exports com.example.greet;   // package nào công khai ra ngoài
}
```

Package **không** được `exports` là hoàn toàn riêng tư đối với module đó — không module nào
khác gọi được, kể cả qua API `public` thông thường (khác hẳn trước Java 9, khi mọi `public`
class trên classpath đều gọi được từ bất kỳ đâu).

## Why?

Nếu không có khai báo tường minh `exports`/`requires`: JVM không có cách nào biết ranh giới
"API công khai có chủ đích" và "chi tiết nội bộ tình cờ là `public`" — mọi thứ `public` đều bị
coi là API, khiến nhà phát triển JDK (hay bất kỳ thư viện nào) không dám tự do thay đổi bất kỳ
class `public` nào, dù nó chưa từng có ý định làm API ổn định. Module System cho phép nói rõ
"đây là API thật sự, đây chỉ là chi tiết cài đặt nội bộ" — một hình thức Encapsulation
(Chapter 08) áp dụng ở quy mô **giữa các thư viện/module**, không chỉ giữa các class trong
cùng một package.

## How?

Module tối giản, biên dịch và chạy được (đã kiểm chứng thật):

```java
// module-info.java
module com.example.greet {
    exports com.example.greet;
}
```

```java
// com/example/greet/Greeter.java
package com.example.greet;
public class Greeter {
    public static void main(String[] args) {
        System.out.println("Xin chao tu module: " + Greeter.class.getModule().getName());
    }
}
```

Biên dịch và chạy bằng `--module-path`/`--module` thay vì `-cp`/tên class trực tiếp:

```
javac -d out module-info.java com/example/greet/Greeter.java
java --module-path out --module com.example.greet/com.example.greet.Greeter
```

Kết quả thật:

```
Xin chao tu module: com.example.greet
```

`Class.getModule().getName()` xác nhận: class này thực sự đang chạy **bên trong** một module
có tên `com.example.greet`, không phải "unnamed module" (classpath truyền thống, xem Deep
Dive).

## Visualization

```
Trước Java 9 (classpath truyền thống):
  Mọi .class trên classpath ─────▶ "UNNAMED MODULE" (một khối lớn,
                                     không phân biệt ranh giới,
                                     mọi public class đều gọi được lẫn nhau)

Từ Java 9 (module path):
  module com.example.greet         module modA
  ┌─────────────────────┐         ┌──────────────────────┐
  │ exports              │         │ (không exports gì)    │
  │  com.example.greet    │◀───────┤ requires               │
  │                       │ dùng   │  com.example.greet     │
  │ package NỘI BỘ        │        │                        │
  │ (không exports)       │───✗───▶│ KHÔNG gọi được, dù     │
  └─────────────────────┘        │ package đó public       │
                                   └──────────────────────┘
```

## Example

Tái hiện đúng và sửa lỗi ở Story — phân biệt rõ `exports` (API public thông thường) và
`opens` (cho phép Reflection sâu):

```java
// module modA — CHỈ exports, KHÔNG opens
module modA {
    exports com.example.secret;
}
```

```java
package com.example.secret;
public class Secret {
    private String value = "bi mat";
}
```

Từ module/code khác cố dùng Reflection đọc field `private` này:

```java
import java.lang.reflect.Field;
import com.example.secret.Secret;

public class Hack {
    public static void main(String[] args) throws Exception {
        Secret s = new Secret();
        Field f = Secret.class.getDeclaredField("value");
        f.setAccessible(true); // XEM Ở ĐÂY
        System.out.println("Doc duoc: " + f.get(s));
    }
}
```

Kết quả thật khi `modA` chỉ `exports` (chưa `opens`):

```
Exception in thread "main" java.lang.reflect.InaccessibleObjectException:
Unable to make field private java.lang.String com.example.secret.Secret.value
accessible: module modA does not "opens com.example.secret" to unnamed module
```

Đúng lỗi đã thấy ở Story! Sửa bằng cách thêm `opens`:

```java
module modA {
    exports com.example.secret; // API public thông thường vẫn dùng được
    opens com.example.secret;   // CHO PHÉP Reflection sâu, kể cả private
}
```

Chạy lại, kết quả thật:

```
Doc duoc: bi mat
```

## Deep Dive

**`exports` và `opens` là hai mức truy cập khác nhau, không phải một** — điểm dễ nhầm nhất
của chapter này:

- **`exports`** — cho phép code ngoài module gọi **API `public` thông thường** (gọi method,
  tạo instance qua constructor `public`...) — đúng cách lập trình bình thường vẫn làm từ trước
  tới nay.
- **`opens`** — cho phép **Reflection sâu** (`setAccessible(true)`, kể cả vào `private`) —
  một mức truy cập mạnh hơn hẳn, cố ý tách riêng khỏi `exports`.

Một module có thể `exports` mà **không** `opens` (như `modA` lúc đầu ở Example) — API public
dùng bình thường được, nhưng Reflection sâu bị chặn. Đây chính là lý do các framework
dựa nặng vào Reflection (Hibernate đọc field `private` của Entity — [Chapter 19](19-reflection.md))
cần module đích phải `opens` (không chỉ `exports`) mới hoạt động đúng — một chi tiết cấu hình
rất thực tế khi làm việc với JPA/Hibernate trên Java 9+ (Phase 6).

## Engineering Insight

**Vì sao thiết kế tách `exports`/`opens` thành hai mức, thay vì một cờ "công khai/không công
khai" duy nhất?** Vì hai nhu cầu này có **mức độ rủi ro khác hẳn nhau**. Một API `public`
thông thường (qua `exports`) được thiết kế có chủ đích, có hợp đồng rõ ràng (chữ ký method,
kiểu tham số) — an toàn để module khác phụ thuộc lâu dài. Reflection sâu (qua `opens`) cho
phép truy cập **vượt qua mọi ràng buộc thiết kế** (Chapter 19: bỏ qua cả `private`/`final`) —
một framework dùng `opens` để đọc/ghi field nội bộ của bạn có thể "biết" những chi tiết cài đặt
mà bạn chưa bao giờ có ý định công khai, và những chi tiết đó có thể đổi bất cứ lúc nào mà
không được coi là "phá vỡ API" (vì nó chưa bao giờ chính thức là API). Tách hai mức này cho
phép module tác giả nói rõ: "tôi cho phép dùng bình thường (`exports`), nhưng KHÔNG cam kết
ổn định nếu bạn dùng Reflection đào sâu (`opens`, nếu có, là một nhượng bộ có ý thức, thường
chỉ mở cho framework cụ thể qua `opens ... to ...`)".

## Historical Note

```
Trước Java 9
    ↓
Classpath truyền thống — mọi .class thuộc "unnamed module" ngầm định,
không có exports/opens tường minh, setAccessible() gần như luôn thành công
    ↓
Java 9 (2017) — Project Jigsaw / JPMS chính thức ra mắt sau NHIỀU NĂM phát
triển (bắt đầu thảo luận từ khoảng 2008) — một trong những thay đổi kiến
trúc lớn nhất trong lịch sử JDK, module hoá cả chính JDK thành hàng chục
module (java.base, java.sql, java.desktop...)
    ↓
Từ Java 9 đến nay — phần lớn ứng dụng/thư viện VẪN chạy trên "classpath
truyền thống" (unnamed module) vì tương thích ngược — module system chủ
yếu được tận dụng RÕ RÀNG nhất ở việc module hoá chính JDK và qua jlink,
chưa trở thành chuẩn mực bắt buộc cho mọi ứng dụng Java viết mới
```

## Myth vs Reality

- **Myth:** "Từ Java 9, mọi dự án Java bắt buộc phải viết `module-info.java`."
  **Reality:** Hoàn toàn không bắt buộc — code không có `module-info.java` chạy trên
  "unnamed module" (giống hệt classpath truyền thống trước Java 9), vẫn hoạt động bình thường.
  Module System là **tuỳ chọn**, không phải yêu cầu bắt buộc cho ứng dụng thông thường.

- **Myth:** "`exports` và `opens` là một, chỉ cần `exports` là đủ cho mọi framework hoạt
  động."
  **Reality:** Xem Example/Deep Dive — thiếu `opens`, Reflection sâu (mà nhiều framework như
  Hibernate cần) sẽ thất bại với `InaccessibleObjectException`, dù package đã `exports` đầy
  đủ.

## Common Mistakes

- **Chỉ `exports` mà quên `opens`** khi module cần được một framework dựa vào Reflection sâu
  sử dụng (JPA Entity, Jackson serialize field private...) — đúng lỗi tái hiện ở Story/
  Example.
- **Viết `module-info.java` cho một dự án không thực sự cần** — với ứng dụng nhỏ, phụ thuộc
  nhiều thư viện chưa module hoá đầy đủ, việc ép dùng module system có thể gây phức tạp không
  tương xứng lợi ích, đặc biệt nếu team chưa quen thuộc với JPMS.
- **Nhầm lẫn module (JPMS) với package** — một module có thể chứa nhiều package, package
  không tự động "thuộc về" một module trừ khi nó nằm trong cấu trúc thư mục của module đó.

## Best Practices

- Nếu dùng framework dựa nặng vào Reflection sâu (Hibernate, một số thư viện JSON), nhớ
  `opens` (không chỉ `exports`) đúng package chứa Entity/DTO liên quan — hoặc dùng
  `opens ... to <module cụ thể>` để giới hạn phạm vi nhượng bộ này chỉ cho đúng framework cần,
  không mở toang cho mọi module khác.
- Không nhất thiết phải áp dụng module system cho mọi dự án — cân nhắc dựa trên quy mô dự án
  và mức độ sẵn sàng của các thư viện phụ thuộc.
- Khi gặp lỗi liên quan module lúc nâng cấp JDK, đọc kỹ thông báo lỗi — nó luôn chỉ rõ chính
  xác cần thêm `exports`/`opens`/`--add-opens` nào.

## Production Notes

**Vấn đề:** ứng dụng chạy ổn định trên Java 8, nâng cấp lên Java 17 (hoặc bất kỳ Java 9+ nào)
thì một thư viện dựa vào Reflection sâu (thường là thư viện cũ, dựa vào chi tiết nội bộ JDK)
bắt đầu ném `InaccessibleObjectException` hoặc `IllegalAccessError`.

- **Triệu chứng:** lỗi chỉ xuất hiện sau khi đổi version JDK, không liên quan tới thay đổi code
  nghiệp vụ nào — đúng tình huống đã nêu ở [Chapter 19 — Production Notes](19-reflection.md).
- **Root cause:** thư viện cố truy cập package JDK nội bộ (chưa `exports`/`opens` cho unnamed
  module) hoặc chính module ứng dụng của bạn thiếu `opens` cho package mà thư viện cần
  Reflection sâu vào.
- **Debug:** đọc chính xác tên package/module trong thông báo lỗi.
- **Solution ngắn hạn:** thêm cờ dòng lệnh `--add-opens <module>/<package>=ALL-UNNAMED` khi
  chạy JVM (giải pháp tạm, không sửa gốc rễ).
- **Solution dài hạn:** nâng cấp thư viện lên phiên bản đã hỗ trợ đúng module system (không
  còn phụ thuộc Reflection sâu vào chi tiết nội bộ JDK), hoặc thêm `opens` tường minh trong
  `module-info.java` của chính ứng dụng nếu vấn đề nằm ở module của bạn.
- **Prevention:** khi nâng cấp version JDK lớn, luôn kiểm tra release note phần liên quan
  module system, và test kỹ các thư viện dựa nặng vào Reflection trước khi triển khai
  production.

## Debug Checklist

- [ ] `InaccessibleObjectException`? → thiếu `opens` cho đúng package/module (xem Example).
- [ ] `IllegalAccessError: ... does not export ...`? → thiếu `exports` hoàn toàn (mức nghiêm
      trọng hơn thiếu `opens` — package thậm chí chưa công khai API public thông thường).
- [ ] Lỗi module chỉ xuất hiện sau khi nâng cấp JDK? → xem đúng quy trình ở Production Notes.

## Source Code Walkthrough

`module-info.java` không biên dịch thành bytecode thông thường như một class — nó biên dịch
thành một file đặc biệt tên `module-info.class`, chứa metadata mô tả module (tên, `requires`,
`exports`, `opens`...) mà JVM đọc **trước cả khi** bắt đầu nạp bất kỳ class nào khác của module
đó — một bước bổ sung vào đúng giai đoạn Loading đã học ở
[Chapter 03](03-class-loader.md), giờ có thêm bước kiểm tra ràng buộc module trước khi cho
phép truy cập xuyên module.

## Summary

Module System (JPMS, Java 9) giải quyết "JAR hell": khai báo tường minh trong
`module-info.java` module nào phụ thuộc (`requires`) module nào, công khai (`exports`) package
nào cho bên ngoài — package không `exports` hoàn toàn riêng tư, dù `public`. `exports` (API
thông thường) và `opens` (Reflection sâu, kể cả `private`) là **hai mức tách biệt** — thiếu
`opens` khiến `setAccessible()` (Chapter 19) thất bại với `InaccessibleObjectException`, dù
package đã `exports` đầy đủ. Module system hoàn toàn tuỳ chọn (không bắt buộc), code không có
`module-info.java` vẫn chạy bình thường trên "unnamed module". `jlink` tận dụng thông tin
module để dựng runtime image tối giản — lời giải cho câu hỏi bỏ ngỏ từ Chapter 02, khép lại
trọn vẹn Phase 1 Foundation.

## Interview Questions

**Junior**

- Module System giải quyết vấn đề gì?
- Một dự án Java có bắt buộc phải có `module-info.java` không?

**Mid**

- `exports` và `opens` trong `module-info.java` khác nhau như thế nào?
- "JAR hell" là gì? Cho một ví dụ cụ thể.

**Senior**

- Giải thích chính xác vì sao một module `exports` một package nhưng chưa `opens` vẫn có thể
  khiến một thư viện dựa vào Reflection (như Hibernate) thất bại — trong khi code gọi API
  `public` thông thường của package đó vẫn hoạt động bình thường.
- Nếu bạn phụ trách nâng cấp một hệ thống lớn từ Java 8 lên Java 17, bạn sẽ lập kế hoạch kiểm
  tra và xử lý rủi ro liên quan tới Module System như thế nào?

## Exercises

- [ ] Chạy lại đúng ví dụ module `com.example.greet` ở phần How?, xác nhận
      `getModule().getName()` trả về đúng tên module.
- [ ] Tái hiện đúng lỗi `InaccessibleObjectException` ở Example (chỉ `exports`, không
      `opens`), sau đó tự sửa bằng cách thêm `opens`, xác nhận Reflection hoạt động trở lại.
- [ ] Thử đổi `opens com.example.secret;` thành `opens com.example.secret to <tên module của
      Hack, nếu Hack cũng là module>;` — tìm hiểu vì sao giới hạn `opens ... to ...` cụ thể
      lại an toàn hơn `opens` mở toang cho mọi module.

## Cheat Sheet

| Khai báo | Ý nghĩa |
| --- | --- |
| `module <tên> { }` | Định nghĩa module |
| `requires <module khác>` | Module này phụ thuộc module khác |
| `exports <package>` | Công khai package cho API public thông thường |
| `opens <package>` | Cho phép Reflection sâu (kể cả `private`) vào package |
| `opens <package> to <module>` | Giới hạn nhượng bộ Reflection chỉ cho module cụ thể |

| Lỗi | Nguyên nhân |
| --- | --- |
| `IllegalAccessError: does not export` | Thiếu `exports` — API public thông thường cũng bị chặn |
| `InaccessibleObjectException` | Có `exports` nhưng thiếu `opens` — chỉ Reflection sâu bị chặn |

## References

- Java Language Specification (JLS) — Chapter 7.7: Module Declarations.
- JEP 261: Module System.
- JEP 220: Modular Run-Time Images (liên quan `jlink`, Chapter 02).
- Oracle: "The Java Tutorials — Trail: Modules".
