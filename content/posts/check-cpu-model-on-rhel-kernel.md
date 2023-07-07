---
title: RHEL kernel 의 CPU model 검사
date: 2019-10-17T01:28:11+09:00
draft: false
author: foobargem
tags: ['kernel', 'rhel', 'cpu']
---


회사에서 RHEL7 을 사용하는 일부 시스템이 있다.
오늘 openvswitch 관련하여 kernel crash 가 일어나 재부팅이 발생을 했다.
관련해서 시스템을 진단하다가 생소한 메시지를 발견했고 RedHat 은 linux kernel 에 CPU model 검사를 한다는 것을 확인하게 되었다.


dmesg 의 일부 내용은 다음과 같다.


```
[    0.000000] Warning: Intel CPU model - this hardware has not undergone testing by Red Hat and might not be certified. Please consult https://hardware.redhat.com for certified hardware.
```


메시지만 보아도 RedHat 이 커널에 무슨짓을 했을것 같은 느낌이다.
RHEL7 의 **<strong>kernel-3.10.0-229.el7.src.rpm</strong>**을 내려받고 소스코드를 확인해 보았다.

**<strong>arch/x86/kernel/setup.c</strong>** 의 **<strong>setup_arch()</strong>** 함수 제일 마지막 부분에서 **<strong>rh_check_supported()</strong>** 를 호출하게 된다.


```
static void rh_check_supported(void)
{
        // 생략

        /* RHEL only supports Intel and AMD processors */
        if ((boot_cpu_data.x86_vendor != X86_VENDOR_INTEL) &&
            (boot_cpu_data.x86_vendor != X86_VENDOR_AMD)) {
                pr_crit("Detected processor %s %s\n",
                        boot_cpu_data.x86_vendor_id,
                        boot_cpu_data.x86_model_id);
                mark_hardware_unsupported("Processor");
        }

        /* Intel CPU family 6, model greater than 60 */
        if ((boot_cpu_data.x86_vendor == X86_VENDOR_INTEL) &&
            ((boot_cpu_data.x86 == 6))) {
                switch (boot_cpu_data.x86_model) {
                case 77: /* Atom Avoton */
                case 70: /* Crystal Well */
                case 69: /* Haswell ULT */
                        break;
                default:
                        if (boot_cpu_data.x86_model > 63) {
                                printk(KERN_CRIT
                                       "Detected CPU family %d model %d\n",
                                       boot_cpu_data.x86,
                                       boot_cpu_data.x86_model);
                                mark_hardware_unsupported("Intel CPU model");
                        }
                        break;
                }
        }

        // 생략
}
```


CPU family 6 중 model number 63 은 **<strong>Haswell (Server)</strong>** 에 해당된다.
crash 발생한 장비의 cpu 의 model number 는 79 **<strong>Broadwell (Server)</strong>** 이었다.
위 코드는 실제 kernel 동작과는 관련이 없는 단순 warning 일 뿐이지만 문제 상황시 RedHat 은 위 내용을 근거로 기술지원 여부를 판단하게 될것 같다.(추측)

kernel 의 CPU 관련 구현들을 알면 알수록 역시 kernel 은 최신 버전을 사용하는게 좋다고 생각이 든다.
