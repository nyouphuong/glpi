---
ngày: 2026-08-17
files: src/Glpi/Mail/SMTP/OauthConfig.php:136, src/Glpi/Mail/SMTP/OauthProvider/Azure.php
commit: 9ce46b030d
---

# Gửi mail qua Microsoft 365: phải OAuth, và phải xin đủ 4 thứ

**Phát hiện**

GLPI 11 có sẵn provider Azure (`MAIL_SMTPOAUTH = 4`), không cần plugin.
Basic auth (user+pass) tới `smtp.office365.com` đã bị Microsoft bỏ. Relay nội bộ thì không dính.

Xin IT — thiếu một mục là tắc:

| # | Xin gì |
|---|---|
| 1 | Bản ghi A trong DNS nội bộ: `glpi.<vùng>` → IP. Miễn phí, không mua tên miền |
| 2 | Chứng chỉ HTTPS từ CA nội bộ (Entra bắt buộc HTTPS, IP thuần có thể bị từ chối) |
| 3 | Hộp thư **có license, đăng nhập tương tác được** + bật **Authenticated SMTP** cho nó |
| 4 | App registration Entra ID → client id, secret, **tenant id** + **file CA gốc công ty** |

App registration:

```
Redirect URI : https://glpi.<vung>/front/smtp_oauth2_callback.php
               http://localhost:8080/front/smtp_oauth2_callback.php   (khai luon ca 2)
Permission   : SMTP.Send (Office 365 Exchange Online) — DELEGATED + admin consent
```

**Vì sao khó tìm**

- **Shared mailbox không dùng được.** Provider đặt `'prompt' => 'login'` + scope `offline_access`
  ⇒ luồng delegated, bắt buộc có người đăng nhập một lần. Shared mailbox không đăng nhập tương tác được.
- **Authenticated SMTP tắt sẵn toàn tenant.** OAuth đúng hết vẫn lỗi, thông báo không gợi ý gì.
- **`tenant` mặc định là `common`** — phải thay bằng Directory ID thật.
- **Redirect URI ghép từ `url_base`** (`OauthConfig.php:136`), phải khớp từng ký tự với Azure.
- **Outbound bắt buộc.** Không cần inbound, nhưng server phải ra được
  `login.microsoftonline.com` (đổi mã lấy token) và `smtp.office365.com:587`.
- **Proxy công ty giải mã TLS** → container không tin CA nội bộ → token exchange fail.
  Đo được: `curl` fail `self-signed certificate in certificate chain`, thêm `-k` thì ra 302
  (đường mạng thông). Vá bằng cách nạp CA công ty:

  ```yaml
  # docker-compose.override.yaml
  services:
    app:
      volumes:
        - ./corp-root-ca.crt:/usr/local/share/ca-certificates/corp-root-ca.crt:ro
  ```
  ```bash
  docker compose exec --user=root app update-ca-certificates
  ```
  Cùng CA này vá luôn lỗi tải Cypress và `docker pull` từ `ghcr.io`.
- **`glpi_useremails` rỗng thì không ai nhận được mail** dù SMTP đúng. Nhập tay hoặc đồng bộ từ AD.

**Trạng thái hiện tại**

```
url_base              = http://localhost:8080     (da sua, truoc thieu cong)
notifications_mailing = 1
smtp_mode             = 4                          MAIL_SMTPOAUTH
smtp_host/port        = smtp.office365.com : 587
smtp_oauth_provider   = Glpi\Mail\SMTP\OauthProvider\Azure
from_email            = glpi-noreply@alphatheta.com
glpi_useremails       = 9 dia chi
```

Thiếu: client id, secret, tenant id, CA gốc. Lên máy chủ thật nhớ sửa `url_base`.

**Kiểm chứng lại**

```bash
docker exec glpi-app curl -sS -o /dev/null https://login.microsoftonline.com && echo "CA ok"
docker exec glpi-db sh -c 'mariadb -uroot -pglpi glpi -N -e \
  "SELECT CONCAT(name,\" = \",COALESCE(value,\"(rong)\")) FROM glpi_configs \
   WHERE context=\"core\" AND (name LIKE \"smtp%\" OR name=\"notifications_mailing\");"'
```

Gửi thử ở *Cài đặt → Thông báo → Cấu hình gửi email*, xem log `files/_log/mail*.log`.
