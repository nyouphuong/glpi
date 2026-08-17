---
ngày: 2026-08-17
files: src/Glpi/Mail/SMTP/OauthConfig.php:136
commit: 9ce46b030d
---

# Việc cần làm để GLPI tự gửi email thông báo

Mục đích: khi có phiếu mới, phiếu được giao, hay cần ký duyệt thì GLPI **tự gửi email** báo
cho đúng người. Hiện chưa gửi được vì thiếu thông tin từ IT.

## Phần đã làm xong

Tao đã cấu hình sẵn trong GLPI, **không cần làm lại**:

```
Bat gui email                 : ROI
Che do gui                    : SMTP + OAuth
May chu gui                   : smtp.office365.com, cong 587
Dia chi dung ten gui          : glpi-noreply@alphatheta.com
Dia chi email cua nguoi dung  : da nhap 9 nguoi
```

## Phần cần xin IT — 4 thứ

Gửi cho IT công ty mẹ, xin đủ 4 mục dưới. **Thiếu một mục là không chạy được.**

**1. Một cái tên cho máy chủ GLPI** — ví dụ `glpi.alphatheta.local`

Không phải mua tên miền. Công ty đã có hệ thống tên nội bộ rồi, IT chỉ cần thêm một dòng
trỏ cái tên đó về địa chỉ IP của máy chủ GLPI. Miễn phí.

*Vì sao cần:* Microsoft không chấp nhận khai bằng địa chỉ IP.

**2. Chứng chỉ HTTPS cho cái tên đó**

Xin từ hệ thống cấp chứng chỉ nội bộ của công ty. Cũng miễn phí, và máy tính nhân viên tin
sẵn nên không hiện cảnh báo đỏ.

*Vì sao cần:* Microsoft bắt buộc dùng `https`, không nhận `http`.

**3. Một hộp thư để đứng tên gửi** — ví dụ `glpi-noreply@alphatheta.com`

Yêu cầu hai điều, nói rõ với IT kẻo họ cấp sai:
- Phải là hộp thư **đăng nhập được bình thường** — đừng xin loại "shared mailbox"
- Phải **bật Authenticated SMTP** cho hộp thư này

*Vì sao cần loại đăng nhập được:* lúc cài đặt phải có người mở trình duyệt, đăng nhập bằng
chính hộp thư đó một lần để bấm đồng ý. Shared mailbox không đăng nhập được nên không làm
được bước này.

**4. Đăng ký ứng dụng bên Microsoft + file chứng chỉ gốc của công ty**

Nhờ quản trị viên Microsoft 365 tạo một "app registration", rồi xin lại **3 con số**:
`Client ID`, `Client secret`, `Tenant ID`.

Khi tạo, họ cần điền đúng mấy thông tin này:

```
Redirect URI:
   https://glpi.alphatheta.local/front/smtp_oauth2_callback.php
   http://localhost:8080/front/smtp_oauth2_callback.php

Quyen: SMTP.Send  (Office 365 Exchange Online), loai Delegated
Can bam "Grant admin consent"
```

Khai luôn cả hai dòng Redirect URI để khỏi phải xin lại lần hai.

Xin thêm **file chứng chỉ gốc của công ty** (đuôi `.crt`). Lý do ở mục "Lỗi hay gặp" bên dưới.

## Khi nhận đủ thì làm gì

1. Đưa tao `Client ID`, `Client secret`, `Tenant ID`, và file `.crt` — tao cắm vào
2. Vào *Cài đặt → Thông báo → Cấu hình gửi email*, bấm nút uỷ quyền
3. Trình duyệt nhảy sang trang đăng nhập Microsoft → đăng nhập **bằng hộp thư ở mục 3**
4. Quay về GLPI là xong. Gửi thử một email để kiểm tra.

## Lỗi hay gặp

**Bấm uỷ quyền thì báo lỗi chứng chỉ.** Máy chủ GLPI không ra được trang đăng nhập của
Microsoft — không phải bị chặn, mà do công ty dùng thiết bị kiểm tra nội dung mạng, nó thay
chứng chỉ của Microsoft bằng chứng chỉ của công ty, GLPI không tin nên chặn lại.

Đây là lý do phải xin file `.crt` ở mục 4. Nạp vào rồi thì hết:

```yaml
# them vao docker-compose.override.yaml
services:
  app:
    volumes:
      - ./corp-root-ca.crt:/usr/local/share/ca-certificates/corp-root-ca.crt:ro
```
```bash
docker compose exec --user=root app update-ca-certificates
```

Nạp xong cũng hết luôn mấy lỗi tải file khác từng gặp trước đây.

**Gửi thử thành công nhưng không ai nhận được.** Do người dùng chưa có địa chỉ email trong
GLPI. Hiện đã nhập cho 9 người. Thêm người mới thì nhớ nhập email, hoặc sau này nối với hệ
thống tài khoản công ty để tự lấy.

**Cấu hình đúng hết mà vẫn lỗi đăng nhập Microsoft.** Kiểm tra IT đã bật Authenticated SMTP
cho hộp thư chưa — mục này bị quên nhiều nhất, và thông báo lỗi không hề nhắc tới nó.

## Kiểm tra nhanh

```bash
# xem may chu da tin chung chi cong ty chua
docker exec glpi-app curl -sS -o /dev/null https://login.microsoftonline.com && echo OK

# xem cau hinh gui mail hien tai
docker exec glpi-db sh -c 'mariadb -uroot -pglpi glpi -N -e \
  "SELECT CONCAT(name,\" = \",COALESCE(value,\"(rong)\")) FROM glpi_configs \
   WHERE context=\"core\" AND (name LIKE \"smtp%\" OR name=\"notifications_mailing\");"'
```

Chuyển sang máy chủ thật thì nhớ sửa lại địa chỉ GLPI trong *Cài đặt → Chung* cho khớp
cái tên ở mục 1, nếu không nút uỷ quyền sẽ báo sai địa chỉ.
