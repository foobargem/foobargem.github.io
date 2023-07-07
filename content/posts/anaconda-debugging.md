---
title: "Anaconda 디버깅 - anaconda 가 거짓말을 했어"
date: 2019-09-20T23:50:24+09:00
draft: false
author: foobargem
tags: ["linux", "centos", "anaconda"]
---

회사에서 PM 은 PXE 로 anaconda 를 구동하여 kickstart 기반으로 OS(CentOS7) 설치를 하고 있다.
kickstart 의 pre script, post script 의 조합으로 OS 구성을 하게 되는데, 잘 동작하면 괜찮지만 script 실행중에 실패가 발생하면 디버깅을 하기가 매우 어렵게 되어있다.
오늘도 동료가 진행하는 서버 구성중 OS 설치가 지속적으로 실패를 했고 디버깅을 하는 과정에서 anaconda(CentOS installer) 의 황당한 버그(?)를 발견하여 간략히 정리를 해둔다.

kickstart 의 post script 선언부에서 logging 설정을 넣으면 post script 실행 과정에 대한 log 를 남기게 된다.
그래서 OS 설치 과정중 post script 의 어디선가 실패하는 것을 확인할 수 있었다.
그리고 문득 anaconda 로 OS 설치하는 과정에 대한 로그도 확인하면 좋을것 같아서 문서를 확인해보니 역시 **logging 설정**이 제공되고 있었다.
방법은 두가지 있는데, 첫번째는 anaconda 부팅 옵션에 logging 관련 설정을 기술하는 것이고, 두번째는 kickstart 에 logging 설정을 기술하는 것이다.

### anaconda boot options

참고: [anaconda boot options](https://rhinstaller.github.io/anaconda/boot-options.html#debugging-and-troubleshooting)

* inst.loglevel
    * loglevel 을 정의한다. 정의한 level 이상의 log 를 대상으로 한다.
    * 기본값: info
    * 정의 가능한 값: debug|info|warning|error|critical
* inst.syslog
    * log 를 전송할 remote host 를 지정한다.
    * v25.14 문서에는 **UDP 514**번 포트가 기본으로 사용된다고 기술되어 있다.


### kickstart options

참고: [kickstart reference](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/installation_guide/sect-kickstart-syntax)

* logging
    * `--host`
    * `--port`
    * `--level`


## 삽질

먼저 가장 간단한 kickstart 에 **logging** 선언을 하고 OS 설치를 진행하면서 syslog 수집서버(UDP 514 만 Listen)로 전달되는지 확인을 했다.

```
skipx
text
install

logging --host=192.168.100.11 --level=debug
```

그런데 syslog 수집서버에 전달이 안되어서 anaconda shell 로 `/etc/rsyslog.conf` 설정을 확인해보니
remote host 설정이 **@@192.168.100.11 (TCP 514)** 로 설정이 되어 있음을 알수 있었다.

![anaconda_rsyslog_conf](/images/anaconda_rsyslog_conf.png)


그래서 **@192.168.100.11 (UDP 514)** 로 설정을 변경후 rsyslog 재시작을 해주니 syslog 수집서버로 전달이 잘 됨을 확인했다.

```
*.* @192.168.100.11
```

이번에는 kickstart 의 logging 설정을 주석처리 하고 anaconda boot options 로 시도를 해보았다.
하지만 역시 OS 설치 과정의 로그는 syslog 수집 서버로 전달되지 않았고 이유는 마찬가지로 TCP 로 설정이 되었기 때문이었다.

![anaconda_boot_option](/images/anaconda_boot_option.png)

사내 공통으로 사용중인 syslog 수집 서버라서 바로 TCP 로 변경할수 없어서 꼼수를 시도해 보았고 잘 동작함을 확인했다.
그것은 kickstart 의 **pre script** 에 rsyslog 설정을 추가후 rsyslog 서비스를 재시작 하도록 하는 것이다.

```
%pre
#!/bin/bash
cat &gt;&gt;/etc/rsyslog.conf &lt;&lt;EOF
*.* @192.168.100.11
EOF
systemctl restart rsyslog
%end
```

다시 OS 설치를 시도해보니 기대한 대로 anaconda 의 syslog 가 수집서버로 잘 전달됨을 확인할수 있었다.
그래서 실제 사내 시스템에 적용을 했고, OS 설치 과정의 로그를 kibana 에서 쉽게 확인할 수 있게 되었다.

몇가지 의문점은 다음주에 출근해서 더 확인을 해보도록 하자.
* anaconda 의 버그인가? 아니면 문서가 잘못 적혀 있는 것인가?
* 왜 TCP 를 기본으로 설정 했을까?
