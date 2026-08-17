# Sổ tay codebase

> **ĐỌC FILE NÀY TRƯỚC KHI ĐI TÌM BẤT CỨ THỨ GÌ TRONG CODE.**
> Mỗi phát hiện = 1 dòng ở đây + 1 file riêng. Chỉ mở file khi thấy dòng đó liên quan.
> Giữ mỗi dòng **dưới 15 từ** — đây là file bị đọc thường xuyên thứ hai sau `CLAUDE.md`.

Format: `- [tiêu đề](file.md) — mồi nhử để biết có nên mở không`

## Kiến trúc & layout
- [Symfony 6.4 bọc ngoài, 97% code nghiệp vụ vẫn là PHP 2003](kien-truc-hybrid-symfony-legacy.md) — đọc trước khi tìm route hay class

## Cạm bẫy / hành vi bất ngờ
- [Repo trên ổ Windows làm mỗi request chậm ~100 lần](bind-mount-windows-lam-cham-100-lan.md) — không phải lỗi app

## Quy ước ngầm (code không nói ra)
- [Tùy biến phải nằm trong plugins/, route plugin bị tự thêm tiền tố](tuy-bien-bang-plugin-khong-sua-core.md) — đọc trước khi viết tính năng mới

## Tích hợp ngoài / hạ tầng
- [Bản chạy nằm ở WSL không phải D:\, và mô hình nhánh main/riêng](quy-trinh-hai-ban-repo-va-mo-hinh-nhanh.md) — đọc trước khi sửa file hay pull upstream
- [DB nằm trong Docker volume, backup phải gồm 3 phần](backup-va-noi-luu-du-lieu.md) — lệnh dump/restore đã verify, và cách mất sạch DB

---

## Note có còn đúng không?

Mỗi note ghi `commit:` là hash lúc viết. Kiểm tra file đó đã đổi chưa:

```bash
git log --oneline <commit>..HEAD -- <đường/dẫn/file>
```

Rỗng = note còn nguyên giá trị. Có output = code đã đổi, **verify lại trước khi tin**,
sửa note rồi cập nhật `commit:`.

## Luật ghi note

**GHI khi:** tốn hơn ~5 phút / hơn 3 file mới hiểu ra một điều không hiển nhiên.

**KHÔNG ghi:**
- Thứ đọc code là thấy ngay (tên hàm làm gì, kiểu dữ liệu…)
- Chuyện chỉ đúng một lần (giá trị debug, kết quả một lần chạy)
- Thứ đã có trong `docs/ba/` hoặc `docs/governance/`
- Lịch sử sửa lỗi — cái đó để git log lo

Note sai còn tệ hơn không có note. Thà bỏ trống còn hơn ghi đoán.
