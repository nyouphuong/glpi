---
ngày: 2026-08-15
files: docker-compose.yaml:10, Makefile:47
commit: d739565a49
---

# Chạy GLPI từ ổ Windows qua Docker bind-mount làm mỗi request chậm ~100 lần

**Phát hiện**

`docker-compose.yaml:10` bind-mount source vào container: `".:/var/www/glpi:rw"`.
Nếu repo nằm trên ổ Windows (`D:\GLPI`) và dùng Docker Desktop, **mọi thao tác file nhỏ
đi qua lớp dịch 9P/virtiofs** và chậm thảm hại. GLPI chạm hàng nghìn file mỗi request nên
ăn trọn hình phạt này.

Số đo thật, cùng một container, đọc bằng PHP:

| Vị trí | `stat` | `read` |
|---|---|---|
| Bind-mount `D:\` | 367 op/s | **130 file/s** |
| Named volume ext4 | 408.388 op/s | **47.631 file/s** |

→ chậm hơn **~1.100×** khi stat, **~366×** khi đọc. Đọc 1.500 file mất 11,5 giây.
Sau khi chuyển sang ext4, cùng phép đo đó còn **0,67 giây**.

Hệ quả đo được trên trang `/front/central.php`:

| | Repo ở `D:\` | Repo ở WSL2 ext4 |
|---|---|---|
| central.php | 9–27 s | **0,13–0,21 s** |
| login | 16–17 s | 0,40 s |
| ticket/computer/user.php | 24–27 s | 0,25–0,62 s |

**Vì sao khó tìm**

Triệu chứng giống hệt "app viết tệ" hoặc "thiếu cache", nên rất dễ đi sai đường. Đã thử và
**đều không phải nguyên nhân**: tắt Xdebug (đang bật `mode=develop,debug`), đặt
`opcache.validate_timestamps=0`, nâng `max_accelerated_files` lên 65407, chuyển
`GLPI_ENVIRONMENT_TYPE` từ `development` sang `production`, chuyển `files/_cache` sang
named volume. Tất cả cộng lại chỉ kéo 27s xuống ~9s.

Bằng chứng loại trừ dứt điểm:
- Debug bar GLPI báo **8.635 ms tổng nhưng chỉ 109 ms SQL** → không phải DB.
- File PHP tối giản (không bootstrap GLPI) chạy **0,02 s** → không phải Apache/PHP.
- Opcache lành mạnh: 1.887 script cached, hit rate 97,5%, không OOM restart.

Nói cách khác: không có tinh chỉnh PHP nào vá được hình phạt I/O 366×. **Phải đưa source
ra khỏi filesystem Windows**, không có đường vòng.

**Cách xử lý**

Đặt repo trong ext4 của WSL2 (`~/GLPI`) rồi chạy `docker compose` từ đó. Kèm theo:

- Đặt `git config core.autocrlf false` cho bản WSL. Bản trên `D:\` có `autocrlf=true` nên
  toàn bộ working tree là CRLF, khiến `bin/console` chết với `env: 'php\r': No such file`
  (shebang dính `\r`), và `install/mysql/*.sql` đổi nội dung → **hash schema lệch**, GLPI
  báo "Update needed" dù cùng commit.
- Khi copy file giữa hai bản phải chuyển CRLF→LF (`sed 's/\r$//'`), không thì `eslint` văng
  hàng nghìn lỗi `linebreak-style`.
- File nào có sửa mà muốn commit thì để **LF**. Thực nghiệm: cùng một file, để CRLF thì
  `git diff` báo 1413+/1399− (cả file), để LF thì còn đúng 114+/63−.
- Clone bằng `git clone /mnt/d/GLPI ~/GLPI` (đọc packfile, ~44 s) thay vì copy cây thư mục
  (20k file lẻ qua bind-mount thì rất lâu). `vendor/` và `node_modules/` cài lại tại chỗ,
  đừng copy.

**ĐỪNG giữ `opcache.validate_timestamps=0` sau khi đã chuyển sang ext4**

Lúc còn ở bind-mount có thể bạn đặt `opcache.validate_timestamps=0` để bớt stat file.
Chuyển sang ext4 rồi thì **phải bật lại `=1`**, vì:

- Trên ext4 stat rẻ (408.388 op/s) nên bật lại gần như không tốn gì — đo thực tế
  `central.php` vẫn 0,14–0,18 s.
- Để `=0` thì **sửa file PHP xong web vẫn chạy code cũ** cho tới khi reload Apache. Bẫy này
  đã làm mất công debug thật: sửa route trong plugin, copy file, xoá cache Symfony, mà route
  vẫn ra đường dẫn cũ — chỉ vì opcache giữ bản biên dịch cũ.

Cấu hình nên dùng (đặt trong `/usr/local/etc/php/custom_conf.d/`, là một named volume nên
sống qua `docker compose up` nhưng **không** sống qua `docker compose down -v`):

```ini
opcache.validate_timestamps=1
opcache.revalidate_freq=0
opcache.max_accelerated_files=65407   ; mặc định 10000 quá nhỏ cho GLPI + vendor
opcache.memory_consumption=512
```

Xdebug thì vẫn nên tắt khi không debug (`xdebug.mode=off`) — image dev bật sẵn
`mode=develop,debug`, tốn đáng kể.

**Kiểm chứng lại**

```bash
docker compose exec app php -r '$t=microtime(true); $n=0;
  foreach (new RecursiveIteratorIterator(new RecursiveDirectoryIterator("src")) as $f)
    { if ($f->isFile()) { @file_get_contents($f->getPathname()); if (++$n>=1500) break; } }
  printf("%d file trong %.2fs\n", $n, microtime(true)-$t);'
```

Dưới ~1 giây là đang chạy trên filesystem native. Trên 10 giây là đang dính bind-mount Windows.
