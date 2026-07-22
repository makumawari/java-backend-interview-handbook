---
tags:
  - Java
  - JVM
  - Compilation
  - Bytecode
  - Foundation
---

# Một chương trình Java chạy như thế nào?

> Phase: Phase 1 — Java Foundation
> Chapter slug: `mot-chuong-trinh-java-chay-nhu-the-nao`

## Metadata

```yaml
Chapter: Một chương trình Java chạy như thế nào?
Phase: Phase 1 — Java Foundation
Difficulty: ★★
Importance: ★★★★
Interview Frequency: 60%
Prerequisites:
  - Biết cú pháp Java cơ bản (class, method, biến) — không cần biết gì về JVM
Used Later:
  - JDK, JRE, JVM (Chapter 02)
  - Class Loader (Chapter 03)
  - Runtime Data Areas (Chapter 04)
  - Execution Engine (Chapter 05)
  - JIT Compiler (Chapter 06)
  - Garbage Collection (Chapter 07)
  - JVM Tuning, Docker (Phase 8 — Production)
Estimated Reading: 20 phút
Estimated Practice: 30 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

Bạn mở terminal, gõ hai lệnh quen thuộc:

```
javac Main.java
java Main
```

Chữ `Hello World` hiện ra sau chưa đầy một giây. Bạn đã gõ hai lệnh này hàng trăm lần, nhưng
thử dừng lại 3 giây và tự hỏi: **giữa hai lệnh đó, máy tính đã làm những gì?**

Nếu bạn từng học qua Python, bạn sẽ thấy lạ: Python chỉ cần `python main.py` — một lệnh
duy nhất, chạy thẳng. Nếu bạn từng học qua C, bạn sẽ thấy lạ theo cách khác: C biên dịch ra
một file thực thi (`.exe`, `a.out`) chạy thẳng trên hệ điều hành đó, không chạy được trên hệ
điều hành khác.

Java thì làm cả hai việc, theo một cách khác hẳn: **biên dịch (như C) nhưng không biên dịch
ra máy thật — mà biên dịch ra một "máy giả lập".** File `Main.class` sinh ra sau `javac`
không phải là chương trình chạy được trên Windows/Linux/macOS. Nó là chương trình chạy được
trên **JVM** — một chiếc máy tính không tồn tại vật lý.

Câu hỏi thật sự của chapter này không phải "làm sao chạy được Java", mà là:

> Tại sao Java lại chọn con đường vòng này — biên dịch ra một máy giả lập, rồi máy giả lập
> đó mới thực sự chạy chương trình? Cái giá phải trả và cái lợi nhận được là gì?

Trả lời được câu hỏi này, bạn sẽ hiểu vì sao toàn bộ Phase 1 của handbook được xếp theo đúng
thứ tự: JDK/JRE/JVM là gì (Chapter 02) → JVM tìm và nạp class như thế nào (Chapter 03) → JVM
cấp phát bộ nhớ ở đâu (Chapter 04) → JVM thực thi bytecode ra sao (Chapter 05-06) → JVM dọn
rác thế nào (Chapter 07). Chapter này là tấm bản đồ; các chapter sau là từng vùng đất trên
bản đồ đó.

## Interview Question (Central)

> Khi bạn gõ `java Main` trong terminal, điều gì thực sự xảy ra — từ lúc gõ Enter đến lúc
> `Hello World` xuất hiện trên màn hình?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Phân biệt được rạch ròi hai giai đoạn: **biên dịch** (`javac`) và **thực thi** (`java`)
- [ ] Giải thích được bytecode là gì, vì sao nó không phải là "mã máy" và không phải là
      "mã nguồn"
- [ ] Kể tên được 4 việc chính mà JVM làm khi chạy một chương trình: nạp class, cấp phát bộ
      nhớ, thực thi bytecode, dọn rác — làm tiền đề để đi sâu từng việc ở các chapter sau
- [ ] Tự tay quan sát được bytecode thật bằng `javap -c` và quá trình nạp class thật bằng
      `-Xlog:class+load`
- [ ] Giải thích được vì sao "Java chậm vì là ngôn ngữ thông dịch" là một quan niệm sai

## Prerequisites

- Từng viết và chạy được một chương trình Java trước đây, kể cả chỉ nhớ mang máng cú pháp
  `class`, `public static void main`, khai báo biến. Chapter này **không yêu cầu** bạn đã
  biết gì về JVM, bytecode, hay compiler — đó chính là thứ chapter này sẽ xây từ đầu.

## Used Later

Bức tranh tổng thể ở chapter này sẽ được zoom vào từng mảnh ở các chapter kế tiếp:

- **JDK, JRE, JVM** (Chapter 02) — `javac` nằm ở đâu trong JDK, `java` chạy trên thành phần
  nào.
- **Class Loader** (Chapter 03) — bước "nạp class" ở phần How? bên dưới thực chất diễn ra
  như thế nào.
- **Runtime Data Areas** (Chapter 04) — "cấp phát bộ nhớ" gồm những vùng nào, Heap khác
  Stack ra sao.
- **Execution Engine, JIT Compiler** (Chapter 05-06) — Interpreter và JIT được nhắc ở đây
  chỉ là lời giới thiệu, cơ chế thật sẽ được mổ xẻ kỹ.
- **Garbage Collection** (Chapter 07) — vì sao Java không cần bạn tự giải phóng bộ nhớ như C.
- **JVM Tuning, Docker** (Phase 8) — thời gian khởi động JVM (điều bạn vừa thấy khi gõ
  `java Main`) là một vấn đề thật trong production khi chạy hàng trăm container.

## Problem

Giả sử bạn viết đúng một file `Main.java`. Bạn có hai lựa chọn thiết kế nếu là người tạo ra
ngôn ngữ Java vào năm 1995:

1. **Biên dịch thẳng ra mã máy** (giống C): nhanh, nhưng file thực thi sinh ra trên macOS sẽ
   không chạy được trên Windows — phải biên dịch lại cho từng hệ điều hành.
2. **Thông dịch từng dòng source code** (giống Python thời kỳ đầu): chạy được ở mọi nơi có
   trình thông dịch, nhưng chậm vì phải phân tích cú pháp lại mỗi lần chạy.

Java cần chạy được trên mọi thiết bị — máy chủ, máy tính cá nhân, và (mục tiêu ban đầu của
Java) cả tivi, lò vi sóng. Chọn phương án 1 thì mất tính di động. Chọn phương án 2 thì mất
tốc độ. Java phải giải quyết bài toán tưởng như không thể: vừa chạy được ở mọi nơi, vừa
nhanh.

## Concept

Java giải bài toán trên bằng cách **chèn thêm một tầng trung gian**: thay vì biên dịch thẳng
ra mã máy của CPU thật, `javac` biên dịch ra **bytecode** — một tập lệnh cho một "CPU ảo"
không tồn tại vật lý, gọi là **JVM (Java Virtual Machine)**.

```
Mã nguồn (.java)  →  Bytecode (.class)  →  JVM thông dịch/biên dịch bytecode → CPU thật
     con người đọc        JVM đọc              (khác nhau tuỳ Windows/Linux/macOS)
```

Bytecode giống mã nguồn ở chỗ nó không phụ thuộc hệ điều hành hay CPU — file `Main.class`
sinh ra trên macOS chạy y hệt trên Windows, miễn là máy đó có JVM. Bytecode giống mã máy ở
chỗ nó đã được biên dịch sẵn — JVM không cần phân tích cú pháp `if`, `for`, dấu ngoặc như
lúc thông dịch source code, nên nhanh hơn thông dịch source thuần tuý rất nhiều.

Đây chính là khẩu hiệu nổi tiếng của Java: **"Write Once, Run Anywhere"** — viết một lần,
chạy ở bất kỳ đâu có JVM.

## Why?

Nếu không có tầng bytecode/JVM này:

- Mỗi công ty viết phần mềm Java sẽ phải build ra nhiều bản khác nhau cho từng hệ điều hành
  khách hàng dùng — đúng vấn đề mà C/C++ gặp phải và Java sinh ra để giải quyết.
- Không thể có JIT Compiler tối ưu lúc runtime dựa trên hành vi thực tế của chương trình
  (xem Chapter 06) — vì compiler tĩnh của C chỉ nhìn thấy mã nguồn, không nhìn thấy chương
  trình đang chạy nhanh/chậm ở đâu.
- Không thể có Garbage Collector đứng giữa chương trình và bộ nhớ thật để tự động dọn rác
  (xem Chapter 07) — vì C biên dịch thẳng ra mã máy, không có "người đứng giữa" để can thiệp.

Tầng JVM chính là thứ cho phép toàn bộ những khả năng đó tồn tại. Cái giá phải trả: mỗi lần
chạy `java Main`, JVM phải khởi động từ đầu (nạp class, cấp phát bộ nhớ...) — không có sẵn
một file thực thi để chạy thẳng như C. Đây là gốc rễ của "JVM startup time", một vấn đề rất
thật trong production (xem phần Production Notes).

## How?

Toàn bộ hành trình từ `javac Main.java` đến `Hello World` xuất hiện trên màn hình gồm hai
giai đoạn tách biệt:

**Giai đoạn 1 — Compile-time (chỉ chạy một lần, bởi `javac`):**

1. `javac` đọc `Main.java`, kiểm tra cú pháp và kiểu dữ liệu (type checking).
2. Sinh ra bytecode, lưu vào file `.class` — mỗi class (kể cả class không phải `public`)
   sinh ra một file `.class` riêng.

**Giai đoạn 2 — Run-time (chạy lại từ đầu mỗi lần gõ `java`):**

3. Lệnh `java Main` khởi động một **tiến trình JVM** mới.
4. **Class Loader** tìm file `Main.class`, đọc, kiểm tra tính hợp lệ (verify), rồi nạp vào
   bộ nhớ JVM.
5. JVM cấp phát các vùng nhớ runtime (**Runtime Data Areas**): Method Area/Metaspace chứa
   thông tin class, Heap chứa object, Stack chứa biến cục bộ và lời gọi method.
6. **Execution Engine** bắt đầu chạy method `main()`: phần lớn bytecode được **Interpreter**
   thông dịch từng lệnh; những đoạn code chạy đi chạy lại nhiều lần ("hot code") được
   **JIT Compiler** biên dịch sang mã máy thật để chạy nhanh hơn ở những lần sau.
7. **Garbage Collector** âm thầm chạy nền, dọn các object trong Heap không còn được tham
   chiếu tới nữa.
8. Chương trình kết thúc, JVM thoát.

## Visualization

```
Main.java  (con người đọc được)
     │
     │  javac Main.java
     ▼
Main.class  (bytecode — JVM đọc được, không phụ thuộc OS/CPU)
     │
     │  java Main   ← khởi động một tiến trình JVM MỚI, mỗi lần chạy lại từ đầu
     ▼
┌────────────────────────── Tiến trình JVM ───────────────────────────┐
│                                                                       │
│  (1) Class Loader                                                    │
│       └─ tìm, đọc, verify Main.class            [chi tiết: Ch. 03]  │
│                          │                                           │
│                          ▼                                           │
│  (2) Runtime Data Areas                                              │
│       └─ cấp phát Method Area / Heap / Stack     [chi tiết: Ch. 04]  │
│                          │                                           │
│                          ▼                                           │
│  (3) Execution Engine                                                │
│       ├─ Interpreter: thông dịch bytecode        [chi tiết: Ch. 05] │
│       └─ JIT Compiler: biên dịch "hot code"                          │
│          sang mã máy thật                        [chi tiết: Ch. 06] │
│                          │                                           │
│                          ▼                                           │
│  (4) Garbage Collector (chạy nền song song)      [chi tiết: Ch. 07] │
│                                                                       │
└───────────────────────────────────────────────────────────────────┘
     │
     ▼
"Hello World" in ra màn hình, JVM thoát
```

## Example

```java
public class Main {
    public static void main(String[] args) {
        Product product = new Product("Áo thun", 199_000);
        System.out.println("Sản phẩm: " + product.getName() + " - Giá: " + product.getPrice());
    }
}

class Product {
    private final String name;
    private final int price;

    Product(String name, int price) {
        this.name = name;
        this.price = price;
    }

    String getName() { return name; }
    int getPrice() { return price; }
}
```

Biên dịch và chạy:

```
javac Main.java
java Main
```

Kết quả:

```
Sản phẩm: Áo thun - Giá: 199000
```

**Thử ngay:** sau khi `javac Main.java`, chạy `ls *.class`. Bạn sẽ thấy **hai** file:
`Main.class` và `Product.class` — dù cả hai class cùng nằm trong một file `.java`. Đây là
manh mối đầu tiên cho Chapter 03 (Class Loader): JVM nạp class theo đơn vị `.class`, không
theo đơn vị file `.java`.

## Deep Dive

**Nhìn thẳng vào bytecode.** Chạy `javap -c Main.class` trên đúng ví dụ ở trên, bạn sẽ thấy
method `main` được biên dịch thành (output thật, đã chạy thử):

```
public static void main(java.lang.String[]);
    Code:
       0: new           #7    // class Product
       3: dup
       4: ldc           #9    // String Áo thun
       6: ldc           #11   // int 199000
       8: invokespecial #12   // Method Product."<init>":(Ljava/lang/String;I)V
      11: astore_1
      12: getstatic     #15   // Field java/lang/System.out:Ljava/io/PrintStream;
      15: aload_1
      16: invokevirtual #21   // Method Product.getName:()Ljava/lang/String;
      19: aload_1
      20: invokevirtual #25   // Method Product.getPrice:()I
      23: invokedynamic #29,0 // InvokeDynamic makeConcatWithConstants:(...)Ljava/lang/String;
      28: invokevirtual #33   // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      31: return
```

Đây không phải mã máy (CPU không hiểu `invokespecial`), cũng không phải mã nguồn (không còn
dấu vết `new Product(...)` như một khối, mà tách thành `new` + `dup` + gọi constructor riêng)
— nó nằm ở giữa. Mỗi lệnh bytecode là một hành động rất nhỏ, rất tường minh: `new` (cấp phát
object rỗng trên Heap), `invokespecial` (gọi constructor), `getstatic` (lấy field tĩnh),
`invokevirtual` (gọi method thông qua cơ chế đa hình)... Chính vì bytecode đã "chặt nhỏ"
chương trình thành các bước tường minh như vậy, JVM thông dịch nó nhanh hơn nhiều so với việc
thông dịch trực tiếp cú pháp `.java`.

Chú ý lệnh `23: invokedynamic` — đây là cách trình biên dịch hiện đại (Java 9+) dịch phép nối
chuỗi `"..." + product.getName() + ...`. Thay vì sinh sẵn code `StringBuilder.append(...)` lặp
đi lặp lại như các phiên bản Java cũ, `javac` để lại một "chỗ trống" (`invokedynamic`) và để
JVM tự quyết định chiến lược nối chuỗi tối ưu nhất **lúc chạy** — một ví dụ rất cụ thể cho ý
"JVM tối ưu lúc runtime" ở phần Engineering Insight bên dưới.

**Nhìn thẳng vào quá trình nạp class.** Chạy chương trình với cờ:

```
java -Xlog:class+load=info Main
```

Bạn sẽ thấy hàng trăm dòng log, trong đó có `Main` và `Product` — đúng thứ tự JVM nạp chúng
vào bộ nhớ. (`-Xlog:class+load` là cờ logging hợp nhất từ Java 9+/JEP 158; JVM cũ hơn dùng
`-verbose:class`, vẫn còn hoạt động như một alias.) Đây chính là hành vi của Class Loader mà
Chapter 03 sẽ mổ xẻ chi tiết: JVM không nạp toàn bộ class ngay từ đầu, mà nạp "lazy" — class
nào được dùng tới mới nạp.

## Engineering Insight

**Tại sao không compile thẳng ra mã máy như C, rồi cứ thế mà chạy — nhanh hơn hẳn?**

Vì compiler tĩnh (như `gcc`) chỉ nhìn thấy *mã nguồn*, không nhìn thấy *chương trình đang
chạy*. Nó không biết được: vòng lặp nào chạy 3 lần, vòng lặp nào chạy 3 triệu lần; nhánh
`if` nào gần như luôn đúng, nhánh nào gần như không bao giờ vào. JIT Compiler của JVM (xem
Chapter 06) đứng ở vị trí đặc biệt: nó quan sát chương trình **khi đang chạy thật**, đo đếm
method nào được gọi nhiều, rồi mới quyết định bỏ công biên dịch tối ưu phần đó sang mã máy.
Đổi lại, JVM phải trả giá bằng thời gian khởi động (phải nạp class, thông dịch, "làm nóng"
trước khi JIT phát huy tác dụng) — điều bạn sẽ gặp lại ở Phase 8 khi bàn về JVM Tuning và cold
start của container.

Nói cách khác: C tối ưu **lúc biên dịch**, dựa trên thông tin tĩnh. JVM tối ưu **lúc chạy**,
dựa trên hành vi thật. Đây không phải Java "chậm hơn hay nhanh hơn" C một cách tuyệt đối — đó
là hai triết lý tối ưu khác nhau, đánh đổi khác nhau.

## Historical Note

```
Java 1.0 (1996)
    ↓  Thuần thông dịch bytecode, không có JIT — rất chậm, bị chê "Java chậm"
J2SE 1.3 / HotSpot (2000)
    ↓  Giới thiệu JIT Compiler thích ứng (adaptive) — tốc độ tăng vọt
Java 9 (2017)
    ↓  JEP 158 — Unified JVM Logging (-Xlog thay cho các cờ -verbose/-XX rời rạc)
Java 11 (2018)
    ↓  JEP 330 — chạy thẳng "java Main.java" không cần javac tường minh (xem Cheat Sheet)
GraalVM Native Image (Spring Boot 3+)
    ↓  Biên dịch AOT (Ahead-Of-Time) thẳng ra file thực thi native — bỏ hẳn bước
       "khởi động JVM" để có cold-start gần như tức thì trong container
```

Định kiến "Java chậm" bắt nguồn từ Java 1.0 — thời chưa có JIT. Định kiến đó đã lỗi thời hơn
20 năm nhưng vẫn còn phổ biến (xem Myth vs Reality). Dòng cuối (GraalVM) cho thấy: ngay cả
bản thân hệ sinh thái Java cũng đang tối ưu tiếp cái giá "khởi động JVM" mà chapter này vừa
nói tới — không phải để phủ nhận JVM, mà để có thêm lựa chọn khi cold-start quan trọng hơn
peak performance (ví dụ: serverless function).

## Myth vs Reality

- **Myth:** "Java là ngôn ngữ thông dịch (interpreted), nên chậm hơn C."
  **Reality:** Java là mô hình **lai (hybrid)** — bytecode được cả thông dịch lẫn JIT biên
  dịch sang mã máy thật. Với "hot code", tốc độ sau khi JIT "làm nóng" tiệm cận mã native.
  Cái chậm thật sự nằm ở giai đoạn khởi động (nạp class, chưa kịp JIT), không nằm ở bản chất
  ngôn ngữ.

- **Myth:** "`javac` biên dịch ra chương trình chạy được, giống hệt C biên dịch ra `.exe`."
  **Reality:** `javac` chỉ biên dịch ra bytecode trung gian. Bytecode **không tự chạy được**
  trên hệ điều hành — nó cần JVM làm "người phiên dịch" lúc runtime. Đây là khác biệt cốt lõi
  so với C.

## Common Mistakes

- **Nhầm `javac` với `java`.** Người mới thường không phân biệt rõ: `javac` chỉ biên dịch
  (không chạy chương trình), `java` chỉ chạy (cần đã có sẵn `.class`, hoặc dùng cách chạy
  file nguồn trực tiếp ở Cheat Sheet bên dưới).
- **Đổi tên file `.java` không khớp tên class `public`.** Java bắt buộc: nếu class là
  `public`, tên file phải trùng tên class (`Main.java` cho `class Main`). Sai tên → lỗi biên
  dịch khó hiểu với người mới.
- **Biên dịch bằng JDK version cao, chạy bằng JRE/JVM version thấp hơn** → gặp
  `UnsupportedClassVersionError` (xem Debug Checklist) — vì bytecode có đánh số phiên bản,
  JVM cũ không đọc được bytecode "từ tương lai".

## Best Practices

- Dùng `java -version` để xác nhận đang chạy đúng JDK/JVM mong muốn — đặc biệt quan trọng
  khi máy cài song song nhiều phiên bản JDK (rất phổ biến khi làm nhiều dự án).
- Khi chỉ muốn thử nhanh một file `.java` (script, học tập, demo), dùng cách chạy trực tiếp
  ở Cheat Sheet thay vì `javac` rồi `java` hai bước — đỡ sinh file `.class` rác.
- Khi build để deploy production, luôn chỉ định rõ `--release <version>` cho `javac` thay vì
  để mặc định theo máy build — tránh trường hợp máy build và máy chạy production lệch version
  JDK mà không ai để ý.

## Production Notes

**Vấn đề:** ứng dụng chạy tốt trên máy dev, nhưng khi deploy lên server production thì crash
ngay lúc khởi động với `java.lang.UnsupportedClassVersionError` hoặc
`NoClassDefFoundError`.

- **Triệu chứng:** ứng dụng không khởi động được; log báo lỗi ngay trong vài trăm
  mili-giây đầu tiên, trước khi bất kỳ dòng log nghiệp vụ nào kịp in ra.
- **Nguyên nhân gốc (`UnsupportedClassVersionError`):** file `.class` được biên dịch bằng
  JDK mới hơn JVM đang chạy trên server (ví dụ: build bằng JDK 21, server production vẫn cài
  JRE 17). Bytecode có major version number; JVM cũ từ chối nạp bytecode "từ tương lai".
- **Nguyên nhân gốc (`NoClassDefFoundError`):** class từng biên dịch thành công (có mặt lúc
  `javac`), nhưng **lúc runtime**, Class Loader không tìm thấy file `.class` tương ứng trên
  classpath — thường do thiếu `.jar` dependency lúc deploy, hoặc classpath cấu hình sai.
- **Debug:** so sánh `java -version` giữa máy build (CI/CD) và máy production; kiểm tra
  classpath thực tế lúc chạy (`java -verbose:class` hoặc `-Xlog:class+load=info` như ở phần
  Deep Dive để xem class nào JVM thực sự tìm thấy/không tìm thấy).
- **Solution:** thống nhất version JDK giữa build và runtime (Docker image cố định version
  là cách phổ biến nhất); đảm bảo mọi dependency được đóng gói đầy đủ (fat jar/uber jar hoặc
  layer đúng trong image).
- **Prevention:** pin chính xác version JDK trong Dockerfile/CI pipeline, không dùng
  `latest`; thêm bước kiểm tra `java -version` như một health check ngay đầu pipeline deploy.

## Debug Checklist

Chương trình không chạy được ngay từ bước khởi động — kiểm tra theo đúng thứ tự pipeline ở
phần How?:

- [ ] Lỗi xảy ra lúc `javac` (compile-time) hay lúc `java` (run-time)? Đọc kỹ thông báo lỗi —
      compile-time luôn báo rõ số dòng trong `.java`.
- [ ] Nếu là `ClassNotFoundException`/`NoClassDefFoundError`: classpath có trỏ đúng chỗ chứa
      `.class`/`.jar` không?
- [ ] Nếu là `UnsupportedClassVersionError`: so `java -version` (runtime) với version JDK lúc
      build.
- [ ] Nếu báo lỗi tên class không khớp file: tên file `.java` có trùng tên `public class`
      bên trong không?
- [ ] Nếu nghi ngờ sai class được nạp (ví dụ có 2 bản `.jar` cùng chứa 1 class): chạy với
      `-Xlog:class+load=info` để xem chính xác JVM nạp class đó từ file nào.

## Source Code Walkthrough

Ở mức tổng quan (chưa cần đọc source C++ của HotSpot — việc đó dành cho các chapter sau khi
đã có đủ nền tảng), trình tự khi gõ `java Main` là:

```
java (launcher, src/java.base/share/native/launcher/main.c trong OpenJDK)
   ↓
JNI_CreateJavaVM()          — tạo tiến trình JVM
   ↓
Class Loader nạp Main.class
   ↓
JVM tìm method "public static void main(String[])" trong class Main
   ↓
Execution Engine bắt đầu thực thi bytecode của main()
```

Chapter 03 (Class Loader) sẽ đi sâu vào bước "Class Loader nạp Main.class" — cụ thể là ba
class loader phân cấp (Bootstrap, Platform, Application) và cơ chế delegation giữa chúng.

## Summary

Một chương trình Java chạy qua hai giai đoạn tách biệt: **compile-time** (`javac` biến
`.java` thành bytecode `.class`, chạy một lần) và **run-time** (`java` khởi động một tiến
trình JVM mới mỗi lần, nạp class, cấp phát bộ nhớ, thực thi bytecode qua Interpreter + JIT,
và dọn rác qua GC). Tầng bytecode/JVM ở giữa chính là thứ đánh đổi lấy tính di động
("Write Once, Run Anywhere") và khả năng tối ưu lúc runtime (JIT, GC) — đổi lại bằng chi phí
khởi động mà C không có. Bốn việc JVM làm mỗi lần chạy — nạp class, cấp phát bộ nhớ, thực thi,
dọn rác — chính là bốn chủ đề của Chapter 03 đến 07.

## Interview Questions

**Junior**

- `javac` và `java` khác nhau như thế nào? Cái nào tạo ra file, cái nào không?
- Bytecode là gì? Nó có phải là mã máy (machine code) không?

**Mid**

- Khi bạn gõ `java Main`, liệt kê các bước JVM thực hiện cho tới khi chương trình bắt đầu
  chạy `main()`.
- Vì sao file `.class` biên dịch bằng JDK 21 lại không chạy được trên máy chỉ cài JRE 8?
- Một file `.java` chứa 2 class thì sinh ra bao nhiêu file `.class`? Vì sao?

**Senior**

- JIT Compiler giải quyết vấn đề gì mà một compiler tĩnh như `gcc` không giải quyết được?
  Đánh đổi là gì?
- So sánh mô hình thực thi của Java (bytecode + JVM), Python (thông dịch thuần), và C
  (biên dịch thẳng ra native). Trong bối cảnh nào mỗi mô hình có lợi thế?
- Vì sao GraalVM Native Image lại hấp dẫn với các ứng dụng chạy trên serverless/container,
  trong khi JVM truyền thống thì không?

## Exercises

- [ ] Viết một file `.java` chứa 2 class như ví dụ ở trên. Biên dịch, đếm số file `.class`
      sinh ra, giải thích vì sao.
- [ ] Chạy `javap -c` lên file `.class` vừa biên dịch. Đối chiếu từng nhóm lệnh bytecode với
      dòng source code tương ứng — đặc biệt tìm lệnh `invokevirtual` ứng với
      `System.out.println(...)`.
- [ ] Cố tình biên dịch bằng `javac --release 21 Main.java`, sau đó tìm cách chạy bằng một
      JVM cũ hơn (nếu có sẵn) hoặc tra cứu thông báo lỗi `UnsupportedClassVersionError` sẽ
      trông như thế nào — tự giải thích ý nghĩa từng phần trong thông báo lỗi đó.
- [ ] Chạy chương trình với `java -Xlog:class+load=info Main`, lọc (`grep`) ra các dòng có
      chữ "Product" — xác nhận `Product` được nạp trước hay sau khi `main()` bắt đầu chạy.

## Cheat Sheet

| Lệnh | Ý nghĩa |
| --- | --- |
| `javac Main.java` | Biên dịch, sinh ra `Main.class` (và `.class` cho mọi class khác trong file) |
| `java Main` | Chạy — cần **tên class** đã biên dịch, không phải tên file |
| `java Main.java` | (Java 11+, JEP 330) Biên dịch + chạy trong một bước, không ghi `.class` ra đĩa — tiện cho script/demo nhanh |
| `javap -c Main.class` | Xem bytecode dạng người đọc được |
| `java -version` | Kiểm tra đang chạy JDK/JVM phiên bản nào |
| `java -Xlog:class+load=info Main` | Xem JVM nạp class nào, theo thứ tự nào (Java 9+; bản cũ hơn dùng `-verbose:class`) |

## References

- Java Virtual Machine Specification (Oracle) — Chapter 5: Loading, Linking, and
  Initializing.
- JEP 158: Unified JVM Logging.
- JEP 330: Launch Single-File Source-Code Programs.
- OpenJDK source — `src/java.base/share/native/launcher/main.c` (điểm khởi đầu của lệnh
  `java`).
