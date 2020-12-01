---
layout: post
title: "[SOLVED] A start job is running for Raise network interfaces (5min 1s)"
author: "nonetype"
categories: dev
tags: vm
---

qemu 부팅이 너무 느려요!

---

# 목차

1. TOC
{:toc}

---

# Too slow..

qemu 부팅시 네트워크 작업 속도가 너무 느리다..
기존 10-20초면 부팅되던 이미지가 1분 넘게 네트워크를 잡고 있다.

```sh
[ ***  ] A start job is running for Raise ne…rk interfaces (1min 53s / 5min 2s)
```

`/lib/systemd/system/networking.service` 파일 내의 `TimeoutStartSec` 값을 `10sec`으로 변경하여 해결했다.


**변경 후 네트워크가 되지 않는다..**
추가적인 방법을 찾아봐야겠다.


# References
<https://hoguinside.blogspot.com/2018/03/a-start-job-is-running-for-raise.html>
