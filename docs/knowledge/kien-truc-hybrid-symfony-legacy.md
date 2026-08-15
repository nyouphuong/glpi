---
ngày: 2026-08-15
files: src/Glpi/Controller/LegacyFileLoadController.php:53-100, src/Glpi/Kernel/Kernel.php, src/Glpi/Controller/CentralController.php:50
commit: d739565a49
---

# GLPI chạy trên Symfony 6.4, nhưng 97% code nghiệp vụ vẫn là PHP thuần thời 2003

**Phát hiện**

Đây là codebase **migrate dở**, hai thời đại chồng lên nhau. Symfony 6.4 thật
(`symfony/framework-bundle`, kernel ở `src/Glpi/Kernel/Kernel.php`, entrypoint duy nhất
`public/index.php`), Twig cho view, Laminas cho mail/i18n. **Không có Doctrine** — tầng DB
tự viết (`DBmysql` + `CommonDBTM` kiểu Active Record).

Nhưng phần lớn code vẫn là kiểu cũ:

| Kiểu | Đường dẫn | Quy mô |
|---|---|---|
| Class nghiệp vụ, **không namespace** | `src/*.php` | 623 file / 290.202 dòng |
| Trang web, mỗi file 1 trang | `front/*.php` | 315 file / 23.202 dòng |
| Endpoint AJAX, mỗi file 1 endpoint | `ajax/*.php` | 119 file / 10.066 dòng |
| Code hiện đại PSR-4 `Glpi\` | `src/Glpi/**` | 47 thư mục |
| View | `templates/**.twig` | 478 file |

86 class trong `src/*.php` kế thừa `CommonDBTM`. Ví dụ chuỗi thừa kế thật:
`Ticket extends CommonITILObject extends CommonDBTM`.

**Vì sao khó tìm**

Nhìn thư mục gốc sẽ tưởng là 2 dự án khác nhau ghép lại, và **không thấy được cầu nối**.
Cầu nối là `LegacyFileLoadController`: nó là một Symfony controller mà bên trong `require`
thẳng file PHP cũ, hứng output bằng `ob_start`/`ob_get_clean` rồi bọc thành `Response`.
Nên URL `/front/central.php` trông y hệt PHP 2005 nhưng thực tế đã đi qua toàn bộ HTTP
kernel của Symfony.

Hệ quả thực tế: **debug bar Symfony vẫn hiện trên các trang legacy**, tên route dạng
`@front_central_legacy`. Thấy tên đó thì biết trang đang chạy qua cầu nối này chứ không
phải một controller viết tay. Một số trang đã được nâng lên controller thật, ví dụ
`#[Route('/front/central.php', name: 'front_central_legacy')]` ở `CentralController.php:50`.

Đừng đi tìm `config/routes.yaml` kiểu Symfony chuẩn — route khai bằng **attribute**
`#[Route]` rải trong `src/Glpi/Controller/*`, phần còn lại rơi vào cầu nối legacy.
`routes/` chỉ có đúng 1 file cho môi trường development. `inc/` chỉ còn 2 file tàn dư,
đừng tưởng nó quan trọng.

**Neo code**

- `public/index.php` — entrypoint duy nhất
- `src/Glpi/Kernel/Kernel.php` — kernel Symfony
- `src/Glpi/Controller/LegacyFileLoadController.php:64` — chỗ `require` file cũ
- `src/Glpi/Controller/CentralController.php:50` — ví dụ route legacy đặt tên tường minh
- `src/CommonDBTM.php` — class gốc của toàn bộ tầng nghiệp vụ cũ
- `dependency_injection/services.php` — khai báo service Symfony

**Lần theo một trang bất kỳ**

`/front/computer.php` → `front/computer.php` (script cũ) → dùng class `src/Computer.php`
→ render `templates/**.twig`.

**Kiểm chứng lại**

```bash
grep -rn "_legacy" src/Glpi/ --include=*.php     # các route legacy được đặt tên
ls src/*.php | wc -l                             # còn bao nhiêu class chưa namespace
```

Con số giảm dần theo thời gian nghĩa là upstream đang migrate tiếp.
