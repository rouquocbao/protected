OneChanger Guard – Hướng dẫn đầy đủ (Kernel →
Framework)
Mục tiêu: boot.img chỉ hoạt động với ROM của bạn. Nếu boot.img được dùng với ROM khác
(framework không hợp lệ), kernel sẽ panic sau N giây.
 CẢNH BÁO QUAN TRỌNG - Kernel panic có rủi ro bootloop cứng. - Khi test, luôn để
timeout dài (180–300s). - Đảm bảo máy có Recovery / Download Mode để cứu.

1. Kiến trúc tổng thể
```
Kernel (Guard)
 ├─ Tạo sysfs: /sys/kernel/onechanger/rom_ok
 ├─ Start timer (N giây)
 └─ Nếu rom_ok != 1 → panic()
Framework (SystemServer)
 ├─ Chạy muộn khi Android đã lên
 ├─ Tính hash file bí mật trong /system
 ├─ So với hash từ boot (kernel)
 └─ Nếu hợp lệ → ghi 1 vào sysfs
``
