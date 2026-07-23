---
tags:
  - Java
  - Lambda
  - ModernJava
---

# Lambda Expression

> Phase: Phase 4 — Modern Java
> Chapter slug: `lambda`

## Metadata

```yaml
Chapter: Lambda
Phase: Phase 4 — Modern Java
Difficulty: ★★
Importance: ★★★★★
Interview Frequency: 85%
Prerequisites:
  - Phase 1, Chapter 08 — OOP Fundamentals (anonymous class)
Used Later:
  - Chapter 02 — Functional Interface
  - Chapter 04 — Stream
Estimated Reading: 15 phút
Estimated Practice: 15 phút
```

## Story

> Xem lại: Phase 1, Chapter 08 (OOP Fundamentals) — muốn truyền một "hành vi" (một đoạn logic)
> như tham số cho một phương thức khác, cách duy nhất trước Java 8 là viết một **anonymous
> class** implement một interface có một phương thức duy nhất.

So sánh trực tiếp hai cách viết cùng một logic (tính tổng các chữ số của một số):

```java
DigitSum anon = new DigitSum() {
    @Override
    public int sum(int n) {
        int total = 0;
        while (n > 0) { total += n % 10; n /= 10; }
        return total;
    }
};

DigitSum lambda = n -> {
    int total = 0;
    while (n > 0) { total += n % 10; n /= 10; }
    return total;
};
```

Cả hai cho ra đúng cùng một kết quả (`sum(12345) = 15`), nhưng lambda ngắn hơn hẳn — không cần
lặp lại tên interface, tên phương thức, hay từ khoá `@Override`. Nhìn sâu hơn vào bytecode sinh
ra, sự khác biệt còn lớn hơn nhiều so với vẻ ngoài "chỉ là cú pháp gọn hơn".

## Interview Question (Central)

> Lambda expression trong Java là gì? Nó khác anonymous class như thế nào — cả về cú pháp lẫn
> cách JVM biên dịch/thực thi?

## Objectives

- [ ] Viết được lambda expression đúng cú pháp cho một functional interface (Chapter 02)
- [ ] Tự tay chứng minh bằng thực nghiệm: anonymous class sinh ra một `.class` file riêng,
      lambda thì không (dùng `invokedynamic`)
- [ ] Hiểu khái niệm "effectively final" khi lambda capture biến từ scope bên ngoài

## Prerequisites

- Phase 1, Chapter 08 — hiểu anonymous class, interface với một phương thức trừu tượng.

## Used Later

- **Chapter 02 (Functional Interface)** — lambda chỉ có thể gán cho một functional interface.
- **Chapter 04 (Stream)** — hầu hết thao tác trên Stream nhận lambda làm tham số.

## Problem

Trước Java 8, cách duy nhất để truyền một đoạn logic (không phải dữ liệu) như tham số là viết
một anonymous class implement một interface. Cú pháp này cồng kềnh: phải lặp lại tên interface,
tên phương thức, từ khoá `new`/`@Override` — cho một logic có thể chỉ vỏn vẹn một dòng. Ngoài
ra, mỗi anonymous class thực sự sinh ra một **`.class` file riêng** tại thời điểm biên dịch,
tăng số lượng class phải nạp khi ứng dụng khởi động.

## Concept

**Lambda expression** là cú pháp ngắn gọn để biểu diễn một **instance của functional interface**
(interface có đúng một phương thức trừu tượng, Chapter 02) — chỉ cần viết tham số và thân logic,
không cần khai báo lại tên interface/phương thức. Cú pháp: `(tham_so) -> bieu_thuc_hoac_khoi_lenh`.

## Why?

Lambda không chỉ là "đường cú pháp" (syntactic sugar) thuần tuý cho anonymous class — nó được
biên dịch theo cơ chế **khác hẳn**: dùng lệnh bytecode `invokedynamic` cùng `LambdaMetafactory`,
**trì hoãn** việc tạo class thực tế cho lambda tới tận **runtime** (lần đầu lambda đó được thực
thi), thay vì tạo sẵn một `.class` file tại thời điểm biên dịch như anonymous class. Thiết kế
này giúp giảm số lượng class phải nạp khi khởi động ứng dụng (đặc biệt quan trọng khi một
codebase có hàng nghìn lambda), đồng thời cho JVM có cơ hội tối ưu hoá việc tạo instance lambda
tại runtime.

## How?

```java
// Cu phap day du (khoi lenh, co return tuong minh)
Runnable r1 = () -> { System.out.println("Hello"); };

// Cu phap rut gon (bieu thuc don, tu dong return)
Function<Integer, Integer> square = x -> x * x;

// Nhieu tham so
BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b;
```

## Visualization

```
Anonymous class:                    Lambda:

  Bien dich:                          Bien dich:
   -> sinh NGAY 1 file .class          -> KHONG sinh file .class rieng
      (ClassName$1.class)                 chi sinh 1 lenh "invokedynamic"

  Runtime:                            Runtime:
   -> new ClassName$1() nhu           -> LAN DAU goi toi, LambdaMetafactory
      class binh thuong                  MOI tao ra instance implement
                                          functional interface (lazy)
```

## Example

```java
import java.util.function.Function;
import java.util.function.IntBinaryOperator;

public class LambdaDemo {
    interface DigitSum {
        int sum(int n);
    }

    public static void main(String[] args) {
        DigitSum anon = new DigitSum() {
            @Override
            public int sum(int n) {
                int total = 0;
                while (n > 0) { total += n % 10; n /= 10; }
                return total;
            }
        };

        DigitSum lambda = n -> {
            int total = 0;
            while (n > 0) { total += n % 10; n /= 10; }
            return total;
        };

        System.out.println("Anonymous class: sum(12345) = " + anon.sum(12345));
        System.out.println("Lambda: sum(12345) = " + lambda.sum(12345));

        int multiplier = 10; // effectively final
        Function<Integer, Integer> multiply = x -> x * multiplier;
        System.out.println("Lambda capture 'multiplier': multiply(5) = " + multiply.apply(5));

        IntBinaryOperator add = (a, b) -> a + b;
        System.out.println("IntBinaryOperator: add(3, 4) = " + add.applyAsInt(3, 4));
    }
}
```

Kết quả thật (JDK 17):

```
Anonymous class: sum(12345) = 15
Lambda: sum(12345) = 15
Lambda capture 'multiplier': multiply(5) = 50
IntBinaryOperator: add(3, 4) = 7
```

Cả hai cách viết cho đúng `15`. Nhưng kiểm tra các file `.class` thực tế sinh ra sau biên dịch:

```
$ ls LambdaDemo*.class
LambdaDemo$1.class          <- anonymous class: MOT class file rieng
LambdaDemo$DigitSum.class   <- interface
LambdaDemo.class            <- class chinh, KHONG co LambdaDemo$2.class cho lambda!
```

Chỉ có **`LambdaDemo$1.class`** (cho anonymous class) — không hề có `LambdaDemo$2.class` hay bất
kỳ file nào khác cho lambda, dù cả hai đều implement cùng interface `DigitSum`. Xem bytecode của
`main()` bằng `javap -c` xác nhận trực tiếp:

```
0: new           #7        // class LambdaDemo$1        <- anonymous class: new object binh thuong
...
8: invokedynamic #10,  0   // InvokeDynamic #0:sum:()LLambdaDemo$DigitSup;  <- lambda: invokedynamic!
```

## Deep Dive

**Vì sao lambda không sinh `.class` file tại thời điểm biên dịch?** Trình biên dịch `javac`
chỉ sinh ra một lệnh `invokedynamic` trỏ tới một **bootstrap method** (`LambdaMetafactory.metafactory`)
cùng phần thân lambda được biên dịch thành một **phương thức private tĩnh ẩn** trong chính class
chứa nó (không phải một class riêng). Lần đầu tiên JVM thực thi lệnh `invokedynamic` đó (runtime,
không phải compile-time), nó gọi `LambdaMetafactory` để **tạo động** (dùng kỹ thuật tương tự
Reflection nhưng hiệu quả hơn — dùng `MethodHandle`) một class implement functional interface
tương ứng, rồi cache lại kết quả này (`CallSite`) để các lần gọi tiếp theo không phải tạo lại.
Đây là lý do lambda thường có overhead khởi tạo lần đầu cao hơn một chút so với anonymous class
đã có sẵn class, nhưng từ lần thứ hai trở đi (đã cache) không có sự khác biệt đáng kể.

## Engineering Insight

**"Effectively final" nghĩa là gì, và vì sao lambda bắt buộc yêu cầu nó?** Một biến local được
gọi là "effectively final" nếu nó **chỉ được gán giá trị đúng một lần** (dù không có từ khoá
`final` tường minh) — như `multiplier` trong Example. Lambda chỉ được phép "capture" (dùng) biến
local thoả điều kiện này. Lý do: lambda có thể được thực thi **sau này**, có thể trên một thread
**khác** hoàn toàn với thread đã tạo ra nó (điển hình khi truyền lambda vào một task bất đồng bộ,
Phase 3) — nếu biến local gốc đã bị thay đổi hoặc thậm chí đã ra khỏi scope (biến local nằm trên
stack, có thể đã bị giải phóng), lambda sẽ đọc một giá trị sai hoặc không tồn tại. Yêu cầu
"effectively final" đảm bảo lambda luôn nắm giữ đúng **giá trị đã được sao chép tại thời điểm
tạo**, loại bỏ hoàn toàn khả năng đọc sai giá trị do thay đổi sau đó — bất kể lambda được thực
thi khi nào, trên thread nào.

## Historical Note

Lambda expression ra đời ở Java 8 (2014, JSR 335) — một trong những thay đổi ngôn ngữ lớn nhất
kể từ Generics (Java 5, 2004). Động lực chính là hỗ trợ Stream API (Chapter 04) theo phong cách
lập trình hàm (functional programming), vốn đã phổ biến ở các ngôn ngữ khác (Scala, JavaScript)
từ trước đó nhiều năm.

## Myth vs Reality

- **Myth:** "Lambda chỉ là cú pháp gọn hơn cho anonymous class, về bản chất kỹ thuật giống hệt
  nhau."
  **Reality:** Đã chứng minh bằng thực nghiệm — cơ chế biên dịch hoàn toàn khác nhau (sinh sẵn
  `.class` file vs `invokedynamic` trì hoãn tới runtime), dẫn tới khác biệt về số lượng class
  nạp khi khởi động và thời điểm class thực sự được tạo.

- **Myth:** "Lambda có thể dùng bất kỳ biến local nào từ scope bên ngoài, giống hệt closure
  trong JavaScript."
  **Reality:** Chỉ được dùng biến "effectively final" — xem Engineering Insight để hiểu lý do kỹ
  thuật đằng sau giới hạn này.

## Common Mistakes

- **Cố gán lại giá trị cho biến local đã bị lambda capture** — lỗi biên dịch "variable used in
  lambda expression should be final or effectively final", vì gán lại phá vỡ điều kiện
  "effectively final".
- **Viết lambda quá dài, nhiều dòng logic phức tạp** — làm giảm khả năng đọc hiểu, nên tách
  thành một phương thức riêng và dùng method reference (Chapter 03) khi logic đủ phức tạp.
- **Nhầm lẫn lambda có thể thay đổi (mutate) biến local bên ngoài** — lambda chỉ đọc được giá
  trị đã sao chép, không thể gán lại biến local gốc (khác với field của object, vẫn thay đổi
  được bình thường).

## Best Practices

- Ưu tiên lambda ngắn gọn, một biểu thức duy nhất khi có thể; tách thành phương thức riêng +
  method reference (Chapter 03) khi logic dài hơn vài dòng.
- Nhớ rằng lambda chỉ capture giá trị "effectively final" — không dùng lambda để thay đổi trạng
  thái biến local bên ngoài; dùng một object có field (ví dụ mảng 1 phần tử, hoặc `AtomicInteger`
  — Phase 3, Chapter 12) nếu thực sự cần một "hộp chứa" thay đổi được.
- Không nên "ép" mọi anonymous class thành lambda một cách máy móc — với logic phức tạp, đa
  phương thức, class thông thường vẫn là lựa chọn rõ ràng hơn.

## Debug Checklist

- [ ] Lỗi biên dịch "variable ... should be final or effectively final"? → kiểm tra biến local
      có bị gán lại ở đâu đó trong cùng scope không.
- [ ] Lambda chạy chậm bất thường ở lần gọi đầu tiên? → bình thường, do chi phí khởi tạo
      `LambdaMetafactory` lần đầu (xem Deep Dive) — các lần gọi sau đã được cache, không còn
      overhead này.

## Summary

Lambda expression cho phép viết ngắn gọn một instance của functional interface, thay thế
anonymous class cồng kềnh. Đã chứng minh bằng thực nghiệm: anonymous class sinh ra một `.class`
file riêng ngay lúc biên dịch, trong khi lambda dùng `invokedynamic`, trì hoãn việc tạo class
thực tế tới lần đầu thực thi tại runtime — không sinh thêm file `.class` nào. Lambda chỉ được
phép capture biến local "effectively final" (chỉ gán một lần), đảm bảo giá trị nhất quán bất kể
lambda được thực thi khi nào, trên thread nào.

## Interview Questions

- Coding: Write a Lambda expression to find the sum of digits of a number.

**Mid**

- Lambda khác anonymous class như thế nào về cách biên dịch và thực thi?
- "Effectively final" là gì? Vì sao lambda yêu cầu điều kiện này?

## Exercises

- [ ] Chạy lại `LambdaDemo` ở trên, tự kiểm tra bằng `ls *.class` và `javap -c` để xác nhận sự
      khác biệt bytecode giữa anonymous class và lambda.
- [ ] Viết một lambda implement `Comparator<String>` để sắp xếp danh sách chuỗi theo độ dài
      giảm dần.
- [ ] Thử cố tình gán lại một biến local đã bị lambda capture, quan sát lỗi biên dịch chính
      xác.

## Cheat Sheet

| | Anonymous Class | Lambda |
| --- | --- | --- |
| `.class` file | Sinh ngay lúc biên dịch | Không sinh riêng (dùng `invokedynamic`) |
| Cú pháp | Dài (lặp lại tên interface/method) | Ngắn gọn |
| `this` bên trong | Trỏ tới chính anonymous class | Trỏ tới class bao quanh (lexical scoping) |
| Capture biến local | Phải final/effectively final | Phải final/effectively final |
| Đa phương thức trừu tượng | Được | Không (chỉ functional interface) |

## References

- JLS §15.27 — Lambda Expressions.
- JSR 335 — Lambda Expressions for the Java Programming Language.
