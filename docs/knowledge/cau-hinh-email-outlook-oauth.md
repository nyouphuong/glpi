---
ngày: 2026-08-17
files: src/Glpi/Mail/SMTP/OauthConfig.php:136, src/Glpi/Mail/SMTP/OauthProvider/Azure.php, front/smtp_oauth2_callback.php
commit: 9ce46b030d
---

# Gửi email qua Microsoft 365 phải dùng OAuth, và phải xin ĐỦ BA thứ chứ không phải một

**Phát hiện**

GLPI 11 có sẵn provider **Azure** cho SMTP OAuth (`src/Glpi/Mail/SMTP/OauthProvider/Azure.php`),
không cần plugin. Chế độ gửi mail khai bằng hằng số `MAIL_SMTPOAUTH = 4`
(`src/autoload/constants.php`), bên cạnh `MAIL_SMTP`, `MAIL_SMTPSSL`, `MAIL_SMTPTLS`.

**Đừng dùng basic auth (user + mật khẩu).** Microsoft đã ngừng hỗ trợ trên Exchange Online.
Cấu hình được thì cũng sẽ chết khi Microsoft siết tiếp.

Ba thứ phải xin IT (thiếu một là tắc):

| # | Xin gì | Ai cấp |
|---|---|---|
| 1 | Hộp thư **có license, đăng nhập tương tác được** (vd `glpi-noreply@...`) | IT / Exchange |
| 2 | App registration Entra ID → Client ID, Client secret, **Tenant ID** | Quản trị tenant |
| 3 | Bật **Authenticated SMTP** cho đúng hộp thư ở mục 1 | Quản trị Exchange |

Tham số app registration:

```
Redirect URI : https://<host-glpi>/front/smtp_oauth2_callback.php
Permission   : SMTP.Send  (Office 365 Exchange Online) — loai DELEGATED
Can admin consent
```

Redirect URI lấy từ `OauthConfig.php:136`, phải khớp **từng ký tự** với ô trong Azure.

Cấu hình phía GLPI (*Cài đặt → Thông báo → Cấu hình gửi email*): chế độ **SMTP+OAuth**,
provider **Azure**, host `smtp.office365.com`, port `587`, điền Client ID / Secret /
Tenant ID, rồi bấm uỷ quyền → đăng nhập bằng chính hộp thư gửi → GLPI lưu refresh token.

**Vì sao khó tìm**

Ba cái bẫy, cái nào cũng làm mất hàng giờ mà thông báo lỗi không gợi ý gì:

1. **Đừng xin shared mailbox.** IT hay cấp shared mailbox cho việc kiểu này vì không tốn
   license. Nhưng shared mailbox **mặc định không đăng nhập tương tác được**, mà provider
   Azure của GLPI lại bắt buộc có bước đó — mã nguồn đặt `'prompt' => 'login'` và xin scope
   `offline_access`, tức luồng *delegated*, phải có người mở trình duyệt đăng nhập một lần.
   Phải là hộp thư có license.

2. **OAuth đúng hết vẫn lỗi nếu chưa bật Authenticated SMTP.** Exchange Online có công tắc
   riêng cho từng hộp thư (`SmtpClientAuthenticationDisabled`), **tắt sẵn theo mặc định toàn
   tenant**. Nhiều người tưởng dùng OAuth thì khỏi cần — không phải.

3. **Tenant ID đừng để `common`.** Mặc định của provider là `common` (dùng cho app chia sẻ
   nhiều tổ chức). Với tenant nội bộ phải điền đúng Directory ID.

Thêm: **refresh token lưu trong DB, mã hoá bằng `config/glpicrypt.key`**. Mất khoá đó thì
phải uỷ quyền lại từ đầu — xem [note sao lưu](backup-va-noi-luu-du-lieu.md).

**Trạng thái hiện tại**

```
notifications_mailing = 0     <- dang TAT
smtp_mode             = 0     <- MAIL_MAIL, chua cau hinh
smtp_oauth_provider   = (rong)
```

Chưa cấu hình gì. Bật thông báo cùng trang cấu hình SMTP.

**Đường tạm nếu xin lâu**

Thủ tục xin ở công ty mẹ có thể vài tuần. GLPI **vẫn chạy được không cần email** — người dùng
tự vào xem phiếu. Nhưng thiếu thông báo là rủi ro thật cho giai đoạn thí điểm: phiếu giao cho
nhóm mà không ai được báo thì nằm im.

Hỏi IT một câu ngắn: *"có SMTP relay nội bộ cho ứng dụng gửi mail không?"* — loại relay lọc
theo IP, không cần xác thực. Nếu có thì dùng chế độ `MAIL_SMTP` trỏ vào đó, chạy được ngay,
khỏi chờ Entra ID.

**Kiểm chứng lại**

```bash
docker exec glpi-db sh -c 'mariadb -uroot -pglpi glpi -N -e \
  "SELECT CONCAT(name,\" = \",COALESCE(value,\"(rong)\")) FROM glpi_configs \
   WHERE context=\"core\" AND (name LIKE \"smtp%\" OR name=\"notifications_mailing\");"'
```

Sau khi cấu hình xong, gửi thử từ trang *Cấu hình gửi email* rồi kiểm tra `files/_log/mail*.log`.

**Ghi thêm — đăng nhập bằng SSO thì KHÁC chuyện này**

Đừng nhầm hai việc. Phần trên là **gửi mail** bằng OAuth. Còn **đăng nhập vào GLPI bằng SSO**
(kiểu Microsoft Entra ID) thì bản này **không hỗ trợ sẵn**: quét mã nguồn ra 0 file OIDC,
0 file SAML; chỉ có `AuthLDAP` (đấu AD kiểu cũ), `AuthMail`, CAS, và cơ chế *external auth*
qua HTTP header. Muốn SSO thật thì phải đặt reverse proxy xác thực phía trước.
Mấy bảng `glpi_oauth*` trong DB là GLPI **làm chủ** OAuth cho API của nó, không phải chiều ngược lại.
