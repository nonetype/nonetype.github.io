---
layout: post
title: "IDT Overwrite in Linux kernel"
author: "nonetype"
categories: pwn
tags: linux_kernel
---

IDT Overwrite and mitigations in Linux Kernel

# 목차


<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [목차](#목차)
- [Abstract](#abstract)
- [The IDTR](#the-idtr)
  - [Leak IDTR](#leak-idtr)
    - [Result](#result)
- [The IDT Gate](#the-idt-gate)
- [IDT Overwrite](#idt-overwrite)
- [Mitigation](#mitigation)
- [References](#references)

<!-- /code_chunk_output -->


# Abstract
`인터럽트와 인터럽트 레지스터, 인터럽트 테이블 소개 부분`
커널에는 다양한 인터럽트가 존재한다. '3 / 0'같은 연산을 수행할 때 발생하는 `Zero-Division Error`, User-mode에서 Kernel-mode로 진입하는 `System call` 등이 대표적인 인터럽트의 예다.

이런 인터럽트가 발생하면, 이를 핸들링 하기 위한 함수를 호출하는데, 이 때 해당 인터럽트의 핸들링 함수를 찾기 위해 사용하는 것이 IDT, 'Interrupt Descriptor Table' 이다.

이 IDT 내의 함수 포인터를 변조하면 임의 코드 실행이 가능하게 되는데, 이를 위해 IDT의 구조와 IDTR(IDT Register), 어떤 방식으로 임의 코드 실행을 만들어 내는지 알아보자.


# The IDTR
KASLR이 적용되고 있기 때문에, IDT의 Base Address 또한 가변적이다. 따라서 IDT Base Address는 IDTR(Interrupt Descriptor Table Register)의 값으로 존재하게 된다.
간단하게 다음과 같이 나타낼 수 있다.

{% gist 3015348ffef969a9531c05f333072ac9 %}

IDT 변조를 위해서는 IDTR의 값을 뽑아올 수 있어야 하는데, 실제로 커널의 IDT Register의 값을 확인하는 방법은 생각보다 간단하다.
커널 자체에서 idt reigster의 값을 가져올 수 있는 명령어, `sidt` 명령을 지원하므로 아래와 같이 인라인 어셈블리를 통해 IDT Register가 가르키고 있는 IDT Base를 출력할 수 있다.

## Leak IDTR
```c
#include <stdio.h>
#include <stdint.h>

struct idtr {
	uint16_t limit;
	uint64_t base;
} __attribute__ ((packed));

void printIDTRBase(){
	struct idtr idtr;
	asm("sidt %0" : "=m" (idtr));
	
	printf("idtr: %llx\n", idtr.base);
	return;
}

int main() {

    printIDTRBase();
    return 0;
}
```

### Result

http://coffeenix.net/doc/develop/ia-interrupt/ch1-1.html
https://itguava.tistory.com/16
https://wiki.kldp.org/KoreanDoc/html/EmbeddedKernel-KLDP/idt.html
https://flslg.tistory.com/entry/IDT-Interrupt-Descriptor-Table
https://m.blog.naver.com/PostView.nhn?blogId=il0veu2&logNo=56772995&proxyReferer=https%3A%2F%2Fwww.google.com%2F (bookmarked)



# The IDT Gate
`인터럽트 테이블의 구조체 구조 및 확인법 / IDT Gate의 DPL(인터럽트 권한)도 설명해야 함`
https://github.com/NJhyo/bobfuzzer/blob/master/Exploit/NoneType/2019-09-23-Linux-Kernel-Exploit-Development-lab4.md#idt-overwrite

```
(gdb) x/32gx 0xffffffff81dd6000
0xffffffff81dd6000:	0x816a8e0000108120	0x00000000ffffffff
0xffffffff81dd6010:	0x81698e040010edb0	0x00000000ffffffff
0xffffffff81dd6020:	0x81698e030010f1c0	0x00000000ffffffff
0xffffffff81dd6030:	0x8169ee040010edf0	0x00000000ffffffff
0xffffffff81dd6040:	0x816aee0000108140	0x00000000ffffffff
0xffffffff81dd6050:	0x816a8e0000108160	0x00000000ffffffff
0xffffffff81dd6060:	0x816a8e0000108180	0x00000000ffffffff
0xffffffff81dd6070:	0x816a8e00001081a0	0x00000000ffffffff
0xffffffff81dd6080:	0x816a8e02001081c0	0x00000000ffffffff
0xffffffff81dd6090:	0x816a8e00001081f0	0x00000000ffffffff
0xffffffff81dd60a0:	0x816a8e0000108210	0x00000000ffffffff
0xffffffff81dd60b0:	0x816a8e0000108240	0x00000000ffffffff
0xffffffff81dd60c0:	0x81698e010010ee30	0x00000000ffffffff
0xffffffff81dd60d0:	0x81698e000010eed0	0x00000000ffffffff
0xffffffff81dd60e0:	0x81698e000010ef00	0x00000000ffffffff
0xffffffff81dd60f0:	0x816a8e0000108270	0x00000000ffffffff
(gdb)
```
<!--![55392467](https://i.imgur.com/XmpVSvW.jpg)-->

# IDT Overwrite
`IDT Overwrite를 위해 생성한 매크로(x64), 그리고 그 함수를 이용하여 오버라이트하는 과정, 최종적으로 오버라이트 한 인터럽트를 호출하는 것 까지`
https://resources.infosecinstitute.com/hooking-idt/#gref

# Mitigation
`해당 기법을 막기 위해 추가된 보호기법 설명 (ro_after_init 외 다수, 아래 링크에 존재)`
https://kernsec.org/wiki/index.php/Exploit_Methods/Function_pointer_overwrite

# References
`레퍼런스 문서 주소 추가`