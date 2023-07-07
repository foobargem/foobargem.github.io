---
title: 라즈베리파이와 wifi - Operation not possible due to RF-kill
date: 2020-06-15T12:06:58+09:00
draft: false
author: foobargem
tags: ['raspberrypi', 'raspbian', 'rfkill']
---

Raspberry Pi OS(이전의 Raspbian) 를 부팅한뒤
wpa_supplicant, interfaces 정보를 적절하게 설정하고
wlan0 인터페이스를 활성화 하려고 하니 이런 오류가 발생!

```
root@raspberrypi:~# ifup wlan0
Internet Systems Consortium DHCP Client 4.4.1
Copyright 2004-2018 Internet Systems Consortium.
All rights reserved.
For info, please visit https://www.isc.org/software/dhcp/

RTNETLINK answers: Operation not possible due to RF-kill
Listening on LPF/wlan0/dc:a6:32:b2:5e:db
Sending on   LPF/wlan0/dc:a6:32:b2:5e:db
Sending on   Socket/fallback
DHCPDISCOVER on wlan0 to 255.255.255.255 port 67 interval 5
send_packet: Network is down
dhclient.c:2445: Failed to send 300 byte long packet over wlan0 interface.
receive_packet failed on wlan0: Network is down
DHCPDISCOVER on wlan0 to 255.255.255.255 port 67 interval 12
send_packet: Network is down
dhclient.c:2445: Failed to send 300 byte long packet over wlan0 interface.
^CGot signal Interrupt, terminating...
ifup: failed to bring up wlan0
```

rfkill 에 의해서 wlan0 가 활성화 할수 없는 것이 원인이다.

rfkill 은 wifi, 블루투스 같은 전파 송수신 장치를 제어하는 리눅스 커널의 서브시스템이다.

현재 상태는 wlan 장치가 soft blocked 된 상태로 unblock 해주면 사용 가능해진다.

```
root@raspberrypi:~# rfkill list
0: phy0: Wireless LAN
        Soft blocked: yes
        Hard blocked: no
1: hci0: Bluetooth
        Soft blocked: no
        Hard blocked: no

root@raspberrypi:~# rfkill unblock 0

root@raspberrypi:~# rfkill list
0: phy0: Wireless LAN
        Soft blocked: no
        Hard blocked: no
1: hci0: Bluetooth
        Soft blocked: no
        Hard blocked: no
```

## 참고
* [https://lwn.net/Articles/677839/](https://lwn.net/Articles/677839/)
* [https://access.redhat.com/documentation/ko-kr/red_hat_enterprise_linux/6/html/power_management_guide/rfkill](https://access.redhat.com/documentation/ko-kr/red_hat_enterprise_linux/6/html/power_management_guide/rfkill)
