---
title: Raspberry Pi 에서 온습도 센서(DHT-11)로 온/습도 측정하기
date: 2016-03-27T11:59:47+09:00
draft: false
author: foobargem
tags: ['raspberrypi']
---

## DHT11 온습도 센서

* [http://eleparts.co.kr/EPXF43N9](http://eleparts.co.kr/EPXF43N9)
* manual: [http://www.micropik.com/PDF/dht11.pdf](http://www.micropik.com/PDF/dht11.pdf)
* Python library: [https://github.com/adafruit/Adafruit_Python_DHT](https://github.com/adafruit/Adafruit_Python_DHT)


## 시험환경

* Raspberry PI B

GPIO layout:
![GPIO Layout](http://www.raspberrypi-spy.co.uk/wp-content/uploads/2012/09/Raspberry-Pi-GPIO-Layout-Revision-2.png)

## 시험

### 센서연결

* pin #1 - 3V
* pin #6 - Ground
* pin #16 - GPIO23 (data)

### 설치

```
$ sudo python setup.py install
```

### 예제코드 수정 및 실행

예제코드: [https://github.com/adafruit/Adafruit_Python_DHT/blob/master/examples/simpletest.py](https://github.com/adafruit/Adafruit_Python_DHT/blob/master/examples/simpletest.py)


```
#!/usr/bin/python
import Adafruit_DHT

sensor = Adafruit_DHT.DHT11

# GPIO23 (pin no: #16)
pin = 23

humidity, temperature = Adafruit_DHT.read_retry(sensor, pin)

if humidity is not None and temperature is not None:
    print "Temp={0:0.1f}*C Humidity={1:0.1f}%".format(temperature, humidity)
else:
    print "Failed to get reading."
```


**실행결과**

```
$ sudo python read.py
Temp=24.0*C Humidity=21.0%
```

## ToDo

* 매일 아침 창문 밖의 온습도 측정하여 휴대폰으로 알람 발송.
* 매 시간 단위로 온습도 측정하여 influxdb 에 저장. (그 후로는 나중에 생각하자..)
