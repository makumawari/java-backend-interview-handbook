---
tags:
  - Java
  - NIO
  - Foundation
---

# NIO

> Phase: Phase 1 — Java Foundation
> Chapter slug: `nio`

## Metadata

```yaml
Chapter: NIO
Phase: Phase 1 — Java Foundation
Difficulty: ★★★
Importance: ★★★
Interview Frequency: 35%
Prerequisites:
  - Chapter 20 — IO
Used Later:
  - Virtual Thread (Phase 3) — Virtual Thread giải quyết bài toán "nhiều kết nối đồng thời" theo hướng khác hẳn NIO
  - Netty, Reactive Programming (Phase 5) — WebFlux/Netty xây trực tiếp trên nền Channel/Selector ở đây
Estimated Reading: 20 phút
Estimated Practice: 15 phút
```

> Tags: xem front matter ở đầu file (spec/part-5-knowledge-graph.md §17) — dùng cho tag
> filter trên site MkDocs, không lặp lại trong block Metadata này.

## Story

> Xem lại: [Chapter 20](20-io.md) — `java.io` dùng mô hình **blocking**: gọi `read()`, thread
> hiện tại **dừng lại chờ** cho tới khi dữ liệu sẵn sàng.

Với việc đọc file trên đĩa, blocking không phải vấn đề lớn — thao tác nhanh, thread chờ không
lâu. Nhưng thử tưởng tượng một server xử lý **10.000 kết nối mạng đồng thời**, mỗi kết nối có
thể im lặng hàng giây (client gõ chậm, mạng chậm). Với mô hình blocking I/O truyền thống, cách
duy nhất để xử lý 10.000 kết nối cùng lúc là tạo **10.000 thread** — mỗi thread block chờ đúng
một kết nối. Nhưng thread không hề rẻ: mỗi thread chiếm một vùng Stack riêng (Chapter 04, mặc
định thường 512KB-1MB) — 10.000 thread có thể ngốn hàng GB bộ nhớ chỉ để... chờ đợi trong im
lặng.

Đây chính là bài toán mà NIO (New I/O, Java 1.4) ra đời để giải quyết: làm sao **một thread
duy nhất** có thể theo dõi hàng nghìn kết nối cùng lúc, không cần một thread riêng cho mỗi kết
nối.

## Interview Question (Central)

> NIO giải quyết vấn đề gì mà IO truyền thống (`java.io`) không giải quyết được? Buffer và
> Channel trong NIO đóng vai trò gì?

## Objectives

Sau chapter này bạn sẽ:

- [ ] Giải thích được vấn đề "một thread một kết nối" của blocking I/O truyền thống không mở
      rộng (scale) được khi số kết nối lớn
- [ ] Phân biệt `Buffer`/`Channel` (NIO cũ, Java 1.4) với `Path`/`Files` (NIO.2, Java 7)
- [ ] Dùng thành thạo `Files`/`Path` cho các tác vụ file thông thường — API hiện đại hơn hẳn
      `File` truyền thống
- [ ] Hiểu ở mức khái niệm cơ chế non-blocking qua `Selector` (không cần tự viết server NIO
      hoàn chỉnh ở chapter nền tảng này)

## Prerequisites

- Chapter 20 — hiểu mô hình blocking I/O và giới hạn của nó.

## Used Later

- **Virtual Thread** (Phase 3) — giải quyết đúng bài toán "nhiều kết nối đồng thời" nhưng theo
  hướng hoàn toàn khác NIO (làm cho blocking I/O trở nên "rẻ" thay vì tránh blocking) — chapter
  đó sẽ so sánh trực tiếp hai triết lý.
- **Netty, Reactive Programming** (Phase 5) — WebFlux (Spring) và Netty (nền tảng nhiều
  framework hiệu năng cao) xây trực tiếp trên `Channel`/`Selector` của NIO — hiểu nền tảng ở
  đây giúp đọc hiểu code Reactive dễ dàng hơn nhiều.

## Problem

Với `java.io`, số lượng kết nối đồng thời server xử lý được bị giới hạn trực tiếp bởi số
thread có thể tạo — mà số thread lại bị giới hạn bởi bộ nhớ (Chapter 04) và chi phí chuyển
ngữ cảnh (context switch) của hệ điều hành. Cần một mô hình cho phép **một thread giám sát
nhiều kết nối cùng lúc**, chỉ thực sự xử lý (đọc/ghi) kết nối nào **đã sẵn sàng** có dữ liệu,
thay vì mỗi kết nối phải "khoá" một thread ngồi chờ.

## Concept

NIO giới thiệu ba khái niệm cốt lõi:

- **Buffer** — một vùng nhớ có cấu trúc (không phải stream tuần tự đơn giản như `java.io`),
  hỗ trợ đọc/ghi hai chiều, có vị trí (`position`) và giới hạn (`limit`) tường minh.
- **Channel** — đại diện cho một kết nối tới nguồn dữ liệu (file, socket mạng...), đọc/ghi
  thông qua `Buffer`, có thể cấu hình **non-blocking**.
- **Selector** — cho phép **một thread** đăng ký theo dõi **nhiều Channel** cùng lúc, chỉ
  được "đánh thức" khi có Channel nào đó thực sự sẵn sàng đọc/ghi — đây chính là cơ chế giải
  quyết bài toán 10.000 kết nối ở Story.

## Why?

Nếu không có `Selector`: dù `Channel` hỗ trợ non-blocking, code vẫn phải tự liên tục "hỏi"
(poll) từng Channel xem đã sẵn sàng chưa — lãng phí CPU nếu có hàng nghìn Channel. `Selector`
uỷ quyền việc "theo dõi hàng nghìn Channel cùng lúc" xuống tận hệ điều hành (dựa trên cơ chế
gốc như `epoll` trên Linux, `kqueue` trên macOS/BSD) — cực kỳ hiệu quả, một thread Java có thể
giám sát hàng chục nghìn kết nối mà gần như không tốn CPU khi chúng đang im lặng.

## How?

**NIO.2 (Java 7)** — bổ sung `Path`/`Files`, thay thế `File` cũ, dùng cho tác vụ file thông
thường (không cần non-blocking):

```java
Path path = Path.of("product.txt");
Files.writeString(path, "Ao thun,199000");
List<String> lines = Files.readAllLines(path);
long size = Files.size(path);
```

**NIO gốc (Java 1.4)** — `Buffer`/`Channel`, dùng khi cần kiểm soát chi tiết hơn (thường ở
tầng framework, ít khi tự viết trực tiếp trong code nghiệp vụ hàng ngày):

```java
try (FileChannel channel = FileChannel.open(path, StandardOpenOption.READ)) {
    ByteBuffer buffer = ByteBuffer.allocate(64);
    int bytesRead = channel.read(buffer);
    buffer.flip(); // chuyển từ chế độ GHI sang chế độ ĐỌC
    byte[] data = new byte[bytesRead];
    buffer.get(data);
}
```

## Visualization

```
Mô hình blocking I/O (Chapter 20):
  Thread 1 ── block chờ ──▶ Kết nối 1
  Thread 2 ── block chờ ──▶ Kết nối 2
  Thread 3 ── block chờ ──▶ Kết nối 3
  ...
  Thread N ── block chờ ──▶ Kết nối N        (N thread cho N kết nối)

Mô hình NIO với Selector:
                    ┌──────────────┐
  Kết nối 1 ────────┤              │
  Kết nối 2 ────────┤              │
  Kết nối 3 ────────┤   Selector    │◀── MỘT thread duy nhất theo dõi
  ...               │  (1 thread)   │    TẤT CẢ kết nối
  Kết nối N ────────┤              │
                    └──────┬───────┘
                            │  chỉ "đánh thức" khi có Channel
                            │  THỰC SỰ sẵn sàng đọc/ghi
                            ▼
                  Xử lý đúng Channel đó
```

## Example

**NIO.2 — Path/Files, dùng hàng ngày cho tác vụ file:**

```java
import java.nio.file.*;
import java.util.List;

public class NioDemo {
    public static void main(String[] args) throws Exception {
        Path path = Path.of("product-nio.txt");
        Files.write(path, List.of("Ao thun,199000", "Quan jean,399000"));

        List<String> lines = Files.readAllLines(path);
        lines.forEach(l -> System.out.println("Doc: " + l));

        System.out.println("Kich thuoc file: " + Files.size(path) + " bytes");
        Files.delete(path);
    }
}
```

Kết quả thật:

```
Doc: Ao thun,199000
Doc: Quan jean,399000
Kich thuoc file: 32 bytes
```

**NIO gốc — Channel/Buffer, mức thấp hơn:**

```java
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;
import java.nio.file.*;
import java.nio.charset.StandardCharsets;

public class NioDemo2 {
    public static void main(String[] args) throws Exception {
        Path path = Path.of("nio-channel-demo.txt");
        Files.writeString(path, "Hello NIO Channel");

        try (FileChannel channel = FileChannel.open(path, StandardOpenOption.READ)) {
            ByteBuffer buffer = ByteBuffer.allocate(64);
            int bytesRead = channel.read(buffer);
            buffer.flip();
            byte[] data = new byte[bytesRead];
            buffer.get(data);
            System.out.println("Doc qua Channel+Buffer: "
                + new String(data, StandardCharsets.UTF_8));
        }
        Files.delete(path);
    }
}
```

Kết quả thật:

```
Doc qua Channel+Buffer: Hello NIO Channel
```

## Deep Dive

**`buffer.flip()` — vì sao bắt buộc phải gọi trước khi đọc?** `ByteBuffer` có hai chế độ dùng
chung một vùng nhớ: **ghi** (nạp dữ liệu vào, `position` tăng dần theo dữ liệu đã ghi) và
**đọc** (lấy dữ liệu ra, đọc từ đầu tới `limit`). `channel.read(buffer)` ở Example **ghi** dữ
liệu vào buffer — sau lời gọi đó, `position` đang ở cuối phần vừa ghi. Nếu đọc ngay lúc này,
chương trình sẽ đọc từ vị trí sai (cuối buffer, nơi chưa có gì). `flip()` làm đúng một việc:
đặt `limit = position` hiện tại (đánh dấu "đây là điểm kết thúc dữ liệu thật"), rồi
`position = 0` (quay về đầu) — chuyển buffer từ "vừa ghi xong" sang "sẵn sàng đọc từ đầu". Quên
gọi `flip()` là lỗi cực kỳ phổ biến với người mới học NIO, gây đọc ra dữ liệu rỗng hoặc sai.

## Engineering Insight

**Vì sao `Selector` hiệu quả tới mức xử lý được hàng chục nghìn kết nối trên một thread, dù
nghe có vẻ "không thể"?** Vì `Selector` không tự mình liên tục hỏi (busy-poll) từng Channel —
nó **uỷ quyền việc theo dõi xuống tận hệ điều hành**, thông qua các cơ chế I/O multiplexing
gốc của OS (`epoll` trên Linux, `kqueue` trên macOS/BSD, `IOCP` trên Windows). Những cơ chế
này được chính hệ điều hành tối ưu ở tầng kernel — khi một thread gọi `selector.select()`, nó
thực sự **block** (giống blocking I/O!) nhưng chỉ **một thread duy nhất** block, chờ tín hiệu
"có ít nhất một trong hàng nghìn Channel đã sẵn sàng" từ kernel, thay vì hàng nghìn thread mỗi
cái block chờ đúng một Channel. Đây là lý do NIO không "loại bỏ" blocking hoàn toàn — nó **gom
việc chờ đợi lại**, biến N lần chờ (trên N thread) thành 1 lần chờ (trên 1 thread), một sự đánh
đổi thông minh giữa mô hình lập trình phức tạp hơn hẳn (code non-blocking khó viết, khó debug
hơn blocking rất nhiều) và khả năng mở rộng vượt trội.

## Historical Note

```
Java 1.4 (2002)
    ↓
java.nio ra đời — Buffer, Channel, Selector — giải quyết bài toán
scale số lượng kết nối đồng thời, nhưng API tương đối phức tạp, khó viết
đúng (nhiều dự án chọn dùng qua thư viện bọc như Netty thay vì tự viết)
    ↓
Java 7 (2011)
    ↓
NIO.2 (JSR 203) — Path/Files/WatchService — API hiện đại, dễ dùng hơn
NHIỀU cho các tác vụ file thông thường, dần thay thế java.io.File cũ
    ↓
Java 21 (2023) — Virtual Thread (Phase 3) — hướng giải quyết HOÀN TOÀN
KHÁC cho đúng bài toán "nhiều kết nối đồng thời": thay vì tránh blocking
(NIO), làm cho blocking I/O trở nên cực rẻ bằng thread ảo do JVM quản lý
```

## Myth vs Reality

- **Myth:** "NIO thay thế hoàn toàn `java.io`, nên luôn dùng NIO cho mọi tác vụ file."
  **Reality:** Với tác vụ file thông thường (đọc/ghi file cấu hình, log, xử lý dữ liệu tuần
  tự không cần hiệu năng cực cao), `Files`/`Path` (NIO.2) hay `Buffered...` (`java.io`,
  Chapter 20) đều phù hợp — sự khác biệt hiệu năng không đáng kể cho phần lớn use case. NIO
  gốc (`Channel`/`Selector`) chỉ thực sự cần thiết khi bài toán là **hàng nghìn kết nối mạng
  đồng thời**, không phải cho việc đọc một file cấu hình.

- **Myth:** "NIO loại bỏ hoàn toàn khái niệm blocking."
  **Reality:** Xem Engineering Insight — `selector.select()` vẫn block, chỉ là **gom** hàng
  nghìn lần chờ thành một lần chờ trên một thread, không phải loại bỏ blocking hoàn toàn.

## Common Mistakes

- **Quên `buffer.flip()`** trước khi đọc — lỗi phổ biến nhất với người mới học NIO gốc, xem
  Deep Dive.
- **Dùng `Channel`/`Selector` cho tác vụ file đơn giản** — phức tạp hoá không cần thiết, nên
  dùng `Files`/`Path` (NIO.2) hoặc `java.io` có buffer (Chapter 20) cho hầu hết trường hợp.
- **Tự viết server NIO non-blocking hoàn chỉnh từ đầu** trong dự án thực tế thay vì dùng thư
  viện đã được kiểm chứng kỹ (Netty) — API `Selector` mức thấp rất dễ viết sai (quên xử lý
  read một phần, quên đăng ký lại interest set...), một trong những lý do phần lớn hệ sinh
  thái Java chuyển sang dùng Netty thay vì tự viết trực tiếp trên NIO gốc.

## Best Practices

- Với tác vụ file thông thường trong code nghiệp vụ, ưu tiên `Files`/`Path` (NIO.2) — API
  hiện đại, ít lỗi hơn `File` cũ.
- Không tự viết code `Selector` mức thấp trong dự án thực tế trừ khi có lý do rất đặc thù —
  dùng framework đã kiểm chứng (Netty, hoặc các framework HTTP hiện đại xây trên nó, Phase 5).
- Khi cần xử lý nhiều kết nối đồng thời trong dự án mới (Java 21+), cân nhắc Virtual Thread
  (Phase 3) trước — thường đơn giản hơn nhiều so với lập trình non-blocking trực tiếp, trong
  khi vẫn đạt khả năng mở rộng tương đương cho phần lớn use case thực tế.

## Production Notes

Chapter này thuần nền tảng khái niệm — NIO gốc hiếm khi được viết trực tiếp trong code nghiệp
vụ (thường ẩn sau framework như Netty/Spring WebFlux). Ảnh hưởng production thực tế đáng chú ý
nhất: khi một dịch vụ dùng framework reactive/non-blocking (Phase 5, Phase 9) đột nhiên có độ
trễ cao hoặc "treo", nguyên nhân đôi khi là do vô tình gọi một API **blocking** (ví dụ JDBC
truyền thống, Chapter 20) ngay trong luồng xử lý non-blocking — chặn đứng đúng thread duy nhất
đang phục vụ `Selector` cho hàng nghìn kết nối khác, gây nghẽn toàn bộ hệ thống chỉ vì một chỗ
gọi sai — hiểu rõ mô hình "một thread phục vụ nhiều kết nối" ở chapter này giải thích chính
xác vì sao lỗi này nghiêm trọng hơn nhiều so với blocking một thread bình thường.

## Debug Checklist

- [ ] Đọc buffer NIO ra dữ liệu rỗng/sai? → kiểm tra đã gọi `flip()` trước khi đọc chưa.
- [ ] Dịch vụ dùng framework reactive/non-blocking bị treo/độ trễ tăng vọt bất thường? → nghi
      ngờ có lời gọi blocking (JDBC thường, Thread.sleep...) vô tình chạy trên thread đang
      phục vụ non-blocking I/O — xem Production Notes.
- [ ] Cần đọc/ghi file đơn giản, phân vân dùng API nào? → mặc định chọn `Files`/`Path`
      (NIO.2), chỉ cân nhắc `Channel`/`Selector` khi có nhu cầu non-blocking thực sự rõ ràng.

## Source Code Walkthrough

`Selector.select()` (hiện thực trong HotSpot) cuối cùng gọi xuống implementation riêng theo
từng hệ điều hành — `EPollSelectorImpl` trên Linux (gọi `epoll_wait` của kernel),
`KQueueSelectorImpl` trên macOS/BSD — đây là bằng chứng cụ thể cho Engineering Insight:
Selector không tự sáng chế cơ chế theo dõi hàng nghìn kết nối, nó **uỷ quyền trực tiếp** cho
cơ chế I/O multiplexing đã được hệ điều hành tối ưu sẵn ở tầng kernel, JVM chỉ đóng vai trò
lớp API thống nhất, đa nền tảng bên trên.

## Summary

NIO (Java 1.4) giới thiệu `Buffer`/`Channel`/`Selector` để giải quyết bài toán mà blocking I/O
truyền thống (Chapter 20) không mở rộng được: xử lý hàng nghìn kết nối đồng thời mà không cần
một thread riêng cho mỗi kết nối. `Selector` uỷ quyền việc theo dõi hàng nghìn Channel xuống
cơ chế I/O multiplexing của hệ điều hành (`epoll`/`kqueue`), gom nhiều lần chờ thành một lần
chờ trên một thread. NIO.2 (Java 7, `Path`/`Files`) là một nâng cấp riêng biệt, hiện đại hơn
cho tác vụ file thông thường, không liên quan trực tiếp tới non-blocking. Virtual Thread
(Phase 3) là hướng giải quyết mới, khác triết lý, cho cùng bài toán "nhiều kết nối đồng thời".

## Interview Questions

**Junior**

- NIO khác IO truyền thống ở điểm gì?
- `Files`/`Path` (NIO.2) có liên quan tới non-blocking I/O không?

**Mid**

- Giải thích vai trò của `Buffer`, `Channel`, `Selector` trong NIO.
- Vì sao phải gọi `flip()` trước khi đọc dữ liệu từ một `ByteBuffer` vừa ghi?

**Senior**

- Giải thích cơ chế `Selector` uỷ quyền xuống `epoll`/`kqueue` của hệ điều hành — vì sao điều
  này cho phép một thread theo dõi hiệu quả hàng chục nghìn kết nối?
- Một dịch vụ dùng Spring WebFlux (reactive, non-blocking) bị treo định kỳ khi tải cao. Bạn
  nghi ngờ điều gì đầu tiên, dựa trên hiểu biết về mô hình "một thread phục vụ nhiều kết nối"
  ở chapter này?

## Exercises

- [ ] Chạy lại đúng hai ví dụ `NioDemo` và `NioDemo2` ở trên.
- [ ] Cố tình bỏ dòng `buffer.flip()` trong `NioDemo2`, chạy lại, quan sát và giải thích kết
      quả sai lệch.
- [ ] Tìm hiểu và thử viết một `WatchService` (NIO.2) đơn giản theo dõi một thư mục, in ra mỗi
      khi có file mới được tạo — một use case thực dụng khác của NIO.2 chưa nhắc tới trong
      chapter.

## Cheat Sheet

| Thành phần | Vai trò |
| --- | --- |
| `Buffer` | Vùng nhớ có cấu trúc, đọc/ghi hai chiều (`position`, `limit`) |
| `Channel` | Kết nối tới nguồn dữ liệu, đọc/ghi qua Buffer, hỗ trợ non-blocking |
| `Selector` | Một thread theo dõi nhiều Channel cùng lúc |
| `Path`/`Files` (NIO.2) | API file hiện đại, thay thế `File` cũ |

| Muốn | Dùng |
| --- | --- |
| Đọc/ghi file thông thường | `Files`/`Path` (NIO.2) |
| Kiểm soát buffer mức thấp | `Channel`/`ByteBuffer` (NIO gốc) |
| Xử lý hàng nghìn kết nối mạng đồng thời | `Selector`, hoặc thực tế hơn: dùng Netty/framework reactive |

## References

- Oracle: "The Java Tutorials — Trail: Custom Networking", phần NIO.
- JSR 51: New I/O APIs for the Java Platform.
- JSR 203: More New I/O APIs for the Java Platform ("NIO.2").
- Java SE API Documentation — gói `java.nio`, `java.nio.file`, `java.nio.channels`.
