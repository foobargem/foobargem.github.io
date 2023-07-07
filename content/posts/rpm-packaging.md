---
title: RPM packaging
date: 2016-06-11T11:36:38+09:00
draft: false
author: foobargem
tags: ['linux', 'packaging', 'centos', 'rpm']
---


## 배경

VMware VM 중 1대의 Disk 공간이 부족해서 resize 를 하려고 했는데 parted-3.1 (CentOS 7) 에는 resize command 가 없어졌다.
구글링을 해보니 parted-3.2 에 resizepart 가 추가되었다고 한다.

* [http://savannah.gnu.org/forum/forum.php?forum_id=8042](http://savannah.gnu.org/forum/forum.php?forum_id=8042)

그래서 소스코드를 직접 빌드하고서 내친김에 rpm packaging 까지 하게 되었다.


## rpm 패키징

### Build directory 생성

```
$ mkdir rpmbuild
$ cd rpmbuild
$ mkdir SOURCES SPECS BUILD BUILDROOT RPMS SRPMS
$ ls
BUILD  BUILDROOT  RPMS  SOURCES  SPECS  SRPMS
```

### Spec file 작성

```
$ cd SPECS
$ vim parted-3.2.spec
```

**parted-3.2.spec**

gist: [https://gist.github.com/foobargem/f260b896fe030a382b7b18f43633a8a4#file-parted-3-2-spec](https://gist.github.com/foobargem/f260b896fe030a382b7b18f43633a8a4#file-parted-3-2-spec)


```
%global _enable_debug_package 0
%global debug_package %{nil}
%define dist .el7.nhnent
%define _prefix /opt/parted

Name: parted
Version: 3.2
Release: 1%{?dist}
Summary: The GNU disk partition manipulation program

License: GPLv3+
URL: http://www.gnu.org/software/parted
Source0: parted-3.2.tar.xz

%description
The GNU Parted program allows you to create, destroy, resize, move,
and copy hard disk partitions. Parted can be used for creating space
for new operating systems, reorganizing disk usage, and copying data
to new hard disks.

%prep
%setup -q

%build
%configure
make %{?_smp_mflags}

%install
%make_install

%clean
rm -rf $RPM_BUILD_ROOT

%files
%{_prefix}
#%{_datadir}/*
#%{_includedir}/*
#%{_libdir}/*
#%{_sbindir}/*

%changelog
* Fri Jun 10 2016 makerpm
- build and packaging
```

### Build

```
$ cd SPECS
$ rpmbuild -bb parted-3.2.spec
```

오류 없으면 rpmbuilds/RPMS/x86_64/parted-3.2-1.el7.nhnent.x86_64.rpm 파일이 생성된다.


### 설명

```
%global _enable_debug_package 0
%global debug_package %{nil}
```

debug package 를 생성하지 않음

```
%define dist .el7.nhnent
%define _prefix /opt/parted
```

main stream package 와 구분하기 위해서 release name 에 nhnent 라는 접미사를 붙였고
package files 의 prefix 를 /opt/parted 로 지정했다.


## 에피소드

* prefix 와 %files

prefix 를 /opt/parted 로 하고 %files 에 각 파일을 기술해줬더니
패키지 삭제시 empty directory 가 남아있는 이슈가 있었다.
그래서 %files 를 %{_prefix} 롤 지정했다.

만약 /usr 디렉토리에 sbin, share, lib 등으로 나누어 설치하게 하려면
%files 에 아래와 같이 기술해줘야 한다.


```
%files
%{_datadir}/*
%{_includedir}/*
%{_libdir}/*
%{_sbindir}/*
```

## 참고

* [http://www.tldp.org/HOWTO/RPM-HOWTO/build.html](http://www.tldp.org/HOWTO/RPM-HOWTO/build.html)
* [https://fedoraproject.org/wiki/How_to_create_an_RPM_package](https://fedoraproject.org/wiki/How_to_create_an_RPM_package)
* [https://fedoraproject.org/wiki/Packaging:DistTag](https://fedoraproject.org/wiki/Packaging:DistTag)
* [https://fedoraproject.org/wiki/How_to_create_a_GNU_Hello_RPM_package](https://fedoraproject.org/wiki/How_to_create_a_GNU_Hello_RPM_package)
