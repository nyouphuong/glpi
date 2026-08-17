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

**Đừng dùng basic auth (user + mật khẩu) khi gửi tới Microsoft 365.** Microsoft đã khai tử
xác thực bằng mật khẩu trên Exchange Online.

Phân biệt cho rõ, hai chuyện hay bị gộp làm một:

| Gửi tới đâu | SMTP + mật khẩu | Ghi chú |
|---|---|---|
| `smtp.office365.com` (Microsoft 365) | **Đã bỏ** | Bắt buộc OAuth |
| Relay nội bộ của công ty | **Vẫn dùng được** | Không phải máy Microsoft nên chính sách đó không áp dụng; relay thường lọc theo IP, không cần mật khẩu |

Tức "SMTP với mật khẩu bị bỏ" chỉ đúng với Microsoft 365. Giao thức SMTP không bị bỏ.

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
GLPI tự ghép nó từ cấu hình `url_base`, nên `url_base` phải đúng trước khi uỷ quyền.

**Phải có TÊN MIỀN NỘI BỘ trước — nhưng không phải mua gì.** Công ty dùng IP nội bộ, không có
tên miền công cộng, vẫn làm được:

- Entra ID **bắt buộc HTTPS** cho redirect URI, chỉ nới cho `localhost`. IP thuần rất có thể
  bị từ chối — đừng đặt cược kế hoạch vào đó.
- Công ty đã chạy AD thì **đã có sẵn vùng DNS nội bộ**. Chỉ cần xin **một bản ghi A**:
  `glpi.<vùng-nội-bộ>` → `192.168.x.x`. Miễn phí, IT thêm một dòng.
- Chứng chỉ HTTPS xin từ **CA nội bộ** (AD Certificate Services). Máy trong domain tin sẵn,
  không hiện cảnh báo. CA công cộng không cấp cho tên nội bộ hay IP riêng được.

**Mẹo tiết kiệm một vòng thủ tục:** app registration cho khai **nhiều redirect URI cùng lúc**.
Xin một lần, khai cả hai:

```
http://localhost:8080/front/smtp_oauth2_callback.php          <- thu tren may
https://glpi.<vung-noi-bo>/front/smtp_oauth2_callback.php     <- chay that
```

**Không cần mở firewall ra internet.** Luồng OAuth chạy qua trình duyệt của người uỷ quyền
(trình duyệt → Microsoft → trình duyệt → GLPI nội bộ). Microsoft **không bao giờ tự gọi vào**
máy chủ GLPI. Địa chỉ nội bộ hoàn toàn ổn, miễn người bấm uỷ quyền đang ở trong mạng công ty.
Đây là câu trả lời sẵn cho IT khi họ hỏi "có phải mở firewall không".

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

**OAuth KHÔNG thay thế SMTP — nó chỉ thay ô mật khẩu**

Hay bị hiểu nhầm là phải cấu hình hai thứ. Thật ra `smtp_mode` chỉ nhận **một** giá trị:

```
MAIL_MAIL      = 0   ham mail() cua PHP, khong qua SMTP
MAIL_SMTP      = 1   SMTP + mat khau
MAIL_SMTPSSL   = 2   SMTP + mat khau, ma hoa SSL
MAIL_SMTPTLS   = 3   SMTP + mat khau, ma hoa TLS
MAIL_SMTPOAUTH = 4   SMTP + OAuth        <- chon cai nay cho M365
```

Bốn dòng dưới đều là SMTP, chỉ khác cách mã hoá và cách xác thực. `host`, `port`, địa chỉ
người gửi **giữ nguyên** ở mọi chế độ — đó là *gửi tới đâu*, không liên quan xác thực.

**Người nhận email: không có danh sách nào để duy trì**

GLPI không lưu danh sách gửi. Nó lưu **vai trò**, tra ra người thật lúc gửi:

| Bảng | Số dòng (bản sạch) | Là gì |
|---|---|---|
| `glpi_notifications` | 82 | Sự kiện cài sẵn: phiếu mới, được giao, cần duyệt, sắp trễ… |
| `glpi_notificationtargets` | 189 | **Ai nhận** — khai theo vai trò, không theo địa chỉ |
| `glpi_notificationtemplates` | 32 | Mẫu nội dung |
| `glpi_queuednotifications` | 0 | Hàng đợi chờ gửi |

Vai trò kiểu *người yêu cầu · nhóm phụ trách · người phê duyệt · người theo dõi*. Nhân viên
vào/ra, đổi phòng ban thì danh sách người nhận **tự đúng**, không phải sửa cấu hình.

Gửi bằng hàng đợi, không chặn thao tác người dùng — cron `queuednotification` chạy mỗi 60 giây.
(`mailgate` mỗi 10 phút là chiều ngược lại: đọc mail đến để tạo phiếu, cần thêm quyền **đọc**
hộp thư chứ không chỉ quyền gửi.)

**Trạng thái hiện tại**

```
notifications_mailing = 0     <- dang TAT
smtp_mode             = 0     <- MAIL_MAIL, chua cau hinh
smtp_oauth_provider   = (rong)
glpi_useremails       = 0 dong  <- CAI BAY, xem duoi
```

**Cạm bẫy `glpi_useremails = 0`:** chưa người dùng nào có địa chỉ email. Cấu hình SMTP xong,
gửi thử thành công, mà **vẫn không ai nhận được gì** — rất dễ tưởng SMTP hỏng. Email vào hệ
thống bằng hai đường: nhập tay từng người (hợp lúc thí điểm), hoặc **đồng bộ từ AD** (AD đã có
sẵn email mọi nhân viên). Đây là lý do thứ hai để đấu AD, ngoài chuyện đăng nhập.

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
