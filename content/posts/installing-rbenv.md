---
title: Installing rbenv on OS X
date: 2013-12-01T22:12:48+09:00
draft: false
author: foobargem
tags: ['ruby']
---

늘 하는 얘기지만 Linux 에서는 이런 고민은 안해도 된다.
OS X Mavericks 에서는 ruby-2.0.0p247 이 시스템에 설치되어 있다.
그래도 최신버전의 ruby와 GEM 을 편리하게 관리하려면 rbenv 만한 것도 없는것 같다.

[Rbenv 프로젝트 페이지](https://github.com/sstephenson/rbenv) 의 가이드를 따라 설치를 하면
irb 나 rails console 에서 한글을 입력하지 못하게 될것이고 openssl 관련한 이슈가 생길것이다. (이전의 경험이라 지금은 확인을 안해봤다.)

이를 위해서 openssl 과 readline 을 ruby 를 빌드할때 참조시켜줘야 한다.
brew 의 설치는 [Homebrew 프로젝트 페이지](http://brew.sh/) 를 참조하면 된다.

## openssl 설치

```
brew install openssl
```

## readline 설치

```
brew install readline
```

## rbenv 설치

```
$ cd  # move to home dir.
$ git clone https://github.com/sstephenson/rbenv.git ~/.rbenv
$ echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bash_profile
$ echo 'eval "$(rbenv init -)"' >> ~/.bash_profile
$ source .bash_profile
$ type rbenv
#=> "rbenv is a function"
```

자세한 내용은 [Rbenv 프로젝트 페이지](https://github.com/sstephenson/rbenv) 를 참조.

## ruby-build 설치

rbenv 의 plugin 으로 설치하는 것을 권장.

```
$ git clone https://github.com/sstephenson/ruby-build.git ~/.rbenv/plugins/ruby-build
```

자세한 내용은 [ruby-build 프로젝트 페이지](https://github.com/sstephenson/ruby-build) 를 참조.

## ruby 설치

```
$ rbenv install --list   # 설치가능한 버전확인
$ CONFIGURE_OPTS="--with-readline-dir=$(brew --prefix readline) --with-openssl-dir=$(brew --prefix openssl)" rbenv install 2.0.0-p353
$ rbenv global 2.0.0-p353
$ rbenv rehash
$ ruby --version
```

이것으로 완료!!
즐거운 루비 곡갱이질을.. :)
