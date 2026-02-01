OneChanger Guard – Hướng dẫn đầy đủ (Kernel → Framework)Tài liệu này hướng dẫn thiết lập cơ chế bảo vệ để boot.img chỉ hoạt động với bản ROM của bạn. Nếu được dùng với ROM khác, kernel sẽ tự động thực hiện lệnh panic sau một khoảng thời gian $N$ giây.[!CAUTION]
CẢNH BÁO QUAN TRỌNG
* Kernel panic có rủi ro gây bootloop cứng.
* Khi test, luôn để timeout dài (180–300s).
* Đảm bảo máy có Recovery / Download Mode để cứu.1. Kiến trúc tổng thể Kernel (Guard) Tạo sysfs: /sys/kernel/onechanger/rom_ok.Khởi động bộ đếm giờ (N giây).Nếu rom_ok != 1 thì thực hiện panic().Framework (SystemServer) Chạy khi Android đã khởi động xong.Tính toán hash của file bí mật trong /system.So sánh với hash từ bootloader/kernel truyền vào.Nếu hợp lệ, ghi giá trị 1 vào sysfs.Kết quả: ROM không chính chủ sẽ không có code framework để xác thực, dẫn đến không ghi được vào sysfs và gây ra Kernel Panic.PHẦN A — KERNEL: ONECHANGER GUARD A1. Tạo driver kernel Đường dẫn: drivers/android/onechanger_guard.c C#include <linux/module.h> [cite: 24]
#include <linux/init.h> [cite: 25]
#include <linux/kobject.h> [cite: 26]
#include <linux/sysfs.h> [cite: 28]
#include <linux/workqueue.h> [cite: 29]
#include <linux/jiffies.h> [cite: 29]
#include <linux/panic.h> [cite: 30]
#include <linux/mutex.h> [cite: 30]

#define ONECHANGER_TIMEOUT_SEC 180 [cite: 31]

static struct kobject *onechanger_kobj; [cite: 32]
static struct delayed_work panic_work; [cite: 33]
static bool rom_ok = false; [cite: 33, 72]
static DEFINE_MUTEX(onechanger_lock); [cite: 34]

static void onechanger_panic_work(struct work_struct *work) { [cite: 35]
    bool ok; [cite: 37]
    mutex_lock(&onechanger_lock); [cite: 38]
    ok = rom_ok; [cite: 39]
    mutex_unlock(&onechanger_lock); [cite: 40]
    if (!ok) panic("OneChangerGuard: ROM not authorized"); [cite: 41, 43]
}

static ssize_t rom_ok_show(struct kobject *kobj, struct kobj_attribute *attr, char *buf) { [cite: 44, 45, 47]
    bool ok; [cite: 49]
    mutex_lock(&onechanger_lock); [cite: 50]
    ok = rom_ok; [cite: 51]
    mutex_unlock(&onechanger_lock); [cite: 52]
    return scnprintf(buf, PAGE_SIZE, "%d\n", ok ? 1 : 0); [cite: 53]
}

static ssize_t rom_ok_store(struct kobject *kobj, struct kobj_attribute *attr, const char *buf, size_t count) { [cite: 54, 55, 56]
    if (buf[0] == '1') { [cite: 58]
        mutex_lock(&onechanger_lock); [cite: 59]
        rom_ok = true; [cite: 60]
        mutex_unlock(&onechanger_lock); [cite: 61]
        cancel_delayed_work_sync(&panic_work); [cite: 62]
    }
    return count; [cite: 64]
}

static struct kobj_attribute rom_ok_attr = __ATTR(rom_ok, 0660, rom_ok_show, rom_ok_store); [cite: 67, 68]

static int __init onechanger_guard_init(void) { [cite: 69]
    int ret; [cite: 71]
    onechanger_kobj = kobject_create_and_add("onechanger", kernel_kobj); [cite: 75]
    if (!onechanger_kobj) return -ENOMEM; [cite: 76, 77]
    ret = sysfs_create_file(onechanger_kobj, &rom_ok_attr.attr); [cite: 78]
    if (ret) { [cite: 78]
        kobject_put(onechanger_kobj); [cite: 80]
        return ret; [cite: 81]
    }
    INIT_DELAYED_WORK(&panic_work, onechanger_panic_work); [cite: 82]
    schedule_delayed_work(&panic_work, ONECHANGER_TIMEOUT_SEC * HZ); [cite: 83]
    pr_info("OneChanger Guard loaded, timeout=%ds\n", ONECHANGER_TIMEOUT_SEC); [cite: 84]
    return 0; [cite: 84]
}

late_initcall(onechanger_guard_init); [cite: 85]
MODULE_LICENSE("GPL"); [cite: 86]
A2. Makefile & Kconfig drivers/android/Makefile: obj-$(CONFIG_ONECHANGER_GUARD) += onechanger_guard.o drivers/android/Kconfig: Thêm config ONECHANGER_GUARD với giá trị mặc định là y.Defconfig: CONFIG_ONECHANGER_GUARD=y.PHẦN B — FILE BÍ MẬT TRONG SYSTEM Tạo file bí mật: Dùng lệnh head -c 64 /dev/urandom để tạo blob ngẫu nhiên.Copy vào system image: Thêm đường dẫn vào PRODUCT_COPY_FILES trong build script.Tính hash: Dùng sha256sum để lấy mã Hex của file.Gắn vào boot: Thêm mã hash vào BOARD_BOOTCONFIG trong BoardConfig.mk.PHẦN C — FRAMEWORK (SYSTEMSERVER) Framework sẽ thực hiện so sánh mã hash thực tế của file bí mật với mã hash mong đợi được lưu trong thuộc tính hệ thống ro.boot.onechanger_sig. Nếu khớp, hàm writeSysfsOne sẽ ghi '1' vào /sys/kernel/onechanger/rom_ok để xác thực ROM.4. Trình tự test an toàn Set ONECHANGER_TIMEOUT_SEC = 180 trong kernel.Build và flash ROM đầy đủ.Boot vào Android và dùng ADB để kiểm tra các thông số ro.boot.onechanger_sig, file bí mật và giá trị trong rom_ok.5. Lưu ý bảo mật & giới hạn Người dùng có quyền Root có thể can thiệp vào sysfs, có thể khắc phục bằng SELinux.Cơ chế này nhằm chống việc sử dụng trái phép (anti-reuse), không phải giải pháp bảo mật tuyệt đối.
