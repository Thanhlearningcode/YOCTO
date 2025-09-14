# Xây dựng Image cho Raspberry Pi 4 với Yocto Project

## 1. Yêu cầu phần cứng và phần mềm

### 1.1. Yêu cầu phần cứng
Để quá trình build mượt mà, nên chuẩn bị:
- **CPU**: Tối thiểu 4 nhân (quad-core) hoặc cao hơn
- **RAM**: Tối thiểu 8GB (khuyến nghị 16GB+)
- **Ổ đĩa**: Trống tối thiểu 50GB (khuyến nghị 100GB+). **SSD** sẽ nhanh hơn HDD
- **Thiết bị**: Raspberry Pi 4 (1/2/4/8GB), thẻ **microSD ≥ 16GB**, nguồn tốt, cáp HDMI/USB-serial (tuỳ nhu cầu)

### 1.2. Yêu cầu phần mềm
- **Hệ điều hành host**: Ubuntu 20.04 LTS trở lên hoặc Debian 10+. (Fedora/CentOS dùng được nhưng có thể cần cấu hình thêm)

### 1.3. Cấu hình tham khảo
- Hệ điều hành: Ubuntu 22.04  
- RAM: 8GB  
- Ổ cứng: 100GB SSD  
- CPU: 4 nhân, 2 luồng, 3.2GHz  

---

## 2. Chuẩn bị môi trường

### 2.1. Cài đặt các công cụ cần thiết

**Ubuntu / Debian**
```bash
sudo apt-get update
sudo apt-get install gawk wget git-core diffstat unzip texinfo gcc-multilib \
  build-essential chrpath socat cpio python3 python3-pip python3-pexpect \
  xz-utils debianutils iputils-ping libsdl1.2-dev xterm vim zstd liblz4-tool
Fedora

bash
Copy
Edit
sudo dnf install gawk make wget tar bzip2 gzip python3 unzip perl patch \
  diffutils diffstat git cpp gcc gcc-c++ glibc-devel texinfo chrpath \
  ccache perl-Data-Dumper perl-Text-ParseWords perl-Thread-Queue perl-bignum socat \
  python3-pexpect findutils which file cpio python3-pip xz SDL-devel xterm zstd lz4
3. Tải mã nguồn Yocto Project
3.1. Chuẩn bị thư mục & clone
bash
Copy
Edit
mkdir -p ~/yocto
cd ~/yocto

# Poky (chọn nhánh cho phù hợp, ví dụ dunfell để đồng bộ với hướng dẫn)
git clone -b dunfell git://git.yoctoproject.org/poky
cd poky

# Layer cho Raspberry Pi + OE
git clone -b dunfell git://git.yoctoproject.org/meta-raspberrypi
git clone -b dunfell git://git.openembedded.org/meta-openembedded
3.2. Cấu trúc chính trong Poky (nhắc lại nhanh)
bitbake/ – công cụ BitBake

meta/ – layer lõi (OE-Core)

meta-poky/ – distro tham chiếu Poky

meta-yocto-bsp/ – BSP mẫu

oe-init-build-env – script thiết lập môi trường build

4. Xây dựng image cho Raspberry Pi 4
Raspberry Pi 4 là ARM64, dùng MACHINE: raspberrypi4-64.

4.1. Chuyển nhánh (nếu cần)
bash
Copy
Edit
cd ~/yocto/poky
git checkout dunfell
4.2. Khởi tạo môi trường build
bash
Copy
Edit
source oe-init-build-env
Sau lệnh này sẽ có thư mục build/ chứa cấu hình.

4.3. Tùy chỉnh file cấu hình
4.3.1. conf/local.conf
Mở poky/build/conf/local.conf và cập nhật:

conf
Copy
Edit
# Tiết kiệm dung lượng (xóa work sau build xong)
INHERIT += "rm_work"

# Định dạng gói (mặc định là RPM, có thể giữ nguyên)
PACKAGE_CLASSES ?= "package_rpm"

# Quan trọng: target board
MACHINE ?= "raspberrypi4-64"

# Một số tiện ích khuyến nghị (tuỳ chọn)
EXTRA_IMAGE_FEATURES ?= "debug-tweaks"

# (Tuỳ chọn) Thêm Wi-Fi/Bluetooth tools & firmware
IMAGE_INSTALL:append = " linux-firmware-brcm43455 pi-bluetooth iw wpa-supplicant bluez5"
4.3.2. conf/bblayers.conf
Thêm meta-raspberrypi và meta-openembedded/meta-oe:

conf
Copy
Edit
BBLAYERS ?= " \
  /home/<user>/yocto/poky/meta \
  /home/<user>/yocto/poky/meta-poky \
  /home/<user>/yocto/poky/meta-yocto-bsp \
  /home/<user>/yocto/poky/meta-raspberrypi \
  /home/<user>/yocto/poky/meta-openembedded/meta-oe \
"
Thay <user> bằng tên người dùng/đường dẫn thực tế.

4.4. Tiến hành build
Bạn có thể build image tối thiểu hoặc có giao diện:

bash
Copy
Edit
# Image console tối thiểu
bitbake core-image-minimal

# (Tuỳ chọn) Image có giao diện Sato
# bitbake core-image-sato
4.5. Vị trí image đầu ra
Sau khi build xong, image nằm tại:

swift
Copy
Edit
build/tmp/deploy/images/raspberrypi4-64/
Thường có các định dạng như .wic.bz2, .rpi-sdimg, v.v.

4.6. Ghi image vào thẻ microSD
Xác định thiết bị thẻ nhớ:

bash
Copy
Edit
lsblk
Giải nén & ghi image (ví dụ dùng .wic.bz2):

bash
Copy
Edit
cd ~/yocto/poky/build/tmp/deploy/images/raspberrypi4-64/

# Chọn đúng file .wic.bz2 sinh ra
cp core-image-minimal-raspberrypi4-64-*.wic.bz2 ./temp.wic.bz2
bzip2 -d temp.wic.bz2

# CẢNH BÁO: thay /dev/sdX bằng thiết bị thẻ nhớ chính xác!
sudo dd if=temp.wic of=/dev/sdX bs=4M status=progress conv=fsync
sync
(Tuỳ chọn) Nếu có .bmap, dùng bmaptool sẽ nhanh hơn:

bash
Copy
Edit
sudo apt-get install bmap-tools
sudo bmaptool copy core-image-minimal-raspberrypi4-64-*.wic.bz2 /dev/sdX
4.7. Khởi động Raspberry Pi 4
Cắm thẻ microSD vào Pi 4, cấp nguồn.

Kết nối HDMI để theo dõi boot (nếu image đồ hoạ) hoặc UART/SSH (nếu image console).

Với debug-tweaks, tài khoản root thường không mật khẩu khi truy cập terminal trực tiếp.

Cấu hình Wi-Fi (nếu cần):

bash
Copy
Edit
# ví dụ đơn giản với wpa_cli / wpa_supplicant
# hoặc tạo /etc/wpa_supplicant/wpa_supplicant.conf
5. Kết luận
Bạn đã:

Chuyển từ môi trường QEMU sang Raspberry Pi 4 (raspberrypi4-64)

Thêm meta-raspberrypi, cấu hình local.conf & bblayers.conf phù hợp

Build image và flash vào thẻ microSD để chạy trên phần cứng thực

Yocto cho phép tuỳ biến rất cao. Từ đây, bạn có thể thêm package, firmware, UI (Sato/Wayland), hoặc tích hợp CI/CD theo nhu cầu.