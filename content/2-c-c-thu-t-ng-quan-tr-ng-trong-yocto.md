# Các thuật ngữ cơ bản trong Yocto (Raspberry Pi 4)

Bài viết này đặc biệt phù hợp với những bạn mới bắt đầu tiếp cận Yocto và muốn nắm rõ các thuật ngữ thường được sử dụng, với ví dụ áp dụng cho **Raspberry Pi 4 (64-bit)**.

---

## 1. Image
Image là một recipe định nghĩa những thành phần sẽ có trong root file system (RFS). Yocto cung cấp một số image mặc định trong Poky, chẳng hạn như `core-image-minimal`, đây là một image tối thiểu để hệ thống có thể khởi động.

Ví dụ: `core-image-minimal.bb`  

```bitbake
SUMMARY = "A small image just capable of allowing a device to boot."
IMAGE_INSTALL = "packagegroup-core-boot ${CORE_IMAGE_EXTRA_INSTALL}"
IMAGE_LINGUAS = " "
LICENSE = "MIT"
inherit core-image
IMAGE_ROOTFS_SIZE ?= "8192"
IMAGE_ROOTFS_EXTRA_SPACE_append = "${@bb.utils.contains("DISTRO_FEATURES", "systemd", " + 4096", "" ,d)}"
```

- **IMAGE_INSTALL**: Các gói cài vào RFS.  
- **CORE_IMAGE_EXTRA_INSTALL**: Thêm gói tùy biến.  
- **IMAGE_ROOTFS_SIZE**: Kích thước rootfs.  
- **inherit core-image**: Kế thừa class định nghĩa image cơ bản.  

---

## 2. Machine file
Machine mô tả phần cứng cụ thể để build image. Với Raspberry Pi 4, file cấu hình nằm trong `meta-raspberrypi/conf/machine/raspberrypi4-64.conf`.

Ví dụ rút gọn:  

```conf
#@TYPE: Machine
#@NAME: Raspberry Pi 4 Model B
#@DESCRIPTION: Machine configuration for Raspberry Pi 4 64-bit

PREFERRED_PROVIDER_virtual/kernel ?= "linux-raspberrypi"
UBOOT_MACHINE = "rpi_4_defconfig"
KERNEL_IMAGETYPE = "Image"

SERIAL_CONSOLES ?= "115200;ttyAMA0"

MACHINE_FEATURES += "wifi bluetooth vc4graphics"
```

- **UBOOT_MACHINE**: chỉ định cấu hình U-Boot cho Pi 4.  
- **KERNEL_IMAGETYPE**: dùng `Image` (ARM64) thay vì `bzImage`.  
- **SERIAL_CONSOLES**: UART mặc định qua `ttyAMA0`.  
- **MACHINE_FEATURES**: có thêm WiFi, Bluetooth, GPU VideoCore IV.  

---

## 3. Distro
Distro là một bản phân phối hoàn chỉnh. Mặc định Yocto dùng `poky`, bạn có thể kiểm tra trong `conf/local.conf`:  

```conf
DISTRO ?= "poky"
```

File cấu hình distro `poky.conf` định nghĩa:  

- **DISTRO**: tên bản phân phối.  
- **DISTRO_FEATURES**: các tính năng bật sẵn (như opengl, wayland, vulkan).  
- **PREFERRED_VERSION_linux-yocto**: kernel mặc định, có thể override bằng kernel của Pi (`linux-raspberrypi`).  

---

## 4. Local.conf
`local.conf` nằm trong `build/conf/` dùng để tùy chỉnh cho máy build:  

- **MACHINE**:  
  ```conf
  MACHINE ?= "raspberrypi4-64"
  ```
- **IMAGE_FSTYPES**: định dạng image, ví dụ `.wic.bz2`.  
- **BB_NUMBER_THREADS** và **PARALLEL_MAKE**: điều chỉnh luồng CPU để tăng tốc build.  
- **TMPDIR**: thư mục chứa output build.  

---

## 5. Kết luận
Trong bài viết này, chúng ta đã tìm hiểu các thuật ngữ cơ bản trong Yocto, áp dụng cho **Raspberry Pi 4**. Việc hiểu rõ Image, Machine, Distro và Local.conf sẽ giúp bạn dễ dàng tùy chỉnh và xây dựng hệ điều hành nhúng theo nhu cầu.  

---
