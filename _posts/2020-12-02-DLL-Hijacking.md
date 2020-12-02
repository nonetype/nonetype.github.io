---
layout: post
title: "DLL Hijacking Note"
author: "nonetype"
date: 2020-12-01 15:41:00 +0900
categories: [pwn, windows]
tags: [dll hijacking]
---

Note for DLL Hijacking

- [The path check order for loading DLL](#the-path-check-order-for-loading-dll)
- [Interesting DLLs](#interesting-dlls)
  - [Windows 10](#windows-10)
    - [WindowsCoreDeviceInfo.dll](#windowscoredeviceinfodll)
  - [Windows 7](#windows-7)
- [Interesting tools](#interesting-tools)
  - [Siofra](#siofra)
- [Helpful articles](#helpful-articles)


# The path check order for loading DLL

1. The directory from which the application loaded
2. 32-bit System directory (C:\Windows\System32)
3. 16-bit System directory (C:\Windows\System)
4. Windows directory (C:\Windows)
5. The current working directory (CWD)
6. Directories in the PATH environment variable (first system and then user)

# Interesting DLLs

## Windows 10

### WindowsCoreDeviceInfo.dll

**Status**

- Hijackable (Verified)

**Full path**

- C:\\Windows\\System32\\WindowsCoreDeviceInfo.dll

**Loaded process**

- svchost.exe (SYSTEM perm)
- usocoreworker.exe (SYSTEM perm)

**References**

- [https://itm4n.github.io/usodllloader-part1/](https://itm4n.github.io/usodllloader-part1/)

## Windows 7

In Windows 7, I found some dlls which checked by some processes but not exist. (Just place your dll to the path)

**But not clear it's hijackable dll**

- csrss.exe (SYSTEM perm)
    - Note
        - Only EN version Windows?
    - Dlls
        - C:\\Windows\\System32\\en-US\\Microsoft.Windows.Common-Controls.DLL
        - C:\\Windows\\System32\\en\\Microsoft.Windows.Common-Controls.DLL
- powershell.exe (user perm)
    - Dlls
        - C:\\Windows\\System32\\WindowsPowerShell\\v.1.0\\ATL.DLL
        - C:\\Windows\\System32\\WindowsPowerShell\\v.1.0\\USERENV.dll
- SearchProtocolHost.exe (SYSTEM perm)
    - Note
        - Checked twice every 6mins
    - Dlls
        - C:\\Windows\\System32\\msfte.dll
        - C:\\Windows\\System32\\msTracer.dll
- svchost.exe (SYSTEM perm)
    - Note
        - Check interval is long or need some behavior
    - Dlls
        - C:\\Users\user\AppData\Local\VirtualStore\Windows\System32\msxml3.dll
        - C:\\Windows\\SysWOW64\\vga.dll
        - C:\\Users\\<Username>\\Appdata\\Local\\VirtualStore\\Windows\\SYSTEM32\\ntdll.dll
            - checked by Mpengine.dll
- winsat.exe (user perm)
    - Dlls
        - C:\\Windows\\System32\\gdiplus.dll
- wmiprvse.exe (SYSTEM perm)
    - Note
        - Check interval is long or need some behavior
    - Dlls
        - C:\\Windows\\System32\\wbem\\CRYBTBASE.dll
        - C:\\Windows\\System32\\wbem\\ntmarta.dll
        - C:\\Windows\\System32\\wbem\\CRYPTSP.dll
        - C:\\Windows\\System32\\wbem\\RpcRtRemote.dll
- mscorsvw.exe (unknown perm)
    - Note
        - Check interval is long or need some behavior
    - Dlls
        - C:\\Windows\\SysWOW64\\rpcss.dll

# Interesting tools

## Siofra

- [https://github.com/Cybereason/siofra](https://github.com/Cybereason/siofra)

```
Siofra is a tool designed to identify and exploit DLL hijacking vulnerabilities 
in Windows programs. It is able to simulate the Windows loader in order to give 
visibility into all of the dependencies (and corresponding vulnerabilities) of 
a PE on disk, or alternatively an image file in memory corresponding to an active 
process. More significantly, the tool has the ability to easily generate DLLs to 
exploit these types of vulnerabilities via PE infection with dynamic shellcode creation. 
These infected DLLs retain the code (DllMain, exported functions) as well as the 
resources of a DLL to seamlessly preserve the functionality of the application loading 
them, while at the same time allowing the researcher to specify an executable payload 
to be either run as a separate process or loaded into the target as a module. Additionally, 
the tool contains automated methods of combining UAC auto-elevation criteria with 
the aforementioned functionality in order to scan for UAC bypass vulnerabilities.
```

# Helpful articles

- [https://www.wietzebeukema.nl/blog/hijacking-dlls-in-windows](https://www.wietzebeukema.nl/blog/hijacking-dlls-in-windows)
- [https://itm4n.github.io/windows-dll-hijacking-clarified/](https://itm4n.github.io/windows-dll-hijacking-clarified/)
