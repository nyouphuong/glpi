---
ngày: 2026-08-15
files: src/Glpi/Routing/PluginRoutesLoader.php:60-70, src/Glpi/DependencyInjection/PluginContainer.php:175-180, plugins/atcustom/
commit: d739565a49
---

# Tùy biến nội bộ phải nằm trong `plugins/`, và route của plugin bị tự thêm tiền tố

**Phát hiện**

Muốn thêm tính năng riêng mà vẫn `git pull` được từ upstream thì **không sửa core**, viết
plugin. Lý do rất cụ thể: `.gitignore` của GLPI có `/plugins/*`, nên toàn bộ thư mục plugin
nằm ngoài repo upstream — pull về không bao giờ conflict.

GLPI 11 cấp cho plugin **Router + DI container thật của Symfony**, nên code mới viết được
theo kiểu hiện đại thay vì thêm file vào `front/` hay `ajax/`:

- Mọi class trong `plugins/<key>/src/Controller/` được nạp tự động vào container
  (`PluginContainer.php:175-180`), namespace bắt buộc là `GlpiPlugin\<Ucfirst(key)>\Controller\`.
  Hằng `NS_PLUG` = `GlpiPlugin\` (`src/autoload/constants.php:63`).
- Route khai bằng attribute `#[Route]`, khớp lúc runtime qua `PluginsRouterListener`
  (priority 375, sau `LegacyRouterListener` 400).
- Quyền truy cập khai bằng `#[SecurityStrategy(...)]`, danh sách chiến lược ở
  `src/Glpi/Http/Firewall.php:56-81` (`no_check`, `authenticated`, `central_access`…).

Plugin tối thiểu chạy được cần **3 file**:

```
plugins/<key>/setup.php                      plugin_version_<key>(), plugin_init_<key>()
plugins/<key>/hook.php                       plugin_<key>_install(), plugin_<key>_uninstall()
plugins/<key>/src/Controller/XxxController.php
```

Thiếu `hook.php` thì `plugin:install` báo *"function plugin_<key>_install is missing"* và
không cài được. Cài/bật bằng:

```bash
php bin/console plugin:install <key> --username=glpi -n
php bin/console plugin:activate <key> -n
```

**Vì sao khó tìm**

`PluginRoutesLoader` **tự thêm tiền tố** đường dẫn và tên route
(`PluginRoutesLoader.php`, chỗ `addPrefix`/`addNamePrefix`):

| Khai trong controller | URL thật | Tên route thật |
|---|---|---|
| `#[Route('/health', name: 'health')]` | `/plugins/<key>/health` | `@<key>:health` |

Viết đường dẫn tuyệt đối `#[Route('/plugins/atcustom/health')]` sẽ ra
`/plugins/atcustom/plugins/atcustom/health` → 404, mà **thông báo lỗi không hề gợi ý điều đó**
(chỉ là trang "Item not found" bình thường). Mỗi route còn được đăng ký 2 lần, thêm một bản
dưới `/marketplace/<key>/`.

Thêm nữa `debug:router` **không** liệt kê route plugin — chúng nằm ở router riêng
(`glpi_plugin_router`), khớp lúc runtime. Đừng dùng `debug:router` để kết luận route không tồn tại.

**Neo code**

- `src/Glpi/Routing/PluginRoutesLoader.php` — chỗ thêm tiền tố, và nơi dump được toàn bộ route plugin
- `src/Glpi/Kernel/Listener/RequestListener/PluginsRouterListener.php:87` — ném 404 nếu plugin chưa loaded
- `src/Glpi/DependencyInjection/PluginContainer.php:177` — chỉ nạp khi tồn tại `src/Controller/`
- `plugins/atcustom/` — bộ khung mẫu đang chạy được

**Kiểm chứng lại**

```bash
curl -s http://localhost:8080/plugins/atcustom/health
# => {"plugin":"atcustom","version":"0.1.0","status":"ok"}
```

Nếu 404, dump route thật thay vì đoán — tạo file tạm trong `front/` (script legacy có bootstrap GLPI):

```php
$loader = new \Glpi\Routing\PluginRoutesLoader(\Glpi\Application\Environment::get()->value);
foreach ($loader->load('plugins_routes') as $name => $r) { echo "$name  {$r->getPath()}\n"; }
```

Nhớ xoá file tạm sau khi xong. Và nếu sửa file PHP mà không thấy đổi gì, kiểm tra
`opcache.validate_timestamps` — xem [note về bind-mount](bind-mount-windows-lam-cham-100-lan.md).
