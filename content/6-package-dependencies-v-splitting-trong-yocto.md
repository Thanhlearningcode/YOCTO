# Package Dependencies và Package Splitting trong Yocto

Trong bài viết này, chúng ta sẽ cùng khám phá hai khái niệm quan trọng trong **Yocto**:  
**Package Dependencies** (phụ thuộc của gói) và **Package Splitting** (phân chia gói).  
Đây là các khái niệm cốt lõi giúp tối ưu hoá hiệu suất và tăng tính linh hoạt khi xây dựng hệ thống nhúng hoặc phân phối Linux.

Khi làm việc với Yocto, việc quản lý phụ thuộc gói và chia gói đóng vai trò thiết yếu trong việc **giảm dung lượng**, **cải thiện hiệu quả**, và **nâng cao khả năng bảo trì** hệ thống.

---

## 1. Package Dependencies

**Package Dependencies** là các yêu cầu cần thiết để một gói hoạt động bình thường. Nếu một gói cần thư viện hoặc các gói khác, Yocto sẽ đảm bảo các yêu cầu này được thiết lập trước (hoặc sau) khi build package. Có hai nhóm phụ thuộc chính:

- **Build time dependencies**: Phụ thuộc cần trong **quá trình build** → dùng biến `DEPENDS`.
- **Run time dependencies**: Phụ thuộc cần **khi chạy** → dùng biến `RDEPENDS_${PN}`.

### 1.1. Build time dependencies

Tại thời điểm build, một số thư viện hoặc package khác là cần thiết để đảm bảo quá trình build diễn ra thành công. Trong Yocto, biến **`DEPENDS`** dùng để chỉ định các gói/thư viện cần có trong **sysroots** trước khi `do_compile()` chạy.

#### 1.1.1. Thiết lập môi trường

```bash
~/yocto/poky$ source oe-init-build-env
```

#### 1.1.2. Tạo mã nguồn và recipe

Tạo thư mục chứa recipe **`demo-dependencies`** trong `meta-thanhdeptrai/recipes-apps`, sau đó thêm source & recipe theo cấu trúc:

```bash
~/yocto/poky/meta-thanhdeptrai/recipes-apps$ mkdir demo-dependencies
~/yocto/poky/meta-thanhdeptrai/recipes-apps/demo-dependencies$ tree
.
├── files
│   ├── depend_buildtime.c
│   ├── depend_runtime.c
│   ├── package_split.c
│   └── Makefile
└── demo-dependencies.bb
```

**`depend_buildtime.c`**

```c
#include <stdio.h>
#include <curl/curl.h>

int main(void)
{
    CURL *curl;
    CURLcode res;

    printf("thanhdeptrai\n");
    curl = curl_easy_init();
    if (curl) {
        printf("Build dependency check: OK\n");
        curl_easy_cleanup(curl);
    }

    return 0;
}
```

**`depend_runtime.c`**

```c
#include <stdio.h>
#include <stdlib.h>

int main(void)
{
    printf("thanhdeptrai\n");
    int status = system("bash -c 'echo This is executed by bash at runtime!'");
    if (status == -1) {
        perror("Failed to run bash command");
        return 1;
    }

    return 0;
}
```

**`package_split.c`**

```c
#include <stdio.h>

int main(void)
{
    printf("thanhdeptrai: package splitting!\n");
    return 0;
}
```

**`Makefile`**

```makefile
CC = gcc
CFLAGS = -Wall

all: depend_buildtime depend_runtime package_split

depend_buildtime: depend_buildtime.c
	$(CC) depend_buildtime.c $(CFLAGS) -lcurl -o depend_buildtime

depend_runtime: depend_runtime.c
	$(CC) depend_runtime.c $(CFLAGS) -o depend_runtime

package_split: package_split.c
	$(CC) package_split.c $(CFLAGS) -o package_split

clean:
	rm -rf depend_buildtime package_split depend_runtime
```

**`demo-dependencies.bb`**

```bitbake
DESCRIPTION = "Demo program showcasing package dependencies and splitting"
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302"

S = "${WORKDIR}"
SRC_URI = "file://depend_buildtime.c \
           file://depend_runtime.c \
           file://package_split.c \
           file://Makefile"

# DEPENDS = "curl"

EXTRA_OEMAKE = "CC='${CC}' CFLAGS='${CFLAGS} -Wl,--hash-style=gnu'"

do_compile() {
    oe_runmake
}

do_install() {
    install -d ${D}${bindir}
    install -m 0755 depend_buildtime ${D}${bindir}/depend_buildtime
    install -m 0755 depend_runtime ${D}${bindir}/depend_runtime
    install -m 0755 package_split ${D}${bindir}/package_split
}
```

#### 1.1.3. Build recipe

```bash
~/yocto/poky$ bitbake demo-dependencies
```

**Lỗi thường gặp** (thiếu thư viện curl):

```
| depend_buildtime.c:2:10: fatal error: curl/curl.h: No such file or directory
|     2 | #include <curl/curl.h>
|       |          ^~~~~~~~~~~~~
```

**Nguyên nhân**: `DEPENDS` chưa khai báo `curl`, nên sysroots không có headers/lib phục vụ build.  
**Khắc phục**: Bỏ comment `DEPENDS = "curl"` trong `.bb`, rồi build lại.

Khi build thành công sẽ thấy:

```
NOTE: Tasks Summary: Attempted 3353 tasks of which 3328 didn't need to be rerun and all succeeded.
```

#### 1.1.4. Thêm libcurl vào image (runtime)

```conf
# build/conf/local.conf
IMAGE_INSTALL_append = " libcurl"
```

### 1.2. Run time dependencies

**Run time dependencies** là các gói, thư viện hoặc công cụ cần để chương trình **chạy**. Khai báo bằng `RDEPENDS_${PN}`.

**Ví dụ lỗi** với `depend_runtime` do gọi `bash` qua `system()` nhưng image **không có** gói `bash`:

```
sh: bash: not found
```

**Khắc phục** trong recipe:

```bitbake
RDEPENDS_${PN} = "bash"
```

Và trong image (nếu muốn có sẵn):

```conf
IMAGE_INSTALL_append = " bash"
```

Build lại image:

```bash
~/yocto/poky$ bitbake core-image-minimal
~/yocto/poky$ runqemu qemux86-64 nographic
```

Chạy lại test:

```
root@qemux86-64:~# depend_runtime
thanhdeptrai
This is executed by bash at runtime!
```

---

## 2. Package Splitting

Trong nhiều trường hợp, **không cần** cài đặt toàn bộ nội dung của một package vào image. Để tối ưu dung lượng và cho phép quản lý linh hoạt, Yocto hỗ trợ **chia package** thành các **subpackages** dùng các biến `PACKAGES` và `FILES_*`.

### 2.1. Recipe mẫu tách gói

```bitbake
DESCRIPTION = "Demo program showcasing package dependencies and splitting"
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302"

S = "${WORKDIR}"
SRC_URI = "file://depend_buildtime.c \
           file://depend_runtime.c \
           file://package_split.c \
           file://Makefile"

DEPENDS = "curl"
RDEPENDS_${PN} = "bash"
RDEPENDS_package-split = "bash"

EXTRA_OEMAKE = "CC='${CC}' CFLAGS='${CFLAGS} -Wl,--hash-style=gnu'"

# Khai báo các package con
PACKAGES =+ "demo-tools package-split"

# Gộp depend_buildtime và depend_runtime vào package demo-tools
FILES_demo-tools = "${bindir}/depend_buildtime \
                    ${bindir}/depend_runtime"

# Khai báo package riêng cho package_split
FILES_package-split = "${bindir}/package_split"

do_compile() {
    oe_runmake
}

do_install() {
    install -d ${D}${bindir}
    install -m 0755 depend_buildtime ${D}${bindir}/depend_buildtime
    install -m 0755 depend_runtime ${D}${bindir}/depend_runtime
    install -m 0755 package_split ${D}${bindir}/package_split
}
```

### 2.2. Giải thích

- `PACKAGES =+ "demo-tools package-split"`: khai báo thêm **hai** subpackages.
- `FILES_demo-tools`: gom **`depend_buildtime`** & **`depend_runtime`** vào cùng một subpackage **demo-tools**.
- `FILES_package-split`: tách **`package_split`** thành gói riêng **package-split**.
- `RDEPENDS_package-split = "bash"`: nếu subpackage cần phụ thuộc runtime riêng, có thể khai báo tại đây.
- Khi build, BitBake sẽ sinh ra nhiều gói (ipk/deb/rpm) tương ứng với từng subpackage.

### 2.3. Cài đặt có chọn lọc

Chỉ muốn cài **demo-tools** (không cài `package-split`) vào image:

```conf
IMAGE_INSTALL_append = " demo-tools"
```

Build & chạy:

```bash
bitbake core-image-minimal
runqemu qemux86-64 nographic
```

Kiểm thử:

```
root@qemux86-64:~# depend_buildtime
thanhdeptrai
Build dependency check: OK

root@qemux86-64:~# depend_runtime
thanhdeptrai
This is executed by bash at runtime!

root@qemux86-64:~# package_split
-sh: package_split: command not found
```

---

## 3. Kết luận

- **Package Dependencies** đảm bảo các gói/thư viện cần thiết có mặt đúng lúc (build-time qua `DEPENDS`, runtime qua `RDEPENDS_*`).  
- **Package Splitting** cho phép chia nhỏ gói thành các **subpackage** để **tối ưu dung lượng** và **cài đặt linh hoạt**.  
- Áp dụng đúng `DEPENDS`, `RDEPENDS_*`, `PACKAGES`, `FILES_*` giúp hệ thống **ổn định**, **nhẹ**, và **dễ bảo trì**.
