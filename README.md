# OneChanger Guard – Hướng dẫn đầy đủ (Kernel → Framework)

**Mục tiêu:** `boot.img` chỉ hoạt động với ROM của bạn. Nếu boot.img được dùng với ROM khác (framework không hợp lệ), kernel sẽ panic sau **N giây**.

---

## ⚠️ CẢNH BÁO QUAN TRỌNG

- Kernel panic có rủi ro **bootloop cứng**
- Khi test, luôn để **timeout dài (180–300s)**
- Đảm bảo máy có **Recovery / Download Mode** để cứu

---

# 1️⃣ Kiến trúc tổng thể

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
*(giữ nguyên như trước)*

---

# PHẦN B — FILE BÍ MẬT TRONG SYSTEM
*(giữ nguyên như trước)*

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

## C3. Helper & authorize logic (trong class SystemServer)

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

# PHẦN D — FRAMEWORK AUTHORIZE (BẢN GỌN)

## D1. File

```
frameworks/base/services/java/com/android/server/SystemServer.java
```

## D2. Logic

- Đọc `androidboot.onechanger_sig`
- Hash `/system/etc/.onechanger_blob`
- Nếu khớp → ghi `/sys/kernel/onechanger/rom_ok`

## D3. Gọi sớm

```java
new Thread(() -> {
    try { Thread.sleep(25000); } catch (Exception ignored) {}
    oneChangerAuthorizeIfValid();
}).start();
```

---

# 4️⃣ Trình tự test an toàn
*(giữ nguyên như trước)*

---

# 5️⃣ Lưu ý bảo mật & giới hạn
*(giữ nguyên như trước)*
