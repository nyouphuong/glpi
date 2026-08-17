---
ngày: 2026-08-17
files: docker-compose.yaml:24-37, Makefile:137-155
commit: 9ce46b030d
---

# Dữ liệu nằm trong Docker volume, và dump DB một mình KHÔNG phải là backup đủ

**Dữ liệu nằm ở đâu**

| Thứ | Nơi lưu | Ghi chú |
|---|---|---|
| Database | volume `glpi_db` → `/var/lib/docker/volumes/glpi_db/_data` | Nằm **trong VM của Docker Desktop**, không phải trên `D:\` cũng không phải trong home WSL |
| Tài liệu người dùng upload | `files/_uploads`, `_pictures`, `_inventories`, `_rss`, `_graphs`, `_plugins` | Nằm cùng thư mục source |
| Cache / session / tmp | volume `glpi_glpi_cache`, `glpi_glpi_sessions`, `glpi_glpi_tmp` | **Vứt được**, không cần backup |
| Khoá mã hoá + cấu hình DB | `config/glpicrypt.key`, `config/config_db.php`, `config/oauth.pem`, `oauth.pub` | `config/` bị gitignore → **không có trong git** |

Thông số kết nối (khai ở `docker-compose.yaml:31-35`):

```
database : glpi          host (trong mang compose): db
root     : glpi / glpi   user: glpi / glpi
port 3306 KHONG publish ra host -> chi truy cap duoc qua container
```

Quy mô hiện tại: 442 bảng, 46,5 MB dữ liệu, 245 MB trên đĩa.

**Cạm bẫy chết người**

1. **`docker compose down -v` xoá sạch volume `glpi_db`** — mất toàn bộ database, không cứu
   được. `docker compose down` (không có `-v`) thì an toàn.
2. **Dump DB một mình là backup THIẾU.** Thiếu `config/glpicrypt.key` thì các trường được mã
   hoá trong DB (mật khẩu LDAP, mật khẩu mailbox…) không giải mã lại được — dump vẫn phục hồi
   nhưng mấy phần đó thành rác. Thiếu `files/` thì mất hết tài liệu người dùng đã upload.
   **Backup đủ = DB dump + `files/` + `config/`.**
3. **`docker compose` phải chạy từ thư mục WSL** (`/home/phong/GLPI`). Chạy từ `D:\GLPI` sẽ
   tạo lại container với bind-mount về `D:` và phá mất setup — xem
   [note quy trình 2 bản repo](quy-trinh-hai-ban-repo-va-mo-hinh-nhanh.md).
   Riêng `docker exec` / `docker start` thì chạy từ đâu cũng được vì không đụng compose file.

**Backup — lệnh đã chạy thật và verify**

```bash
mkdir -p .dump                      # .dump/ da nam trong .gitignore
TS=$(date +%Y-%m-%d_%H-%M-%S)

# 1. Database
docker exec glpi-db sh -c \
  'mariadb-dump -uroot -pglpi --single-transaction --routines --triggers --events glpi' \
  | gzip > ".dump/glpi_${TS}.sql.gz"

# 2. Tai lieu nguoi dung + khoa ma hoa (chay trong WSL, tai /home/phong/GLPI)
tar czf ".dump/files_${TS}.tar.gz" \
    --exclude='files/_cache' --exclude='files/_sessions' --exclude='files/_tmp' \
    files/ config/
```

`--single-transaction` để dump nhất quán mà không khoá bảng (InnoDB).

Kiểm tra dump vừa tạo:

```bash
gunzip -t .dump/glpi_*.sql.gz                        # toan ven gzip
gunzip -c .dump/glpi_*.sql.gz | grep -c 'CREATE TABLE'   # phai ra 442
```

**Phục hồi**

Cách an toàn để **thử** dump mà không đụng DB thật — phục hồi vào một DB riêng rồi đối chiếu:

```bash
docker exec glpi-db sh -c 'mariadb -uroot -pglpi -e "
  DROP DATABASE IF EXISTS glpi_restore_test;
  CREATE DATABASE glpi_restore_test CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"'

gunzip -c .dump/glpi_<TS>.sql.gz | docker exec -i glpi-db sh -c 'mariadb -uroot -pglpi glpi_restore_test'

docker exec glpi-db sh -c 'mariadb -uroot -pglpi -N -e "
  SELECT CONCAT(table_schema,\": \",COUNT(*)) FROM information_schema.tables
  WHERE table_schema IN (\"glpi\",\"glpi_restore_test\") GROUP BY table_schema;"'
# hai dong phai ra so bang bang nhau

docker exec glpi-db sh -c 'mariadb -uroot -pglpi -e "DROP DATABASE glpi_restore_test;"'
```

Phục hồi **đè lên DB thật** (mất dữ liệu hiện tại, cân nhắc kỹ):

```bash
docker exec glpi-db sh -c 'mariadb -uroot -pglpi -e "
  DROP DATABASE IF EXISTS glpi;
  CREATE DATABASE glpi CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"'
gunzip -c .dump/glpi_<TS>.sql.gz | docker exec -i glpi-db sh -c 'mariadb -uroot -pglpi glpi'
```

Nhớ giải nén kèm `files/` và `config/` cho khớp cùng thời điểm, rồi
`docker compose exec app php bin/console cache:clear`.

**Kiểm chứng lại**

Đã chạy thật ngày 17/08/2026: dump ra 117 KB (bản cài mới, gần như chưa có dữ liệu), phục hồi
vào `glpi_restore_test` cho **442 bảng khớp với bản gốc**, và ticket test có mặt đầy đủ.

`Makefile:137-155` có sẵn target `db-dump` / `db-restore` làm đúng việc này, nhưng **máy này
chưa cài `make`** nên phải gõ lệnh thô như trên.
