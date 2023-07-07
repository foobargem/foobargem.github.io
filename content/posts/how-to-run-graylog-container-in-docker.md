---
title: Docker container 로 Graylog 구동하기
date: 2020-05-03T12:54:44+09:00
draft: false
author: foobargem
tags: ['docker', 'graylog']
---

Ubuntu 의 cloud-config 기반의 자동설치를 시험하다 디버깅을 위해 syslog 수집이 필요해서 graylog 를 docker 로 구성을 해보았다.

단순히 디버깅 목적이라 production 에서 사용하려면 많은 검증이 필요하다.


## 준비

환경은 Debian 10(buster) 기준이다.

필요한 것
* VM
* docker
* docker-compose

우선 TOAST Cloud 에서 Debian 10 기반의 VM을 하나 생성한다.
VM 접속후 docker 및 docker-compose 를 설치한다.
아래와 같은 스크립트면 docker, docker-compose 설치 및 debian user 에 docker 권한을 추가해준다.

```
sudo apt update
sudo apt -y install \
	apt-transport-https ca-certificates curl \
	gnupg gnupg-agent software-properties-common

curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -

sudo add-apt-repository \
	"deb [arch=amd64] https://download.docker.com/linux/debian \
   	$(lsb_release -cs) \
   	stable"

sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io

sudo curl -L "https://github.com/docker/compose/releases/download/1.25.5/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

sudo usermod -a -G docker debian
```

## graylog container 구동

```
$ git clone https://github.com/foobargem/docker-graylog.git
$ cd docker-graylog
$ cp docker-compose.yml.example docker-compose.yml
```

docker-compose.yml 의 GRAYLOG_HTTP_EXTERNAL_URI 를 외부에서 접속 가능한 Floating IP 나 Domain name 으로 변경한다.


```
$ docker-compose up -d
```

graylog dashboard (http://{Floating_IP_of_VM}:9000/) 로 접속

* user: admin
* passwd: admin


## Syslog inputs 설정

![configure_graylog_inputs_01](/images/2020/05/graylog_syslog_setup01.png)

![configure_graylog_inputs_02](/images/2020/05/graylog_syslog_setup02.png)

UDP 514 port 로 syslog 에 대한 inputs 를 설정한다.
System -> Inputs 로 이동후 Syslog UDP 를 선택한다.


* global 체크
* title 작성
* 그 외의 기본 설정은 유지

save 버튼을 클릭하면 Syslog UDP inputs 가 동작을 시작한다.


## 동작 확인

graylog 의 Streams > All messages 로 이동후 Update every **5 Seconds** 설정을 해둔다.

rsyslog.conf 에 remote 전송 설정 추가

```
$ sudo bash -c 'cat >>/etc/rsyslog.conf <<EOF
*.*  @127.0.0.1:514;RSYSLOG_SyslogProtocol23Format
EOF'

$ sudo systemctl restart rsyslog
$ logger hello-graylog
```

![graylog_stream_messages](/images/2020/05/graylog_stream_messages.png)

logger 명령으로 보낸 메시지가 확인된다.


## 참고

* [https://docs.graylog.org/en/3.2/index.html](https://docs.graylog.org/en/3.2/index.html)
* [https://docs.graylog.org/en/3.2/pages/configuration/elasticsearch.html](https://docs.graylog.org/en/3.2/pages/configuration/elasticsearch.html)
* [https://www.elastic.co/guide/en/elasticsearch/reference/6.8/docker.html](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/docker.html)
