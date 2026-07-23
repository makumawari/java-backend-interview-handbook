---
tags:
  - NoSQL
  - MongoDB
  - Database
---

# NoSQL & Non-Relational Data

> Phase: Phase 7 — Database
> Chapter slug: `nosql`

## Metadata

```yaml
Chapter: NoSQL & Non-Relational Data
Phase: Phase 7 — Database
Difficulty: ★★★
Importance: ★★★★
Interview Frequency: 65%
Prerequisites:
  - Chapter 02 — JOIN
  - Chapter 05 — Transaction (Database ACID Transaction)
Used Later:
  - Kiến trúc lựa chọn công nghệ lưu trữ (Phase 8)
Estimated Reading: 18 phút
Estimated Practice: 15 phút
```

## Story

> Xem lại: [Chapter 02](02-join.md) — lấy "tên khách hàng kèm đơn hàng" trong SQL cần `JOIN` hai
> bảng tách biệt (`customers`, `orders`). Với MongoDB (mô hình document), cùng dữ liệu đó được
> lưu **gộp chung** trong một document duy nhất — không cần join gì cả.

```javascript
db.customers.find({name: "Nguyen Van A"}, {name:1, orders:1, _id:0})
```

```json
{
  "name": "Nguyen Van A",
  "orders": [
    { "status": "NEW", "items": [{"product": "Laptop", "qty": 1}, {"product": "Mouse", "qty": 2}] },
    { "status": "SHIPPED", "items": [{"product": "Desk", "qty": 1}] }
  ]
}
```

Một câu truy vấn **duy nhất** trả về đầy đủ khách hàng, đơn hàng, **và** từng sản phẩm trong đơn
hàng — lồng nhau trực tiếp trong một document, không có khái niệm "bảng riêng biệt cần join".

## Interview Question (Central)

> SQL (relational) và NoSQL (document) khác nhau như thế nào về cách mô hình hoá dữ liệu? Khi
> nào nên chọn NoSQL thay vì SQL?

## Objectives

- [ ] Hiểu mô hình document (NoSQL) lưu dữ liệu **lồng nhau, gộp chung** thay vì tách thành
      nhiều bảng chuẩn hoá (normalized) như SQL
- [ ] Tự tay chứng minh bằng thực nghiệm: một truy vấn MongoDB lấy được dữ liệu "customer + đơn
      hàng + sản phẩm" mà không cần join
- [ ] Tự tay chứng minh **schema flexibility**: các document trong cùng một collection có thể có
      cấu trúc field khác nhau, không cần migration

## Prerequisites

- Chapter 02 — hiểu JOIN trong SQL, để đối chiếu trực tiếp với cách NoSQL tránh nhu cầu join.
- Chapter 05 — hiểu ACID, nền tảng để so sánh đảm bảo giao dịch giữa hai mô hình.

## Used Later

- **Kiến trúc lựa chọn công nghệ lưu trữ** (Phase 8) — quyết định SQL hay NoSQL (hoặc kết hợp cả
  hai — polyglot persistence) là một quyết định kiến trúc quan trọng khi thiết kế hệ thống.

## Problem

Mô hình quan hệ (SQL, Phase 6-7) chuẩn hoá dữ liệu thành nhiều bảng để **tránh trùng lặp**
(mỗi thông tin chỉ lưu đúng một nơi) — nhưng đổi lại, đọc dữ liệu "kết hợp" từ nhiều bảng luôn
cần `JOIN` (Chapter 02), và với dữ liệu có cấu trúc **lồng sâu, thay đổi linh hoạt theo từng bản
ghi** (ví dụ mỗi sản phẩm trong catalog có thuộc tính hoàn toàn khác nhau — sách có "số trang",
laptop có "CPU"/"RAM"), việc ép buộc vào một schema bảng cố định trở nên gượng ép, đòi hỏi nhiều
bảng phụ hoặc cột thường xuyên `NULL`.

## Concept

**NoSQL** (không chỉ một công nghệ, mà một nhóm mô hình dữ liệu thay thế mô hình quan hệ) gồm
nhiều loại: **document** (MongoDB — lưu dữ liệu dạng JSON/BSON lồng nhau), **key-value** (Redis),
**column-family** (Cassandra), **graph** (Neo4j). Với mô hình **document**, dữ liệu **liên quan
chặt chẽ, thường được đọc cùng nhau** được **gộp chung** vào một document duy nhất (denormalization — phản chuẩn hoá, chủ động chấp nhận trùng lặp dữ liệu để đổi lấy tốc độ đọc), thay vì tách
thành nhiều bảng chuẩn hoá cần join lại khi đọc.

## Why?

Việc gộp dữ liệu liên quan vào một document phản ánh đúng **mẫu hình truy cập** phổ biến của
nhiều ứng dụng thực tế: phần lớn khi cần thông tin một khách hàng, ứng dụng **cũng cần luôn** đơn
hàng của họ — nếu dữ liệu này luôn được đọc cùng nhau, việc tách thành bảng riêng (SQL) chỉ tạo
thêm chi phí join không cần thiết ở **mọi lần đọc**, đổi lấy lợi ích "không trùng lặp dữ liệu"
— một lợi ích không quan trọng bằng tốc độ đọc trong nhiều tình huống thực tế. Schema linh hoạt
(mỗi document không bắt buộc cùng cấu trúc) phù hợp khi dữ liệu **thực sự đa dạng** giữa các bản
ghi (ví dụ catalog sản phẩm đa dạng loại) — SQL buộc phải định nghĩa trước **mọi** cột có thể có
(dẫn tới nhiều cột `NULL`), trong khi document chỉ lưu **đúng những field bản ghi đó thực sự có**.

## How?

```javascript
// Document long nhau - KHONG can JOIN de lay du lieu lien quan
db.customers.insertOne({
  name: "Nguyen Van A",
  orders: [
    { status: "NEW", items: [{ product: "Laptop", qty: 1 }] }
  ]
});

db.customers.find({ name: "Nguyen Van A" }); // 1 truy van, co DAY DU du lieu long nhau
```

## Visualization

```
SQL (chuan hoa, tach bang):              NoSQL/Document (phan chuan hoa, gop chung):

  customers        orders                  customers (1 document = 1 khach hang + moi thu lien quan)
  ┌──────────┐    ┌───────────┐            ┌────────────────────────────────┐
  │ id, name │    │ id, cust_id│           │ { name: "...",                  │
  └──────────┘    │ status     │           │   orders: [                     │
       │          └───────────┘            │     { status: "...",            │
       │  can JOIN de ket hop  │            │       items: [{product,qty}] } │
       └───────────────────────┘            │   ] }                          │
                                            └────────────────────────────────┘
  Doc: 2 buoc (JOIN)                       Doc: 1 buoc (khong join)
  Uu diem: KHONG trung lap du lieu         Uu diem: doc NHANH, khong can join
```

## Example

**Truy vấn lồng nhau, không cần JOIN:**

```javascript
db.customers.find({name: "Nguyen Van A"}, {name:1, orders:1, _id:0})
```

Kết quả thật (MongoDB 7.0.39):

```json
{
  "name": "Nguyen Van A",
  "orders": [
    { "status": "NEW", "items": [{"product": "Laptop", "qty": 1}, {"product": "Mouse", "qty": 2}] },
    { "status": "SHIPPED", "items": [{"product": "Desk", "qty": 1}] }
  ]
}
```

**Schema linh hoạt — mỗi document có thể khác cấu trúc:**

```javascript
db.customers.insertMany([
  { name: "Nguyen Van A", orders: [...] },
  { name: "Tran Thi B", orders: [...] },
  { name: "Le Van C", orders: [], loyaltyTier: "GOLD" } // CHI document nay co field nay
]);

db.customers.find({}, {name:1, loyaltyTier:1, _id:0});
```

Kết quả thật:

```json
{ "name": "Nguyen Van A" }
{ "name": "Tran Thi B" }
{ "name": "Le Van C", "loyaltyTier": "GOLD" }
```

Chỉ **"Le Van C"** có field `loyaltyTier` — hai document còn lại **không hề có** field này (không
phải `null`, mà hoàn toàn **không tồn tại** trong document đó). Không có `ALTER TABLE` nào được
chạy, không có migration nào cần thiết để thêm field này cho một bản ghi cụ thể — trái ngược
hoàn toàn với SQL, nơi thêm một cột mới (dù chỉ cần cho một số dòng) đòi hỏi `ALTER TABLE` áp
dụng cho **toàn bộ** bảng, và mọi dòng khác sẽ có giá trị `NULL` cho cột đó.

## Deep Dive

**Đánh đổi cụ thể của việc "gộp chung, phản chuẩn hoá" là gì — không phải chỉ có lợi ích?** Chấp
nhận trùng lặp dữ liệu (denormalization) đổi lấy tốc độ đọc, nhưng phải trả giá ở phía **ghi**:
nếu cùng một thông tin xuất hiện ở nhiều document (ví dụ tên sản phẩm được nhúng vào **mọi** đơn
hàng chứa sản phẩm đó), khi thông tin gốc thay đổi (đổi tên sản phẩm), cần **cập nhật tất cả**
các bản sao đó — không có ràng buộc `FOREIGN KEY` nào tự động đảm bảo tính nhất quán (khác với
SQL, nơi sửa một dòng trong bảng gốc tự động phản ánh đúng ở mọi nơi tham chiếu tới, nhờ chuẩn
hoá). Đây chính là đánh đổi cốt lõi giữa hai mô hình: SQL tối ưu cho **tính nhất quán và tránh
trùng lặp** (đánh đổi lấy chi phí join khi đọc), NoSQL document tối ưu cho **tốc độ đọc** (đánh
đổi lấy rủi ro không nhất quán khi dữ liệu trùng lặp không được đồng bộ đúng cách).

## Engineering Insight

**Quyết định "SQL hay NoSQL" trong thực tế nên dựa trên tiêu chí nào, không phải theo trào lưu
công nghệ?** Câu hỏi cốt lõi: **mẫu hình truy cập dữ liệu thực tế là gì?** Nếu ứng dụng thường
xuyên cần truy vấn dữ liệu theo **nhiều chiều khác nhau, linh hoạt** (ví dụ "tìm mọi đơn hàng của
một sản phẩm cụ thể", không chỉ "tìm đơn hàng của một khách hàng") — mô hình quan hệ với khả năng
`JOIN` linh hoạt (Chapter 02) phù hợp hơn nhiều so với document (vốn tối ưu cho **một** hướng
truy cập chính, đã "đóng gói" sẵn trong cấu trúc lồng nhau). Nếu ứng dụng có mẫu hình truy cập
**cố định, có thể dự đoán trước** (luôn đọc "khách hàng kèm đơn hàng của họ" theo cùng một cách),
và cần **tốc độ đọc cực cao** ở quy mô lớn, document model phát huy lợi thế rõ rệt. Trong thực tế,
nhiều hệ thống lớn dùng **polyglot persistence** — kết hợp cả hai (SQL cho dữ liệu giao dịch cốt
lõi cần tính nhất quán cao, NoSQL cho dữ liệu cần đọc nhanh, ít thay đổi cấu trúc) thay vì chọn
tuyệt đối một trong hai.

## Historical Note

Thuật ngữ "NoSQL" (ban đầu là tên một công cụ database cụ thể năm 1998, sau này được cộng đồng
tái sử dụng làm tên gọi chung cho phong trào rộng hơn) trở nên phổ biến từ khoảng 2009, gắn liền
với làn sóng các công ty công nghệ quy mô lớn (Google, Amazon, Facebook) công bố các hệ thống
lưu trữ phân tán riêng (BigTable, Dynamo, Cassandra) để giải quyết bài toán mở rộng quy mô
(scale) mà mô hình quan hệ truyền thống thời điểm đó gặp khó khăn. MongoDB (2009) là một trong
những database document phổ biến nhất ra đời trong làn sóng này.

## Myth vs Reality

- **Myth:** "NoSQL luôn nhanh hơn SQL, nên là lựa chọn mặc định cho ứng dụng hiện đại."
  **Reality:** Xem Engineering Insight — hiệu năng phụ thuộc hoàn toàn vào mẫu hình truy cập dữ
  liệu; với truy vấn cần linh hoạt theo nhiều chiều, SQL với JOIN thường phù hợp hơn.

- **Myth:** "NoSQL không có schema, hoàn toàn tự do, không cần thiết kế cấu trúc dữ liệu trước."
  **Reality:** Đã chứng minh bằng thực nghiệm — schema **linh hoạt hơn** SQL (không bắt buộc
  mọi document giống nhau), nhưng vẫn cần thiết kế cấu trúc hợp lý; "tự do hoàn toàn" dễ dẫn tới
  dữ liệu không nhất quán nếu không có kỷ luật thiết kế.

## Common Mistakes

- **Chọn NoSQL chỉ vì "nghe nói nhanh hơn" mà không phân tích mẫu hình truy cập dữ liệu thực
  tế** — có thể khiến các truy vấn cần linh hoạt trở nên khó khăn hơn hẳn so với SQL.
- **Nhúng (embed) dữ liệu có tần suất thay đổi cao vào nhiều document khác nhau** — gây khó khăn
  đồng bộ khi cần cập nhật, đánh đổi không xứng đáng với lợi ích tốc độ đọc.
- **Thiết kế document quá lồng sâu, quá lớn** — MongoDB có giới hạn kích thước document (16MB),
  và document quá lớn làm chậm cả đọc lẫn ghi.

## Best Practices

- Chọn mô hình dữ liệu dựa trên **mẫu hình truy cập thực tế**, không dựa trên trào lưu công nghệ.
- Nhúng (embed) dữ liệu **ít thay đổi, luôn đọc cùng nhau**; tách riêng (reference, tương tự
  foreign key) dữ liệu **thay đổi thường xuyên** hoặc **cần truy vấn độc lập** theo nhiều chiều.
- Cân nhắc polyglot persistence (kết hợp SQL + NoSQL) cho hệ thống lớn với nhiều loại dữ liệu có
  đặc tính truy cập khác nhau.

## Debug Checklist

- [ ] Truy vấn NoSQL cần lọc theo nhiều chiều khác nhau trở nên phức tạp/chậm? → cân nhắc liệu
      mẫu hình truy cập có thực sự phù hợp với document model, hay nên dùng SQL.
- [ ] Dữ liệu trùng lặp giữa các document không đồng bộ đúng sau khi cập nhật? → kiểm tra chiến
      lược cập nhật có cover đủ mọi bản sao đã nhúng dữ liệu đó không.
- [ ] Document quá lớn, chậm đọc/ghi? → cân nhắc tách bớt dữ liệu ít liên quan trực tiếp ra
      collection riêng, dùng tham chiếu thay vì nhúng toàn bộ.

## Summary

NoSQL (document, MongoDB) lưu dữ liệu liên quan **gộp chung, lồng nhau** trong một document,
khác với SQL chuẩn hoá thành nhiều bảng cần `JOIN` (Chapter 02). Đã chứng minh bằng thực nghiệm:
một truy vấn MongoDB duy nhất trả về đầy đủ khách hàng + đơn hàng + sản phẩm lồng nhau, không cần
join; các document trong cùng collection có thể có cấu trúc field khác nhau hoàn toàn (một
document có `loyaltyTier`, các document khác không hề có field này) mà không cần
`ALTER TABLE`/migration. Đánh đổi cốt lõi: SQL tối ưu tính nhất quán, tránh trùng lặp (đổi lấy
chi phí join); NoSQL document tối ưu tốc độ đọc (đổi lấy rủi ro không nhất quán khi dữ liệu
trùng lặp không đồng bộ đúng). Quyết định chọn mô hình nào nên dựa trên mẫu hình truy cập dữ liệu
thực tế, không theo trào lưu công nghệ — nhiều hệ thống lớn kết hợp cả hai (polyglot
persistence).

## Interview Questions

- SQL (relational) và NoSQL (document) khác nhau như thế nào về cách mô hình hoá dữ liệu?
- Khi nào nên chọn NoSQL thay vì SQL?

**Senior**

- Giải thích đánh đổi cụ thể giữa "chuẩn hoá" (SQL) và "phản chuẩn hoá" (NoSQL document). Rủi ro
  gì khi nhúng dữ liệu trùng lặp vào nhiều document?
- "Polyglot persistence" là gì? Cho ví dụ một hệ thống thực tế nên kết hợp cả SQL và NoSQL.

## Exercises

- [ ] Chạy lại các câu lệnh ở Example trên một MongoDB thật, xác nhận kết quả đúng như mô tả.
- [ ] Mô hình hoá cùng dữ liệu `customers`/`orders`/`order_items` (đã dùng xuyên suốt Phase 7)
      theo cả hai cách: SQL chuẩn hoá và MongoDB document lồng nhau — so sánh số câu truy vấn
      cần thiết cho cùng một yêu cầu nghiệp vụ.
- [ ] Thử cập nhật tên một sản phẩm đã được nhúng vào nhiều đơn hàng khác nhau trong MongoDB,
      quan sát cần bao nhiêu thao tác để đồng bộ toàn bộ bản sao.

## Cheat Sheet

| | SQL (Relational) | NoSQL (Document, ví dụ MongoDB) |
| --- | --- | --- |
| Cấu trúc dữ liệu | Chuẩn hoá, nhiều bảng | Phản chuẩn hoá, gộp chung, lồng nhau |
| Lấy dữ liệu liên quan | Cần `JOIN` | Một truy vấn, không cần join |
| Schema | Cố định, `ALTER TABLE` áp dụng toàn bảng | Linh hoạt, mỗi document có thể khác cấu trúc |
| Tối ưu cho | Tính nhất quán, tránh trùng lặp | Tốc độ đọc theo mẫu hình cố định |
| Rủi ro chính | Chi phí join khi đọc | Đồng bộ dữ liệu trùng lặp khi ghi |

## References

- MongoDB Documentation — Data Modeling.
- Martin Fowler — "NoSQL Distilled" (2012, tổng quan các mô hình NoSQL).
