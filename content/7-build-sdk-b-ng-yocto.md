# Hướng dẫn Build & Sử dụng **SDK Yocto** cho **Raspberry Pi 4 (raspberrypi4-64)**

Tài liệu này chuyển thể từ bài viết SDK tổng quan và **được đi�u chỉnh cho Raspberry Pi 4 (ARM64)**.  
Bạn sẽ biết cách **tạo SDK chuẩn / eSDK**, **cài đặt**, **thiết lập môi trư�ng**, và **biên dịch thử** một ứng dụng với toolchain chéo.

---

## 1) SDK trong Yocto là gì?
**SDK (Software Development Kit)** cung cấp:

- **Cross toolchain**: biên dịch trên host (x86_64) ra binary cho **RPi4 (ARM64)**.  
- **Sysroots**: header & thư viện tương ứng.  
- **Script thiết lập môi trư�ng**: đặt biến `PATH`, `CC`, `CXX`, `PKG_CONFIG_PATH`, v.v.

> Với **Raspberry Pi 4**, khuyến nghị dùng máy 64-bit và **MACHINE = `raspberrypi4-64`** để nhận **toolchain aarch64**.

---

## 2) Chuẩn bị dự án Yocto cho RPi 4
Tạo workspace và clone các layer (ví dụ nhánh *dunfell* để đồng bộ hướng dẫn):

```bash
mkdir -p ~/yocto && cd ~/yocto

# Poky (dunfell ví dụ)
git clone -b dunfell git://git.yoctoproject.org/poky
cd poky

# Layer Raspberry Pi + OpenEmbedded
git clone -b dunfell git://git.yoctoproject.org/meta-raspberrypi
git clone -b dunfell git://git.openembedded.org/meta-openembedded

# Thiết lập môi trư�ng build
source oe-init-build-env
```

Cập nhật **`conf/bblayers.conf`** để thêm layer:
```conf
BBLAYERS ?= " \
  /home/<user>/yocto/poky/meta \
  /home/<user>/yocto/poky/meta-poky \
  /home/<user>/yocto/poky/meta-yocto-bsp \
  /home/<user>/yocto/poky/meta-raspberrypi \
  /home/<user>/yocto/poky/meta-openembedded/meta-oe \
"
```
> �ổi `<user>` cho phù hợp đư�ng dẫn máy bạn.

Cập nhật **`conf/local.conf`** (phần quan tr�ng):
```conf
MACHINE ?= "raspberrypi4-64"   # ARM64 cho Raspberry Pi 4
PACKAGE_CLASSES ?= "package_rpm"
# (tuỳ ch�n) tối ưu dung lượng: INHERIT += "rm_work"
```

---

## 3) Các cách **tạo SDK**

### Cách A — Thêm toolchain vào **image** (phát triển **trực tiếp trên target**)
Trong `conf/local.conf`:
```conf
EXTRA_IMAGE_FEATURES ?= "tools_sdk"
```
Rồi build image:
```bash
bitbake core-image-minimal
```
> Cách này cài **gcc/make/headers** vào **Raspberry Pi 4**. Phù hợp demo nhanh trên target nhưng **không khuyến nghị** cho sản xuất do giới hạn tài nguyên.

### Cách B — Tạo **Standard SDK** (khuyến nghị)
Tạo **file cài đặt SDK** trên host:
```bash
bitbake core-image-minimal -c populate_sdk
```
Sau khi xong, tìm trong:
```
build/tmp/deploy/sdk/
```
Bạn sẽ thấy file dạng:
```
poky-glibc-x86_64-core-image-minimal-*-raspberrypi4-64-toolchain-<ver>.sh
```
> Tên thật có thể khác đôi chút theo cấu hình. Cứ dựa **mẫu có `raspberrypi4-64`** là đúng.

### Cách C — Tạo **Extensible SDK (eSDK)**
```bash
bitbake core-image-minimal -c populate_sdk_ext
```
**eSDK** bao gồm devtool và metadata, tiện cho việc **sửa recipe, thử nghiệm, cập nhật** ngay trên host.

---

## 4) Cài đặt & Thiết lập môi trư�ng SDK trên host

### 4.1 Cài đặt
Chạy script `.sh` trong thư mục `deploy/sdk`:
```bash
cd build/tmp/deploy/sdk/
./poky-glibc-x86_64-core-image-minimal-*-raspberrypi4-64-toolchain-*.sh
```
- Ch�n nơi cài (mặc định thư�ng là `/opt/poky/<ver>`).

### 4.2 Nạp biến môi trư�ng (quan tr�ng)
Tìm **script environment** cho **aarch64** rồi `source`:
```bash
# gợi ý tìm nhanh
ls /opt/poky/*/environment-setup-*aarch64*

# ví dụ (tên thực tế có thể khác nhau)
source /opt/poky/3.1.33/environment-setup-aarch64-poky-linux
```
Kiểm tra nhanh:
```bash
echo "$CC"
$CC --version
```
Bạn nên thấy toolchain **`aarch64-poky-linux-gcc`**.

---

## 5) Thử nghiệm SDK bằng ứng dụng C đơn giản

### 5.1 Viết mã nguồn
Tạo file `~/hello_SDK.c`:
```c
#include <stdio.h>

int main(void) {
    printf("thanhdeptrai: hello SDK for Raspberry Pi 4 (aarch64)!\n");
    return 0;
}
```

### 5.2 Biên dịch bằng cross toolchain
```bash
# �ảm bảo bạn đã 'source environment-setup-aarch64-poky-linux'
$CC -o ~/hello_SDK_app ~/hello_SDK.c
```

### 5.3 Chép và chạy trên target
**Cách 1 — SSH tới Raspberry Pi 4 thực tế**
```bash
# thay 192.168.x.x là IP của Pi 4
scp ~/hello_SDK_app root@192.168.x.x:/home/root/
ssh root@192.168.x.x
/home/root/hello_SDK_app
```
**Cách 2 — Nếu bạn đang dùng QEMU aarch64**
```bash
scp ~/hello_SDK_app root@<qemu-ip>:/home/root/
ssh root@<qemu-ip> /home/root/hello_SDK_app
```

Kết quả mong đợi:
```
thanhdeptrai: hello SDK for Raspberry Pi 4 (aarch64)!
```

---

## 6) Mẹo & Tùy biến SDK

- **Ch�n image khác**: SDK phụ thuộc vào image (ví dụ `core-image-sato` nếu cần đồ h�a).
- **Thêm gói vào SDK** (khi tạo):
  ```conf
  # conf/local.conf hoặc distro/layer
  TOOLCHAIN_TARGET_TASK += " packageA packageB "
  TOOLCHAIN_HOST_TASK   += " nativesdk-cmake nativesdk-pkgconfig "
  ```
- **eSDK + devtool**: tiện cho quy trình chỉnh sửa recipe, `devtool modify`, `devtool build`, `devtool update-recipe` ngay trên host.

---

## 7) Lợi ích chính khi dùng SDK
- **Nhanh & g�n**: Không phải rebuild toàn bộ image khi chỉnh ứng dụng.  
- **Nhất quán**: Toolchain/headers giống target → ít lỗi “khác môi trư�ng�.  
- **Thân thiện IDE**: Dùng VS Code, CLion… với toolchain aarch64.  
- **Tiết kiệm tài nguyên target**: Build trên host, không làm nặng Pi 4.

---

## 8) Tóm tắt nhanh các lệnh quan tr�ng

```bash
# Chuẩn bị & cấu hình cho RPi4 (ARM64)
source oe-init-build-env
# (sửa local.conf: MACHINE ?= "raspberrypi4-64")

# Tạo SDK chuẩn
bitbake core-image-minimal -c populate_sdk

# Tạo eSDK
bitbake core-image-minimal -c populate_sdk_ext

# Cài đặt SDK
cd build/tmp/deploy/sdk/
./poky-glibc-x86_64-core-image-minimal-*-raspberrypi4-64-toolchain-*.sh

# Nạp môi trư�ng cross-compile
source /opt/poky/*/environment-setup-aarch64-poky-linux

# Build app mẫu
$CC -o ~/hello_SDK_app ~/hello_SDK.c
```

---

### Ghi chú tương thích
- Ví dụ nhánh **dunfell (3.1.x)** phổ biến với cộng đồng RPi, nhưng bạn có thể dùng bản mới hơn (kirkstone/nanbield...) miễn layer tương thích.  
- Tên file SDK `.sh` và script `environment-setup-*` **có thể khác** theo **distro/TCLIBC/MACHINE**; dùng `ls` để dò đúng tên.

Chúc bạn build SDK thành công và code “mượt� cho **Raspberry Pi 4**! 🚀
