---
title: LG 그램 랩탑에서 Linux kernel 빌드하기
date: 2015-07-04T11:06:36+09:00
draft: false
author: foobargem
tags: ['linux', 'kernel', 'firmware', 'wireless']
---

회사에서 지급받은 랩탑은 LG 그램이다.
OS는 LMDE(Linux Mint Debian Edition) 을 설치해서 잘 쓰고 있다.

처음에는 i915 driver 버그로 LED 밝기조절이 안되고 밝기조절 기능을 활성화 하는경우 시스템이 멈추는 문제가 있었다.
삽질을 하던중 KLDP 에서 커널 업데이트하면 해결된다는 정보를 알게 되었다. [https://kldp.org/node/151205](https://kldp.org/node/151205) 
읽고서 바로 3.18.8 커널로 빌드를 했고 문제는 깔끔하게 해결되었다. :D

커널 업데이트 후 wireless device (Intel Wireless 7620) 가 인식이 안되었는데 원인은 firmware 버전이 맞지 않아서였다.
그래서　[linux wireless driver website](https://wireless.wiki.kernel.org/en/users/Drivers/iwlwifi) 에서
커널버전에 맞는 firmware 를 다운로드 받아 설치를 했고 잘 동작함을 확인했다.

이번주에 오랜만에 커널을 4.1.1 버전으로 업데이트하고 wireless device 가 동작을 안해서
이전의 firmware 설치한 기억을 잊어버리고 dmesg, log 를 추적해서 firmware 버전이 안맞는것이
원인이라는 것을 찾아서 다시 4.1 버전 이상에 맞는 firmware 를 설치했다.

또 삽질하는 것을 방지하고자 커널 빌드과정을 정리한다.


### prepare

##### Kernel source

https://www.kernel.org/

##### Wireless device firmware

설치하려는 커널 버전에 맞는 firmware 다운로드

```
# tar xvzf iwlwifi-7260-ucode-25.30.13.0.tgz
# cp iwlwifi-7260-13.ucode /lib/firmware
```

### kernel build

##### Configuration

설정 하나하나를 살펴보고 활성/모듈/비활성을 선택해야 한다.
그러나 지금버전 커널의 config 파일를 기반으로 설정을 하면 좀 수월하다.
그래서 /boot/config-$(uname -r) 파일을 복사해온다.


```
# cd /usr/src
# tar xvf linux-4.1.1.tar.xz
...
# cd linux-4.1.1
# cp /boot/config-$(uname -r) /usr/src/linux-4.1.1/.config
# make menuconfig
// Load -> .config 후 전체 설정을 확인
```


##### Build

빌드 수행시 -jN (N은 cpu core 개수의 1/2정도) 옵션으로 병렬로 할수 있다.
make install 이 수행되면 자동으로 update-grub 가 수행된다.


```
# make -j2
# make modules_install install
```

##### Done

```
# reboot
```

### 참고

* [https://www.kernel.org/](https://www.kernel.org/)
* [http://kernelnewbies.org/KernelBuild](http://kernelnewbies.org/KernelBuild)
* [https://wireless.wiki.kernel.org/en/users/Drivers/iwlwifi](https://wireless.wiki.kernel.org/en/users/Drivers/iwlwifi)
* [https://kldp.org/node/151205](https://kldp.org/node/151205)
