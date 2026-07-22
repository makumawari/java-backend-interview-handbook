# Specification Changelog

## v1.3 (2026-07-23)

Bật tag filter thật trên site MkDocs (`mkdocs.yml` plugin `tags`).

### Part 5 — Knowledge Graph

- §17 "Chapter Tags": làm rõ Tags phải nằm ở **YAML front matter thật** (khối `---` đầu
  file, trước H1) chứ không chỉ trong code block "Metadata" ở thân bài — MkDocs Material
  chỉ đọc tag từ front matter thật, không đọc được YAML lồng trong fenced code block.

### book/ scaffold + mkdocs.yml

- Mỗi chapter giờ có front matter `---\ntags:\n  - TODO\n---` ở đầu file; dòng `Tags:`
  trong block Metadata thân bài được thay bằng ghi chú trỏ sang front matter (tránh 2
  nguồn sự thật).
- `mkdocs.yml`: bỏ option `tags_file` (đã deprecated ở mkdocs-material 9.7.7), thay bằng
  trang `book/tags.md` chứa directive `<!-- material/tags -->` — cú pháp đúng của phiên
  bản plugin hiện tại.
- Đã verify bằng `mkdocs serve` + browser thật: trang `/tags/` hiển thị đúng danh sách
  139 chapter dưới tag "TODO" (placeholder, sẽ tách nhỏ khi viết tag thật cho từng
  chapter); mở một chapter (HashMap) thấy chip tag ngay trên tiêu đề.

## v1.2 (2026-07-23)

Sau khi so sánh phạm vi roadmap với hai cuốn sách được tham chiếu ở Part 1
(*Modern Java in Action*, *Spring in Action* 6th ed.), phát hiện 3 mảng hoàn toàn vắng mặt
dù cả hai sách đều dành riêng nhiều chapter cho chúng. Thêm vào Part 4:

- Phase 5 Spring: `18 Reactive Programming (Project Reactor, WebFlux, Mono/Flux, RSocket)`
  — tương ứng Part 3 "Reactive Spring" (4 chapter) của Spring in Action.
- Phase 7 Database: `10 NoSQL & Non-Relational Data` — tương ứng ch. "Working with
  nonrelational data" của Spring in Action.
- Phase 8 Production: `07 Messaging (JMS, RabbitMQ, Spring Integration)` — tương ứng ch.
  "Sending messages asynchronously" + "Integrating Spring" của Spring in Action; đặt cạnh
  Kafka (đã có từ v1.0) để so sánh trade-off trực tiếp.

`book/` regenerate lại: 139 chapter (từ 136).

## v1.1 (2026-07-23)

Sửa các vấn đề phát hiện khi scaffold repo `book/` lần đầu từ v1.0.

### Part 2 — Chapter Standard

- Thêm §25 "Hiệu chỉnh độ sâu theo Importance": bảng tiêu chí ở §24 trước đây bắt gần như
  mọi mục là bắt buộc cho **mọi** chapter bất kể độ quan trọng. Giờ mức độ bắt buộc của
  từng mục (Deep Dive, Engineering Insight, Historical Note, Myth vs Reality, Source Code
  Walkthrough...) phụ thuộc Importance ★★★★★/★★★★/★★★/★★-★ của chapter.

### Part 3 — Book-wide Standard

- Thêm §2.1 "Slug (roadmap) vs Title (nội dung)": §2 quy định tên chapter phải là câu hỏi,
  nhưng toàn bộ roadmap ở Part 4 lại toàn danh từ ("Lock", "Bean", "Cache"). Giờ làm rõ:
  roadmap dùng slug ngắn (danh từ), còn H1 thật trong file nội dung mới cần viết dạng câu
  hỏi.

### Part 4 — Roadmap

- Đánh số thứ tự cho toàn bộ Phase 2–10 (trước đây chỉ Phase 1 có số 01–20, các Phase còn
  lại chỉ trình bày dạng flow không số).
- Thêm §3.3 ghi chú tường minh về 3 khái niệm bị trùng tên có chủ đích giữa các Phase:
  **Transaction** (Spring / JPA / Database), **Cache/Caching** (Spring / Architecture), và
  **Lock** (Java-level / Database-level). Mỗi chapher giờ có hậu tố `(Góc nhìn)` phân biệt.
- Thêm 5 chapter mới — lộ ra từ 9 câu hỏi phỏng vấn không map được vào bất kỳ chapter nào
  khi scaffold lần đầu:
  - Phase 1: `08 OOP Fundamentals` (Polymorphism, Encapsulation, Inheritance, Abstraction,
    Overloading vs Overriding, `final`, `super`)
  - Phase 1: `17 Enum` (được nhắc tới như ví dụ độ sâu ở Part 3 §11 và Part 4 §3.1 nhưng
    trước đây chưa từng là chapter thật)
  - Phase 2: `14 Comparable vs Comparator`
  - Phase 3: `14 Callable vs Runnable`
  - Phase 6: `04 Multi-Datasource Configuration`
  - Phase 9: `07 Resilience Patterns` (Circuit Breaker, Retry, Bulkhead, Timeout)

### Part 5 — Knowledge Graph

- Thêm §21 "Quan hệ giữa các cơ chế Mapping": Interview/Production/Debug Mapping (trong
  từng chapter), Cross-Reference Index, Production Playbook và Glossary từng có phạm vi
  chồng chéo không rõ ràng. Giờ mỗi cơ chế có vai trò và mức chi tiết riêng, tránh trôi dạt
  (drift) khi sách phát triển.

### book/ scaffold

- Regenerate lại toàn bộ `book/` theo roadmap v1.1 (135 chapter thay vì 130).
- 9 câu hỏi trước đây ở `book/phase-10-interview/00-unmapped-gap-questions.md` giờ đã có
  chapter đích thực — file này được cập nhật thành lịch sử "đã giải quyết" thay vì gap còn
  tồn đọng.
