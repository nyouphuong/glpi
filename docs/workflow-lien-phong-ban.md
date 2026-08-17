# Workflow liên phòng ban trong GLPI

> Bám theo bản đang chạy (GLPI 11.0.9-dev). Mọi tính năng nhắc dưới đây **đã kiểm chứng có
> trong DB/code của bản này**, không phải tính năng nghe nói.

---

## Nguyên tắc: ticket KHÔNG phải chỉ của IT

Đây là hiểu nhầm lớn nhất. Trong GLPI, cái quyết định "ai xử lý" là **nhóm được gán**, không
phải "phòng IT". Nên bất kỳ bộ phận nào cũng làm nhà cung cấp dịch vụ được: kho, bảo trì, nhân
sự, hành chính.

Cơ chế định tuyến:

```
Người gửi  →  chọn Form / Danh mục  →  Quy tắc tự gán  →  Nhóm phụ trách  →  Người xử lý
```

## Năm khối xây dựng (đều có sẵn)

| Khối | Bảng trong DB | Dùng để |
|---|---|---|
| **Nhóm** (Group) | `glpi_groups` | "Kho", "Bảo trì", "Nhân sự" — đơn vị nhận việc |
| **Danh mục + Quy tắc** | `glpi_itilcategories`, `glpi_rules` | Tự gán ticket về đúng nhóm |
| **Form** (service catalog) | 12 bảng `glpi_forms_*` | Biểu mẫu cho người không rành hệ thống |
| **Phê duyệt** | `glpi_ticketvalidations`, `glpi_validationsteps` | Cần sếp duyệt trước khi làm |
| **Đặt chỗ / mượn** | `glpi_reservations`, `glpi_reservationitems` | Mượn tài sản theo lịch |

Form tạo ra được **Ticket, Change hoặc Problem** (`FormDestinationTicket/Change/Problem`), và
form set sẵn được người phụ trách (`AssigneeField`) lẫn tài sản liên quan (`AssociatedItemsField`).

---

## Ca của bạn: sản xuất mượn đồ từ kho

Có **3 cách**, khác nhau về độ chặt. Chọn theo mức độ cần kiểm soát:

### Cách 1 — Đặt chỗ thuần (nhẹ nhất)

Dùng khi món đồ dùng chung, mượn trả trong ngày, không cần ai duyệt.

1. Kho đánh dấu thiết bị là **cho phép đặt chỗ** (`Tài sản → chọn thiết bị → tab Đặt chỗ`)
2. Sản xuất vào **Các công cụ → Đặt chỗ**, xem lịch, chọn khung giờ trống
3. Đến hạn thì trả, kho xác nhận

**Được:** thấy lịch, không đụng nhau, không cần thao tác của kho.
**Không được:** không có phê duyệt, không có vết "ai chịu trách nhiệm nếu hỏng".

### Cách 2 — Ticket qua Form (khuyên dùng)

Dùng khi cần vết yêu cầu rõ ràng và kho phải chủ động xử lý.

1. Tạo Form **"Mượn thiết bị"**: mượn gì, số lượng, từ ngày → đến ngày, lý do, dự án nào
2. Form đặt đích là **Ticket**, gán sẵn nhóm **Kho**
3. Sản xuất điền form → ticket tự về hàng đợi của Kho
4. Kho xử lý: chuẩn bị đồ → chuyển trạng thái **Đang xử lý** → giao đồ → **Đã giải quyết**
5. Sản xuất trả đồ → xác nhận → **Đã đóng**

**Được:** có lịch sử, thống kê được ai mượn nhiều, món nào hay mượn, mượn bao lâu.

### Cách 3 — Form + Phê duyệt + Đặt chỗ (chặt nhất)

Dùng cho thiết bị đắt tiền hoặc mượn dài ngày.

1. Sản xuất điền form
2. Ticket sinh ra kèm **yêu cầu phê duyệt** gửi quản đốc sản xuất
3. Duyệt xong ticket mới chuyển sang nhóm Kho (chưa duyệt thì kho không thấy)
4. Kho tạo **đặt chỗ** cho đúng thiết bị theo khoảng thời gian đã duyệt
5. Trả đồ → đóng ticket

**Được:** có chữ ký duyệt, có lịch giữ chỗ, có vết đầy đủ. **Mất:** thêm một bước chờ.

> Bản này hỗ trợ **phê duyệt nhiều bước** (`glpi_validationsteps`), cấu hình được kiểu
> "cần 1 người bất kỳ duyệt" hay "cần tất cả duyệt".

---

## Catalog workflow

Ký hiệu: **T** = Ticket · **F** = Form · **V** = Phê duyệt · **R** = Đặt chỗ · **C** = Change · **P** = Problem · **Prj** = Dự án

### A. Kho & tài sản

| # | Workflow | Ai gửi → ai xử lý | Dùng |
|---|---|---|---|
| 1 | Mượn thiết bị ngắn hạn | Sản xuất → Kho | R |
| 2 | Mượn thiết bị dài ngày / đắt tiền | Sản xuất → Quản đốc → Kho | F+V+R |
| 3 | Cấp phát vật tư tiêu hao | Mọi phòng → Kho | F+T |
| 4 | Báo mất / hỏng thiết bị mượn | Người mượn → Kho | T (gắn tài sản) |
| 5 | Trả thiết bị trước hạn | Người mượn → Kho | T |
| 6 | Kiểm kê định kỳ | Kho → các phòng | T định kỳ |
| 7 | Điều chuyển tài sản giữa phòng | Phòng A → Kho → Phòng B | F+V |

### B. Sản xuất & bảo trì

| # | Workflow | Ai gửi → ai xử lý | Dùng |
|---|---|---|---|
| 8 | Báo hỏng máy móc dây chuyền | Sản xuất → Bảo trì | T (ưu tiên cao) |
| 9 | Bảo trì phòng ngừa theo lịch | Bảo trì tự tạo | T định kỳ |
| 10 | Máy hỏng lặp lại nhiều lần | Bảo trì → phân tích | P (gom nhiều T) |
| 11 | Cải tiến dây chuyền | Sản xuất → Kỹ thuật → BGĐ | F+V+C |
| 12 | Yêu cầu phụ tùng thay thế | Bảo trì → Kho → Mua hàng | F+V |
| 13 | Dừng máy khẩn cấp | Sản xuất → Bảo trì | T ưu tiên cao nhất |

### C. Nhân sự

| # | Workflow | Ai gửi → ai xử lý | Dùng |
|---|---|---|---|
| 14 | Nhân viên mới vào (onboarding) | HR → IT + Kho + Hành chính | F → nhiều T |
| 15 | Nhân viên nghỉ việc (offboarding) | HR → IT + Kho | F → nhiều T (thu hồi tài sản) |
| 16 | Đổi phòng ban / chức danh | HR → IT (đổi quyền) | F+T |
| 17 | Xin cấp thiết bị làm việc | Nhân viên → Trưởng phòng → IT | F+V+T |
| 18 | Đăng ký đào tạo | Nhân viên → HR | F+V |

### D. Hành chính & mua sắm

| # | Workflow | Ai gửi → ai xử lý | Dùng |
|---|---|---|---|
| 19 | Đặt phòng họp / xe công tác | Mọi phòng → Hành chính | R |
| 20 | Đề nghị mua sắm | Phòng ban → Trưởng phòng → Mua hàng | F+V |
| 21 | Yêu cầu sửa chữa cơ sở vật chất | Mọi phòng → Hành chính | T |
| 22 | Gia hạn hợp đồng nhà cung cấp | Hệ thống nhắc → Mua hàng | T từ cảnh báo hợp đồng |
| 23 | Cấp văn phòng phẩm | Mọi phòng → Hành chính | F+T |

### E. IT cổ điển

| # | Workflow | Ai gửi → ai xử lý | Dùng |
|---|---|---|---|
| 24 | Sự cố không dùng được máy | Nhân viên → IT | T |
| 25 | Cấp quyền truy cập hệ thống | Nhân viên → Trưởng phòng → IT | F+V |
| 26 | Nâng cấp/thay đổi hệ thống | IT → BGĐ | C+V |
| 27 | Sự cố lặp lại nhiều nơi | IT → phân tích nguyên nhân | P |
| 28 | Triển khai phần mềm mới | IT → nhiều phòng | Prj |

### F. Cấp quản lý

| # | Workflow | Ai gửi → ai xử lý | Dùng |
|---|---|---|---|
| 29 | Dự án đầu tư thiết bị | BGĐ → nhiều phòng | Prj (gồm nhiều T) |
| 30 | Báo cáo SLA hàng tháng | Tự động | Thống kê |

---

## Cần cấu hình gì trước khi chạy được

Hiện trạng bản này: **1 nhóm, 0 danh mục ITIL, 0 form, 0 SLA**. Tức chưa chạy được workflow nào
ở trên. Thứ tự dựng:

1. **Tạo nhóm** cho từng bộ phận nhận việc: Kho, Bảo trì, Hành chính, HR, IT
   (*Quản trị → Nhóm*). Không có nhóm thì không định tuyến được.
2. **Tạo danh mục ITIL** theo cây 2 tầng: `Kho > Mượn thiết bị`, `Kho > Cấp vật tư`,
   `Bảo trì > Hỏng máy`… Mỗi danh mục gán **nhóm phụ trách mặc định**.
3. **Quy tắc gán tự động** (*Quản trị → Quy tắc → Quy tắc gán ticket*): theo danh mục → nhóm.
4. **Tạo Form** cho những yêu cầu người dùng hay gõ sai. Bắt đầu 3–5 form thôi.
5. **Bật đặt chỗ** cho các thiết bị cho mượn (tab Đặt chỗ trên từng tài sản).
6. **Phê duyệt** chỉ bật ở nơi thật sự cần chữ ký — bật tràn lan là mọi thứ tắc.

## Ba lời khuyên từ thực tế

- **Đừng dựng cả 30 workflow cùng lúc.** Chọn 2–3 cái đau nhất (thường là mượn thiết bị và
  báo hỏng máy), chạy 1 tháng, rồi mở rộng. Dựng hết một lần thì không ai theo.
- **Nhóm phải có người thật và phải được báo.** Ticket rơi vào nhóm không ai xem là mất niềm
  tin ngay lần đầu. Cấu hình thông báo trước khi công bố.
- **Danh mục ít thôi.** 10 danh mục người ta chọn đúng, 50 danh mục người ta chọn bừa và số
  liệu thành rác.
