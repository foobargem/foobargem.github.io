---
title: v4.19 kernel 에서 xfs 의 barrier/nobarrier mount option 이 제거됨
date: 2020-03-31T17:36:02+09:00
draft: false
author: foobargem
tags: ['linux', 'xfs']
---


kernel 변화에 따른 성능변화를 측정하기 위해서
CentOS 7.4 / 3.10.0-693.21.1.el7.x86_64 기반의 시스템에
elrepo 의 kernel(5.5.x) 을 설치하고 재부팅을 했다.


그런데, 오랜 시간이 지나도 ssh 접속이 되지 않았다.
(beta 장비라서 OOB 구성이 안되어 있음)
어쩔수 없이 IDC 엔지니어에게 요청을 해서
다시 3.10.0-693.21.1.el7.x86_64 로 부팅을 했다.


동일 형상의 VM 을 가지고 elrepo 의 kernel 을 설치하고
console 화면을 확인해보니 부팅 중에 root filesystem 을 마운트 하지 못하여
maintenance mode 가 되어 있는게 원인이었다.


/etc/fstab 을 확인해보고 검색을 해보니
kernel 4.19 부터는 barrier, nobarrier 옵션이 사라졌다고 한다.

/etc/fstab 에서 nobarrier 옵션을 제거하니 문제없이 부팅 성공!!


## 참고

* [kernel commit - xfs: remove deprecated barrier/nobarrier mount](https://github.com/torvalds/linux/commit/1c02d502c20809a2a5f71ec16a930a61ed779b81)
* [xfs manpage](http://man7.org/linux/man-pages/man5/xfs.5.html)
