# Hướng dẫn sử dụng GLPI

> Viết cho **bản đang chạy tại `localhost:8080`** (GLPI 11.0.9-dev), số liệu lấy trực tiếp từ
> DB ngày 17/08/2026. Đây là hướng dẫn để bắt đầu dùng hiệu quả, không phải sách tra cứu đầy đủ —
> tài liệu chính thức ở https://glpi-project.org/documentation/

---

## 1. Hai giao diện, đừng nhầm

GLPI có **hai giao diện tách biệt**, quyết định bởi cột `interface` của profile:

| Giao diện | Ai thấy | Làm được gì |
|---|---|---|
| **helpdesk** | Người dùng cuối | Chỉ gửi yêu cầu, xem ticket của mình, đọc FAQ |
| **central** | Nhân viên IT | Toàn bộ: tài sản, ticket, hợp đồng, cấu hình |

Trong bản này chỉ profile **Self-Service** là `helpdesk`, 7 profile còn lại đều `central`.

Nếu người dùng than "tôi đăng nhập mà chẳng thấy gì" — gần như chắc chắn họ đang ở giao diện
helpdesk. Đó là **đúng thiết kế**, không phải lỗi phân quyền.

## 2. Tài khoản mặc định

| Tài khoản | Profile | Dùng để |
|---|---|---|
| `glpi` | Super-Admin | Quản trị toàn hệ thống |
| `tech` | Technician | Kỹ thuật viên xử lý ticket |
| `normal` | Observer | Chỉ xem, không sửa |
| `post-only` | Self-Service | Người dùng cuối gửi yêu cầu |
| `glpi-system` | (không profile) | Tài khoản nội bộ, **đừng đụng** |

> ⚠️ **Mật khẩu mặc định trùng tên tài khoản.** GLPI hiện cảnh báo đỏ ngay trên dashboard cho
> tới khi đổi. Đây là việc đầu tiên phải làm trước khi cho ai truy cập.
> Đổi tại: **Quản trị → Người dùng → chọn user → tab Mật khẩu**.
> Tài khoản không dùng thì vô hiệu hoá thay vì xoá (giữ lịch sử ticket).

## 3. Vòng đời ticket

Chín trạng thái, lấy từ `src/CommonITILObject.php:120-128`:

| Giá trị | Trạng thái | Ý nghĩa |
|---|---|---|
| 1 | Mới (INCOMING) | Vừa tạo, chưa ai nhận |
| 2 | Đang xử lý — đã gán | Đã có người phụ trách |
| 3 | Đang xử lý — đã lên lịch | Có mốc thời gian dự kiến |
| 4 | Chờ (WAITING) | Treo vì chờ bên thứ ba / chờ người dùng |
| 5 | Đã giải quyết (SOLVED) | IT xong việc, **chờ người báo xác nhận** |
| 6 | Đã đóng (CLOSED) | Kết thúc, tính vào thống kê |
| 7/8/10 | Chấp nhận / Theo dõi / Chờ duyệt | Dùng khi bật quy trình phê duyệt |

**Chỗ hay dùng sai nhất: SOLVED ≠ CLOSED.** `SOLVED` là "IT nghĩ đã xong", `CLOSED` là "người
dùng xác nhận xong". Đóng thẳng sang CLOSED thì mất luôn vòng phản hồi và số liệu tỷ lệ
reopen. Nên để GLPI tự đóng sau N ngày kể từ SOLVED nếu không ai phản đối
(**Cài đặt → Chung → Hỗ trợ**).

**Độ ưu tiên không tự gõ.** GLPI tính `priority` từ ma trận `urgency × impact`
(`computePriority()`, `src/CommonITILObject.php:3362`):

- **Urgency** — gấp tới mức nào, do người báo cảm nhận
- **Impact** — ảnh hưởng bao nhiêu người, do IT đánh giá
- **Priority** — máy suy ra, dùng để xếp hàng đợi

Sửa ma trận tại **Cài đặt → Chung → Hỗ trợ**. Để người báo tự chọn priority là hỏng cơ chế —
ai cũng chọn "rất khẩn cấp".

## 4. Entity — cấu trúc tổ chức

Hiện chỉ có **1 entity: `Root entity`**.

Entity là cách GLPI chia tách dữ liệu theo chi nhánh/phòng ban: mỗi entity có tài sản, ticket,
người dùng riêng, và **quyền được cấp theo từng entity**. Con kế thừa cấu hình của cha.

Chỉ tạo thêm entity khi thật sự cần cô lập dữ liệu (ví dụ nhiều công ty con, nhiều chi nhánh
không được thấy dữ liệu của nhau). Một phòng IT phục vụ một công ty thì **để nguyên một entity**
— chia sớm là tự làm khổ mình, vì mọi thứ sau đó đều phải chọn entity.

## 5. Thứ tự dựng hệ thống

Hiện trạng: **0 danh mục ITIL, 0 SLA, 91 quy tắc cài sẵn, 1 ticket test, chưa có tài sản nào.**
Tức hệ thống mới cài xong, chưa cấu hình nghiệp vụ. Thứ tự nên làm:

1. **Đổi mật khẩu mặc định** (mục 2)
2. **Cài đặt → Chung**: tên tổ chức, múi giờ, ngôn ngữ mặc định
3. **Quản trị → Người dùng**: tạo user thật, hoặc đấu LDAP/AD nếu có
4. **Danh mục ITIL** (*Cài đặt → Dropdown → Danh mục ITIL*) — hiện đang **rỗng**. Không có
   danh mục thì ticket không phân loại được, sau này không thống kê nổi. Bắt đầu 5–10 mục thôi,
   thiếu thì thêm; đẻ ra 50 mục ngay từ đầu là không ai chọn đúng.
5. **Nhập tài sản** — thủ công, import CSV, hoặc bật inventory tự động (mục 7)
6. **SLA** (*Cài đặt → Dropdown → SLA*) — hiện **rỗng**. Chỉ làm khi đã có cam kết thời gian
   thật với người dùng, đừng làm cho đẹp
7. **Quy tắc tự động** — đã có sẵn 91 quy tắc; thêm quy tắc gán ticket theo danh mục ở
   *Quản trị → Quy tắc*

## 6. Ba khối chức năng chính

Theo đúng menu bên trái:

**Asset** — quản lý tài sản. Máy tính, màn hình, thiết bị mạng, máy in, điện thoại, phần mềm,
rack, tủ. Mỗi tài sản có vòng đời (trạng thái, vị trí, người dùng, ngày mua, bảo hành). Đây là
nền của mọi thứ khác: gắn ticket vào tài sản thì mới trả lời được "cái máy này hỏng mấy lần rồi".

**Hỗ trợ** — ITIL. Ticket (sự cố + yêu cầu), Vấn đề (problem — nguyên nhân gốc của nhiều sự cố),
Thay đổi (change — sửa đổi có kiểm soát), Kế hoạch, Thống kê.

**Quản lý** — hợp đồng, nhà cung cấp, ngân sách, tài liệu, giấy phép phần mềm. Gắn hợp đồng với
tài sản để biết cái nào còn bảo hành, cái nào sắp hết hạn.

Hai khối còn lại: **Các công cụ** (KB, ghi chú, RSS, báo cáo), **Quản trị / Cài đặt** (người dùng,
nhóm, entity, quy tắc, cấu hình).

## 7. Dùng cho hiệu quả — mấy điểm đáng giá nhất

**Bật inventory tự động thay vì gõ tay.** GLPI 11 có sẵn agent kiểm kê (GLPI Agent) cài lên máy
trạm, tự đẩy cấu hình phần cứng/phần mềm về. Nhập tay vài trăm máy là không bền: dữ liệu sẽ lỗi
thời trong vài tuần.

**Gắn ticket vào tài sản, luôn luôn.** Ticket không gắn tài sản thì chỉ là cái email có số thứ tự.
Gắn rồi mới có lịch sử theo thiết bị, và mới biết nên sửa hay nên thay.

**Knowledge Base trả lời trước khi ticket sinh ra.** Mỗi lần xử lý xong một sự cố lặp lại, viết
1 bài KB rồi bật hiển thị trong FAQ. Người dùng tự tra, ticket giảm thật.

**Dùng Problem cho nguyên nhân gốc.** 20 ticket "mạng chậm" nên gom về 1 Problem. Đóng Problem
thì đóng được cả cụm, và báo cáo mới phản ánh đúng.

**Đừng bật hết tính năng ngay.** GLPI rất rộng (DCIM, dự án, ngân sách, đặt chỗ...). Bật hết từ
đầu là không ai dùng cái nào cho tới nơi. Chạy tốt Asset + Ticket trước đã.

**Dashboard sửa được.** Trang chủ là dashboard sửa được — thêm/bớt/kéo thả card, tạo dashboard
riêng cho từng nhóm. Nút bút chì ở góc phải để vào chế độ sửa.

## 8. Sao lưu

Hệ thống chạy trong Docker, **dữ liệu không nằm trong thư mục source**. Backup đủ gồm 3 phần
(DB + `files/` + `config/`) — lệnh cụ thể đã kiểm chứng trong
[docs/knowledge/backup-va-noi-luu-du-lieu.md](knowledge/backup-va-noi-luu-du-lieu.md).

Riêng `config/glpicrypt.key` mà mất thì mật khẩu LDAP/mailbox lưu trong DB **không giải mã lại
được**, dù có dump đầy đủ.
