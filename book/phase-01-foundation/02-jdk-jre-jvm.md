---
tags:
  - Java
  - JVM
  - JDK
  - Foundation
---

# JDK, JRE, JVM

> Phase: Phase 1 — Java Foundation
> Chapter slug: `jdk-jre-jvm`

## Metadata

```yaml
Chapter: JDK, JRE, JVM
Phase: Phase 1 — Java Foundation
Difficulty: ★★
Importance: ★★★★
Interview Frequency: 85%
Prerequisites:
  - Chapter 01 — Một chương trình Java chạy như thế nào?
Used Later:
  - Class Loader (Chapter 03)
  - Toàn bộ handbook (JAVA_HOME, version JDK là nền cho mọi ví dụ code)
  - Docker, JVM Tuning (Phase 8 — Production)
Estimated Reading: 15 phút
Estimated Practice: 20 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Chapter 01](01-mot-chuong-trinh-java-chay-nhu-the-nao.md) — bạn đã chạy
> `javac Main.java` rồi `java Main`.

Nhưng `javac` và `java` là hai chương trình nằm ở đâu trên máy bạn? Khi bạn "cài Java", bạn
đã cài cái gì? Thử chạy lệnh sau trên chính máy bạn đang học:

```
java -version
```

Rồi thử:

```
javac -version
```

Nếu lệnh thứ hai báo `command not found`, bạn vừa chạm đúng vào chủ đề của chapter này: máy
bạn có thể **chạy** được Java nhưng không **biên dịch** được Java — vì hai lệnh đó không đến
từ cùng một thứ. Rất nhiều người mới nhầm lẫn "cài Java" là cài một thứ duy nhất. Thực ra có
ba khái niệm lồng vào nhau, và phân biệt được chúng sẽ cứu bạn khỏi hàng giờ debug lỗi
`javac: command not found` trong CI/CD sau này.

## Interview Question (Central)

> JDK, JRE, JVM khác nhau như thế nào? Cài JDK có phải là cài JVM không?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Vẽ được sơ đồ lồng nhau JDK ⊃ JRE ⊃ JVM và biết chính xác mỗi lớp thêm gì
- [ ] Biết `javac`, `java`, `jar`, `javap`, `jshell` nằm trong thành phần nào
- [ ] Giải thích được vì sao từ Java 11, Oracle/OpenJDK không còn phát hành JRE độc lập nữa
- [ ] Biết JVM là một **đặc tả (specification)**, HotSpot chỉ là một cách hiện thực nó
- [ ] Tránh được lỗi `JAVA_HOME` trỏ nhầm, gây `javac: command not found` trong CI

## Prerequisites

- Chapter 01 — đã hiểu khái niệm bytecode, biết `javac` biên dịch, `java` chạy.

## Used Later

- **Class Loader** (Chapter 03) — JVM sẽ được bóc tách tiếp: JVM nạp class bằng cách nào.
- **Toàn bộ handbook** — mọi ví dụ code từ đây về sau đều giả định bạn hiểu JDK là gì để cài
  đặt môi trường đúng.
- **Docker, JVM Tuning** (Phase 8) — production thường chỉ cần JRE-tương-đương (không cần
  `javac`) để giảm kích thước image, một quyết định chỉ hợp lý nếu bạn hiểu rõ JDK/JRE khác
  nhau ở đây.

## Problem

Bạn cần hai việc rất khác nhau:

1. **Lúc code**: cần biên dịch (`javac`), đóng gói (`jar`), debug (`jdb`), xem bytecode
   (`javap`) — một bộ công cụ cho *nhà phát triển*.
2. **Lúc chạy chương trình đã build xong** (ví dụ: máy khách chỉ cần chạy app, không sửa
   code): chỉ cần *thực thi* được bytecode — không cần biên dịch, không cần công cụ dev.

Nếu gộp chung một gói duy nhất, máy chỉ cần chạy chương trình sẽ phải tải về cả một bộ công
cụ phát triển không dùng tới — nặng và thừa. Đây là lý do Java tách thành nhiều lớp.

## Concept

Ba khái niệm lồng vào nhau như búp bê Nga, từ trong ra ngoài:

- **JVM (Java Virtual Machine)** — lớp trong cùng. Chỉ làm một việc: thực thi bytecode (như
  Chapter 01 đã mô tả). Tự bản thân JVM không biết `System.out.println` là gì — nó cần thư
  viện.
- **JRE (Java Runtime Environment)** = JVM + **thư viện chuẩn** (`java.lang`, `java.util`,
  `java.io`...). Đủ để **chạy** một chương trình Java đã biên dịch sẵn.
- **JDK (Java Development Kit)** = JRE + **công cụ phát triển** (`javac`, `jar`, `javadoc`,
  `jdb`, `javap`, `jshell`...). Đủ để **viết và build** chương trình Java.

## Why?

Nếu không tách lớp:

- Không thể phân phối một bản "chỉ để chạy" nhẹ hơn cho end-user/server production — luôn
  phải kéo theo toàn bộ công cụ dev không dùng tới.
- Không thể có nhiều cách **hiện thực JVM khác nhau** cho cùng một đặc tả (xem Engineering
  Insight) — nếu JDK/JVM là một khối cứng, không ai viết được HotSpot, GraalVM, hay
  Eclipse OpenJ9 để cạnh tranh nhau về hiệu năng trên cùng một chuẩn bytecode.

## How?

```
JDK (Java Development Kit)
 └── JRE (Java Runtime Environment)
      └── JVM (Java Virtual Machine)
      └── Thư viện chuẩn: java.lang, java.util, java.io, ...
 └── Công cụ dev: javac, jar, javadoc, jdb, javap, jshell, ...
```

Khi bạn gõ `javac Main.java` (Chapter 01), bạn dùng công cụ nằm ở lớp **JDK**. Khi bạn gõ
`java Main`, bạn dùng công cụ nằm ở lớp **JRE** (cụ thể là launcher gọi vào JVM). Một máy chỉ
cài JRE (không có JDK) vẫn chạy `java Main` được — miễn là đã có sẵn `Main.class` — nhưng
không compile được gì cả.

## Visualization

```
┌─────────────────────────── JDK ────────────────────────────┐
│  Công cụ dev: javac · jar · javadoc · jdb · javap · jshell  │
│                                                                │
│  ┌───────────────────────── JRE ─────────────────────────┐  │
│  │  Thư viện chuẩn: java.lang, java.util, java.io, ...    │  │
│  │                                                          │  │
│  │  ┌────────────────── JVM ──────────────────┐           │  │
│  │  │  Class Loader · Runtime Data Areas ·      │           │  │
│  │  │  Execution Engine · Garbage Collector     │           │  │
│  │  │  [chi tiết: Chapter 03-07]                │           │  │
│  │  └────────────────────────────────────────────┘         │  │
│  └──────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

## Example

Xác nhận trên chính máy bạn: JDK 17 thật sự chứa nhiều công cụ hơn chỉ `javac`/`java`
(output thật, chạy trên macOS với JDK 17):

```
$ ls $JAVA_HOME/bin
jar jarsigner java javac javadoc javap jcmd jconsole jdb jdeprscan jdeps
jfr jhsdb jimage jinfo jlink jmap jmod jpackage jps jrunscript jshell
jstack jstat jstatd keytool rmiregistry serialver
```

Hơn 25 công cụ — `javac`/`java` chỉ là hai trong số đó. Thử luôn `jshell` (REPL của Java,
có từ Java 9) để chạy thử code Java không cần file, không cần `javac`:

```
$ jshell
jshell> 1 + 1
$1 ==> 2
jshell> System.out.println("Hello từ JDK")
Hello từ JDK
jshell> /exit
```

`jshell` chỉ tồn tại trong JDK — một máy chỉ cài JRE sẽ không có lệnh này.

## Deep Dive

**JVM là một đặc tả, không phải một sản phẩm.** *Java Virtual Machine Specification* (JLS đi
kèm) chỉ mô tả **hành vi** JVM phải có — không quy định JVM phải viết bằng ngôn ngữ gì, tối
ưu ra sao. Bất kỳ ai cũng có thể viết một chương trình tuân theo đặc tả đó và gọi nó là một
JVM. Trên thị trường tồn tại nhiều JVM khác nhau, cùng chạy được một file `.class`:

- **HotSpot** — JVM mặc định của OpenJDK/Oracle JDK (JVM bạn đang dùng ở các ví dụ trong
  handbook này).
- **GraalVM** — có chế độ chạy như JVM thông thường (với JIT Compiler viết bằng Java, gọi là
  Graal Compiler) và chế độ biên dịch AOT ra Native Image (đã nhắc ở Chapter 01).
- **Eclipse OpenJ9** — JVM do IBM phát triển, tối ưu cho footprint bộ nhớ nhỏ, hay dùng trong
  container.

Đây chính xác là lý do "Write Once, Run Anywhere" hoạt động được ở tầng công nghiệp: bytecode
là hợp đồng chung, còn JVM nào thực thi hợp đồng đó nhanh nhất là cuộc cạnh tranh riêng của
từng nhà cung cấp.

## Engineering Insight

Vì sao từ Java 11, Oracle/OpenJDK **không còn phát hành JRE độc lập** nữa (chỉ còn JDK)?

Trước Java 9, JRE tồn tại như một sản phẩm dành cho *người dùng cuối* — cài để chạy applet
trên trình duyệt, hoặc chạy ứng dụng desktop Java mà không cần biết lập trình. Nhưng:

1. Applet trong trình duyệt đã bị khai tử (các trình duyệt lớn ngừng hỗ trợ Java Plugin
   khoảng 2015-2017 vì lý do bảo mật).
2. Project Jigsaw (Java 9, module system — sẽ gặp lại ở Chapter 20 Module System) mở ra công
   cụ `jlink`, cho phép **tự đóng gói một runtime tối giản** chỉ chứa đúng module ứng dụng
   cần — linh hoạt hơn hẳn một JRE "một size cho tất cả".

Nói cách khác: nhu cầu "chỉ cần chạy, không cần dev tools" vẫn còn (rất cần trong production/
container), nhưng cách đáp ứng đã đổi từ "tải sẵn một gói JRE cố định" sang "tự build runtime
tối giản bằng `jlink` từ chính JDK". Đây là điều Phase 8 (JVM Tuning, Docker) sẽ khai thác khi
tối ưu kích thước container image.

## Historical Note

```
Trước Java 9
    ↓
JRE là sản phẩm độc lập, cài riêng cho end-user (chạy applet, ứng dụng desktop)
    ↓
Java 9 (2017) — Project Jigsaw, module hoá JDK, sinh ra jlink
    ↓
Java 11 (2018) — Oracle/OpenJDK ngừng phát hành JRE độc lập, chỉ còn JDK
    ↓
Hiện tại — muốn một "JRE tối giản", tự build bằng jlink từ JDK
```

## Myth vs Reality

- **Myth:** "Cài JRE là đủ để lập trình Java."
  **Reality:** JRE không có `javac` — không biên dịch được. Cần JDK để phát triển.

- **Myth:** "JVM là một phần mềm cụ thể, cứ nói 'JVM' là nói đến HotSpot."
  **Reality:** JVM là một đặc tả (specification). HotSpot chỉ là hiện thực phổ biến nhất,
  không phải hiện thực duy nhất (xem Deep Dive).

- **Myth:** "Từ Java 11 không còn JRE nữa, khái niệm JRE đã biến mất."
  **Reality:** Khái niệm JRE (JVM + thư viện chuẩn, đủ để *chạy* chương trình) vẫn còn —
  chỉ là không còn được **phân phối như một sản phẩm cài đặt riêng** nữa.

## Common Mistakes

- **`JAVA_HOME` trỏ vào thư mục JRE thay vì JDK** (phổ biến trên máy cũ vẫn còn cài JRE
  riêng, hoặc cấu hình CI sai) → build script gọi `javac` thất bại với
  `command not found`, trong khi `java` vẫn chạy bình thường — dễ gây bối rối vì "Java vẫn
  chạy được mà".
- **Cài nhiều JDK song song rồi quên đang dùng bản nào.** Rất phổ biến: máy có sẵn
  `jdk-17.jdk` và một bản JDK khác (ví dụ Temurin 26) cùng lúc; `java -version` có thể trả
  về bản khác với bản bạn tưởng đang active.
- **Copy `JAVA_HOME` từ máy khác** mà không kiểm tra máy mình có cài đúng bản đó không —
  gây lỗi khó hiểu, đặc biệt trong shell script CI/CD.

## Best Practices

- Luôn xác nhận bằng `java -version` VÀ `javac -version` — không chỉ một trong hai — trước
  khi debug bất kỳ lỗi build nào.
- Khi làm nhiều dự án cần JDK version khác nhau, dùng công cụ quản lý version
  (`sdkman`, `jenv`, hoặc `asdf`) thay vì đổi `JAVA_HOME` thủ công mỗi lần.
- Trong Dockerfile/CI, pin chính xác image JDK (ví dụ `eclipse-temurin:21-jdk`), không dùng
  tag `latest` — tránh trường hợp build hôm nay và build tuần sau dùng khác version JDK mà
  không ai biết.

## Production Notes

**Vấn đề:** pipeline CI/CD báo lỗi `javac: command not found` dù trước đó vẫn chạy tốt.

- **Triệu chứng:** bước `mvn compile`/`gradle build` thất bại ngay từ đầu, log không có gì
  liên quan tới code của bạn.
- **Root cause:** runner CI được thay đổi image, image mới chỉ cài JRE (hoặc chỉ cài
  runtime tối giản build bằng `jlink`) — đủ để *chạy* nhưng không đủ để *biên dịch*.
- **Debug:** chạy `which javac` và `javac -version` ngay trong chính môi trường CI (thêm một
  step debug tạm thời) để xác nhận `javac` có tồn tại không, PATH có trỏ đúng không.
- **Solution:** đổi lại image CI sang bản có đầy đủ JDK (không phải JRE-only/runtime tối
  giản).
- **Prevention:** phân biệt rõ trong hạ tầng: môi trường **build** (CI) luôn cần JDK đầy đủ;
  môi trường **runtime** (production server chạy app đã build) có thể dùng JRE-tương-đương
  tối giản hơn — nhưng không được lẫn lộn hai loại image này.

## Debug Checklist

- [ ] `java -version` chạy được nhưng `javac -version` báo lỗi? → môi trường chỉ có JRE,
      thiếu JDK.
- [ ] `java -version` trả về version khác bạn mong đợi? → kiểm tra `JAVA_HOME` và `PATH`,
      có thể đang trỏ nhầm JDK trong số nhiều bản cài song song.
- [ ] Build chạy tốt local nhưng fail trên CI? → so sánh `java -version`/`javac -version`
      giữa hai môi trường, đừng giả định chúng giống nhau.

## Source Code Walkthrough

Không cần đọc source ở mức chapter này — JDK/JRE/JVM là ranh giới *phân phối/đóng gói*, không
phải một cơ chế runtime có luồng thực thi để lần theo. Việc "đọc source" thật sự bắt đầu từ
Chapter 03 (Class Loader), khi ta lần theo `java.lang.ClassLoader.loadClass()`.

## Summary

JDK, JRE, JVM là ba lớp lồng nhau: JVM chỉ thực thi bytecode; JRE = JVM + thư viện chuẩn, đủ
để **chạy**; JDK = JRE + công cụ dev (`javac`, `jshell`, `javap`...), đủ để **phát triển**.
JVM bản thân là một **đặc tả**, không phải một sản phẩm cụ thể — HotSpot, GraalVM,
Eclipse OpenJ9 đều là các cách hiện thực khác nhau cho cùng một chuẩn bytecode. Từ Java 11,
JRE không còn được phân phối độc lập, nhưng khái niệm "chỉ cần chạy, không cần biên dịch" vẫn
còn — nay được đáp ứng bằng `jlink` (Chapter 20) thay vì tải sẵn một gói JRE.

## Interview Questions

**Junior**

- JDK, JRE, JVM khác nhau như thế nào?
- `javac` nằm trong JDK hay JRE? `java` thì sao?

**Mid**

- Vì sao một máy chỉ cài JRE vẫn chạy được chương trình Java nhưng không biên dịch được?
- Từ Java 11, vì sao Oracle/OpenJDK không còn phát hành JRE độc lập nữa?

**Senior**

- JVM là một đặc tả hay một sản phẩm cụ thể? Kể tên ít nhất 2 hiện thực JVM khác HotSpot và
  nêu điểm khác biệt chính.
- Trong kiến trúc CI/CD, vì sao nên tách biệt image dùng để build (cần JDK) và image dùng để
  chạy production (chỉ cần JRE-tương-đương)? Đánh đổi là gì nếu gộp chung một image?

## Exercises

- [ ] Chạy `java -version` và `javac -version` trên máy bạn. Nếu có `javac`, tìm thư mục cài
      đặt JDK và liệt kê toàn bộ công cụ trong `bin/`.
- [ ] Mở `jshell`, thử gõ vài dòng code Java (khai báo biến, gọi method) mà không cần tạo
      file `.java` nào.
- [ ] Nếu máy bạn có cài từ 2 JDK trở lên, tìm cách xác định `JAVA_HOME` đang trỏ vào bản
      nào, và thử đổi sang bản khác.

## Cheat Sheet

| Lớp | Gồm | Dùng để |
| --- | --- | --- |
| JVM | Class Loader, Runtime Data Areas, Execution Engine, GC | Thực thi bytecode |
| JRE | JVM + thư viện chuẩn (`java.lang`, `java.util`...) | Chạy chương trình đã biên dịch |
| JDK | JRE + công cụ dev (`javac`, `jar`, `jshell`, `javap`...) | Phát triển + build chương trình |

| Lệnh | Thuộc lớp | Việc gì |
| --- | --- | --- |
| `javac` | JDK | Biên dịch `.java` → `.class` |
| `java` | JRE | Chạy `.class` |
| `jshell` | JDK | REPL, chạy code Java không cần file |
| `javap` | JDK | Xem bytecode |
| `jlink` | JDK | Tự đóng gói runtime tối giản (thay thế vai trò JRE cũ) |

## References

- Java Virtual Machine Specification (Oracle) — Chapter 1: Introduction.
- JEP 220: Modular Run-Time Images.
- Oracle: "JDK 11 Release Notes" — mục loại bỏ JRE/Server JRE installer riêng.
