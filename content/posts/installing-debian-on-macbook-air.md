---
title: Macbook Air 에 Debian(wheezy) 설치하기
date: 2013-12-14T23:09:46+09:00
draft: false
author: foobargem
tags: ['linux', 'debian']
---

## Macbook Air

난 맥북 에어를 좋아한다.
가볍고, 13인치 화면에서 1400x900 해상도가 지원이 되며 전력관리가 우수하기 때문이다.
2010년에 처음으로 맥북 에어를 구입하고 OS X 를 사용하면서 리눅스가 너무너무 그리웠다.
예쁘고 멋진 인터페이스에 나름 안정적인 OS X 였지만
어느날 iTunes 를 업데이트한뒤 iTunes 의 버그로 시스템이 멈추는 현상이 생겼고
그 후로 Update 할때마다 시스템이 죽는게 아닐까 걱정이 되더라. :(

v10.6 > v10.7 > v10.8 > v10.9 까지 판올림을 해오면서 잘 써왔지만 이젠 리눅스로 돌아가기로 결심!


## 준비

Debian wheezy(7.2.0-amd64-netinst) iso 파일을 다운로드 받아서 USB drive 를 만든다.
또한 네트워크 [firmware (firmware-brcm80211)](http://ftp.kr.debian.org/debian/pool/non-free/f/firmware-nonfree/firmware-brcm80211_0.36+wheezy.1_all.deb) 도 다운로드 받아서 다른 USB 에 저장한다.
여유가 있을때 macbook air 에 설치할 수 있는 netinst 로 패키징을 해야겠다.

만들어진 USB 를 꼽고 부팅을 한다. 부팅할때 Option 키를 누르고 있어야 설치디스크 선택 화면이 나온다.
EFI 장치를 선택하고 부팅하면 Debian Installer 화면이 보인다.


## 설치

설치는 installer 인터페이스로 디스크 파티셔닝까지 진행을 한뒤 콘솔로 수동 작업을 해줘야 한다.

파티션은 200MB를 EFI system partition 으로, 30GB를 시스템의 루트(/) 파티션으로, 2GB를 스왑으로 잡았다.
남은 디스크는 OS 설치후 추가 하기 위해 놔두었다.

**Base system**

파티션을 잡은 뒤 Base system 설치가 진행되다가 실패한다. 원인은 정확하게 모르겠다.;;
alt+F2 를 눌러서 콘솔로 접속을 한다. 이시점이면 /target 디렉토리가 존재할것이다. 없다면 mkdir!!

```
# debootstrap --arch amd64 wheezy /target http://ftp.neowiz.com/debian
...
# mount -o bind /proc /target/proc
# mount -o bind /sys /target/sys
# mount -o bind /dev /target/dev
# mount -o bind /dev/pts /target/dev/pts
# chroot /target /bin/bash
# mkdir /boot/efi
# mount /dev/sda1 /boot/efi
```

**패키지 설치**

```
# modprobe efivars
# apt-get install initramfs-tools linux-image-3.2.0-4-amd64 grub-efi-amd64 firmware-brcm80211 wireless-tools wpasupplicant
# update-initramfs -u
# grub-install /dev/sda
# update-grub
# umount /dev/sda1
# exit
# umount /target/dev/pts
# umount /target/dev
# umount /target/sys
# umount /target/proc
# reboot
```

## 완료

이제 부팅이 되면 바로 grub 화면이 보이고 Debian 으로 부팅할 수 있게 된다.

## 남은것들

이제 Macbook Air 가 가지고 있는 하드웨어의 기능을 사용하기 위해 이런저런 삽질이 남았다.

## 참고

* [http://alexandre.delanoe.org/blog/archives/2012/10/index.html#e2012-10-05T09_04_38.txt](http://alexandre.delanoe.org/blog/archives/2012/10/index.html#e2012-10-05T09_04_38.txt)
* [http://dentifrice.poivron.org/laptops/macbookair3,1/](http://dentifrice.poivron.org/laptops/macbookair3,1/)
* [http://blog.felipe-alfaro.com/2006/09/19/installing-refit-on-the-hidden-efi-system-partition/](http://blog.felipe-alfaro.com/2006/09/19/installing-refit-on-the-hidden-efi-system-partition/)
* [https://wiki.debian.org/MacBookAir](https://wiki.debian.org/MacBookAir)
* [https://wiki.debian.org/InstallingDebianOn/Apple/MacBookAir/5-1](https://wiki.debian.org/InstallingDebianOn/Apple/MacBookAir/5-1)
