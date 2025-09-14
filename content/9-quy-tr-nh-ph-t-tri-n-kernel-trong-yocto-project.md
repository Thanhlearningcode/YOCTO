# Phát triển Kernel cho **Raspberry Pi 4** với Yocto Project (core-image-sato)

Phát triển và tùy chỉnh kernel Linux là một nhiệm vụ quan tr�ng trong hệ thống nhúng. Yocto Project cung cấp quy trình có cấu trúc để quản lý, vá (patch), tùy biến và build kernel cho thiết bị. Tài liệu này hướng dẫn **thêm một kernel module mẫu** để đi�u khiển LED qua GPIO27 trên **Raspberry Pi 4 (BCM2711, 64-bit)** và tích hợp vào image **`core-image-sato`**.

> 💡 Yêu cầu: bạn đã có môi trư�ng Yocto với `meta-raspberrypi` và đang build cho **MACHINE = raspberrypi4-64**.

---

## 1) Tổng quan Kernel trong Yocto

Trong Yocto, kernel được build thông qua **recipe** (đặt trong `meta/*/recipes-kernel/linux/`). Có vài nguồn/kernel base thư�ng dùng:

* **Kernel chuẩn** từ kernel.org
* **Kernel nhà cung cấp** (ví dụ Raspberry Pi Foundation)
* **Kernel tùy chỉnh** (fork/nhánh riêng)

Yocto khuyến khích quản lý thay đổi kernel bằng **patch** (git) và/hoặc **config fragment**, nhằm đảm bảo khả năng tái tạo (reproducibility).

---

## 2) Quy trình tích hợp module mẫu (GPIO27)

Mục tiêu: thêm module **`mgpio`** để set GPIO27 **HIGH** khi nạp module, và **LOW** khi remove.

### 2.1) Cấu trúc module

```
mgpio/
├── Kconfig
├── Makefile
└── mgpio.c
```

**`mgpio.c`**

```c
#include <linux/module.h>
#include <linux/gpio.h>

#define GPIO_NUMBER_27  27
#define LOW  0
#define HIGH 1

static int __init mgpio_driver_init(void)
{
    /* Cấp quy�n và set GPIO27 làm output */
    gpio_request(GPIO_NUMBER_27, "gpio_27");
    gpio_direction_output(GPIO_NUMBER_27, LOW);

    /* Set GPIO27 HIGH */
    gpio_set_value(GPIO_NUMBER_27, HIGH);
    pr_info("mgpio: GPIO27 set to HIGH, status: %d\n", gpio_get_value(GPIO_NUMBER_27));
    return 0;
}

static void __exit mgpio_driver_exit(void)
{
    /* Trả GPIO27 v� LOW trước khi r�i */
    gpio_set_value(GPIO_NUMBER_27, LOW);
    gpio_free(GPIO_NUMBER_27);
    pr_info("mgpio: GPIO27 set to LOW (module exit)\n");
}

module_init(mgpio_driver_init);
module_exit(mgpio_driver_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("thanhdeptrai");
MODULE_DESCRIPTION("GPIO Driver for Raspberry Pi 4 (BCM2711)");
```

**`Makefile`**

```make
EXTRA_CFLAGS = -Wall
obj-$(CONFIG_MGPIO) = mgpio.o
```

**`Kconfig`**

```Kconfig
menu "mgpio device driver"

config MGPIO
    tristate "mgpio device driver"
    depends on ARM64 || ARM
    help
      Simple demo driver: set GPIO27 HIGH on load, LOW on unload.

endmenu
```

> 📌 Lưu ý: Trên kernel mới API GPIO legacy có thể đã bị cảnh báo/khuyên dùng gpiod. � đây dùng API cũ cho mục đích demo tối giản.

---

### 2.2) Thiết lập môi trư�ng

```bash
# Trong cây nguồn Poky
source oe-init-build-env

# �ảm bảo MACHINE là Raspberry Pi 4 (64-bit)
# poky/build/conf/local.conf
# MACHINE ?= "raspberrypi4-64"
```

---

### 2.3) Trích xuất nguồn kernel bằng devtool

```bash
cd ~/yocto/poky/build
devtool modify virtual/kernel
```

* Mã nguồn kernel sẽ xuất hiện ở:
  `build/workspace/sources/linux-raspberrypi`

Tạo thư mục module và thêm 3 file ở trên:

```bash
cd workspace/sources/linux-raspberrypi
mkdir -p drivers/mgpio
# đặt Kconfig, Makefile, mgpio.c vào drivers/mgpio/
```

---

### 2.4) Khai báo module với kernel tree

**Sửa `drivers/Makefile`** (thêm dòng này cuối file):

```make
obj-$(CONFIG_MGPIO) += mgpio/
```

**Sửa `drivers/Kconfig`** (thêm dòng này cuối file):

```Kconfig
source "drivers/mgpio/Kconfig"
```

---

### 2.5) Bật cấu hình module trong defconfig

�ối với **Raspberry Pi 4 / ARM64**, cập nhật defconfig tương ứng (thư�ng là **`arch/arm64/configs/bcm2711_defconfig`**):

```diff
+ CONFIG_MGPIO=m
```

> Nếu dự án của bạn dùng defconfig tên khác, hãy chỉnh đúng file dưới `arch/arm64/configs/…`.
> (Cách thay thế nâng cao: dùng **config fragment** thay vì sửa defconfig trực tiếp.)

---

### 2.6) Tạo patch từ thay đổi

```bash
cd workspace/sources/linux-raspberrypi
git add drivers/mgpio/ drivers/Kconfig drivers/Makefile arch/arm64/configs/bcm2711_defconfig
git commit -m "Add mgpio driver (GPIO27) for Raspberry Pi 4 (BCM2711)"
git format-patch -1
```

Patch vừa tạo có dạng `0001-Add-mgpio-driver-….patch`.

---

### 2.7) �ưa patch vào meta-layer của bạn

Tạo thư mục trong layer tùy chỉnh, ví dụ `meta-thanhdeptrai`:

```bash
mkdir -p ../../meta-thanhdeptrai/recipes-kernel/linux/linux-raspberrypi
cp 0001-Add-mgpio-driver-*.patch ../../meta-thanhdeptrai/recipes-kernel/linux/linux-raspberrypi/
```

Tạo **`linux-raspberrypi_%.bbappend`**:

```
meta-thanhdeptrai/
└── recipes-kernel/
    └── linux/
        ├── linux-raspberrypi/
        │   └── 0001-Add-mgpio-driver-*.patch
        └── linux-raspberrypi_%.bbappend
```

**`linux-raspberrypi_%.bbappend`**

```bitbake
FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"

# Thêm patch vào SRC_URI
SRC_URI += "file://0001-Add-mgpio-driver-*.patch"

# Tự động nạp module khi boot (tùy ch�n)
KERNEL_MODULE_AUTOLOAD += "mgpio"
```

---

### 2.8) Build kernel (và/hoặc image)

```bash
# Chỉ build kernel
bitbake virtual/kernel

# Hoặc build image mục tiêu (tự kéo kernel mới)
bitbake core-image-sato
```

Sau khi build image, flash ra thẻ microSD như bình thư�ng. Nếu dùng `KERNEL_MODULE_AUTOLOAD`, module **mgpio** sẽ tự load khi khởi động (GPIO27 sẽ ở mức HIGH). Nếu không, có thể thử thủ công:

```bash
# Trên thiết bị:
modprobe mgpio
dmesg | tail -n 50
# ... kiểm tra log "GPIO27 set to HIGH ..."
rmmod mgpio
```

---

## 3) Kết luận

Bạn vừa thêm một **kernel module tùy chỉnh** vào kernel **Raspberry Pi 4 (BCM2711, ARM64)** trong Yocto bằng quy trình tiêu chuẩn:

* Sửa code trong **workspace** qua `devtool modify virtual/kernel`
* Commit và **tạo patch**
* Thêm **.bbappend** + **patch** vào **meta-layer** riêng
* Build lại **kernel/image**

Cách làm này đảm bảo thay đổi được quản lý sạch sẽ, **tái tạo được** và dễ chia sẻ trong nhóm. Từ ví dụ `mgpio`, bạn có thể mở rộng để tích hợp driver thật của dự án, thêm config fragments, hoặc chia nh� patch theo best practices.
