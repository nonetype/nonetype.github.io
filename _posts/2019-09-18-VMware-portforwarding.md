---
layout: post
title: "VMware Port Forwarding for Debugging"
author: "nonetype"
categories: dev
tags: vm
---

Qemu Linux Kernel 디버깅을 위해 VMware 포트포워딩을 해보자.

---

# 목차

1. TOC
{:toc}

---

# VMware network setting

VMware 창의 `Edit - Virtual Network Editor`를 열어준 후, 오른쪽 밑 Help 버튼 위의 `Change Settings`를 클릭한다.

![vne_nat](/assets/vne_nat.PNG)
포트포워딩 설정할 가상 머신의 네트워크 연결 타입을 클릭한다.
나는 NAT로 설정되어 있어서 `Type: NAT`를 클릭했다.

이후 `NAT Settings` 버튼을 클릭한다.

![vne_nat_setting](/assets/vne_nat_setting.PNG)

위와 같은 창이 뜨게 되는데, `Add...` 버튼을 클릭한다.

![vne_nat_setting_add](/assets/vne_nat_setting_add.PNG)

`Host port`는 호스트에서 열 포트,

`Type`은 TCP 그대로 유지,

`Virtual machine IP address`는 가상머신의 IP

`Virtual machine port`는 qemu kernel debugging을 할 것이기 때문에 1234로 입력한다.

![vne_nat_setting_add_comp](/assets/vne_nat_setting_add_comp.PNG)

이후 OK를 눌러 저장하고 나온다.


# IDA Remote Debugger Attach

디버깅할 Qemu Machine을 실행시킨 후, IDA로 vmlinux 파일을 열자.

![vne_ida](/assets/vne_ida.PNG)

`Debugger - Process options` 클릭

`Hostname`: localhost
`Port`: 11234로 설정한다.

`Debugger - Attach to Process`를 클릭해서 Remote Debugging을 시작하면 된다!

# References
<https://crehacktive3.blog.me/221155296135>
