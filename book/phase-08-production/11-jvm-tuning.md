---
tags:
  - JVM Tuning
  - Garbage Collector
  - Humongous Allocation
---

# JVM Tuning: Chọn và Cấu hình Garbage Collector

> Phase: Phase 8 — Production
> Chapter slug: `jvm-tuning`

## Metadata

```yaml
Chapter: JVM Tuning
Phase: Phase 8 — Production
Difficulty: ★★★★★
Importance: ★★★★
Interview Frequency: 55%
Prerequisites:
  - Phase 1, Chapter 07 — Garbage Collection
  - Phase 8, Chapter 08 — Docker
Used Later:
  - Chapter 12 — Performance Tuning
Estimated Reading: 15 phút
Estimated Practice: 12 phút
```

## Story

Chạy cùng một chương trình (liên tục cấp phát mảng `byte[512 * 1024]` — 512KB — trong 8 giây, giữ
lại 200MB dữ liệu sống lâu dài) với ba bộ thu gom rác khác nhau, cùng `-Xmx512m -Xms512m`:

```bash
java -Xmx512m -XX:+UseG1GC -Xlog:gc:file=gc-g1.log GcDemo
java -Xmx512m -XX:+UseSerialGC -Xlog:gc:file=gc-serial.log GcDemo
java -Xmx512m -XX:+UseParallelGC -Xlog:gc:file=gc-parallel.log GcDemo
```

Phân tích log GC thật, đếm số lần pause và tổng thời gian:

```
G1GC:       2456 pause, tong 312.0ms, trung binh 0.127ms/pause, pause LON NHAT 0.826ms
SerialGC:    407 pause, tong  48.3ms, trung binh 0.119ms/pause, pause LON NHAT 14.940ms
ParallelGC:  338 pause, tong  92.2ms, trung binh 0.273ms/pause, pause LON NHAT 11.312ms
```

Kết quả gây bất ngờ: G1GC — bộ thu gom được thiết kế để **giảm** pause time — lại có **tổng thời
gian GC cao nhất** (312ms, gấp 6 lần Serial) trong bài test này, dù pause tối đa của nó thấp nhất
(0.826ms so với gần 15ms của Serial). Nguyên nhân: `byte[512 * 1024]` = 524288 byte dữ liệu, cộng
thêm header object (~16 byte) và độ dài mảng, vượt quá **50% kích thước region mặc định của G1**
(1MB) — biến mỗi object này thành **"humongous object"**, kích hoạt một đường xử lý cấp phát tốn
kém riêng của G1, xảy ra **2456 lần** trong bài test.

## Interview Question (Central)

> Có những loại Garbage Collector nào trong JVM hiện đại, và nên chọn loại nào cho ứng dụng
> production? "Humongous Allocation" trong G1GC là gì, và vì sao nó có thể khiến G1 chậm hơn các
> GC đơn giản hơn trong một số tình huống?

## Objectives

- [ ] Tự tay đo và so sánh số lần pause, tổng thời gian GC, và pause lớn nhất của ba GC khác nhau
      trên cùng một workload
- [ ] Hiểu đúng cơ chế "humongous allocation" của G1GC — nguyên nhân chính xác (kích thước object
      so với kích thước region) và hệ quả hiệu năng thật đã đo được
- [ ] Biết cách đọc log GC (`-Xlog:gc`) để chẩn đoán vấn đề GC thực tế, không chỉ dựa vào lý
      thuyết

## Prerequisites

- Phase 1, Chapter 07 — khái niệm Garbage Collection, thế hệ trẻ/già (young/old generation).
- Phase 8, Chapter 08 — JVM container-aware ergonomics, nền tảng để hiểu `-Xmx` tương tác thế nào
  với giới hạn container.

## Used Later

- **Chapter 12 (Performance Tuning)** — GC là một trong những nghi phạm hàng đầu khi điều tra độ
  trễ bất thường ở production.

## Problem

JVM cung cấp nhiều thuật toán Garbage Collector khác nhau, mỗi loại đánh đổi giữa **thông lượng**
(throughput — tổng thời gian ứng dụng chạy thực sự, không bị GC chiếm dụng) và **độ trễ** (latency
— thời gian một lần pause cụ thể kéo dài bao lâu). Chọn sai GC cho đặc thù workload — hoặc dùng
đúng GC nhưng với pattern cấp phát bộ nhớ không phù hợp với thuật toán đó — có thể gây ra độ trễ
hoặc tổng chi phí GC cao hơn hẳn mức cần thiết, như đã thấy ở Story.

## Concept

Ba GC phổ biến trong JDK 17:

- **Serial GC** (`-XX:+UseSerialGC`) — đơn giản nhất, thu gom bằng **một thread duy nhất**, dừng
  toàn bộ ứng dụng (stop-the-world) trong mỗi lần GC. Phù hợp ứng dụng nhỏ, heap nhỏ, ít CPU.
- **Parallel GC** (`-XX:+UseParallelGC`) — dùng **nhiều thread** để thu gom song song, vẫn
  stop-the-world nhưng nhanh hơn nhờ chạy song song. Tối ưu cho **thông lượng**.
- **G1 GC** (`-XX:+UseG1GC`, mặc định từ JDK 9) — chia heap thành nhiều **region** nhỏ, thu gom
  từng phần theo mục tiêu pause time đặt trước (`-XX:MaxGCPauseMillis`), tối ưu cho **độ trễ thấp
  và ổn định**, đặc biệt với heap lớn.

**Humongous object** — trong G1, một object có kích thước lớn hơn **50% kích thước một region**
được cấp phát trực tiếp vào một chuỗi region liền kề dành riêng (không qua vùng cấp phát nhanh
thông thường), và được xử lý theo một đường logic riêng, tốn kém hơn cấp phát bình thường.

## Why?

Vì không có một thuật toán GC nào tối ưu cho **mọi** tình huống — giảm pause time tối đa (G1, ZGC)
luôn đi kèm chi phí phụ (bookkeeping theo region, concurrent marking) mà GC đơn giản (Serial) không
cần tới — JDK cung cấp nhiều lựa chọn để khớp đúng với đặc thù từng ứng dụng. G1 mặc định vì phù
hợp với đa số ứng dụng hiện đại (heap tương đối lớn, cần pause time dự đoán được), nhưng "mặc định
tốt cho đa số" không có nghĩa là "tối ưu cho mọi workload cụ thể" — như bài test ở Story cho thấy,
với đúng pattern cấp phát (nhiều object cỡ trung bình gần ngưỡng humongous), G1 có thể tệ hơn hẳn
Serial về tổng chi phí GC.

## How?

```bash
# Bat log GC chi tiet
java -Xmx512m -Xms512m -XX:+UseG1GC -Xlog:gc:file=gc.log -jar app.jar

# Doi GC
java -Xmx512m -XX:+UseParallelGC -jar app.jar

# Xem kich thuoc region cua G1 dang duoc dung
java -XX:+PrintFlagsFinal -version | grep G1HeapRegionSize
```

```java
// Doan code tao ra humongous allocation neu region size = 1MB:
byte[] data = new byte[512 * 1024]; // 524288 byte + header > 50% cua 1MB region
```

## Visualization

```
G1 Region (kich thuoc mac dinh vi du 1MB = 1048576 byte):

  Nguong humongous = 50% region = 524288 byte

  byte[512*1024] = 524288 byte du lieu + ~16 byte header
                  = ~524304 byte  >  524288 byte  --> HUMONGOUS!

  Object binh thuong:                    Humongous object:
  ┌──────┐                                ┌──────┬──────┬──────┐
  │Region│ <- cap phat nhanh              │Region│Region│Region│ <- can 1+ region LIEN KE
  └──────┘    trong TLAB                  └──────┴──────┴──────┘    rieng, cham hon
```

## Example

**So sánh thật — cùng workload, cùng `-Xmx512m`, ba GC khác nhau:**

| GC | Số lần pause | Tổng thời gian GC | Pause trung bình | Pause lớn nhất |
| --- | --- | --- | --- | --- |
| G1GC | 2456 | 312.0ms | 0.127ms | 0.826ms |
| SerialGC | 407 | 48.3ms | 0.119ms | 14.940ms |
| ParallelGC | 338 | 92.2ms | 0.273ms | 11.312ms |

**Log thật của G1GC — phần lớn pause ghi rõ nguyên nhân "G1 Humongous Allocation":**

```
[0.087s][info][gc] GC(0) Pause Young (Concurrent Start) (G1 Humongous Allocation) 229M->229M(512M) 0.826ms
[0.093s][info][gc] GC(2) Pause Young (Concurrent Start) (G1 Humongous Allocation) 251M->251M(512M) 0.198ms
[0.098s][info][gc] GC(4) Pause Young (Concurrent Start) (G1 Humongous Allocation) 273M->273M(512M) 0.260ms
```

**Log thật của SerialGC — pause "Allocation Failure" thông thường, số lượng ít hơn nhiều nhưng mỗi
lần lâu hơn đáng kể:**

```
[0.035s][info][gc] GC(0) Pause Young (Allocation Failure) 136M->130M(494M) 12.400ms
[0.052s][info][gc] GC(1) Pause Young (Allocation Failure) 266M->200M(494M) 14.940ms
[0.057s][info][gc] GC(2) Pause Young (Allocation Failure) 336M->200M(494M) 3.554ms
```

**Xác nhận kích thước region gây ra ngưỡng humongous:**

```bash
java -Xmx512m -Xms512m -XX:+UseG1GC -XX:+PrintFlagsFinal -version | grep G1HeapRegionSize
```
```
size_t G1HeapRegionSize = 1048576   {product} {ergonomic}
```

`byte[512 * 1024]` = 524288 byte dữ liệu — **chính xác bằng 50%** của 1048576 byte — cộng thêm
header object khiến kích thước thật sự **vượt** ngưỡng 50%, biến object này thành humongous ở
**mọi lần cấp phát**.

## Deep Dive

**Vì sao G1 xử lý humongous object tốn kém hơn hẳn object thông thường, dẫn tới tổng thời gian GC
cao hơn dù mỗi pause riêng lẻ vẫn rất nhanh?** Object thông thường được cấp phát cực nhanh trong
một vùng đệm riêng cho từng thread (TLAB — Thread-Local Allocation Buffer), gần như không cần đồng
bộ hoá. Humongous object **bỏ qua hoàn toàn** con đường này — nó cần tìm một chuỗi region **liền
kề** đủ lớn, cấp phát trực tiếp (thường ngay vào old generation logic dù object có thể "chết" rất
nhanh), và trong nhiều trường hợp **kích hoạt một chu kỳ GC ngay lập tức** để dọn chỗ nếu không đủ
region liền kề trống — chính xác là điều xảy ra ở Story: mỗi lần cấp phát `byte[512*1024]` mới
kích hoạt một "Pause Young (Concurrent Start) (G1 Humongous Allocation)" riêng biệt, giải thích vì
sao số lần pause của G1 (2456) cao hơn Serial (407) tới 6 lần — mỗi humongous object gần như buộc
GC phải can thiệp, thay vì được gom chung vào một đợt dọn dẹp thông thường như Serial/Parallel làm
khi Young Generation đầy.

## Engineering Insight

**Bài học tổng quát nào rút ra được, áp dụng cho việc thiết kế cấu trúc dữ liệu trong ứng dụng
thực tế dùng G1 (GC mặc định của hầu hết ứng dụng Java hiện đại)?** Cần tránh tạo ra các object có
kích thước **dao động quanh ngưỡng 50% region size** một cách có hệ thống — ví dụ buffer đọc file
cố định 512KB/1MB, hay batch xử lý dữ liệu với kích thước cố định gần ranh giới này. Hai hướng
khắc phục thực tế: (1) **tăng kích thước region** bằng `-XX:G1HeapRegionSize` (có thể đặt tường
minh, ví dụ 4MB hoặc 8MB, để object 512KB không còn vượt ngưỡng 50% nữa) — quyết định lại đánh đổi
với số lượng region ít hơn cho cùng heap size; hoặc (2) **tránh cấp phát lặp lại object cỡ lớn cố
định**, dùng buffer pool tái sử dụng thay vì tạo mới liên tục. Bài học rộng hơn: kích thước GC
region **không phải chi tiết implementation vô hình** — nó ảnh hưởng trực tiếp và đo lường được
tới hiệu năng thực tế nếu pattern cấp phát của ứng dụng vô tình "cộng hưởng" xấu với nó.

## Historical Note

G1 (Garbage-First) được giới thiệu thử nghiệm từ JDK 7 (2011), trở thành GC **mặc định** từ JDK 9
(2017, JEP 248), thay thế Parallel GC vốn là mặc định trước đó — phản ánh sự dịch chuyển ưu tiên
của cộng đồng Java từ "tối đa hoá thông lượng" sang "độ trễ thấp, ổn định, dự đoán được", phù hợp
với xu hướng ứng dụng web/microservices cần phản hồi nhanh hơn là xử lý batch thuần tuý. Các GC thế
hệ mới hơn (ZGC từ JDK 15, Shenandoah) đẩy mục tiêu này xa hơn nữa — nhắm tới pause time dưới 1ms
bất kể kích thước heap, bằng cách làm hầu hết công việc GC **đồng thời** (concurrent) với ứng dụng
thay vì dừng hẳn.

## Myth vs Reality

- **Myth:** "G1 luôn là lựa chọn tốt nhất vì nó là GC mặc định và mới hơn Serial/Parallel."
  **Reality:** Đã đo bằng thực nghiệm — với đúng pattern cấp phát (object cỡ gần ngưỡng humongous),
  G1 có tổng thời gian GC cao gấp 6 lần Serial GC trên cùng workload. "Mặc định" và "mới hơn" không
  đồng nghĩa "tối ưu cho mọi trường hợp".
- **Myth:** "Pause time trung bình thấp đồng nghĩa GC đó hiệu quả hơn."
  **Reality:** G1 có pause trung bình thấp nhất (0.127ms) nhưng tổng thời gian GC (312ms) lại cao
  nhất do số lượng pause quá nhiều — tổng chi phí GC quan trọng không kém pause time từng lần.

## Common Mistakes

- **Chọn GC dựa thuần vào lý thuyết/mặc định mà không đo bằng workload thực tế của chính ứng
  dụng** — như đã thấy, kết quả thực tế có thể ngược hẳn kỳ vọng lý thuyết.
- **Cấp phát lặp lại object có kích thước cố định gần ngưỡng 50% region size của G1** — vô tình
  kích hoạt humongous allocation liên tục, như đã tái hiện ở Story.
- **Không bật log GC (`-Xlog:gc`) ở production** — không có dữ liệu thật để chẩn đoán khi hiệu năng
  bất thường, phải đoán mò thay vì phân tích số liệu cụ thể.

## Best Practices

- Luôn bật `-Xlog:gc` (hoặc chi tiết hơn `-Xlog:gc*`) ở production — chi phí ghi log rất nhỏ so
  với giá trị chẩn đoán mang lại khi cần điều tra sự cố.
- Benchmark GC bằng chính workload thực tế của ứng dụng trước khi quyết định, không chỉ dựa vào
  khuyến nghị chung chung.
- Nếu nghi ngờ humongous allocation, kiểm tra `G1HeapRegionSize` và đối chiếu với kích thước các
  object lớn phổ biến trong ứng dụng (buffer, batch, mảng cố định).
- Với ứng dụng cần pause time cực thấp bất kể heap lớn tới đâu, cân nhắc ZGC/Shenandoah thay vì cố
  tinh chỉnh G1 quá mức.

## Debug Checklist

- [ ] Ứng dụng có nhiều GC pause bất thường dù heap chưa đầy? → bật `-Xlog:gc*`, tìm dòng chứa
      "Humongous Allocation" — nếu xuất hiện nhiều, nghi ngờ object cỡ lớn gần ngưỡng 50% region.
- [ ] Tổng thời gian ứng dụng "mất" vào GC cao bất thường? → tính tổng thời gian pause từ log GC,
      so sánh với các GC khác trên cùng workload (như đã làm ở Story) trước khi kết luận.
- [ ] Cần biết kích thước region G1 đang dùng? → `-XX:+PrintFlagsFinal -version | grep
      G1HeapRegionSize`.

## Summary

JVM cung cấp nhiều GC với đánh đổi khác nhau giữa thông lượng và độ trễ — G1 (mặc định từ JDK 9)
tối ưu độ trễ qua cơ chế region, nhưng object lớn hơn 50% kích thước region trở thành "humongous
object", được cấp phát theo đường xử lý tốn kém riêng. Đã đo bằng thực nghiệm: với workload cấp
phát liên tục object 512KB (vừa đủ vượt ngưỡng 50% của region 1MB mặc định), G1 tạo ra 2456 lần
pause với tổng 312ms GC time — cao hơn 6 lần so với Serial GC (407 pause, 48ms) trên cùng
`-Xmx512m` — dù pause tối đa của G1 vẫn thấp hơn nhiều (0.826ms so với 14.9ms). Bài học cốt lõi:
chọn GC và đánh giá hiệu năng GC cần dựa trên đo lường thực tế với workload cụ thể, không chỉ dựa
vào lý thuyết hay việc GC đó là mặc định.

## Interview Questions

**Mid**

- Sự khác biệt giữa Serial GC, Parallel GC, và G1 GC?
- "Humongous Allocation" trong G1GC là gì?

**Senior**

- Vì sao G1GC — được thiết kế để giảm pause time — lại có thể có tổng thời gian GC cao hơn Serial
  GC trong một số workload cụ thể? Cách chẩn đoán và khắc phục?
- Ngưỡng xác định một object là "humongous" trong G1 được tính như thế nào? Điều gì xảy ra khi
  kích thước object dao động quanh ngưỡng này?

## Exercises

- [ ] Chạy lại `GcDemo` với `-XX:G1HeapRegionSize=4m`, xác nhận số lần "Humongous Allocation" giảm
      mạnh hay biến mất hoàn toàn so với region size mặc định 1MB.
- [ ] Đổi kích thước object cấp phát trong `GcDemo` xuống `256 * 1024` (rõ ràng dưới 50% region),
      so sánh lại số lần pause và tổng thời gian GC của G1.
- [ ] Thử ZGC (`-XX:+UseZGC`) trên cùng workload, so sánh pause lớn nhất với ba GC đã đo ở Story.

## Cheat Sheet

| GC | Ưu tiên | Số thread thu gom | Phù hợp |
| --- | --- | --- | --- |
| Serial | Đơn giản, ít overhead | 1 | Ứng dụng nhỏ, heap nhỏ |
| Parallel | Thông lượng | Nhiều (song song) | Batch job, ưu tiên tổng thời gian xử lý |
| G1 (mặc định) | Độ trễ thấp, ổn định | Nhiều (theo region) | Đa số ứng dụng web/production hiện đại |
| ZGC/Shenandoah | Độ trễ cực thấp | Chủ yếu đồng thời | Heap rất lớn, yêu cầu pause < 1ms |

| Điều kiện humongous (G1) | Hệ quả |
| --- | --- |
| Object > 50% kích thước region | Cấp phát qua đường riêng, tốn kém hơn TLAB thông thường |

## References

- OpenJDK — JEP 248 (G1 làm GC mặc định).
- OpenJDK Documentation — G1 Garbage Collector Tuning Guide (Humongous Objects).
- Oracle — HotSpot VM Options (`-Xlog:gc`, `-XX:G1HeapRegionSize`).
