---
tags:
  - Docker
  - Container
  - JVM Ergonomics
---

# Docker: Đóng gói ứng dụng Java và JVM trong Container

> Phase: Phase 8 — Production
> Chapter slug: `docker`

## Metadata

```yaml
Chapter: Docker
Phase: Phase 8 — Production
Difficulty: ★★★★
Importance: ★★★★
Interview Frequency: 65%
Prerequisites:
  - Phase 1, Chapter 07 — Garbage Collection
Used Later:
  - Chapter 09 — Kubernetes
  - Chapter 11 — JVM Tuning
Estimated Reading: 15 phút
Estimated Practice: 12 phút
```

## Story

Chạy image ứng dụng Spring Boot trong container với giới hạn bộ nhớ khác nhau, rồi kiểm tra
`MaxHeapSize` thật mà JVM tự chọn:

```bash
docker run --rm --entrypoint java demo-app:multistage -XX:+PrintFlagsFinal -version | grep MaxHeapSize
docker run --rm --memory=512m --entrypoint java demo-app:multistage -XX:+PrintFlagsFinal -version | grep MaxHeapSize
docker run --rm --memory=1g --entrypoint java demo-app:multistage -XX:+PrintFlagsFinal -version | grep MaxHeapSize
```

Kết quả thật (máy host có 16GB RAM, Docker VM được cấp 7.82GB):

```
Khong gioi han:        MaxHeapSize = 2099249152   (~1.955 GB, = 1/4 cua 7.82GB Docker VM)
--memory=512m:          MaxHeapSize = 134217728    (128MB, = 1/4 cua 512MB)
--memory=1g:            MaxHeapSize = 268435456    (256MB, = 1/4 cua 1024MB)
```

JVM (JDK 17) **không** dùng bộ nhớ của máy host (16GB) hay toàn bộ Docker VM (7.82GB) khi có giới
hạn `--memory` — nó đọc đúng **giới hạn cgroup của chính container**, và tự động đặt heap bằng
đúng 1/4 giới hạn đó trong mọi trường hợp thử nghiệm. Đây là bằng chứng trực tiếp, đo được, rằng
JVM hiện đại **nhận biết container** một cách chính xác — một hành vi không phải lúc nào cũng đúng
với các phiên bản Java cũ hơn (xem Deep Dive).

## Interview Question (Central)

> JVM có tự động nhận biết giới hạn bộ nhớ của container Docker hay không? Nếu không cấu hình gì
> thêm, một ứng dụng Java chạy trong container với `--memory=512m` có nguy cơ bị OOM-killed không?

## Objectives

- [ ] Tự tay chứng minh bằng thực nghiệm JVM (JDK 17) tự động điều chỉnh `MaxHeapSize` theo đúng
      giới hạn bộ nhớ cgroup của container, không phải bộ nhớ vật lý của host
- [ ] Viết multi-stage Dockerfile giảm kích thước image thực tế, đo bằng số liệu thật
- [ ] Chạy container với user không phải root, xác nhận bằng lệnh `id` thật bên trong container

## Prerequisites

- Phase 1, Chapter 07 — cơ chế Garbage Collection, nền tảng để hiểu vì sao kích thước heap ảnh
  hưởng trực tiếp tới tần suất/độ trễ GC.

## Used Later

- **Chapter 09 (Kubernetes)** — giới hạn bộ nhớ container (`resources.limits.memory`) là nền tảng
  trực tiếp cho khái niệm này ở tầng Kubernetes.
- **Chapter 11 (JVM Tuning)** — hiểu đúng cách JVM đặt heap mặc định là bước đầu tiên trước khi
  tinh chỉnh thủ công bằng `-Xmx`.

## Problem

Một tiến trình Java chạy trong container thực chất vẫn là một tiến trình bình thường trên **cùng
kernel Linux** với host — container không phải máy ảo, nó chỉ là tiến trình bị giới hạn tài nguyên
bằng cơ chế **cgroup** (control group) của kernel. Nếu JVM không biết đọc giới hạn cgroup này, nó
sẽ tự đặt kích thước heap dựa trên bộ nhớ **của toàn bộ host** (có thể lớn hơn giới hạn container
rất nhiều) — dẫn tới heap được cấp phát vượt quá giới hạn thực tế, và container bị kernel **OOM
Killed** ngay khi JVM cố dùng nhiều bộ nhớ hơn mức cho phép, dù bản thân ứng dụng chưa thực sự
"leak" gì cả.

## Concept

Từ **JDK 10** trở đi (và được backport về JDK 8u191), JVM có tính năng **container-aware
ergonomics**, bật mặc định qua cờ `-XX:+UseContainerSupport`. Khi khởi động, JVM tự đọc giới hạn bộ
nhớ từ cgroup (`/sys/fs/cgroup/memory.max` hoặc tương đương) thay vì gọi API bộ nhớ vật lý của
toàn hệ điều hành. Mặc định, `MaxHeapSize` được đặt bằng **25%** (`MaxRAMPercentage`) giới hạn bộ
nhớ phát hiện được — chính xác như đã đo ở Story.

## Why?

Vì container là cơ chế **giới hạn tài nguyên ở tầng kernel**, không phải cô lập bộ nhớ vật lý,
tiến trình bên trong container vẫn về mặt kỹ thuật "nhìn thấy" tổng bộ nhớ của host nếu chỉ gọi
API bộ nhớ hệ điều hành thông thường. JVM container-aware phải chủ động đọc **file cgroup** (một
cơ chế riêng của Linux, khác hẳn API bộ nhớ tiêu chuẩn) để biết giới hạn **thực sự áp dụng cho nó**
— đây là lý do tính năng này cần được thêm tường minh vào JVM (không tự động "có sẵn" từ đầu), và
vì sao các phiên bản JDK cũ hơn JDK 8u191 hoàn toàn không có khả năng này, dẫn tới OOM Kill phổ
biến khi mới bắt đầu container hoá ứng dụng Java trong giai đoạn 2014-2018.

## How?

```dockerfile
# Multi-stage: stage 1 build, stage 2 chi chua runtime + jar
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn -q dependency:go-offline
COPY src ./src
RUN mvn -q package -DskipTests

FROM eclipse-temurin:17-jre-jammy
WORKDIR /app
RUN groupadd -r appgroup && useradd -r -g appgroup appuser
COPY --from=build /app/target/demo-0.0.1-SNAPSHOT.jar app.jar
USER appuser
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

```bash
docker build -t demo-app:multistage .
docker run --memory=512m -p 8080:8080 demo-app:multistage
```

## Visualization

```
Khong container-aware (JDK cu, hoac tat UseContainerSupport):

  Host 16GB RAM --> JVM doc "tong bo nho he thong" --> MaxHeapSize dua tren 16GB
  Container gioi han --memory=512m --> JVM co dung > 512MB --> KERNEL OOM-KILL container

Co container-aware (JDK 10+, mac dinh bat):

  Container gioi han --memory=512m --> JVM doc cgroup memory.max = 512MB
                                    --> MaxHeapSize = 25% x 512MB = 128MB
                                    --> JVM tu gioi han minh, KHONG vuot qua 512MB
```

## Example

**Bằng chứng JVM đọc đúng giới hạn cgroup, không phải bộ nhớ host — 4 kịch bản thật:**

| Giới hạn container | `MaxHeapSize` thật đo được | Tỉ lệ |
| --- | --- | --- |
| Không giới hạn (Docker VM 7.82GB) | 2099249152 bytes (~1.955 GB) | ~25.0% của 7.82GB |
| `--memory=1g` | 268435456 bytes (256 MB) | 25.0% của 1024MB |
| `--memory=512m` | 134217728 bytes (128 MB) | 25.0% của 512MB |
| `--memory=256m` | 132120576 bytes (~126 MB) | ~49.3% của 256MB |

Ba dòng đầu khớp chính xác tỉ lệ mặc định 25% (`MaxRAMPercentage`). Dòng cuối (`256m`) lại cho tỉ
lệ ~49%, gần gấp đôi — xem giải thích ở Deep Dive.

**Multi-stage build giảm kích thước image thật:**

```bash
docker build -f Dockerfile.naive -t demo-app:naive .      # KHONG multi-stage
docker build -f Dockerfile -t demo-app:multistage .        # CO multi-stage
docker images | grep demo-app
```

```
demo-app:naive         811MB
demo-app:multistage    367MB
```

Image `naive` (build và chạy cùng trong image `maven:3.9-eclipse-temurin-17`, giữ nguyên toàn bộ
JDK + Maven + cache dependency) nặng gấp hơn 2 lần image `multistage` (chỉ giữ JRE + file jar đã
build, loại bỏ hoàn toàn công cụ build khỏi image cuối).

**Container chạy với user không phải root — dữ liệu thật:**

```bash
docker run --rm --entrypoint whoami demo-app:multistage
docker run --rm --entrypoint id demo-app:multistage
```

```
appuser
uid=999(appuser) gid=999(appgroup) groups=999(appgroup)
```

## Deep Dive

**Vì sao giới hạn `256MB` cho tỉ lệ heap ~49% thay vì đúng 25% như các giới hạn khác?** JVM có một
cơ chế bảo vệ bổ sung: nếu tổng bộ nhớ phát hiện được **quá nhỏ**, áp dụng đúng 25%
(`MaxRAMPercentage`) sẽ tạo ra heap quá bé để chạy bất kỳ ứng dụng thực tế nào (ví dụ 25% của
100MB chỉ là 25MB). Để tránh trường hợp này, JVM có tham số `MinRAMPercentage` (mặc định 50%),
được áp dụng khi tổng bộ nhớ phát hiện được nằm dưới một ngưỡng nhỏ (mặc định khoảng 256MB). Dữ
liệu thật ở Example xác nhận đúng: `--memory=256m` nằm ngay ngưỡng này, kết quả ~49% (gần khớp
50%) thay vì 25% — đây không phải sai số đo lường, mà là hành vi ergonomics **có chủ đích**, giúp
container cực nhỏ vẫn có heap đủ dùng thay vì bị "bóp" xuống mức phi thực tế.

## Engineering Insight

**Vì sao chọn build multi-stage với base image runtime là `eclipse-temurin:17-jre-jammy` (JRE)
thay vì tiếp tục dùng chính image `maven:3.9-eclipse-temurin-17` (đầy đủ JDK + Maven) cho cả hai
giai đoạn?** Số liệu thật ở Example (811MB → 367MB) định lượng chính xác lý do: image dùng để
**chạy** ứng dụng production không cần trình biên dịch Java (`javac`, chỉ cần JVM để thực thi
bytecode đã biên dịch sẵn), không cần Maven, không cần cache dependency (`.m2`) hay source code.
Toàn bộ những thứ đó chỉ cần thiết ở **giai đoạn build** — multi-stage build tách hẳn giai đoạn
build (dùng image đầy đủ công cụ) khỏi giai đoạn runtime (chỉ copy đúng một file `.jar` kết quả
sang một base image tối giản hơn). Lợi ích không chỉ là kích thước: image nhỏ hơn cũng đồng nghĩa
**diện tích tấn công bảo mật nhỏ hơn** (ít package/binary hơn = ít lỗ hổng tiềm ẩn hơn) và thời
gian `docker pull` khi triển khai nhanh hơn đáng kể ở quy mô nhiều node.

## Historical Note

Trước JDK 8u191 (phát hành cuối 2018) và JDK 10, JVM hoàn toàn **không nhận biết cgroup** — đây là
giai đoạn đầu của việc container hoá ứng dụng Java trở nên phổ biến, và là nguồn gốc của rất nhiều
sự cố OOM Kill nổi tiếng trong cộng đồng (một tiến trình Java "khoẻ mạnh" theo log ứng dụng nhưng
bị kernel giết đột ngột không rõ lý do). Giải pháp tạm thời thời kỳ đó là cờ thử nghiệm
`-XX:+UnlockExperimentalVMOptions -XX:+UseCgroupMemoryLimitForHeap`. Từ JDK 10 trở đi, khả năng
này được đưa vào chính thức và bật mặc định (`-XX:+UseContainerSupport`), khiến việc "chạy Java
trong container mà không cấu hình `-Xmx` gì cả" trở thành thực hành an toàn và phổ biến như hiện
tại.

## Myth vs Reality

- **Myth:** "JVM không hiểu Docker, nên luôn phải tự set `-Xmx` thủ công khi chạy trong container,
  nếu không sẽ bị OOM Kill."
  **Reality:** Đã chứng minh bằng thực nghiệm — với JDK 10+ (bao gồm JDK 17 dùng trong demo), JVM
  tự động đọc đúng giới hạn cgroup và đặt `MaxHeapSize` hợp lý (25% giới hạn container) mà không
  cần cấu hình gì thêm. Điều này **chỉ đúng với JDK cũ** (trước 8u191).
- **Myth:** "Multi-stage build chỉ giúp Dockerfile gọn hơn về mặt tổ chức, không ảnh hưởng nhiều
  tới kích thước image cuối cùng."
  **Reality:** Đã đo bằng số liệu thật — giảm từ 811MB xuống 367MB, hơn 55%, chỉ bằng cách tách
  giai đoạn build khỏi giai đoạn runtime.

## Common Mistakes

- **Đặt `-Xmx` cố định quá lớn** (ví dụ theo thói quen từ môi trường không container hoá) mà không
  đối chiếu với giới hạn `--memory`/`resources.limits.memory` thực tế của container — vô hiệu hoá
  cơ chế tự bảo vệ mặc định của JVM, dễ gây OOM Kill trở lại.
- **Build image production từ base image đầy đủ JDK + công cụ build** (không multi-stage) — image
  nặng hơn không cần thiết, như đã đo ở Example.
- **Chạy container với user mặc định là root** — vi phạm nguyên tắc least privilege; nếu ứng dụng
  bị khai thác lỗ hổng, tiến trình bên trong container có quyền root ngay từ đầu.

## Best Practices

- Không set `-Xmx` thủ công trừ khi có lý do cụ thể (xem Chapter 11, JVM Tuning) — để JVM
  container-aware tự tính toán hợp lý dựa trên giới hạn container thực tế.
- Luôn dùng multi-stage build cho ứng dụng Java production, tách rõ giai đoạn build (đầy đủ công
  cụ) khỏi giai đoạn runtime (tối giản).
- Luôn tạo và chuyển sang user không phải root (`USER appuser`) trong Dockerfile trước khi
  `ENTRYPOINT` chạy ứng dụng.
- Khi đặt giới hạn `--memory` rất nhỏ (dưới ~256MB), lưu ý JVM chuyển sang `MinRAMPercentage`
  (mặc định 50%) thay vì 25% — tính toán heap dự kiến dựa trên hành vi thực tế này, không giả định
  luôn là 25%.

## Debug Checklist

- [ ] Container Java bị kernel OOM-killed dù log ứng dụng không báo lỗi gì? → kiểm tra phiên bản
      JDK có hỗ trợ `UseContainerSupport` không (JDK 10+, hoặc 8u191+); kiểm tra có đang set
      `-Xmx` thủ công lớn hơn giới hạn container không.
- [ ] Cần xác nhận JVM đang đọc đúng giới hạn container? → chạy
      `java -XX:+PrintFlagsFinal -version | grep MaxHeapSize` bên trong container, đối chiếu với
      giới hạn `--memory`/`resources.limits.memory` đã đặt.
- [ ] Image quá nặng, deploy chậm? → kiểm tra Dockerfile có dùng multi-stage build hay không, có
      đang copy dư thừa công cụ build vào image cuối hay không.

## Summary

JVM hiện đại (JDK 10+, bao gồm JDK 17) có khả năng **container-aware** — tự động đọc giới hạn bộ
nhớ cgroup của container thay vì bộ nhớ vật lý toàn host, và mặc định đặt `MaxHeapSize` bằng 25%
(`MaxRAMPercentage`) giới hạn đó, chuyển sang 50% (`MinRAMPercentage`) khi giới hạn rất nhỏ (~dưới
256MB) — đã chứng minh bằng thực nghiệm với 4 mức giới hạn bộ nhớ khác nhau, kết quả khớp chính xác
với tỉ lệ lý thuyết. Multi-stage build giảm kích thước image thực tế hơn 55% (811MB → 367MB) bằng
cách tách giai đoạn build (cần đầy đủ JDK/Maven) khỏi giai đoạn runtime (chỉ cần JRE + jar). Chạy
container với user không phải root (`USER appuser`, xác nhận bằng `id` thật cho uid=999) là thực
hành bảo mật cơ bản, độc lập với các tối ưu về JVM/kích thước image.

## Interview Questions

**Mid**

- JVM có tự động nhận biết giới hạn bộ nhớ của container Docker hay không?
- Multi-stage build trong Dockerfile giải quyết vấn đề gì?

**Senior**

- Container Java bị OOM-killed dù chưa cấu hình `-Xmx` — nguyên nhân có thể là gì, và cách điều
  tra?
- `MaxRAMPercentage` và `MinRAMPercentage` khác nhau ở điểm nào? Khi nào JVM chuyển từ dùng cái này
  sang cái kia?

## Exercises

- [ ] Chạy `java -XX:+PrintFlagsFinal -version | grep MaxHeapSize` với các giới hạn `--memory`
      khác nhau (128m, 2g, 4g), xác nhận tỉ lệ 25%/50% theo đúng ngưỡng đã mô tả ở Deep Dive.
- [ ] So sánh kích thước image khi dùng base runtime `eclipse-temurin:17-jre-jammy` (Ubuntu) với
      một base tối giản hơn (ví dụ Alpine, nếu tương thích kiến trúc máy).
- [ ] Thử đặt `-Xmx4g` thủ công trong `ENTRYPOINT` rồi chạy với `--memory=512m`, quan sát hành vi
      khi ứng dụng cố cấp phát bộ nhớ vượt giới hạn container.

## Cheat Sheet

| Tham số JVM | Ý nghĩa | Giá trị mặc định |
| --- | --- | --- |
| `-XX:+UseContainerSupport` | Bật nhận biết cgroup (mặc định từ JDK 10+) | Bật |
| `-XX:MaxRAMPercentage` | % bộ nhớ phát hiện được dùng làm `MaxHeapSize` (trường hợp thường) | 25.0 |
| `-XX:MinRAMPercentage` | % dùng khi tổng bộ nhớ phát hiện được rất nhỏ | 50.0 |

| Dockerfile | Kích thước image (demo thật) |
| --- | --- |
| Không multi-stage (build + runtime chung image JDK/Maven) | 811MB |
| Multi-stage (runtime chỉ có JRE + jar) | 367MB |

## References

- OpenJDK — JEP liên quan container awareness (JDK-8146115 và các cải tiến JDK 10+).
- Oracle/Eclipse Temurin Documentation — `-XX:+UseContainerSupport`, `MaxRAMPercentage`.
- Docker Documentation — Multi-stage builds.
