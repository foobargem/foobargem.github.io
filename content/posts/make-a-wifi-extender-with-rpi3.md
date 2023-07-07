---
title: RPi3로 wifi extender 만들기
date: 2020-06-01T21:49:33+09:00
draft: false
author: foobargem
tags: ['linux', 'raspberrypi']
---

굴러다니는 RPi3 를 활용해서 wifi extender (wifi 拡張ポイント)를 만들어 보았다.
LAN cable 을 연장하기 싫어서 wireless nic 2개로 bridge 구성을 했다.

처음에는 단순하게 **hostapd** 를 사용하여 wlan1 을 ap 로 구성하고,
**br0(linux-bridge)** 에 wlan0 를 추가해주면 끝날것으로 생각했다.

그러나 wlan0 를 br0 에 추가하면 아래와 같이 실패가 된다.

```
# brctl addif br0 wlan0
can't add wlan0 to bridge br0: Operation not supported
```

좋은 방법이 없을까 고민을 하다가 **openvswitch** 로 시도를 해보았다.
결과는 성공!! 그러나 속도는 **1/10** 수준이라서.. 만족하지 못함.


## 구성 방법

![network flow](/images/2020/06/wireless_bridge_flow.png)

## veth peering 구성

br0(linux-bridge) 와 br-int(ovs) 를 연결할 peering 링크(veth1, veth1-peer)를 만든다.

```
ip link add name veth1 type veth peer name veth1-peer
ip link set up dev veth1
ip link set up dev veth1-peer
```

## hostapd 구성

### /etc/hostapd/hostapd.conf

```
country_code=JP
interface=wlan1
bridge=br0

ssid=nukki
hw_mode=g
channel=1

wpa=2
wpa_passphrase=1234567890
```

### hostapd 실행

```
# systemctl start hostapd
```


### br0 에 veth1 연결

```
# brctl addif br0 veth1
# brctl show
bridge name bridge id           STP enabled interfaces
br0         8000.126b29519623   no          veth1
                                            wlan1
```

## openvswitch 구성

```
ovs-vsctl add-br br-int
ovs-vsctl set-fail-mode br-int standalone
ovs-vsctl add-port br-int veth1-peer
ovs-vsctl add-port br-int wlan0
```

## 문제점

* 속도가 매우 느리다.
* fast.com 측정 결과 직접 wifi-router 연결했을때의 10% 정도의 속도
* hostapd 를 5GHz 로 구성하면 bridge(br0) 구성이 안됨. (확인 필요)
* on-board wireless adapter 와 usb wireless adapter 의 network interface name 이 부팅 할때마다 순서가 바뀐다.

## 참고

* [https://wiki.debian.org/BridgeNetworkConnections#Bridging_with_a_wireless_NIC](https://wiki.debian.org/BridgeNetworkConnections#Bridging_with_a_wireless_NIC)


정리를 하면서 ebtables 를 사용한 방법이 있는것을 알게 되었는데
다음에 시도를 해봐야겠다.
