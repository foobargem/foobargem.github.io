---
title: stunnel with ulimit
date: 2016-04-13T21:01:22+09:00
draft: false
author: foobargem
tags: ['stunnel', 'ulimit', 'systemd']
---

## 배경

그림과 같이 L4 에 2대의 Swift proxy server 를 바인딩 한 환경에서
https 서비스를 제공하기 위해 L4 에서 SSL Offloading 설정을 했다.

![swift cluster architecture](/images/201604/swift_cluster_architecture.png)


##### 이슈

L4의 SSL Offloading 성능이 떨어짐. 1Gbps 환경에서 download 시 **20MB/s**

속도 문제를 해결하기 위해서 swift proxy server 에서 SSL 처리를 시도해 보았다.
SSL termination 을 위해서 nginx 와 stunnel 을 시험해 보았다.

##### 시험방법

**100 sessions** 로 동시 다운로드 시도시 시스템 리소스 사용율 측정(Load avg.)

* 시험결과
    * nginx: 5 (1 min)
    * stunnel: 0.5 (1 min)

시스템 리소스를 적게 사용하는 stunnel 로 채택!!


## stunnel

### 설치

```
# ./configure –prefix=/usr/local/stunnel –disable-libwrap –with-threads=pthread
# make
# make install
```

### stunnel.conf

```
setuid = www-data
setgid = www-data
pid = /var/run/stunnel/stunnel.pid
;chroot = /var/run/stunnel

; error
debug = 3
output = /var/log/stunnel/stunnel.log

; Protocol version (all, SSLv2, SSLv3, TLSv1)
sslVersion = all

fips = no

[swiftproxy]
accept = 0.0.0.0:443
connect = 127.0.0.1:80
TIMEOUTidle = 600
cert = swift-proxy.crt
key = swift-proxy.nopass.key
```

### process limits

swift 사용량이 늘다보니 아래와 같은 log 가 확인되었다.

```
2016.04.05 12:33:34 LOG3[13353638]: remote socket: Too many open files (24)
```

stunnel process 의 limits 확인

```
# cat /proc/$(cat /var/run/stunnel/stunnel.pid)/limits
Limit                     Soft Limit           Hard Limit           Units
Max cpu time              unlimited            unlimited            seconds
Max file size             unlimited            unlimited            bytes
Max data size             unlimited            unlimited            bytes
Max stack size            8388608              unlimited            bytes
Max core file size        0                    unlimited            bytes
Max resident set          unlimited            unlimited            bytes
Max processes             192299               192299               processes
Max open files            1024                 4096                 files
Max locked memory         65536                65536                bytes
Max address space         unlimited            unlimited            bytes
Max file locks            unlimited            unlimited            locks
Max pending signals       192299               192299               signals
Max msgqueue size         819200               819200               bytes
Max nice priority         0                    0
Max realtime priority     0                    0
Max realtime timeout      unlimited            unlimited            us
```

/etc/security/limits.conf 를 수정해도 프로세스를 기동할 때 적용이 되지 않는다.
적용하려면 init script or service file 을 수정해주면 된다.


### init

/etc/init.d/stunnel

```
...
DEFAULTPIDFILE=/var/run/stunnel/stunnel.pid
DAEMON=/usr/local/stunnel/bin/stunnel
NAME=stunnel
DESC="SSL tunnels"
OPTIONS=""
ENABLED=0
RLIMITS="-n 32000"
...
```

### systemd

systemd 에서는 service file 에서 process 의 resource limits 를 설정해줄 수 있다.

참고: [https://www.freedesktop.org/software/systemd/man/systemd.exec.html](https://www.freedesktop.org/software/systemd/man/systemd.exec.html)


/etc/systemd/system/multi-user.target.wants/stunnel.service

```
[Unit]
Description=SSL tunnel for network daemons
After=syslog.target

[Service]
LimitNOFILE=32000
ExecStart=/usr/local/stunnel/bin/stunnel /usr/local/stunnel/etc/stunnel/stunnel.conf
Type=forking

[Install]
WantedBy=multi-user.target
```


## 적용후

```
# cat /proc/$(cat /var/run/stunnel/stunnel.pid)/limits
...
Max open files            32000                32000                files
```

## 에피소드

다른 동료가 /etc/security/limits.conf 를 수정후 stunnel restart 했으나 max open files 설정은 적용되지 않았다.
이유는 stunnel 의 thread mode 가 pthread 일 때만 limits 설정이 적용됨 확인했다. (역시 manual 과 doc 을 잘 읽어봐야 함)


## 참고

* [http://www.ghostar.org/2014/09/rebooting-linux-temporarily-loses-some-limits-conf-settings/](http://www.ghostar.org/2014/09/rebooting-linux-temporarily-loses-some-limits-conf-settings/)
* [https://www.freedesktop.org/software/systemd/man/systemd.exec.html](https://www.freedesktop.org/software/systemd/man/systemd.exec.html)
