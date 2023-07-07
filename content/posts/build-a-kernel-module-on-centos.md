---
title: "Build a kernel module on CentOS"
date: 2016-06-11T16:56:53+09:00
draft: false
author: foobargem
tags: ["linux", "centos", "kernel"]
---
			
## 발단

bcache (Linux kernel block layer cache) 를 적용하려고 알아보니 CentOS 7 의 커널은 bcache module 이 disabled 였다. 그래서 kernel source 를 받아서 bcache module 을 build 시도했다.

경험하면 할 수록 CentOS 는 서버환경에서 별로이다. - Debian 추천!! :)

## Build 환경 구축

### required packages 설치

```
# yum install rpm-build redhat-rpm-config asciidoc hmaccalc perl-ExtUtils-Embed pesign xmlto \
              audit-libs-devel binutils-devel elfutils-devel elfutils-libelf-devel \
              ncurses-devel newt-devel numactl-devel pciutils-devel python-devel zlib-devel
```

### kernel source 다운로드

http://vault.centos.org/7.N.YYMM/updates/Source/SPackages/ 에서 kernel-$(uname -r).src.rpm 파일을 다운로드 및 설치한다. 설치가 되면 rpmbuild 디렉토리 이하에 관련 files 이 복사되어 있다.

```
# rpm -i http://vault.centos.org/7.2.1511/updates/Source/SPackages/kernel-3.10.0-327.18.2.el7.src.rpm 2&gt;&amp;1 | grep -v exist
# cd ~rpmbuild/SPECS
# rpmbuild -bp --target=$(uname -m) kernel.spec
# cd ~rpmbuild/BUILD
```

### module build

**edit .config**

```
# cd ~rpmbuild/BUILD/kernel-3.10.0-327.13.1.el7/linux-3.10.0-327.13.1.el7.x86_64
# vim .config
```

**.config**

```
CONFIG_BCACHE=m
```

**build**

```
# make modules_install SUBDIRS=drivers/md/bcache
// modules_install: bcache.ko 복사,depmod -a 수행

# modinfo bcache
filename:       /lib/modules/3.10.0-327.13.1.el7.x86_64/kernel/drivers/md/bcache/bcache.ko
author:         Kent Overstreet &lt;kent.overstreet@gmail.com&gt;
license:        GPL
license:        GPL
author:         Kent Overstreet &lt;koverstreet@google.com&gt;
rhelversion:    7.2
srcversion:     744AC1548F61C54B1445EF6
depends:
vermagic:       3.10.0 SMP mod_unload modversions
```

### 모듈 적재

```
# modprobe bcache
insmod: error inserting 'bcache.ko': -1 Invalid module format

# dmesg
...
bcache: no symbol version for module_layout
```

모듈 build 할때 signing 이 안되어서 발생하는 문제.  (signing 하는 방법은 추후에 업데이트 할것!!) 우선은 `-force` 로!!

```
# modprobe -f bcache
# lsmod | grep bcache
bcache                209287  0
```

## 적용후

동일한 조건에서 dm-cache 와 bcache 를 fio 로 시험을 해보았다. 결과는 dm-cache 가 random read, sequential read 가 월등히 빨랐다. CentOS 에서 bcache 모듈을 제외한게 dm-cache 가 좋다고 판단해서 일까..

## 참고

* https://wiki.centos.org/HowTos/Custom_Kernel
* https://wiki.centos.org/HowTos/I_need_the_Kernel_Source
* https://www.kernel.org/doc/Documentation/module-signing.txt
