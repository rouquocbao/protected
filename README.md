# OneChanger Kernel–Framework Guard (Anti Boot.img Reuse)

## Mục tiêu

- Đảm bảo **boot.img chỉ chạy được với ROM do chính bạn build**
- Ngăn người khác lấy `boot.img` của bạn flash sang ROM khác
- Không cần DRM mạnh, chỉ **chặn sử dụng sai mục đích**
- Quyết định **nằm ở kernel**, framework chỉ xác nhận

---

## Tổng quan kiến trúc

```
boot.img
 └─ kernel (inject androidboot.onechanger_sig)
     └─ /proc/cmdline
         └─ androidboot.onechanger_sig=<SHA256>

system.img
 └─ /system/etc/.onechanger_blob   (file bí mật)

framework
 └─ đọc androidboot.onechanger_sig
 └─ hash .onechanger_blob
 └─ nếu khớp → ghi /sys/kernel/onechanger/rom_ok = 1

kernel guard
 └─ chờ timeout
 └─ nếu rom_ok != 1 → panic / reboot
```

---

## PHẦN A — KERNEL GUARD DRIVER

### A1. File driver

```
kernel/samsung/exynos9810/drivers/android/onechanger_guard.c
```

### A2. Chức năng

- Tạo sysfs: `/sys/kernel/onechanger/rom_ok`
- Mặc định `rom_ok = 0`
- Sau `TIMEOUT_SEC`, nếu vẫn 0 → panic/reboot

### A3. Code driver (rút gọn)

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/kobject.h>
#include <linux/sysfs.h>
#include <linux/timer.h>
#include <linux/reboot.h>

#define ONECHANGER_TIMEOUT_SEC 60

static struct kobject *onechanger_kobj;
static int rom_ok;
static struct timer_list guard_timer;

static ssize_t rom_ok_show(struct kobject *kobj,
                           struct kobj_attribute *attr, char *buf)
{
    return sprintf(buf, "%d\n", rom_ok);
}

static ssize_t rom_ok_store(struct kobject *kobj,
                            struct kobj_attribute *attr,
                            const char *buf, size_t count)
{
    if (buf[0] == '1') rom_ok = 1;
    return count;
}

static struct kobj_attribute rom_ok_attr =
    __ATTR(rom_ok, 0660, rom_ok_show, rom_ok_store);

static void guard_timeout(struct timer_list *t)
{
    if (!rom_ok) panic("OneChanger guard");
}

static int __init onechanger_guard_init(void)
{
    onechanger_kobj = kobject_create_and_add("onechanger", kernel_kobj);
    sysfs_create_file(onechanger_kobj, &rom_ok_attr.attr);

    timer_setup(&guard_timer, guard_timeout, 0);
    mod_timer(&guard_timer, jiffies + ONECHANGER_TIMEOUT_SEC * HZ);
    return 0;
}

late_initcall(onechanger_guard_init);
```

### A4. Kconfig & Makefile

`drivers/android/Kconfig`
```
config ONECHANGER_GUARD
    bool "OneChanger ROM Guard"
    default y
```

`drivers/android/Makefile`
```
obj-$(CONFIG_ONECHANGER_GUARD) += onechanger_guard.o
```

Defconfig:
```
CONFIG_ONECHANGER_GUARD=y
```

---

## PHẦN B — INJECT androidboot.onechanger_sig TRONG KERNEL

⚠️ Bắt buộc cho Samsung Exynos

### B1. File

```
arch/arm64/kernel/setup.c
```

### B2. Code inject

```c
#include <linux/string.h>

#define ONECHANGER_SIG "<SHA256_HEX>"

static void __init onechanger_inject_cmdline(void)
{
    const char *key = "androidboot.onechanger_sig=";

    if (strstr(boot_command_line, key)) return;

    if (strlen(boot_command_line) + strlen(key) +
        strlen(ONECHANGER_SIG) + 2 >= COMMAND_LINE_SIZE) return;

    strlcat(boot_command_line, " ", COMMAND_LINE_SIZE);
    strlcat(boot_command_line, key, COMMAND_LINE_SIZE);
    strlcat(boot_command_line, ONECHANGER_SIG, COMMAND_LINE_SIZE);
}
```

Gọi trong `setup_arch()`:

```c
setup_machine_fdt(__fdt_pointer);
onechanger_inject_cmdline();
parse_early_param();
```

Kiểm tra:
```
cat /proc/cmdline | grep onechanger
```

---

## PHẦN C — SYSTEM FILE BÍ MẬT

Tạo file:
```
head -c 64 /dev/urandom > device/samsung/<codename>/onechanger/.onechanger_blob
```

Copy vào system:

```make
PRODUCT_COPY_FILES += \
 device/samsung/<codename>/onechanger/.onechanger_blob:$(TARGET_COPY_OUT_SYSTEM)/etc/.onechanger_blob
```

---

## PHẦN D — FRAMEWORK AUTHORIZE

### D1. File

```
frameworks/base/services/java/com/android/server/SystemServer.java
```

### D2. Logic

- Đọc `androidboot.onechanger_sig`
- Hash `/system/etc/.onechanger_blob`
- Nếu khớp → ghi `/sys/kernel/onechanger/rom_ok`

### D3. Gọi sớm

```java
new Thread(() -> {
    try { Thread.sleep(25000); } catch (Exception ignored) {}
    oneChangerAuthorizeIfValid();
}).start();
```

---

## PHẦN E — TEST

- Boot đúng ROM → không reboot
- Boot.img gắn ROM khác → kernel panic sau timeout

---

## Ghi chú

- Không dùng `ro.boot.*`
- Không dùng `BOARD_KERNEL_CMDLINE`
- Không dùng `init.rc`
- Kernel là điểm quyết định cuối

