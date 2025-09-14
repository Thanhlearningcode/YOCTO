# Devtool trong Yocto: Tạo, chỉnh sửa và kiểm thử recipe nhanh

Trong các bài viết trước, chúng ta đã cùng tìm hiểu cách tạo một **layer** đơn giản trên Yocto và chạy giả lập trên **QEMU**. Tiếp nối chuỗi bài viết về **Yocto Project**, hôm nay chúng ta sẽ khám phá **Devtool** – một công cụ hỗ trợ mạnh mẽ giúp các nhà phát triển dễ dàng **tạo**, **chỉnh sửa** và **kiểm thử** các recipe trong Yocto.

Devtool được thiết kế để **đơn giản hóa quy trình** làm việc với Yocto. Thay vì thao tác thủ công, Devtool cho phép nhanh chóng tạo, sửa và cập nhật recipe; đồng thời build & kiểm thử gói phần mềm trước khi tích hợp vào image cuối cùng.

---

## 1. Tổng quan về Devtool

Devtool dùng để:

- Tạo mới, chỉnh sửa và cập nhật **recipe**  
- **Build** và kiểm thử **package**  
- Tích hợp thay đổi vào **image** nhanh chóng

Thiết lập môi trường trước khi dùng:

```bash
~/yocto/poky$ source oe-init-build-env
```

Xem các subcommand có sẵn:

```bash
~/yocto/poky/build$ devtool -h
```

Một số subcommand phổ biến:
- `add` – Tạo recipe mới
- `modify` – Chỉnh sửa source/recipe hiện có (đưa về workspace)
- `build` – Build recipe hoặc image
- `deploy-target` – Triển khai nhanh gói lên target qua SSH
- `finish` – Hoàn tất, đưa recipe từ workspace vào layer
- `reset` – Hủy thay đổi trong workspace
- `upgrade` – Nâng cấp recipe
- `update-recipe` – Cập nhật recipe theo thay đổi trong source

---

## 2. Các lệnh phổ biến của Devtool

### 2.1. `devtool add`

Tạo recipe mới từ source có sẵn (local hoặc remote). Sau khi chạy, Devtool tạo **workspace** chứa source và recipe.

**Ví dụ**: tạo recipe từ GitHub

```bash
~/yocto/poky/build$ devtool add hello-devtool https://github.com/thanhdeptrai097/Yocto-For-Dummies.git --srcbranch custom-devtool
```

Cấu trúc workspace rút gọn:

```
workspace/
├── appends     # Lưu các thay đổi khi modify/upgrade
├── conf        # Lưu các file cấu hình
├── README
├── recipes     # Lưu các recipe mới được tạo
└── sources     # Lưu source code
```

Đọc source vừa fetch:

```bash
~/yocto/poky/build$ cat workspace/sources/hello-devtool/hello-devtool.c
```

```c
#include <stdio.h>

int main()
{
    printf("thanhdeptrai: hello devtool!\n");
    return 0;
}
```

Recipe tự sinh `hello-devtool_git.bb` (rút gọn):

```bitbake
# Recipe created by recipetool
# (cần chỉnh sửa LICENSE/LIC_FILES_CHKSUM cho phù hợp)
LICENSE = "CLOSED"
LIC_FILES_CHKSUM = ""

SRC_URI = "git://github.com/thanhdeptrai097/Yocto-For-Dummies.git;protocol=https;branch=custom-devtool"

# Modify these as desired
PV = "1.0+git${SRCPV}"
SRCREV = "9684263baeb96af50c937da46a63065c42703963"

S = "${WORKDIR}/git"

# NOTE: no Makefile found, unable to determine what needs to be done
do_configure () { : }
do_compile () { : }
do_install () { : }
```

---

### 2.2. `devtool build`

Build recipe để kiểm thử nhanh package.

Chỉnh `hello-devtool_git.bb` tối thiểu để biên dịch & cài:

```bitbake
LICENSE = "CLOSED"
LIC_FILES_CHKSUM = ""

SRC_URI = "git://github.com/thanhdeptrai097/Yocto-For-Dummies.git;protocol=https;branch=custom-devtool"

PV = "1.0+git${SRCPV}"
SRCREV = "9684263baeb96af50c937da46a63065c42703963"

S = "${WORKDIR}/git"

do_compile () {
    ${CC} ${S}/hello-devtool.c ${LDFLAGS} -o ${S}/hello-devtool
}

do_install () {
    install -d ${D}${bindir}
    install -m 0755 ${S}/hello-devtool ${D}${bindir}
}
```

Build thử:

```bash
~/yocto/poky/build$ devtool build hello-devtool
```

Ví dụ kết quả thành công:

```
Sstate summary: Wanted 0 Found 0 Missed 0 Current 124 (0% match, 100% complete)
NOTE: Executing Tasks
NOTE: hello-devtool: compiling from external source tree ...
NOTE: Tasks Summary: Attempted 531 tasks ... all succeeded.
```

Thêm vào image để test trên target:

```conf
# build/conf/local.conf
IMAGE_INSTALL_append = " hello-devtool"
```

Build & chạy QEMU:

```bash
bitbake core-image-minimal
runqemu qemux86-64 nographic
```

Kiểm tra trên target:

```bash
root@arm:~# /usr/bin/hello-devtool
thanhdeptrai: hello devtool!
```

---

### 2.3. `devtool deploy-target`

Triển khai nhanh recipe đã build lên target qua **SSH** (không cần rebuild image).

**Chuẩn bị**: image có SSH server (dropbear/openssh). Ví dụ:

```conf
# build/conf/local.conf
IMAGE_INSTALL_append = " openssh"
# hoặc
# EXTRA_IMAGE_FEATURES += "ssh-server-openssh"
```

Khởi chạy image rồi deploy:

```bash
# QEMU ví dụ
runqemu qemux86-64 nographic

# Từ host (IP ví dụ 192.168.7.2)
~/yocto/poky/build$ devtool deploy-target hello-devtool root@192.168.7.2
```

Chạy thử trên target:

```bash
root@arm:~# /usr/bin/hello-devtool
thanhdeptrai: hello devtool!
```

---

### 2.4. `devtool finish`

Khi đã ổn, chuyển recipe từ workspace vào một layer của bạn:

```bash
~/yocto/poky/build$ devtool finish -f hello-devtool ../meta-thanhdeptrai/recipes-apps/
```

Trước khi thêm:

```
meta-thanhdeptrai/recipes-apps
└── hello-world
    ├── files
    │   └── hello-world.c
    └── hello-world.bb
```

Sau khi thêm:

```
meta-thanhdeptrai/recipes-apps
├── hello-devtool
│   └── hello-devtool_git.bb
└── hello-world
    ├── files
    │   └── hello-world.c
    └── hello-world.bb
```

---

### 2.5. `devtool modify`

Đưa source/recipe hiện có vào workspace để sửa:

```bash
~/yocto/poky/build$ devtool modify hello-devtool
```

Sửa source, ví dụ:

```c
int main()
{
    printf("thanhdeptrai: hello devtool!\n");
    printf("This line edit in devtool modify.\n");
    return 0;
}
```

Build lại & kiểm thử như trên.

---

### 2.6. `devtool reset`

Hủy bỏ mọi thay đổi trong workspace cho recipe:

```bash
devtool reset hello-devtool
```

---

### 2.7. `devtool update-recipe`

Khi source đã thay đổi (đã commit), cập nhật lại recipe.

Sửa code:

```c
int main()
{
    printf("thanhdeptrai: hello devtool!\n");
    printf("This line edit in devtool modify.\n");
    printf("This line is changed in devtool update-recipe\n");
    return 0;
}
```

Commit & cập nhật:

```bash
~/yocto/poky/build/workspace/sources/hello-devtool$ git add hello-devtool.c
~/yocto/poky/build/workspace/sources/hello-devtool$ git commit -m "Demo devtool update-recipe"
~/yocto/poky/build$ devtool update-recipe hello-devtool
```

Kiểm tra recipe trong layer:

```
meta-thanhdeptrai/recipes-apps/hello-devtool
├── hello-devtool
│   └── 0001-Demo-devtool-update-recipe.patch
└── hello-devtool_git.bb
```

> File patch chứa chi tiết các thay đổi đã ghi nhận.

---

## 3. Kết luận

**Devtool** là công cụ mạnh mẽ và linh hoạt giúp:
- Tạo mới/chỉnh sửa recipe nhanh chóng
- Build & test gói trước khi bake vào image
- Deploy nhanh lên target qua SSH
- Dễ dàng “finish” để đưa recipe vào layer

Từ đây, bạn có thể kết hợp Devtool với **CI/CD**, chuyển sang phần cứng thực (vd. Raspberry Pi 4 với `MACHINE = "raspberrypi4-64"` + `meta-raspberrypi`), hoặc duy trì vòng lặp phát triển rất ngắn nhờ `devtool deploy-target`.
