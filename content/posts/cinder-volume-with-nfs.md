---
title: nfs 를 사용한 cinder-volume 구성시 secure 설정
date: 2019-10-08T00:11:13+09:00
draft: false
author: foobargem
tags: ['openstack', 'cinder', 'nfs']
---


퇴근을 10여분 앞둔 금요일 오후..
갑자기 모니터링 시스템으로 부터 알람이 쏟아지기 시작했다.
진단을 해보니 VM 에 Disk IO 가 정지되어 read-only 상태로 mount 된것을 확인했다.
관련된 log 를 확인하던중 qemu log 에서 **<strong>permission denied (13)</strong>** 오류가 발생한 것을 확인했다.

```
block I/O error in device 'drive-virtio-disk0': Permission denied (13)
block I/O error in device 'drive-virtio-disk0': Permission denied (13)
block I/O error in device 'drive-virtio-disk0': Permission denied (13)
block I/O error in device 'drive-virtio-disk0': Permission denied (13)
```

cinder 의 backend 가 mount 된 directory 와 disk files 의 상태를 조회해보니
일부 disk files 의 소유권이 모두 **<strong>nova:nova</strong>** 로 변경 되어 있었다.

libvirtd 는 qemu process 를 **<strong>qemu:qemu</strong>** 로 구동했기 때문에
disk file 의 소유권이 **<strong>nova:nova</strong>** 로 변경되자 read/write 가 불가능해졌고
**<strong>Permission denied</strong>** 를 계속 출력하고 있던 것이었다.

우선 급하게 other 에게 rw 권한을 부여 해주었다.

```
find . -perm 660 -exec chmod 666 {} +
```

qemu log 에 Permission denied 메시지는 더이상 기록되지 않았다.
하지만 이미 VM에서 filesystem 을 read-only 로 마운트를 한 상태였고,
대상 VM을 모두 재부팅한뒤 복구를 완료했다.

월요일 오전에 의문점을 해결하기 위해 구성을 점검해보았다.


### 왜 같은 mount point 에 있는 disk files 중 일부는 0660 이고, 일부는 0666 인가?

cinder 에서 사용하는 storage 는 NAS이며, nfs 드라이버를 사용하고 있다.
nfs 드라이버에서는 각 backend 에 기술하는 옵션중 secure options 이 존재한다.

**<strong>nas_secure_file_operations</strong>** 와 **<strong>nas_secure_file_permissions</strong>** 가 그것이다.

두가지 설정 모두 3가지 값중 하나를 설정할 수 있다.

* auto: 기본값이며 예상치 못한 짓을 한다.
* true: disk file 의 권한: 660
* false: disk file 의 권한: ugo+rw (666)


cinder-volume.log 를 통해 확인한 것은 volume 이 생성될때 permission 을 660 으로 설정한다는 것이었다.

```
2019-10-07 16:02:32.375 12254 DEBUG oslo_concurrency.processutils [req-55a29436-8cbe-4643-8d5c-0cbea8721f53 - - - - -] Running cmd (subprocess): chmod 660 /var/lib/cinder/mnt/63bf80ae8c90ff6c5fcee0d7b3c5711e/volume-9d9074a3-2f80-4db2-be3a-3f8a65c426c3 execute /usr/lib/python2.7/site-packages/oslo_concurrency/processutils.py:199
2019-10-07 16:02:32.383 12254 DEBUG oslo_concurrency.processutils [req-55a29436-8cbe-4643-8d5c-0cbea8721f53 - - - - -] CMD "chmod 660 /var/lib/cinder/mnt/63bf80ae8c90ff6c5fcee0d7b3c5711e/volume-9d9074a3-2f80-4db2-be3a-3f8a65c426c3" returned: 0 in 0.008s execute /usr/lib/python2.7/site-packages/oslo_concurrency/processutils.py:225
```

cinder.conf 의 backend 설정에는 secure options 가 명시되어 있지 않음에도
cinder volume 은 자꾸만 secure options 이 활성화 된것처럼 volume 을 만들고 있었다.
어쩔수 없이 nfs driver 코드를 살펴보게 되었고 몰랐던 부분을 알게 되었다.

cinder.conf 의 backend 설정에 secure options 가 없다면

**<strong>nas_secure_file_operations</strong>**, **<strong>nas_secure_file_permissions</strong>** 는
모두 **<strong>auto</strong>** 가 된다.

auto 로 값이 설정되면 cinder volume 이 driver 롤 초기화 하는 과정에서 기존에 DB 에 등록되지 않은 backend 일때 
**<strong>.cinderSecureEnvIndicator</strong>** 파일을 몰래 생성(log에 남기긴 함)하고 앞으로 계속 secure options 을 활성화하게 된다.


nfs.NfsDriver.set_nas_security_options()

```
self.configuration.nas_secure_file_permissions = \
    self._determine_nas_security_option_setting(
        self.configuration.nas_secure_file_permissions,
        nfs_mount, is_new_cinder_install)

self.configuration.nas_secure_file_operations = \
    self._determine_nas_security_option_setting(
        self.configuration.nas_secure_file_operations,
        nfs_mount, is_new_cinder_install)
```


remotefs.RemoteFSDriver._determine_nas_security_option_setting()

![_determine_nas_security_option_setting](/images/cinder-volume-with-nfs/openstack-cinder-nfs-secure.png)

참조: https://github.com/openstack/cinder/blob/kilo-eol/cinder/volume/drivers/remotefs.py#L525


### 결론

**auto** 는 사용하지 말고 **true** 또는 **false** 라고 명시하는게 좋다.

kilo 부터 train 까지의 remotefs.py 의 **RemoteFSDriver._determine_nas_security_option_setting** 를 보았지만
**.cinderSecureEnvIndicator** file 생성 과정에 예외처리가 추가된것을 제외하곤 달라진 부분이 없다.
