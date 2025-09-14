# Xây dựng Image cho **Raspberry Pi 4** với Yocto Project

Yocto Project là một công cụ mạnh mẽ cho phép tạo ra các hệ thống Linux tùy chỉnh cho các thiết bị nhúng. Tài liệu này hướng dẫn bạn xây dựng **image hoàn chỉnh cho Raspberry Pi 4** (1/2/4/8GB).

---

## 1. Chuẩn bị môi trường

### 1.1. Yêu cầu phần cứng và phần mềm
Trước khi bắt đầu, hãy đảm bảo bạn có:
- **Máy host**: Ubuntu (khuyến nghị chạy trên VMware/metal) với **≥ 100GB** trống.
- **Thiết bị**: **Raspberry Pi 4**.
- **Bộ nhớ**: thẻ **microSD ≥ 16GB** (khuyến nghị UHS-I/SDHC tốt).
- **Kết nối Internet**: tải package và source code.

### 1.2. Cài đặt các package cần thiết (Ubuntu/Debian)
```bash
sudo apt update
sudo apt install gawk wget git diffstat unzip texinfo gcc build-essential chrpath socat \
  cpio python3 python3-pip python3-pexpect xz-utils debianutils iputils-ping \
  python3-git python3-jinja2 python3-subunit zstd liblz4-tool file locales libacl1
```

> Nếu dùng Fedora/CentOS, bạn có thể cài các gói tương đương theo distro đó.

---

## 2. Thiết lập môi trường Yocto

### 2.1. Tải các meta-layer cần thiết
Trong ví dụ này dùng nhánh **dunfell** để đồng bộ với các hướng dẫn cổ điển (bạn có thể chọn nhánh khác đồng bộ giữa tất cả layer).

```bash
mkdir -p ~/yocto
cd ~/yocto

# Poky (Yocto reference distro)
git clone -b dunfell git://git.yoctoproject.org/poky
cd poky

# Layer cho Raspberry Pi
git clone -b dunfell git://git.yoctoproject.org/meta-raspberrypi

# OpenEmbedded layer bổ sung
git clone -b dunfell git://git.openembedded.org/meta-openembedded
```

### 2.2. Khởi tạo môi trường build
```bash
cd ~/yocto/poky
source oe-init-build-env
```
Sau lệnh trên, thư mục **build/** và các file cấu hình sẽ được tạo.

### 2.3. Cấu hình build

#### 2.3.1. Sửa `conf/local.conf`
Mở `build/conf/local.conf` và điều chỉnh:
```conf
# Chọn máy đích: Raspberry Pi 4 (ARM64)
MACHINE ??= "raspberrypi4-64"

# (Tuỳ chọn) tiết kiệm dung lượng, xóa work sau khi hoàn tất
INHERIT += "rm_work"

# (Tuỳ chọn) thêm công cụ tiện ích khi debug
EXTRA_IMAGE_FEATURES ?= "debug-tweaks"

# (Tuỳ chọn) Wi-Fi/Bluetooth & firmware thường dùng trên RPi4
# IMAGE_INSTALL:append = " linux-firmware-brcm43455 pi-bluetooth iw wpa-supplicant bluez5"
```

#### 2.3.2. Thêm meta-layer vào `conf/bblayers.conf`
Bổ sung **meta-raspberrypi** và **meta-openembedded/meta-oe** (chỉnh sửa đường dẫn theo máy của bạn):
```conf
BBLAYERS ?= " \
  /home/<user>/yocto/poky/meta \
  /home/<user>/yocto/poky/meta-poky \
  /home/<user>/yocto/poky/meta-yocto-bsp \
  /home/<user>/yocto/poky/meta-raspberrypi \
  /home/<user>/yocto/poky/meta-openembedded/meta-oe \
"
```
> Thay `</user>` bằng tên người dùng/đường dẫn thực tế.

---

## 3. Build và flash image

### 3.1. Tiến hành build
Ví dụ build image có giao diện **Sato** (bạn có thể dùng `core-image-minimal` nếu muốn console tối giản):
```bash
cd ~/yocto/poky
bitbake core-image-sato
```
> Thời gian build phụ thuộc CPU/RAM/SSD/Mạng. Lần đầu có thể lâu; các lần sau nhanh hơn nhờ sstate-cache.

### 3.2. Vị trí image đầu ra
Sau khi build xong, image được đặt tại:
```
~/yocto/poky/build/tmp/deploy/images/raspberrypi4-64/
```

### 3.3. Flash image vào thẻ nhớ

**(1) Xác định thiết bị thẻ nhớ**
```bash
lsblk
```

**(2) Giải nén image và ghi vào thẻ**
Ví dụ với file `.wic.bz2` sinh ra từ bước build:
```bash
cd ~/yocto/poky/build/tmp/deploy/images/raspberrypi4-64/

# Chọn đúng file .wic.bz2 của bạn
cp core-image-sato-raspberrypi4-64-*.wic.bz2 temp.wic.bz2
bzip2 -d temp.wic.bz2

# CẢNH BÁO: thay /dev/sdX bằng thiết bị thẻ nhớ CHÍNH XÁC! (ví dụ /dev/sdb)
sudo dd if=temp.wic of=/dev/sdX bs=4M status=progress conv=fsync
sync
```

> (Khuyến nghị) Nếu có file `.bmap`, dùng **bmaptool** sẽ nhanh hơn và an toàn hơn:
> ```bash
> sudo apt install bmap-tools
> sudo bmaptool copy core-image-sato-raspberrypi4-64-*.wic.bz2 /dev/sdX
> ```

### 3.4. Khởi động Raspberry Pi 4
- Cắm thẻ microSD vào **Raspberry Pi 4**, cấp nguồn.
- Kết nối **HDMI** (để xem đồ họa) hoặc **UART/SSH** (image console).
- Nếu bật `debug-tweaks`, thường có thể đăng nhập `root` (không mật khẩu) ở tty cục bộ.
- (Tuỳ chọn) Cấu hình Wi‑Fi với `wpa_supplicant`:
  ```bash
  # Ví dụ nhanh:
  cat <<'EOF' > /etc/wpa_supplicant/wpa_supplicant.conf
  ctrl_interface=/var/run/wpa_supplicant
  update_config=1
  country=US

  network={
      ssid="Ten_WIFI"
      psk="Mat_khau_WIFI"
      key_mgmt=WPA-PSK
  }
  EOF

  # Kích hoạt (tuỳ theo init system của image)
  wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant/wpa_supplicant.conf
  udhcpc -i wlan0  # hoặc dhclient, hoặc systemd-networkd tùy image
  ```

---

## 4. Kết luận
Với hướng dẫn này, bạn đã chuyển từ hướng dẫn dành cho **Raspberry Pi Zero W** sang **Raspberry Pi 4**:
- Thiết lập meta-layer phù hợp (**meta-raspberrypi**, **meta-openembedded**)
- Cấu hình `MACHINE ??= "raspberrypi4-64"`
- Build image (**core-image-sato** hoặc **core-image-minimal**)
- Flash và khởi động trên phần cứng thực

Bạn có thể tiếp tục tuỳ biến image, thêm package/firmware, hoặc tích hợp CI/CD theo nhu cầu dự án.
