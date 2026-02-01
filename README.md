# OneChanger Guard – Hướng dẫn đầy đủ (Kernel → Framework)

**Mục tiêu:**  
`boot.img` chỉ hoạt động với ROM của bạn. Nếu `boot.img` được dùng với ROM khác (framework không hợp lệ), kernel sẽ panic sau N giây.

⚠ **CẢNH BÁO QUAN TRỌNG**
- Kernel panic có rủi ro bootloop cứng  
- Khi test, luôn để timeout dài (180–300s)  
- Đảm bảo máy có Recovery / Download Mode để cứu  

---

## 1. Kiến trúc tổng thể

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
```

ROM khác không có code framework → không ghi sysfs → kernel panic.

---

# PHẦN A — KERNEL: ONECHANGER GUARD

## A1. Tạo driver kernel

**Đường dẫn**
```
kernel/samsung/exynos9810/drivers/android/onechanger_guard.c
```

**Mã nguồn**
```c
#include <linux/module.h>
#include <linux/init.h>
#include <linux/kobject.h>
#include <linux/sysfs.h>
#include <linux/workqueue.h>
#include <linux/jiffies.h>
#include <linux/kernel.h>
#include <linux/mutex.h>

#define ONECHANGER_TIMEOUT_SEC 180

static struct kobject *onechanger_kobj;
static struct delayed_work panic_work;
static bool rom_ok;
static DEFINE_MUTEX(onechanger_lock);

static void onechanger_panic_work(struct work_struct *work)
{
    bool ok;
    mutex_lock(&onechanger_lock);
    ok = rom_ok;
    mutex_unlock(&onechanger_lock);

    if (!ok)
        panic("OneChangerGuard: ROM not authorized");
}

static ssize_t rom_ok_show(struct kobject *kobj,
                           struct kobj_attribute *attr,
                           char *buf)
{
    bool ok;
    mutex_lock(&onechanger_lock);
    ok = rom_ok;
    mutex_unlock(&onechanger_lock);
    return scnprintf(buf, PAGE_SIZE, "%d\n", ok ? 1 : 0);
}

static ssize_t rom_ok_store(struct kobject *kobj,
                            struct kobj_attribute *attr,
                            const char *buf, size_t count)
{
    if (buf[0] == '1') {
        mutex_lock(&onechanger_lock);
        rom_ok = true;
        mutex_unlock(&onechanger_lock);
        cancel_delayed_work_sync(&panic_work);
    }
    return count;
}

static struct kobj_attribute rom_ok_attr =
    __ATTR(rom_ok, 0660, rom_ok_show, rom_ok_store);

static int __init onechanger_guard_init(void)
{
    int ret;
    rom_ok = false;

    onechanger_kobj = kobject_create_and_add("onechanger", kernel_kobj);
    if (!onechanger_kobj)
        return -ENOMEM;

    ret = sysfs_create_file(onechanger_kobj, &rom_ok_attr.attr);
    if (ret) {
        kobject_put(onechanger_kobj);
        return ret;
    }

    INIT_DELAYED_WORK(&panic_work, onechanger_panic_work);
    schedule_delayed_work(&panic_work, ONECHANGER_TIMEOUT_SEC * HZ);

    pr_info("OneChangerGuard loaded, timeout=%ds\n", ONECHANGER_TIMEOUT_SEC);
    return 0;
}
late_initcall(onechanger_guard_init);
MODULE_LICENSE("GPL");
```

---

## A2. Makefile & Kconfig

**kernel/samsung/exynos9810/drivers/android/Makefile**
```
obj-$(CONFIG_ONECHANGER_GUARD) += onechanger_guard.o
```

**kernel/samsung/exynos9810/drivers/android/Kconfig**
```
config ONECHANGER_GUARD
    bool "OneChanger ROM Guard"
    default y
```

**Defconfig**
**kernel/samsung/exynos9810/arch/arm64/configs/exynos9810-starlte_defconfig**
```
CONFIG_ONECHANGER_GUARD=y
```

---

## A3. Kiểm tra kernel

```bash
adb shell ls /sys/kernel/onechanger
adb shell cat /sys/kernel/onechanger/rom_ok
```

---

# PHẦN B — FILE BÍ MẬT TRONG SYSTEM

## B1. Tạo file bí mật khi build ROM

```bash
mkdir -p device/samsung/starlte/onechanger
head -c 64 /dev/urandom > device/samsung/starlte/onechanger/.onechanger_blob
```

**Copy vào system image:**
```
device/samsung/starlte/device.mk
```

```
PRODUCT_COPY_FILES += \
device/samsung/<codename>/onechanger/.onechanger_blob:$(TARGET_COPY_OUT_SYSTEM)/etc/.onechanger_blob
```

Sau khi flash ROM:
```
/system/etc/.onechanger_blob
```

---

## B2. Tính hash và gắn vào boot

```bash
sha256sum device/samsung/<codename>/onechanger/.onechanger_blob
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

# PHẦN C — FRAMEWORK (SYSTEMSERVER)

## C1. File cần patch
```
frameworks/base/services/java/com/android/server/SystemServer.java
```

## C2. Import cần thêm
```java
import android.os.SystemProperties;
import android.util.Slog;
import java.io.File;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.security.MessageDigest;
```

## C3. Helper & authorize logic

```java
private static final String TAG_OC = "OneChanger";
private static final String OC_FILE = "/system/etc/.onechanger_blob";
private static final String OC_EXPECT_PROP = "ro.boot.onechanger_sig";
private static final String OC_SYSFS = "/sys/kernel/onechanger/rom_ok";

private static String sha256File(String path) {
    File f = new File(path);
    if (!f.exists()) return "";
    try (FileInputStream fis = new FileInputStream(f)) {
        MessageDigest md = MessageDigest.getInstance("SHA-256");
        byte[] buf = new byte[8192];
        int r;
        while ((r = fis.read(buf)) > 0) md.update(buf, 0, r);
        byte[] d = md.digest();
        StringBuilder sb = new StringBuilder(d.length * 2);
        for (byte b : d) sb.append(String.format("%02x", b));
        return sb.toString();
    } catch (Throwable t) {
        Slog.e(TAG_OC, "sha256 failed", t);
        return "";
    }
}

private static boolean writeSysfsOne(String path) {
    try (FileOutputStream fos = new FileOutputStream(path)) {
        fos.write('1');
        fos.write('\n');
        fos.flush();
        return true;
    } catch (Throwable t) {
        Slog.e(TAG_OC, "write sysfs failed", t);
        return false;
    }
}

private void oneChangerAuthorizeIfValid() {
    String expected = SystemProperties.get(OC_EXPECT_PROP, "");
    String actual = sha256File(OC_FILE);

    if (expected.isEmpty() || actual.isEmpty()) {
        Slog.e(TAG_OC, "Missing hash");
        return;
    }

    if (!expected.equalsIgnoreCase(actual)) {
        Slog.e(TAG_OC, "Hash mismatch");
        return;
    }

    writeSysfsOne(OC_SYSFS);
}
```

## C4. Gọi authorize khi Android đã lên

```java
new Thread(() -> {
    for (int i = 0; i < 180; i++) {
        if ("1".equals(SystemProperties.get("sys.boot_completed", "0"))) break;
        try { Thread.sleep(1000); } catch (InterruptedException ignored) {}
    }
    oneChangerAuthorizeIfValid();
}, "OneChangerAuth").start();
```

---

# 4. Trình tự test an toàn

- Set `ONECHANGER_TIMEOUT_SEC = 180` (kernel)  
- Build & flash ROM đầy đủ  
- Boot vào Android  

Kiểm tra:
```bash
adb shell getprop ro.boot.onechanger_sig
adb shell ls -l /system/etc/.onechanger_blob
adb shell cat /sys/kernel/onechanger/rom_ok
```

ROM khác + boot.img của bạn → panic sau timeout.

---

# 5. Lưu ý bảo mật & giới hạn

- Người có root có thể `echo 1 > rom_ok` → có thể tăng cường bằng SELinux/token  
- Dev giỏi có thể rebuild kernel bỏ guard  
- Cơ chế này nhằm anti-reuse / anti-mix, không phải DRM tuyệt đối  

---

## Xuất PDF

Mở tài liệu → **Print (Ctrl+P)** → **Save as PDF**
