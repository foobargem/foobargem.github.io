---
title: "Installing Asdf"
date: 2024-05-06T21:30:05+09:00
author: foobargem
tags: ["tools"]
---

python 은 [pyenv](https://github.com/pyenv/pyenv),
ruby 는 [rbenv](https://github.com/rbenv/rbenv),
terraform 은 [tfenv](https://github.com/tfutils/tfenv)

참 다양한 version manager (VM) 이 존재한다. 
그 덕분에 버전을 쉽게 바꿔가며 개발을 할수 있음에 감사한다.

최근에 [asdf-vm](https://asdf-vm.com/guide/introduction.html) 라는 version manager 를 알게 되어
연휴동안 golang, python 을 설정해보았다.

[동작원리](https://asdf-vm.com/guide/introduction.html#how-it-works)는 plugin 마다 shim 이 생성되어 관리되고,
`asdf` 가 `.tool-versions` 정의된 정보를 바탕으로 해당 버전을 실행한다.

[guide](https://asdf-vm.com/guide/getting-started.html) 가 잘 되어 있어서 설치 하는 과정만 간단히 정리해 둔다.


### 설치 (zsh & homebrew)

```zsh
$ brew install asdf

$ echo -e "\n. $(brew --prefix asdf)/libexec/asdf.sh" >> ${ZDOTDIR:-~}/.zshrc
```

### asdf-golang plugin 설치 및 설정

```zsh
$ asdf plugin add golang https://github.com/asdf-community/asdf-golang.git

$ asdf install golang 1.21.9

$ asdf exec go version
go version go1.21.9 darwin/arm64

$ asdf global golang 1.21.9
$ asdf reshim golang
$ rehash

$ go version
go version go1.21.9 darwin/arm64
```

### asdf-python 설치 및 설정

```zsh
$ asdf plugin add asdf-python

$ asdf install python 3.12.3

$ asdf global python 3.12.3
$ asdf reshim python
$ rehash

$ python --version
Python 3.12.3
```

