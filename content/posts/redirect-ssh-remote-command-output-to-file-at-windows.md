---
title: Windows 에서 ssh remote command 결과를 file 로 저장하기
date: 2020-01-22T00:36:16+09:00
draft: false
author: foobargem
tags: ['windows', 'powershell', 'plink']
---

오늘 회사에서 Linux 서버에 있는 어떤 file 의 내용을
Windows 의 notepad 에 복사를 하고 싶은데 좋은 방법이 있냐는 문의를 받았다.

당장 떠오른 방법은 Linux 서버에서 netcat 으로 해당 file 을 전송하게 하고
클라이언트 환경에서 netcat 이나 telnet 으로 내용을 받도록 하는 것이었다.

그러나 windows 환경에서는 제약적인듯 하여 검색을 해보니
putty 의 [plink](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html) 를 사용해서
SSH remote command 실행을 하고 그 결과를 powershell 의 Out-File 함수에 stdin 으로 재전송하면 깔끔하게 file 로 결과를 얻을수 있었다.

![plink|out-file](/images/2020/01/plink_out-file.png)


```
PS C:\Users\administrator> C:\Users\administrator\Downloads\plink.exe
                             -batch -ssh centos@192.168.0.38
                             -i C:\Users\administrator\Documents\codong.ppk
                             "cat /var/log/toast-sysmon-20200121"
                             | Out-File -FilePath c:\Users\administrator\Documents\toast-sysmon.log
```

## 참고

* [plink](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html)
* [PowerShell:Out-File](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/out-file?view=powershell-7)
