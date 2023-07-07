---
title: rabbitmq_exporter rpm packaing
date: 2020-05-03T17:07:28+09:00
draft: false
author: foobargem
tags: ['packaing', 'docker']
---

OpenStack 을 지탱하는 Message queue 인 **RabbitMQ** 를 모니터링 하기 위해서 방안을 찾던중,
이미 퇴사한 친구가 구성해둔 Prometheus 가 있어서 활용하기 위해 rabbitmq_exporter 를 찾아 보았다.

PoC 를 위해서 [https://github.com/kbudde/rabbitmq_exporter](https://github.com/kbudde/rabbitmq_exporter) 를 사용하여 구성을 하기로 했는데,
당장 production 환경을 Docker 같은 container 환경으로 구성할수 없어서 rpm build 를 해야 했다.

이번에는 누구나 언제든 rpmbuild 를 할수 있도록 Docker 기반으로 스크립트를 만들었다.

## rpm packaing

```
$ git clone https://github.com/foobargem/packaging-rabbitmq_exporter.git
$ cd packaging-rabbitmq_exporter
$ make rpm
```

특별한 오류가 없다면 output 디렉토리에 `rabbitmq_exporter-2020.1-1.el7.centos.x86_64.rpm` 이 복사가 된다.

## 참고

* [https://github.com/kbudde/rabbitmq_exporter](https://github.com/kbudde/rabbitmq_exporter)
* [https://github.com/foobargem/packaging-rabbitmq_exporter](https://github.com/foobargem/packaging-rabbitmq_exporter)
