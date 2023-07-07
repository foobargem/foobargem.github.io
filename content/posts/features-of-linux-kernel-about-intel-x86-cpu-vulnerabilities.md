---
title: Intel x86 CPU 취약점에 대한 Linux kernel 의 기능들
date: 2019-09-13T22:32:42+09:00
draft: false
author: foobargem
tags: ['linux', 'kernel', 'intel']
---

몇년 전부터 x86 CPU 의 취약점들이 - [Meltdown and Spectre](https://meltdownattack.com/) 등 - 지속적으로 발견 및 보고되고 있다.

최근  [MaaS - Metal as a Service](https://maas.io/) 를 검토하던중 우연히 /proc/cpuinfo 의 항목중 
**<strong>bugs</strong>** 라는게 생긴것과 sysfs 에 cpu 에 대한 취약점 디렉토리 **/sys/devices/system/cpu/vulnerabilities/** 가 추가 된 것을 알게 되었다.


## /proc/cpuinfo 의 bugs flag

CPU 버그에 대한 workarounds 를 감지 또는 적용했음을 표시해준다.
무려 5년전(2014년 6월)에 [commit](https://github.com/torvalds/linux/commit/80a208bd3948aceddf0429bd9f9b4cd858d526df) 된 기능이다.
회사에서는 주로 CentOS7 을 다루다 보니 발견할 수 없었다.
왜냐면 CentOS7 의 base repository 로 배포되는 커널(3.10)은 bugs flag 기능이 빠져 있기 때문이다.

그래서 elrepo-kernel repository 의 커널을 설치해서 시험을 해보니 /proc/cpuinfo 에 bugs flag 가 표시됨을 확인할 수 있었다.


* CentOS 7
    * base repository 의 kernel: 미반영
    * elrepo-kernel repository 의 kernel: 반영


### CentOS 7.5 (with elrepo-kernel)

```
[root ~]# uname -r
5.2.13-1.el7.elrepo.x86_64

[root ~]# grep bugs /proc/cpuinfo 
bugs        : cpu_meltdown spectre_v1 spectre_v2 spec_store_bypass l1tf mds swapgs
```


## /sys/devices/system/cpu/vulnerabilities/*

또 한가지는 sysfs 에 cpu 에 대한 취약점 디렉토리가 추가가 된 것이다.
2018년 1월에 [commit](https://github.com/torvalds/linux/commit/87590ce6e373d1a5401f6539f0c59ef92dd924a9) 되었다.
CentOS7 은 7.4 기준으로 base repository 의 커널에 반영이 되었다.


### CentOS 7 의 base repository 의 kernel

* 3.10.0-862 >= (7.5)
* 3.10.0-693.19.1.el7 (7.4)


##### CentOS 7.5

```
[root ~]# uname -r
3.10.0-862.14.4.el7.x86_64

[root ~]# grep . /sys/devices/system/cpu/vulnerabilities/*
/sys/devices/system/cpu/vulnerabilities/l1tf:Mitigation: PTE Inversion
/sys/devices/system/cpu/vulnerabilities/meltdown:Mitigation: PTI
/sys/devices/system/cpu/vulnerabilities/spec_store_bypass:Vulnerable
/sys/devices/system/cpu/vulnerabilities/spectre_v1:Mitigation: Load fences, __user pointer sanitization
/sys/devices/system/cpu/vulnerabilities/spectre_v2:Mitigation: Full retpoline
```

##### CentOS 7.4

```
[root ~]$ uname -r
3.10.0-693.21.1.el7.x86_64

[root ~]$ grep . /sys/devices/system/cpu/vulnerabilities/* 
/sys/devices/system/cpu/vulnerabilities/meltdown:Mitigation: PTI
/sys/devices/system/cpu/vulnerabilities/spectre_v1:Mitigation: Load fences
/sys/devices/system/cpu/vulnerabilities/spectre_v2:Mitigation: Full retpoline
```

##### CentOS 7.3

```
[root ~]$ uname -r
3.10.0-514.26.2.el7.x86_64

[root ~]$ grep . /sys/devices/system/cpu/vulnerabilities/*
grep: /sys/devices/system/cpu/vulnerabilities/*: No such file or directory
```


## ToDo

* 어떻게 bugs flag 를 표시하는가?
* sysfs 의 cpu 취약점 항목을 어떻게 표시하는가?


## 참조

* [https://www.reddit.com/r/linux/comments/8k3x3b/til_proccpuinfo_shows_architecture_bugs_such_as/](https://www.reddit.com/r/linux/comments/8k3x3b/til_proccpuinfo_shows_architecture_bugs_such_as/)
* [x86/cpufeature: Add bug flags to /proc/cpuinfo](https://github.com/torvalds/linux/commit/80a208bd3948aceddf0429bd9f9b4cd858d526df)
* [sysfs/cpu: Add vulnerability folder](https://github.com/torvalds/linux/commit/87590ce6e373d1a5401f6539f0c59ef92dd924a9)
