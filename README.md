# OneChanger Guard (Kernel → Framework)

## Mục tiêu
OneChanger Guard là cơ chế bảo vệ **boot.img chỉ hoạt động với ROM tương ứng**. Nếu boot.img bị dùng với ROM khác (framework không hợp lệ), kernel sẽ **panic sau N giây**.

> ⚠️ **CẢNH BÁO**
> - Kernel panic có rủi ro **bootloop cứng**.
> - Khi test, **luôn để timeout dài (180–300s)**.
> - Đảm bảo thiết bị có **Recovery / Download / Fastboot mode** để cứu máy.

---

## Kiến trúc tổng thể

### 1. Kernel (Guard)
- Tạo sysfs:
  ```
  /sys/kernel/onechanger/rom_ok
  ```
- Khi boot:
  - `rom_ok = false`
  - Start `delayed_work` với timeout N giây
- Hết timeout:
  - Nếu `rom_ok != 1` → `panic()`

### 2. Framework (SystemServer)
- Chạy **sau khi Android boot hoàn tất**
- Tính SHA-256 của file bí mật trong `/system`
- So sánh với hash truyền từ bootloader
- Nếu hợp lệ:
  ```
  echo 1 > /sys/kernel/onechanger/rom_ok
  ```

ROM khác không có framework patch → không ghi sysfs → kernel panic.

---

## PHẦN A — KERNEL: OneChanger Guard

### A1. Driver kernel
**Đường dẫn**
```
drivers/android/onechanger_guard.c
```

**Chức năng chính**
- Tạo kobject `onechanger`
- Expose sysfs attribute `rom_ok`
- Start `delayed_work` kiểm tra xác thực ROM
- Panic nếu ROM không hợp lệ

### A2. Kconfig & Makefile
**drivers/android/Makefile**
```
obj-$(CONFIG_ONECHANGER_GUARD) += onechanger_guard.o
```

**drivers/android/Kconfig**
```
config ONECHANGER_GUARD
    bool "OneChanger ROM Guard"
    default y
```

**defconfig**
```
CONFIG_ONECHANGER_GUARD=y
```

### A3. Kiểm tra kernel
```
adb shell ls /sys/kernel/onechanger
adb shell cat /sys/kernel/onechanger/rom_ok
```

---

## PHẦN B — File bí mật trong system

### B1. Tạo file bí mật khi build ROM
```
mkdir -p device/<vendor>/<codename>/onechanger
head -c 64 /dev/urandom > device/<vendor>/<codename>/onechanger/.onechanger_blob
```

Copy vào system image:
```
PRODUCT_COPY_FILES += \
    device/<vendor>/<codename>/onechanger/.onechanger_blob:$(TARGET_COPY_OUT_SYSTEM)/etc/.onechanger_blob
```

Sau khi flash ROM:
```
/system/etc/.onechanger_blob
```

### B2. Gắn hash vào boot
```
sha256sum device/<vendor>/<codename>/onechanger/.onechanger_blob
```

**BoardConfig.mk**
```
BOARD_BOOTCONFIG += androidboot.onechanger_sig=<SHA256_HEX>
```

Runtime property:
```
ro.boot.onechanger_sig
```

---

## PHẦN C — FRAMEWORK (SystemServer)

### C1. File cần patch
```
frameworks/base/services/java/com/android/server/SystemServer.java
```

### C2. Logic xác thực
- Đọc hash từ:
  - `ro.boot.onechanger_sig`
  - `/system/etc/.onechanger_blob`
- So sánh SHA-256
- Nếu khớp → ghi `1` vào sysfs

### C3. Thời điểm chạy
- Thread riêng
- Chờ `sys.boot_completed=1`
- Tránh block SystemServer

---

## Trình tự test an toàn
1. Set `ONECHANGER_TIMEOUT_SEC = 180` (hoặc 300)
2. Build & flash ROM đầy đủ
3. Boot vào Android
4. Kiểm tra:
   ```
   adb shell getprop ro.boot.onechanger_sig
   adb shell ls -l /system/etc/.onechanger_blob
   adb shell cat /sys/kernel/onechanger/rom_ok
   ```
5. Flash **ROM khác + boot.img của bạn** → panic sau timeout

---

## Lưu ý bảo mật & giới hạn
- Người có root có thể:
  ```
  echo 1 > /sys/kernel/onechanger/rom_ok
  ```
- Có thể tăng cường bằng:
  - SELinux rule
  - Token động kernel ↔ framework
- Dev giỏi có thể rebuild kernel bỏ guard

> Cơ chế này nhằm **anti‑reuse / anti‑mix**, **không phải DRM tuyệt đối**.

---

## Gợi ý nâng cấp
- SELinux: chỉ cho `system_server` ghi sysfs
- Token-based authorization
- Thay `panic()` bằng reboot/freeze khi release

---

## License
Kernel module: GPL
Framework patch: AOSP compatible

