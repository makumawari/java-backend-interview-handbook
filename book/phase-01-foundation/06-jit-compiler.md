---
tags:
  - Java
  - JVM
  - JIT
  - Performance
  - Foundation
---

# JIT Compiler

> Phase: Phase 1 — Java Foundation
> Chapter slug: `jit-compiler`

## Metadata

```yaml
Chapter: JIT Compiler
Phase: Phase 1 — Java Foundation
Difficulty: ★★★★
Importance: ★★★★★
Interview Frequency: 65%
Prerequisites:
  - Chapter 01 — Một chương trình Java chạy như thế nào?
  - Chapter 05 — Execution Engine
Used Later:
  - Garbage Collection (Chapter 07)
  - Performance Tuning, JVM Tuning (Phase 8)
  - Virtual Thread, CompletableFuture (Phase 3) — hiệu năng thực thi ảnh hưởng thiết kế concurrency
Estimated Reading: 20 phút
Estimated Practice: 20 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Chapter 05](05-execution-engine.md) — Interpreter thực thi bytecode bằng cách
> thông dịch từng lệnh một, mỗi lần chạy lại từ đầu.

Thử nghĩ: nếu một method được gọi **một triệu lần** trong một request xử lý dữ liệu, Interpreter
có thực sự "đọc lại và thông dịch lại" `iload`, `iadd`, `istore`... một triệu lần, y hệt lần
đầu tiên không? Nếu đúng vậy, Java sẽ không bao giờ nhanh được — vì thông dịch luôn chậm hơn
chạy mã máy thật.

Câu trả lời: JVM **không ngốc như vậy**. Nó theo dõi method nào được gọi nhiều, rồi tự động
biên dịch chính method đó thành mã máy thật — trong lúc chương trình vẫn đang chạy. Đây chính
là JIT Compiler — cỗ máy đứng sau việc Java sau hơn 25 năm bị chê "chậm" (Chapter 01) giờ có
hiệu năng cạnh tranh trực tiếp với các ngôn ngữ biên dịch tĩnh trong rất nhiều bài toán thực
tế.

## Interview Question (Central)

> JIT Compiler tăng tốc thực thi Java như thế nào? Vì sao chương trình Java thường chạy chậm
> hơn ở những lần lặp đầu tiên rồi nhanh dần lên?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Giải thích được cơ chế JIT: đếm lời gọi (invocation counter), ngưỡng "hot", biên dịch
      nền
- [ ] Phân biệt được C1 (client compiler, biên dịch nhanh, tối ưu ít) và C2 (server compiler,
      biên dịch chậm, tối ưu sâu) trong **Tiered Compilation**
- [ ] Đo được bằng thực nghiệm sự khác biệt tốc độ giữa chạy có JIT và chạy thuần Interpreter
      (`-Xint`)
- [ ] Hiểu khái niệm **deoptimization** — JIT có thể "hối hận" và quay lại Interpreter
- [ ] Phân biệt JIT Compiler (runtime, HotSpot) với AOT/GraalVM Native Image (build-time) đã
      nhắc ở Chapter 01

## Prerequisites

- Chapter 01 — biết bytecode, biết JIT tồn tại ở mức giới thiệu.
- Chapter 05 — biết Interpreter thực thi bytecode dựa trên operand stack.

## Used Later

- **Garbage Collection** (Chapter 07) — JIT Compiler và GC đều chạy nền, cạnh tranh tài
  nguyên CPU với thread ứng dụng; hiểu JIT giúp hiểu rõ hơn bức tranh "JVM có nhiều việc chạy
  ngầm" trước khi học tiếp GC.
- **Performance Tuning, JVM Tuning** (Phase 8) — các cờ điều chỉnh ngưỡng JIT, xem log biên
  dịch, là công cụ tối ưu hiệu năng production thực tế.
- **Virtual Thread, CompletableFuture** (Phase 3) — thiết kế các mô hình concurrency hiện đại
  của Java giả định có JIT tối ưu code chạy trên từng thread.

## Problem

Interpreter (Chapter 05) linh hoạt và khởi động nhanh (không tốn thời gian biên dịch trước),
nhưng trả giá bằng việc thông dịch lại từng lệnh **mỗi lần** thực thi — dù method đó được gọi
bao nhiêu lần. Ngược lại, biên dịch **toàn bộ** chương trình sang mã máy trước khi chạy (như
`gcc`) sẽ mất thời gian khởi động lâu, và tệ hơn: biên dịch tĩnh không biết phần code nào thực
sự đáng để tối ưu sâu (dành nhiều thời gian biên dịch) vì nó không thấy được hành vi runtime
thật.

## Concept

JIT (**Just-In-Time**) Compiler là một phần của Execution Engine, chạy **song song** với
Interpreter, không thay thế nó:

1. Chương trình bắt đầu chạy bằng Interpreter (khởi động nhanh).
2. JVM đếm số lần mỗi method được gọi (**invocation counter**) và số lần mỗi vòng lặp quay
   lại (**back-edge counter**).
3. Khi một method/vòng lặp vượt ngưỡng ("hot code"), JIT Compiler biên dịch nó thành mã máy
   thật, **ngay trong lúc chương trình đang chạy** (đây là ý nghĩa "Just-In-Time").
4. Những lần gọi tiếp theo dùng thẳng mã máy đã biên dịch — nhanh hơn thông dịch rất nhiều.

## Why?

Nếu không có JIT: Java mãi mãi chậm hơn C ở mọi tác vụ tính toán nặng, vì luôn phải trả giá
thông dịch. Nếu biên dịch tĩnh toàn bộ (như C): mất khả năng tối ưu dựa trên hành vi runtime
thật (ví dụ: biết chính xác nhánh `if` nào luôn đúng trong lần chạy này, biết chính xác kiểu
dữ liệu thực tế truyền vào một interface để **inline** cuộc gọi ảo). JIT dung hòa cả hai: khởi
động nhanh như thông dịch, đạt tốc độ gần native cho phần code chạy nhiều — chính là
Engineering Insight đã nêu ở Chapter 01, chapter này đi sâu vào **cách** JIT làm được điều đó.

## How?

HotSpot dùng **Tiered Compilation** (biên dịch phân tầng), phối hợp hai compiler:

- **C1 (Client Compiler)** — biên dịch nhanh, tối ưu ở mức vừa phải. Dùng cho code vừa mới
  "nóng lên", cần có mã máy ngay để tăng tốc sớm mà không tốn quá nhiều thời gian biên dịch.
- **C2 (Server Compiler)** — biên dịch chậm hơn nhiều, nhưng tối ưu rất sâu (inline method
  lồng nhau, loại bỏ bounds-check thừa, tối ưu vòng lặp...). Dùng cho code **thực sự** rất
  nóng, đáng để đầu tư thời gian biên dịch kỹ hơn.

```
Bytecode
   │
   ▼
Interpreter chạy ngay (khởi động nhanh, chưa tối ưu)
   │
   │  method/vòng lặp vượt ngưỡng lần 1 (ví dụ ~1500 lần gọi)
   ▼
C1 biên dịch nhanh → mã máy "tạm ổn"
   │
   │  vẫn tiếp tục nóng, vượt ngưỡng lần 2 (ví dụ ~10000 lần gọi)
   ▼
C2 biên dịch kỹ → mã máy tối ưu sâu
```

Nếu một giả định của C2 sai (ví dụ: nó tối ưu dựa trên "interface này luôn chỉ có một class
hiện thực", nhưng sau đó chương trình nạp thêm class hiện thực thứ hai — xem Chapter 03 về nạp
class lazy), JVM thực hiện **deoptimization**: huỷ mã máy đã biên dịch, quay lại chạy bằng
Interpreter cho tới khi đủ dữ liệu để biên dịch lại chính xác hơn.

## Visualization

```
Số lần gọi method
     │
     │              ┌─── C2 (tối ưu sâu, biên dịch chậm)
     │         ┌────┘
     │    ┌────┘  C1 (tối ưu vừa, biên dịch nhanh)
     │────┘
     │  Interpreter (không tối ưu, khởi động ngay)
     └───────────────────────────────────────────▶ Thời gian
     lần gọi 1   ...ngưỡng C1...   ...ngưỡng C2...
```

## Example

So sánh trực tiếp: cùng một đoạn tính toán, chạy có JIT (mặc định) và chạy ép buộc chỉ dùng
Interpreter (`-Xint`, tắt hoàn toàn JIT):

```java
public class JitWarmup2 {
    static long compute(long x) {
        long r = x;
        r ^= (r << 13);
        r ^= (r >>> 7);
        r ^= (r << 17);
        return r;
    }

    public static void main(String[] args) {
        long chunk = 50_000_000L;
        for (int round = 1; round <= 6; round++) {
            long start = System.nanoTime();
            long acc = 0;
            for (long i = 0; i < chunk; i++) {
                acc += compute(acc + i);
            }
            long elapsedMs = (System.nanoTime() - start) / 1_000_000;
            System.out.println("Chunk " + round + " (" + chunk + " lần gọi): "
                + elapsedMs + " ms  [acc=" + acc + "]");
        }
    }
}
```

Chạy bình thường (có JIT, JDK 17):

```
Chunk 1 (50000000 lần gọi): 94 ms   [acc=...]
Chunk 2 (50000000 lần gọi): 135 ms  [acc=...]
Chunk 3 (50000000 lần gọi): 134 ms  [acc=...]
...
```

Chạy với `java -Xint JitWarmup2` (ép chỉ dùng Interpreter, JIT bị tắt hoàn toàn):

```
Chunk 1 (50000000 lần gọi): 964 ms  [acc=...]
Chunk 2 (50000000 lần gọi): 964 ms  [acc=...]
Chunk 3 (50000000 lần gọi): 966 ms  [acc=...]
...
```

Chênh lệch khoảng **7 lần** — bằng chứng thực nghiệm rõ ràng cho giá trị của JIT. Chú ý: các
chunk khi **có** JIT không giảm dần đều đặn (chunk 1 còn nhanh hơn chunk 2 trong lần chạy
thật ở trên) — biên dịch nền chạy trên thread riêng, cạnh tranh CPU với thread chính, nên
đường cong "nóng dần" trong thực tế không mượt như hình vẽ lý thuyết ở phần Visualization.
Đây là điểm khác giữa lý thuyết sách vở và quan sát thực nghiệm — đáng nhớ khi bạn tự đo hiệu
năng thật.

## Deep Dive

**Vì sao ví dụ trên không thấy rõ "chunk sau nhanh hơn chunk trước" mà nhảy thẳng lên tốc độ
cao ngay từ chunk 1?** Vì đoạn code `compute()` rất đơn giản (vài phép XOR/shift) và
50 triệu lần gọi là quá đủ để vượt ngưỡng C1 chỉ trong một phần nhỏ của chunk đầu tiên — quá
trình "nóng lên" diễn ra **trong lòng** chunk 1, không phải giữa các chunk. Với method phức
tạp hơn (nhiều nhánh, gọi lồng nhiều tầng), quá trình biên dịch C1 → C2 kéo dài hơn và dễ quan
sát dạng "chậm dần nhanh lên" rõ hơn qua nhiều lần gọi.

## Engineering Insight

**JIT biết được điều mà `gcc` không bao giờ biết: hành vi runtime THẬT.** Ví dụ kinh điển:
**speculative inlining**. Nếu một interface `Payment` (xem Chapter 08) chỉ có một class hiện
thực (`CreditCardPayment`) thực tế xuất hiện trong suốt quá trình chạy, JIT có thể "đánh cược"
inline thẳng code của `CreditCardPayment.process()` vào chỗ gọi `payment.process()`, bỏ qua
hoàn toàn chi phí gọi hàm ảo (virtual dispatch). Một compiler tĩnh như `gcc` không thể làm
điều này an toàn vì nó không biết chắc lúc runtime có bao nhiêu class hiện thực `Payment`
xuất hiện. Nếu về sau chương trình bất ngờ nạp thêm `MomoPayment` (Chapter 03 — nạp class
lazy), giả định của JIT sai — nó **deoptimize**, quay lại gọi ảo bình thường, rồi biên dịch
lại phiên bản tổng quát hơn. Đây là một minh chứng rất cụ thể cho Myth vs Reality: JIT không
chỉ "biên dịch nhanh hơn", nó biên dịch **thông minh hơn** compiler tĩnh trong nhiều trường
hợp, nhờ nhìn thấy dữ liệu thật.

## Historical Note

```
Java 1.0 (1996)
    ↓  Thuần Interpreter, không JIT — nguồn gốc định kiến "Java chậm"
Java 1.3 / HotSpot (2000)
    ↓  JIT ra đời, nhưng phải CHỌN một trong hai: -client (nhanh khởi động,
       ít tối ưu) hoặc -server (khởi động chậm hơn, tối ưu sâu hơn)
JDK 7 update 4 (2012)
    ↓  Tiered Compilation ra đời — kết hợp cả C1 và C2 tự động,
       không còn phải chọn thủ công giữa hai chế độ
Hiện tại (mặc định từ lâu)
    ↓  Tiered Compilation luôn bật sẵn, HotSpot tự quyết định khi nào
       dùng C1, khi nào leo lên C2
```

> GraalVM Native Image (đã nhắc ở Chapter 01) đi theo hướng khác hẳn: biên dịch AOT
> (Ahead-Of-Time) **trước khi chạy**, không có JIT lúc runtime — đánh đổi lấy cold-start gần
> như tức thì, hy sinh khả năng tối ưu dựa trên hành vi runtime thật mà JIT có được. Đừng
> nhầm GraalVM-làm-JIT (Graal Compiler, một lựa chọn JIT thay HotSpot C2, vẫn chạy runtime)
> với GraalVM Native Image (AOT, không JIT) — đây là hai chế độ khác nhau của cùng một dự án.

## Myth vs Reality

- **Myth:** "Java chậm vì là ngôn ngữ thông dịch."
  **Reality:** Đã bàn ở Chapter 01 — với JIT, phần "hot code" chạy gần bằng tốc độ native.
  Số liệu thực nghiệm ở Example (chênh lệch ~7 lần giữa có/không JIT) là bằng chứng trực
  tiếp.

- **Myth:** "JIT biên dịch một lần rồi xong, giống hệt biên dịch tĩnh nhưng làm lúc runtime."
  **Reality:** JIT có thể **deoptimize** — huỷ bỏ mã đã biên dịch và quay lại Interpreter nếu
  giả định tối ưu (như speculative inlining) không còn đúng nữa. Đây là quá trình động, liên
  tục, không phải "biên dịch một lần cho xong".

## Common Mistakes

- **Benchmark sai cách:** đo thời gian chạy chỉ **một lần duy nhất** một đoạn code rồi kết
  luận về hiệu năng — trong khi lần đầu luôn bị ảnh hưởng nặng bởi chi phí "chưa JIT". Cần đo
  nhiều vòng lặp (warmup) trước khi lấy số liệu thật, hoặc dùng công cụ chuyên benchmark như
  JMH (Java Microbenchmark Harness) thay vì tự đo bằng `System.nanoTime()` một cách ngây thơ
  như ví dụ minh hoạ trong chapter này.
- **Tưởng tăng `-Xmx`/`-Xss` (Chapter 04) sẽ giúp JIT "nóng" nhanh hơn.** JIT không liên quan
  trực tiếp tới kích thước Heap/Stack — nó liên quan tới **tần suất gọi**, không phải bộ nhớ.

## Best Practices

- Khi đo hiệu năng, luôn warmup đủ (chạy nhiều lần trước khi bấm giờ) hoặc dùng JMH — đừng
  tin vào con số đo lần chạy đầu tiên.
- Với ứng dụng cần khởi động cực nhanh (serverless, CLI ngắn hạn), cân nhắc GraalVM Native
  Image (Chapter 01) thay vì JVM truyền thống — vì lợi thế JIT chỉ phát huy khi chương trình
  chạy đủ lâu để "nóng".
- Không cố ép buộc/vi mô quản lý JIT bằng tay trong code nghiệp vụ thông thường — để HotSpot
  tự quyết định, chỉ can thiệp (qua cờ JVM) khi có bằng chứng đo lường cụ thể cần tối ưu.

## Production Notes

**Vấn đề:** dịch vụ có độ trễ (latency) p99 rất cao ngay sau khi vừa deploy/restart, rồi ổn
định dần sau vài phút.

- **Triệu chứng:** ngay sau khi pod/instance mới khởi động, một số request đầu tiên chậm hơn
  hẳn so với vài phút sau đó, dù tải như nhau.
- **Root cause:** đúng hiện tượng "JIT chưa kịp nóng" — những request đầu tiên chạy phần lớn
  bằng Interpreter/C1 (Chapter này), chưa được C2 tối ưu sâu.
- **Debug:** so sánh log độ trễ theo thời gian từ lúc instance khởi động; nếu độ trễ giảm dần
  đều trong vài phút đầu rồi ổn định, đây là dấu hiệu đặc trưng của JIT warmup, không phải
  bug logic.
- **Solution:** thực hiện "warmup" chủ động trước khi instance nhận traffic thật (gửi một số
  request giả lập ngay sau khi khởi động, trước khi đăng ký vào load balancer).
- **Prevention:** đưa bước warmup vào quy trình deploy chuẩn (readiness probe chỉ chuyển
  sang "ready" sau khi đã chạy đủ warmup); với hệ thống scale-out/scale-in thường xuyên
  (autoscaling), cân nhắc GraalVM Native Image nếu độ trễ cold-start là vấn đề nghiêm trọng.

## Debug Checklist

- [ ] Độ trễ cao bất thường chỉ ở những phút đầu sau deploy, rồi tự ổn định? → nghi ngờ JIT
      warmup, không phải bug.
- [ ] Benchmark cho kết quả không ổn định giữa các lần chạy? → kiểm tra đã warmup đủ chưa,
      cân nhắc dùng JMH thay vì đo tay.
- [ ] Nghi ngờ một method không được JIT tối ưu dù chạy rất nhiều? → có thể xem log biên dịch
      bằng `-XX:+PrintCompilation` để xác nhận method đó có được C1/C2 biên dịch không.

## Source Code Walkthrough

Ở mức chapter nền tảng này, không cần đọc source C++ của C1/C2 trong HotSpot (đây là một
trong những phần phức tạp nhất của toàn bộ JVM). Công cụ thực tế để "nhìn thấy" JIT hoạt động
mà không cần đọc source:

```
-XX:+PrintCompilation   → in ra mỗi khi một method được C1/C2 biên dịch
-XX:+PrintInlining      → in ra quyết định inline method của JIT
```

Việc dùng hai cờ trên để quan sát hành vi JIT thật sẽ hữu ích hơn nhiều so với đọc source ở
giai đoạn học tập này — để dành cho Phase 8 (Performance Tuning) khi có đủ ngữ cảnh production
thật để phân tích.

## Summary

JIT Compiler biên dịch bytecode "hot" (chạy nhiều lần) thành mã máy thật ngay trong lúc
chương trình đang chạy, phối hợp với Interpreter qua **Tiered Compilation** (C1 biên dịch
nhanh/tối ưu vừa, C2 biên dịch chậm/tối ưu sâu). Khác với compiler tĩnh, JIT thấy được hành vi
runtime thật nên có thể tối ưu táo bạo hơn (như speculative inlining), và tự **deoptimize**
quay lại Interpreter nếu giả định sai. Đây là lý do chương trình Java thường chậm ở những lần
gọi đầu (chưa JIT) rồi nhanh dần — hiện tượng "JIT warmup" có ảnh hưởng thực tế tới độ trễ
production ngay sau khi deploy/restart.

## Interview Questions

**Junior**

- JIT Compiler là gì? Nó khác Interpreter ở điểm nào?
- Vì sao chương trình Java thường chạy chậm hơn ở lần lặp đầu tiên?

**Mid**

- C1 và C2 khác nhau như thế nào trong Tiered Compilation?
- Deoptimization là gì? Cho một ví dụ tình huống JIT phải deoptimize.

**Senior**

- Giải thích speculative inlining — vì sao JIT làm được điều này mà một compiler tĩnh như
  `gcc` không làm được (an toàn)?
- Trong một hệ thống dùng autoscaling, scale-out liên tục tạo instance mới, JIT warmup ảnh
  hưởng thế nào tới độ trễ p99? Bạn sẽ thiết kế quy trình deploy/readiness-check ra sao để
  giảm thiểu ảnh hưởng đó?

## Exercises

- [ ] Chạy lại đúng ví dụ `JitWarmup2` ở trên, cả chế độ mặc định lẫn `-Xint`, tự đo trên máy
      bạn, xác nhận có chênh lệch tương tự.
- [ ] Chạy chương trình bất kỳ với `-XX:+PrintCompilation`, quan sát danh sách method được
      biên dịch — tìm xem có method nào của chính bạn xuất hiện không.
- [ ] Tìm hiểu và thử chạy cùng một benchmark đơn giản bằng JMH (Java Microbenchmark
      Harness) thay vì tự đo bằng `System.nanoTime()` — so sánh độ tin cậy của hai cách đo.

## Cheat Sheet

| Khái niệm | Ý nghĩa |
| --- | --- |
| Invocation counter | Đếm số lần một method được gọi |
| Back-edge counter | Đếm số lần một vòng lặp quay lại |
| C1 (Client Compiler) | Biên dịch nhanh, tối ưu vừa phải |
| C2 (Server Compiler) | Biên dịch chậm, tối ưu sâu |
| Tiered Compilation | Kết hợp Interpreter → C1 → C2 theo mức độ "nóng" |
| Deoptimization | Huỷ mã đã biên dịch, quay lại Interpreter khi giả định sai |
| `-Xint` | Ép chỉ dùng Interpreter, tắt JIT hoàn toàn (chỉ để thử nghiệm/debug) |
| `-XX:+PrintCompilation` | In log mỗi khi một method được JIT biên dịch |

## References

- Java Virtual Machine Specification (Oracle) — phần liên quan tới thực thi (không quy định
  chi tiết JIT — JIT là chi tiết hiện thực của từng JVM, không phải yêu cầu bắt buộc của đặc
  tả).
- Oracle: "Java HotSpot VM Options" — tài liệu mô tả các cờ `-XX:...` liên quan tới compiler.
- Aleksey Shipilëv và các bài viết chính thức về JMH (OpenJDK Code Tools).
