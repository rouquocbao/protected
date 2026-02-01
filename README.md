OneChanger Guard – Hướng dẫn đầy đủ (Kernel →
Framework)
Mục tiêu: boot.img chỉ hoạt động với ROM của bạn. Nếu boot.img được dùng với ROM khác
(framework không hợp lệ), kernel sẽ panic sau N giây.
 CẢNH BÁO QUAN TRỌNG - Kernel panic có rủi ro bootloop cứng. - Khi test, luôn để
timeout dài (180–300s). - Đảm bảo máy có Recovery / Download Mode để cứu

1. Kiến trúc tổng thể
///
Kernel (Guard)
 ├─ Tạo sysfs: /sys/kernel/onechanger/rom_ok
 ├─ Start timer (N giây)
 └─ Nếu rom_ok != 1 → panic()
Framework (SystemServer)
 ├─ Chạy muộn khi Android đã lên
 ├─ Tính hash file bí mật trong /system
 ├─ So với hash từ boot (kernel)
 └─ Nếu hợp lệ → ghi 1 vào sysfs
///

ROM khác không có code framework → không ghi sysfs → kernel panic.

PHẦN A — KERNEL: ONECHANGER GUARD
A1. Tạo driver kernel
Đường dẫn
drivers/android/onechanger_guard.c

Mã nguồn
///
#include <linux/module.h>
#include <linux/init.h>
#include <linux/kobject.h>
#include <linux/sysfs.h>
#include <linux/workqueue.h>
#include <linux/jiffies.h>
#include <linux/panic.h>
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
