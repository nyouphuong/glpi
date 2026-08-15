---
ngày: 2026-08-15
files: docker-compose.yaml:10, .gitignore:5
commit: 9ce46b030d
---

# Repo chạy từ WSL chứ không phải từ `D:\`, và `main` chỉ là gương của upstream

**Phát hiện**

GLPI tồn tại ở **hai nơi**, rất dễ nhầm bản nào đang chạy:

| Nơi | Vai trò | `core.autocrlf` |
|---|---|---|
| `D:\GLPI` | Mở IDE, giữ git chính, là `origin` của bản WSL | `true` → working tree CRLF |
| `/home/phong/GLPI` (WSL2 ext4) | **Bản Docker thật sự phục vụ `localhost:8080`** | `false` → LF |

**Sửa file ở `D:\` không có hiệu lực gì** cho tới khi đồng bộ sang WSL. Lý do phải tách đôi:
chạy trực tiếp từ ổ Windows chậm hơn ~100 lần — xem
[note bind-mount](bind-mount-windows-lam-cham-100-lan.md).

Mô hình nhánh:

| Nhánh | Vai trò | Commit lên? |
|---|---|---|
| `main` | Gương sao y `upstream/11.0/bugfixes` (GLPI 11.0.9-dev) | **Không bao giờ** |
| nhánh riêng | Code tùy biến, `.claude/`, `docs/`, plugin | Có, và chỉ nhánh này push lên fork |

Giữ `main` sạch là thứ đảm bảo `git merge --ff-only` không bao giờ conflict; mọi va chạm dồn
về bước `rebase` — nơi xử lý chủ động.

**Vì sao khó tìm**

Ba cái bẫy đã cắn thật:

1. **`upstream/main` KHÔNG phải bản 11 mới hơn — nó là GLPI 12.0.0-dev.** Hai dòng lệch nhau
   35 / 551 commit. Gõ `git merge upstream/main` nghe rất hợp lý mà lại kéo nguyên một major
   version khác vào, kèm 4 conflict và bắt migrate DB. Nhánh đúng để cập nhật v11 là
   `upstream/11.0/bugfixes`. Bản phát hành gần nhất là tag `11.0.8`; đầu nhánh bugfixes đang
   cách tag đó 189 commit (tức phần 11.0.9 chưa phát hành).

2. **File untracked bị `git clean -fd` xoá sạch.** `docs/`, `CLAUDE.md`, `.claude/` từng mất
   nguyên một lần vì mới chỉ nằm untracked. `plugins/` sống sót do nằm trong `.gitignore`
   (`clean` không có `-x` thì không đụng file ignored). Chưa `git add` thì git **chưa băm nội
   dung** → không có object nào để `fsck`/`reflog` moi lại. Commit sớm là cách duy nhất.

3. **Gọi lệnh WSL kiểu inline thì kết quả không đáng tin.** Từ Git Bash,
   `wsl -- bash -c '...'` bị nuốt `$VAR`/`$(...)` (từng tạo ra `rsync -a // //`, và nhiều lần
   báo sai trạng thái repo vì `cd ~/GLPI` fail rồi lệnh chạy nhầm ở `/mnt/d/GLPI`), đồng thời
   MSYS đổi `/mnt/c/...` thành `C:/Program Files/Git/mnt/c/...`.

**Thao tác chuẩn**

Đồng bộ file sang bản chạy — **luôn chuyển CRLF→LF**, không thì eslint văng hàng nghìn lỗi
`linebreak-style`:

```bash
sed 's/\r$//' /mnt/d/GLPI/<duong/dan> > /home/phong/GLPI/<duong/dan>
```

Chạy lệnh trong WSL — viết ra file script rồi gọi, đừng inline:

```bash
MSYS_NO_PATHCONV=1 MSYS2_ARG_CONV_EXCL='*' wsl -d Ubuntu-22.04 -- bash "/mnt/c/.../script.sh"
```

Trong script luôn chốt an toàn trước thao tác đệ quy: kiểm tra biến đường dẫn khác rỗng và
không phải `/`, `/home`, `/mnt/*`.

Cập nhật từ upstream khi thấy có gì đáng lấy:

```bash
git fetch upstream
git log --oneline main..upstream/11.0/bugfixes    # xem co gi moi truoc da
git checkout main && git merge --ff-only upstream/11.0/bugfixes
git checkout <nhanh-rieng> && git rebase main && git push --force-with-lease
```

**Kiểm chứng lại**

```bash
docker compose exec app sh -c 'df -h /var/www/glpi | tail -1'
```

Ra `/dev/sd*` là đang chạy từ ext4 (đúng). Ra `D:\` là đang dính bind-mount Windows (sai,
và sẽ chậm ~100 lần).
