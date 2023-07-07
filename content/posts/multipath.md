---
title: Multipath
date: 2014-08-27T11:11:11+09:00
draft: false
author: foobargem
tags: ['linux', 'debian', 'storage']
---


컴퓨터 스토리지에서 multipath I/O 는 결함감내시스템(Fault tolerance) 과 성능(Performance) 향상을 위한 기술을 말한다.

버스(buses), 컨트롤러(controllers), 스위치(switches), 브리지 장치(bridge devices) 로 구성된 대량의 스토리지 장치(mass storage devices)와 컴퓨터 시스템의 CPU 사이에 한개 이상의 경로(path)를 두어서 구성을 한다.

예를 들면 하나의 SCSI 디스크가 연결된 두개의 SCSI 컨트롤러(controllers) 가 같은 컴퓨터에 장착되어 있거나 두개의 파이버 채널(Fiber Channel) 포트(port)가 연결되어 있는 것을 말한다.
만약 한개의 컨트롤러나 포트 또는 스위치가 장애가 발생하면 운영체제(OS)는 장애가 나지 않은 다른 컨트롤러를 통해서 반드시 I/O 작업이 이뤄져야 한다.

Multipath software layers 는 아래와 같은 성능 향상을 위한 기능을 제공한다.

* Dynamic load balancing
* Traffic shaping
* Automatic path management
* Dynamic reconfiguration


## 설정

Debian wheezy 에서 `multipath-tools_0.4.9+git0.4dfdaf2b-7~deb7u2_amd64.deb` 패키지를 설치하여 구성하였다.


### WWN or WWID

WWN(World Wide Name) 또는 WWID(World Wide Identifier) 라고 불리운다.
이것은 FC(Fiber Channel), SAS(Serial Attached SCSI), ATA(Advanced Technology Attachment) 의 스토리지 기술에서 고유한 식별자(unique identifier)로 사용된다.


참고: [http://en.wikipedia.org/wiki/World_Wide_Name](http://en.wikipedia.org/wiki/World_Wide_Name)


### FC 주소 확인

HBA 가 장착된 리눅스 시스템에서 /sys/class/fc_host 디렉토리 이하를 살펴보면 확인할 수 있다.

```
# ls /sys/class/fc_host
host0
# cat /sys/class/fc_host/host0/port_name
0x2100001b32939df8
# cat /sys/class/fc_host/host0/node_name
0x2000001b32939df8
```

### WWN 확인

리눅스 시스템에서는 /sys/class/fc_transport 디렉토리 이하를 살펴보면 확인할 수 있다.

```
# ls /sys/class/fc_transport
target0:0:0  target0:0:1
# cat /sys/class/fc_transport/target0\:0\:0/node_name 
0x2000001b4d01a9f2
# cat /sys/class/fc_transport/target0\:0\:0/port_name 
0x2100001b4d01a9f2
```

### multipath.conf

Debian wheezy 에 4개의 FC path 가 할당 되었다.

```
defaults {
    user_friendly_names yes 
    fast_io_fail_tmo 15  
}

blacklist {
    devnode "^sd[a-z][[0-9]*]"
    wwid 36b82a720d8d964001b5dbbc31d351fe1
}

multipaths {
    multipath {
        wwid 3600601601c203900cff81153e321e411
    }   
    multipath {
        wwid 3600601601c203900d0f81153e321e411
    }   
    multipath {
        wwid 3600601601c203900ee1d1460e321e411
    }   
    multipath {
        wwid 3600601601c203900ef1d1460e321e411
    }   
}
```

### 설정 예

##### defaults

![multipath default](/images/201408/multipath_default.png)

##### path_grouping_policy: multibus

![multipath multibus](/images/201408/multipath_multibus.png)

##### hwhandler=0

![multipath disable-hwhandler](/images/201408/multipath_disable_hwhandler.png)

##### features=0

![multipath disable-features](/images/201408/multipath_disable_features.png)

## failover

Debian wheezy 에서 multipath 를 구성한뒤 failover 시험을 했다.
2개의 HBA에 연결된 port 를 하나 뽑았을 때 I/O 작업이 지속되는지를 체크 했는데 300초 동안 I/O 작업이 멈춰있었다.
실제로는 OCFS2 로 구성되어 있었는데 OCFS2 의 self-fencing 으로인해 재부팅이 되었다.

이를 해결하기 위해서 Debian 시스템을 확인하던 중 dev_loss_tmo 의 기본값이 300초, fast_io_fail_tmo 의 기본값이 off 임을 발견하였다.

그래서 fast_io_fail_tmo 를 15초로 설정해주니 15초의 I/O 지연이 발생하지만 failover 가 정상적으로 수행됨을 확인했다.

## 문제점

multipath.conf 의 man page 에 따라 defaults 섹션에 기술한 설정값이 적용이 안됨을 발견했다.
features=0 을 설정하기 위해서는 multipath 섹션을 따로 기술해주고 no_path_retry fail 을 추가해주면 된다.
이는 좀더 확인이 필요하다.

## 참고

* [https://access.redhat.com/documentation/ko-KR/Red_Hat_Enterprise_Linux/6/html/DM_Multipath/](https://access.redhat.com/documentation/ko-KR/Red_Hat_Enterprise_Linux/6/html/DM_Multipath/)
* [http://www.sourceware.org/lvm2/wiki/MultipathUsageGuide](http://www.sourceware.org/lvm2/wiki/MultipathUsageGuide)
* [http://docs.oracle.com/cd/E19253-01/820-1931/](http://docs.oracle.com/cd/E19253-01/820-1931/)
* [http://www.brentozar.com/archive/2009/05/san-multipathing-part-1-what-are-paths/](http://www.brentozar.com/archive/2009/05/san-multipathing-part-1-what-are-paths/)
* [http://www.brentozar.com/archive/2009/05/san-multipathing-part-2-what-multipathing-does/](http://www.brentozar.com/archive/2009/05/san-multipathing-part-2-what-multipathing-does/)
* [https://www.sit.auckland.ac.nz/Configuring_SAN_access_with_Linux_multipath](https://www.sit.auckland.ac.nz/Configuring_SAN_access_with_Linux_multipath)
