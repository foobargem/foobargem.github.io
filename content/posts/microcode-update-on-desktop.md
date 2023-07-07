---
title: 데스크탑 PC 의 microcode update
date: 2020-01-23T01:00:16+09:00
draft: false
author: foobargem
tags: ['linux', 'debian', 'cpu', 'microcode']
---

kernel-5.4.13 빌드를 한뒤 부팅하고 dmesg 와 sysfs 를 확인해보니 역시나 새로운 intel cpu 취약점을 알려주고 있었다.

```
# dmesg | grep -i micro
[    0.172428] TAA: Vulnerable: Clear CPU buffers attempted, no microcode
[    0.172429] MDS: Vulnerable: Clear CPU buffers attempted, no microcode
[    6.553483] microcode: sig=0x506e3, pf=0x2, revision=0xc6
[    6.553702] microcode: Microcode Update Driver: v2.2.

# grep . /sys/devices/system/cpu/vulnerabilities/*
/sys/devices/system/cpu/vulnerabilities/itlb_multihit:KVM: Mitigation: Split huge pages
/sys/devices/system/cpu/vulnerabilities/l1tf:Mitigation: PTE Inversion; VMX: conditional cache flushes, SMT vulnerable
/sys/devices/system/cpu/vulnerabilities/mds:Vulnerable: Clear CPU buffers attempted, no microcode; SMT vulnerable
/sys/devices/system/cpu/vulnerabilities/meltdown:Mitigation: PTI
/sys/devices/system/cpu/vulnerabilities/spec_store_bypass:Mitigation: Speculative Store Bypass disabled via prctl and seccomp
/sys/devices/system/cpu/vulnerabilities/spectre_v1:Mitigation: usercopy/swapgs barriers and __user pointer sanitization
/sys/devices/system/cpu/vulnerabilities/spectre_v2:Mitigation: Full generic retpoline, IBPB: conditional, IBRS_FW, STIBP: conditional, RSB filling
/sys/devices/system/cpu/vulnerabilities/tsx_async_abort:Vulnerable: Clear CPU buffers attempted, no microcode; SMT vulnerable
```

microcode 를 업데이트 하면 일부는 완화될것으로 생각이 되어 업데이트를 시도!!


Debian 은 **intel-microcode** package 가 필요한데 **non-free** 섹션에 있고
intel-microcode 는 **iucode-tool** package 를 필요로 한다.
iucode-tool 은 contrib 섹션에 포함되어 있다.

## microcode update

/etc/apt/sources.list 편집: non-free, contrib 추가

```diff
-deb http://deb.debian.org/debian/ buster main
+deb http://deb.debian.org/debian/ buster main non-free contrib
```

```
# apt install intel-microcode
```

## 업데이트 후

**mds**, **tsx_async_abort** 가 완화되었다.

```
# dmesg | grep -i micro
[    0.000000] microcode: microcode updated early to revision 0xcc, date = 2019-04-01
[    6.559418] microcode: sig=0x506e3, pf=0x2, revision=0xcc
[    6.559665] microcode: Microcode Update Driver: v2.2.

# grep . /sys/devices/system/cpu/vulnerabilities/*
/sys/devices/system/cpu/vulnerabilities/itlb_multihit:KVM: Mitigation: Split huge pages
/sys/devices/system/cpu/vulnerabilities/l1tf:Mitigation: PTE Inversion; VMX: conditional cache flushes, SMT vulnerable
/sys/devices/system/cpu/vulnerabilities/mds:Mitigation: Clear CPU buffers; SMT vulnerable
/sys/devices/system/cpu/vulnerabilities/meltdown:Mitigation: PTI
/sys/devices/system/cpu/vulnerabilities/spec_store_bypass:Mitigation: Speculative Store Bypass disabled via prctl and seccomp
/sys/devices/system/cpu/vulnerabilities/spectre_v1:Mitigation: usercopy/swapgs barriers and __user pointer sanitization
/sys/devices/system/cpu/vulnerabilities/spectre_v2:Mitigation: Full generic retpoline, IBPB: conditional, IBRS_FW, STIBP: conditional, RSB filling
/sys/devices/system/cpu/vulnerabilities/tsx_async_abort:Mitigation: Clear CPU buffers; SMT vulnerable

# lscpu
Architecture:        x86_64
CPU op-mode(s):      32-bit, 64-bit
Byte Order:          Little Endian
Address sizes:       39 bits physical, 48 bits virtual
CPU(s):              8
On-line CPU(s) list: 0-7
Thread(s) per core:  2
Core(s) per socket:  4
Socket(s):           1
NUMA node(s):        1
Vendor ID:           GenuineIntel
CPU family:          6
Model:               94
Model name:          Intel(R) Core(TM) i7-6700 CPU @ 3.40GHz
Stepping:            3
CPU MHz:             900.958
CPU max MHz:         4000.0000
CPU min MHz:         800.0000
BogoMIPS:            6799.81
Virtualization:      VT-x
L1d cache:           32K
L1i cache:           32K
L2 cache:            256K
L3 cache:            8192K
NUMA node0 CPU(s):   0-7
Flags:               fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault epb invpcid_single pti ssbd ibrs ibpb stibp tpr_shadow vnmi flexpriority ept vpid ept_ad fsgsbase tsc_adjust bmi1 hle avx2 smep bmi2 erms invpcid rtm mpx rdseed adx smap clflushopt intel_pt xsaveopt xsavec xgetbv1 xsaves dtherm ida arat pln pts hwp hwp_notify hwp_act_window hwp_epp md_clear flush_l1d
```

desktop 이니깐 작업은 편하게!!
