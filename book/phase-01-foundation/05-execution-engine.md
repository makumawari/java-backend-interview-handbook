---
tags:
  - Java
  - JVM
  - Interpreter
  - Bytecode
  - Foundation
---

# Execution Engine

> Phase: Phase 1 — Java Foundation
> Chapter slug: `execution-engine`

## Metadata

```yaml
Chapter: Execution Engine
Phase: Phase 1 — Java Foundation
Difficulty: ★★★
Importance: ★★★
Interview Frequency: 40%
Prerequisites:
  - Chapter 01 — Một chương trình Java chạy như thế nào?
  - Chapter 04 — Runtime Data Areas
Used Later:
  - JIT Compiler (Chapter 06) — đi sâu vào phần "biên dịch" của Execution Engine
  - Thread, Concurrency (Phase 3) — mỗi thread có một luồng thực thi bytecode riêng
Estimated Reading: 15 phút
Estimated Practice: 15 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Chapter 01](01-mot-chuong-trinh-java-chay-nhu-the-nao.md) — bạn đã thấy bytecode
> thật của `main()` qua `javap -c`, với các lệnh như `getstatic`, `invokevirtual`.
> [Chapter 04](04-runtime-data-areas.md) đã giới thiệu JVM Stack, nơi mỗi lời gọi method tạo
> ra một "frame".

Nhưng bytecode chỉ là một danh sách lệnh nằm im trong file `.class`. Thứ gì thực sự **đọc**
từng lệnh đó và **làm** đúng như nó nói? Và bytecode không có khái niệm biến `a`, biến `b` như
source code — vậy phép cộng `a + b` được biểu diễn và tính toán như thế nào ở tầng bytecode?

Câu trả lời nằm ở **Execution Engine** — thành phần của JVM đóng vai trò "CPU ảo" thật sự thực
thi từng lệnh bytecode một.

## Interview Question (Central)

> Execution Engine chạy bytecode như thế nào? Tại sao bytecode Java lại được thiết kế dựa
> trên một "ngăn xếp" (stack) thay vì dùng thanh ghi (register) như CPU thật?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Hiểu bytecode JVM là một **stack-based instruction set** — khác với CPU thật (thường
      là register-based)
- [ ] Tự tay lần theo (trace) từng bước operand stack khi thực thi một đoạn bytecode thật
- [ ] Phân biệt được **Interpreter** (thông dịch từng lệnh) với **JIT Compiler** (biên dịch
      sẵn — chi tiết ở Chapter 06)
- [ ] Biết Execution Engine nằm ở đâu trong bức tranh tổng thể JVM đã vẽ ở Chapter 01

## Prerequisites

- Chapter 01 — đã biết bytecode là gì, xem qua `javap -c`.
- Chapter 04 — đã biết JVM Stack, frame method là gì.

## Used Later

- **JIT Compiler** (Chapter 06) — chapter này chỉ giới thiệu Interpreter; JIT Compiler (nửa
  còn lại của Execution Engine) sẽ được mổ xẻ kỹ ở chapter kế tiếp.
- **Thread, Concurrency** (Phase 3) — mỗi thread chạy trên một luồng thực thi bytecode độc
  lập (gắn với JVM Stack và PC Register riêng của thread đó, đã học ở Chapter 04).

## Problem

Bytecode đã được biên dịch sẵn (Chapter 01), nhưng nó vẫn cần một "người" đọc và thực hiện
từng lệnh — giống hệt CPU thật cần đọc và thực hiện từng lệnh mã máy. JVM không có CPU vật lý
riêng của nó; nó phải **giả lập** hành vi đó bằng phần mềm.

Thêm một vấn đề thiết kế: các phép tính trong bytecode (như `a + b`) cần lưu toán hạng tạm
thời ở đâu trước khi tính? CPU thật thường dùng **thanh ghi** (register) — một số ô nhớ cực
nhanh có tên cụ thể (như `EAX`, `EBX`). Nhưng số lượng thanh ghi khác nhau tuỳ kiến trúc CPU
(x86, ARM...) — nếu bytecode thiết kế dựa trên thanh ghi, nó sẽ gắn chặt với một kiến trúc CPU
cụ thể, phá vỡ đúng mục tiêu "chạy được ở mọi nơi" đã bàn ở Chapter 01.

## Concept

Bytecode JVM là **stack-based**: thay vì thanh ghi có tên, mỗi frame (Chapter 04) có một
**operand stack** — một ngăn xếp tạm để chứa toán hạng. Lệnh bytecode thao tác bằng cách
**đẩy (push)** giá trị vào operand stack và **lấy ra (pop)** để tính toán, kết quả lại được
đẩy trở lại.

Execution Engine gồm hai cơ chế phối hợp:

- **Interpreter** — đọc và thực thi bytecode **từng lệnh một**, ngay lập tức, không cần bước
  biên dịch trước. Khởi động nhanh nhưng chạy lại từ đầu mỗi lần gặp cùng một lệnh.
- **JIT Compiler** — theo dõi phần bytecode chạy nhiều lần ("hot code"), biên dịch nó thành
  mã máy thật một lần, dùng lại nhiều lần sau đó. Chi tiết đầy đủ ở Chapter 06.

## Why?

Nếu bytecode dựa trên thanh ghi thay vì stack: mỗi JVM implementation trên mỗi kiến trúc CPU
sẽ phải tự ánh xạ số thanh ghi trừu tượng trong bytecode sang số thanh ghi vật lý khác nhau —
phức tạp và khó đảm bảo hành vi giống hệt nhau trên mọi nền tảng. Stack-based tránh hoàn toàn
vấn đề này: operand stack là một khái niệm trừu tượng, JVM nào cũng hiện thực được bất kể CPU
bên dưới có bao nhiêu thanh ghi thật.

Đánh đổi: thực thi thuần bằng Interpreter trên stack-based bytecode chậm hơn thực thi trực
tiếp trên thanh ghi CPU thật — đây chính xác là lý do JIT Compiler tồn tại (Chapter 06): JIT
biên dịch bytecode stack-based **thành** mã máy register-based thật của CPU, lấy lại tốc độ đã
"hy sinh" vì tính di động.

## How?

Lấy lại đúng ví dụ `add(2, 3)` từ Chapter 01, bytecode thật của method `add`:

```java
static int add(int a, int b) {
    int c = a + b;
    return c;
}
```

```
0: iload_0     // đẩy giá trị biến cục bộ #0 (a) vào operand stack
1: iload_1     // đẩy giá trị biến cục bộ #1 (b) vào operand stack
2: iadd        // lấy 2 giá trị trên đỉnh stack, cộng lại, đẩy kết quả vào stack
3: istore_2    // lấy giá trị trên đỉnh stack, lưu vào biến cục bộ #2 (c)
4: iload_2     // đẩy giá trị biến cục bộ #2 (c) vào operand stack
5: ireturn     // lấy giá trị trên đỉnh stack, trả về, kết thúc method
```

Interpreter thực thi tuần tự từng dòng, đúng theo nghĩa đen của "iload", "iadd", "istore" —
không có bước phân tích cú pháp `a + b` nào cả, vì `javac` đã làm việc đó từ trước (Chapter
01). Interpreter chỉ việc làm theo chỉ dẫn đã được "chặt nhỏ" sẵn.

## Visualization

Lần theo operand stack qua từng bước (giả sử gọi `add(2, 3)`, biến cục bộ #0=2, #1=3):

```
Lệnh          Operand Stack (đỉnh bên phải)      Biến cục bộ
─────────────────────────────────────────────────────────────
(bắt đầu)     [ ]                                 #0=2  #1=3  #2=?
0: iload_0    [ 2 ]                                #0=2  #1=3  #2=?
1: iload_1    [ 2, 3 ]                              #0=2  #1=3  #2=?
2: iadd       [ 5 ]          ← pop 2,3; push 2+3    #0=2  #1=3  #2=?
3: istore_2   [ ]             ← pop 5, lưu vào #2   #0=2  #1=3  #2=5
4: iload_2    [ 5 ]                                #0=2  #1=3  #2=5
5: ireturn    [ ]             ← trả về 5
```

Mỗi lệnh chỉ làm đúng MỘT việc rất nhỏ trên đỉnh stack — đây là bản chất "chặt nhỏ" đã nhắc ở
Chapter 01, và cũng là lý do Interpreter thực thi được mà không cần hiểu ngữ nghĩa cấp cao
như `a + b`.

## Example

Xác nhận bytecode thật khớp với bảng trace ở trên (chạy trên chính máy bạn):

```java
public class AddDemo {
    static int add(int a, int b) {
        int c = a + b;
        return c;
    }
    public static void main(String[] args) {
        System.out.println(add(2, 3));
    }
}
```

```
javac AddDemo.java
javap -c AddDemo.class
```

Kết quả thật (JDK 17) — đúng khớp với bảng trace ở phần Visualization:

```
static int add(int, int);
  Code:
     0: iload_0
     1: iload_1
     2: iadd
     3: istore_2
     4: iload_2
     5: ireturn
```

## Deep Dive

**Tiền tố `i` trong `iload`, `iadd`, `istore`, `ireturn` nghĩa là gì?** Bytecode JVM có các
họ lệnh riêng cho từng kiểu dữ liệu nguyên thuỷ: `i` (int), `l` (long), `f` (float), `d`
(double), `a` (reference — object). Ví dụ `iadd` cộng hai số `int`, còn `ladd` cộng hai số
`long`. Đây là lý do bytecode Java **kiểm tra kiểu ngay ở tầng lệnh**, không cần Interpreter
tự suy luận kiểu dữ liệu lúc chạy như một số ngôn ngữ thông dịch động khác — một phần lý do
Interpreter của JVM nhanh hơn interpreter của ngôn ngữ không có kiểu tĩnh.

## Engineering Insight

**Vì sao operand stack nằm trong từng frame (Chapter 04), không phải một stack chung cho cả
thread?** Vì mỗi lời gọi method là một "phép tính" độc lập — method `add` không được phép nhìn
thấy hay làm nhiễu operand stack của method gọi nó (`main`). Nếu dùng chung một operand stack
cho cả chuỗi lời gọi, một lỗi tính toán trong `add` có thể làm sai lệch dữ liệu của `main` mà
không ai biết. Việc mỗi frame có operand stack riêng (đóng lại hoàn toàn khi method return) là
một dạng cô lập giống hệt tinh thần "per-thread Stack" đã bàn ở Chapter 04 — chỉ khác là ở cấp
độ nhỏ hơn: cô lập giữa các *lời gọi method*, không chỉ giữa các *thread*.

## Historical Note

Thiết kế stack-based cho bytecode không phải phát minh riêng của Java — nó kế thừa truyền
thống từ các máy ảo stack-based trước đó (như UCSD p-System những năm 1970, dùng cho Pascal).
Java (1995) áp dụng lại ý tưởng này đúng vào thời điểm "chạy đa nền tảng" trở thành yêu cầu
sống còn của ngôn ngữ, biến một kỹ thuật cũ thành nền tảng cho "Write Once, Run Anywhere".

## Myth vs Reality

- **Myth:** "Bytecode chạy chậm vì JVM phải 'dịch' `a + b` thành phép cộng mỗi lần chạy."
  **Reality:** `javac` đã "dịch" `a + b` thành `iload, iload, iadd` từ lúc biên dịch (Chapter
  01). Interpreter lúc runtime chỉ thực thi các lệnh đã chặt nhỏ sẵn, không phân tích cú pháp
  lại — đây là lý do bytecode nhanh hơn thông dịch source code thuần tuý.

- **Myth:** "Execution Engine chỉ là Interpreter."
  **Reality:** Execution Engine gồm cả Interpreter và JIT Compiler chạy song song, phối hợp
  với nhau (Chapter 06) — không chỉ đơn thuần thông dịch.

## Common Mistakes

- Nhầm lẫn giữa **biến cục bộ** (local variable array của frame, đánh số #0, #1, #2...) và
  **operand stack** (ngăn xếp tạm để tính toán) — đây là hai vùng khác nhau trong cùng một
  frame, dễ gộp lẫn khi mới đọc bytecode.
- Cho rằng đọc được bytecode nghĩa là hiểu được mọi tối ưu hoá runtime — bytecode chỉ là
  "kịch bản gốc"; JIT Compiler (Chapter 06) có thể biến đổi nó rất nhiều lúc thực thi thật
  (inline method, loại bỏ code thừa...), điều `javap -c` không thể hiện được.

## Best Practices

- Khi cần hiểu vì sao một đoạn code chạy chậm ở mức bytecode, đọc `javap -c` trước để loại
  trừ khả năng do cấu trúc code sinh ra bytecode dư thừa (ví dụ tạo object không cần thiết
  trong vòng lặp), trước khi nghi ngờ tới JIT/GC.
- Không cần thuộc lòng toàn bộ tập lệnh bytecode — chỉ cần hiểu đúng **mô hình** (stack-based,
  chặt nhỏ theo kiểu dữ liệu) là đủ để đọc hiểu bất kỳ đoạn `javap -c` nào khi cần debug sâu.

## Production Notes

Chapter này thuần khái niệm nền tảng — các vấn đề production thật sự liên quan tới thực thi
bytecode (biên dịch JIT chậm khởi động, deoptimization...) thuộc về
[Chapter 06 — JIT Compiler](06-jit-compiler.md), nơi có đủ ngữ cảnh để bàn Production Notes ý
nghĩa hơn là lặp lại ở đây.

## Debug Checklist

- [ ] Nghi ngờ một đoạn code sinh bytecode không như mong đợi (ví dụ boxing/unboxing thừa)?
      → chạy `javap -c` trực tiếp trên class đó, đối chiếu từng dòng với source.
- [ ] Thấy lệnh có tiền tố lạ (`d`, `l`, `f`, `a`...)? → tra lại họ lệnh theo kiểu dữ liệu ở
      phần Deep Dive.

## Source Code Walkthrough

Vòng lặp thực thi chính của Interpreter trong HotSpot (gọi là *bytecode interpreter loop*)
được viết bằng một kỹ thuật gọi là *template interpreter* — sinh sẵn đoạn mã máy nhỏ cho mỗi
loại lệnh bytecode (`iadd`, `iload`...), rồi nhảy (dispatch) tới đúng đoạn mã tương ứng với
lệnh đang đọc. Đây là lý do dù là "thông dịch", Interpreter của HotSpot vẫn nhanh hơn nhiều so
với một trình thông dịch ngây thơ (đọc chuỗi lệnh rồi so sánh if/else từng loại). Chi tiết cơ
chế này thuộc phần rất sâu của HotSpot, không cần thiết ở mức chapter nền tảng này — nhưng
đáng để biết tên gọi khi tra cứu thêm sau này.

## Summary

Execution Engine là "CPU ảo" của JVM, thực thi bytecode bằng cách kết hợp Interpreter (thông
dịch từng lệnh) và JIT Compiler (biên dịch phần hot code — Chapter 06). Bytecode JVM là
**stack-based**: mỗi frame có một operand stack để chứa toán hạng tạm, thay vì dùng thanh ghi
có tên như CPU thật — lựa chọn này giữ cho bytecode độc lập kiến trúc CPU, đúng tinh thần
"Write Once, Run Anywhere" (Chapter 01), đổi lại bằng tốc độ thuần Interpreter chậm hơn, được
bù lại bởi JIT Compiler.

## Interview Questions

**Junior**

- Execution Engine gồm những thành phần nào?
- Operand stack khác biến cục bộ (local variable) như thế nào trong một frame?

**Mid**

- Vì sao bytecode JVM thiết kế dựa trên stack thay vì thanh ghi như CPU thật? Đánh đổi là gì?
- Lệnh `iadd` khác `ladd` ở điểm nào? Vì sao bytecode cần nhiều họ lệnh theo kiểu dữ liệu?

**Senior**

- Trace từng bước operand stack cho một đoạn bytecode bất kỳ bạn tự chọn bằng `javap -c` trên
  một method thật trong project của bạn.
- Giải thích vì sao thiết kế stack-based, dù chậm hơn register-based nếu chạy thuần
  Interpreter, lại không phải là một đánh đổi tồi về tổng thể — liên hệ với vai trò của JIT
  Compiler.

## Exercises

- [ ] Biên dịch và trace bằng tay bytecode của `AddDemo` ở trên — đối chiếu với bảng ở phần
      Visualization, đảm bảo bạn tự làm lại được không cần nhìn đáp án.
- [ ] Viết một method tính `(a + b) * c`, biên dịch, dùng `javap -c` xem bytecode, tự vẽ bảng
      trace operand stack như ví dụ trong chapter.
- [ ] So sánh bytecode của phép cộng `int` (`iadd`) với phép cộng `double` (`dadd`) bằng cách
      viết hai method tương tự nhau chỉ khác kiểu dữ liệu.

## Cheat Sheet

| Tiền tố lệnh | Kiểu dữ liệu |
| --- | --- |
| `i` | int |
| `l` | long |
| `f` | float |
| `d` | double |
| `a` | reference (object) |

| Lệnh mẫu | Ý nghĩa |
| --- | --- |
| `iload_N` | Đẩy biến cục bộ #N (int) vào operand stack |
| `istore_N` | Lấy đỉnh operand stack, lưu vào biến cục bộ #N |
| `iadd` | Pop 2 giá trị, cộng, push kết quả |
| `invokestatic`/`invokevirtual`/`invokespecial` | Gọi method (xem Chapter 01) |
| `ireturn` | Pop đỉnh stack, trả về, kết thúc method |

## References

- Java Virtual Machine Specification (Oracle) — Chapter 2.11: Instruction Set Summary.
- Java Virtual Machine Specification (Oracle) — Chapter 6: The Java Virtual Machine
  Instruction Set.
