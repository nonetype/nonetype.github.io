---
layout: post
title: "Firefox tip note"
author: "nonetype"
categories: pwn
tags: browsers pwn_tips
---

Tip note for Firefox analysis & exploitation

# Index

<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [Index](#index)
- [Firefox note](#firefox-note)
  - [Build](#build)
  - [Debug Array object](#debug-array-object)
    - [Understand Type Masking](#understand-type-masking)
  - [Shape Debugging](#shape-debugging)
  - [What's 0xe5e5e5e5e5e5e5e5 ???](#whats-0xe5e5e5e5e5e5e5e5)
  - [Object storages](#object-storages)
    - [Inline properties](#inline-properties)
    - [Out-of-line properties](#out-of-line-properties)
    - [Elements](#elements)
    - [Summary](#summary)
- [Troubleshooting](#troubleshooting)
  - [TypeError: cannot create 'NoneType' instances while js/src/configure](#typeerror-cannot-create-nonetype-instances-while-jssrcconfigure)
    - [Solution](#solution)
  - [ERROR: Cannot find llvm-objdump](#error-cannot-find-llvm-objdump)
    - [Solution](#solution-1)
  - [ERROR: Could not find autoconf 2.13](#error-could-not-find-autoconf-213)
    - [Solution](#solution-2)
  - [ERROR: Cannot find cbindgen.](#error-cannot-find-cbindgen)
    - [Solution](#solution-3)
  - [ERROR: Could not find clang to generate run bindings for C/C++](#error-could-not-find-clang-to-generate-run-bindings-for-cc)
    - [Solution](#solution-4)
  - [ERROR: could not find Node.js executable later than 8.11](#error-could-not-find-nodejs-executable-later-than-811)
    - [Solution](#solution-5)
  - [ERROR: nasm 2.13 or greater is required for AV1 support](#error-nasm-213-or-greater-is-required-for-av1-support)
    - [Solution](#solution-6)
  - [ERROR: Yasm is required to build with ffvpx, jpeg, libav and vpx](#error-yasm-is-required-to-build-with-ffvpx-jpeg-libav-and-vpx)
    - [Solution](#solution-7)
  - [configure: error: Library requirements](#configure-error-library-requirements)
    - [Solution](#solution-8)
- [References](#references)

<!-- /code_chunk_output -->

# Firefox note

## Build

Firefox 소스 코드는 [여기](https://ftp.mozilla.org/pub/firefox/releases/72.0/source/)에서 받을 수 있다. (`ftp.mozilla.org/pub/firefox/releases/version to build/source`)

```bash
tar -xvf firefox-[version].source.tar.xz
sudo apt-get install nasm yasm clang nodejs rustc cargo llvm python-pip gcc make g++ perl python autoconf autoconf2.13 libgtk2.0-dev -y
mkdir build-[version]
cd build-[version]
../firefox-[version]/js/src/configure --enable-debug
make
./dist/bin/js
```

혹은 아래와 같이 `gecko-dev` Repo를 통해 빌드할 수 있다.

```bash
git clone https://github.com/mozilla/gecko-dev.git
cd gecko-dev
git checkout [commit number]
cd js/src
mkdir build.asserts
cd build.asserts
/bin/sh ../configure.in --enable-debug
make
```

이후 `js> ` 형식의 인터프리터 쉘 창이 뜨면 성공!

## Debug Array object

gdb를 통해 array object가 어떤 방식으로 참조되는지 알아보자.

```cpp
nonetype@box:~/hack/browser/firefox/build-72.0$ gdb dist/bin/js
Reading symbols from dist/bin/js...done.
warning: File "/home/nonetype/hack/browser/firefox/build-72.0/dist/bin/js-gdb.py" auto-loading has been declined by your 'auto-load safe-path' set to "$debugdir:$datadir/auto-load".
To enable execution of this file add
        add-auto-load-safe-path /home/nonetype/hack/browser/firefox/build-72.0/dist/bin/js-gdb.py
line to your configuration file "/home/nonetype/.gdbinit".
To completely disable this security protection add
        set auto-load safe-path /
line to your configuration file "/home/nonetype/.gdbinit".
For more information about this security protection see the
"Auto-loading safe path" section in the GDB manual.  E.g., run from the shell:
        info "(gdb)Auto-loading safe path"
GEF for linux ready, type 'gef' to start, 'gef config' to configure
77 commands loaded for GDB 8.1.0.20180409-git using Python engine 3.6
[*] 3 commands could not be loaded, run 'gef missing' to know why.
Error while writing index for '/home/nonetype/hack/browser/firefox/build-72.0/dist/bin/js': Can't open '/tmp/gef/js.gdb-index' for writing
gef➤  b * js::math_atan
Breakpoint 1 at 0xaf2e60: file /home/nonetype/hack/browser/firefox/firefox-72.0/js/src/jsmath.cpp, line 145.
gef➤  r
Starting program: /home/nonetype/hack/browser/firefox/build-72.0/dist/bin/js
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
[New Thread 0x7ffff648e700 (LWP 12491)]
[New Thread 0x7ffff5aff700 (LWP 12492)]
[Thread 0x7ffff648e700 (LWP 12491) exited]
[New Thread 0x7ffff5900700 (LWP 12493)]
js> a = [0x41414141, 0x42424242, 0x43434343]
[1094795585, 1111638594, 1128481603]
js> a.length = 0xdeadbeef
3735928559
js> Math.atan(a)
Thread 1 "js" hit Breakpoint 1, js::math_atan (cx=0x7ffff5b2a000, argc=0x1, vp=0x7ffff55e4098) at /home/nonetype/hack/browser/firefox/firefox-72.0/js/src/jsmath.cpp:145
145     bool js::math_atan(JSContext* cx, unsigned argc, Value* vp) {
gef➤  x/8gx vp
0x7ffff55e4098: 0xfffe001ee1b8db80      0xfffe001ee1b71140
0x7ffff55e40a8: 0xfffe001ee1b92040      0xfffb001ee1b4e820
0x7ffff55e40b8: 0xfffa000000000000      0x0000000000000011
0x7ffff55e40c8: 0x0000001ee1b3f0e0      0x0000001ee1b2f040
gef➤
```

인자로 전달된 array `a`의 객체 포인터는 `vp+0x10`위치에 존재한다.

여기서 특이한 점이, `vp+0x10` 위치에 존재하는 포인터, `0xfffe001ee1b92040`에서 타입 마스킹[^type_masking](여기서는 `0xfffe`, 다른 타입의 객체일 경우 값이 변화함)을 제거하기 위해 `& 0x7fffffffffff` 연산을 실행하고, 연산의 결과값이 실제 해당 객체가 존재하는 메모리 주소이다.

gdb에서는 다음과 같이 간편하게 계산 가능하다.

```cpp
gef➤  x/32gx 0xfffe001ee1b92040 & 0x7fffffffffff
0x1ee1b92040:   0x0000001ee1b6b820      0x0000001ee1b7cbd8
0x1ee1b92050:   0x0000000000000000      0x0000001ee1b92070
0x1ee1b92060:   0x0000000300000000      0xdeadbeef00000006
0x1ee1b92070:   0xfff8800041414141      0xfff8800042424242
0x1ee1b92080:   0xfff8800043434343      0x0000000000000000
0x1ee1b92090:   0x0000000000000000      0x0000000000000000
0x1ee1b920a0:   0x0000000000000000      0x0000000000000000
0x1ee1b920b0:   0x0000000000000000      0x0000000000000000
0x1ee1b920c0:   0x0000000000000000      0x0000000000000000
0x1ee1b920d0:   0x0000000000000000      0x0000000000000000
0x1ee1b920e0:   0x0000000000000000      0x0000000000000000
0x1ee1b920f0:   0x0000000000000000      0x0000000000000000
0x1ee1b92100:   0x0000000000000000      0x0000000000000000
0x1ee1b92110:   0x0000000000000000      0x0000000000000000
0x1ee1b92120:   0x0000000000000000      0x0000000000000000
0x1ee1b92130:   0x0000000000000000      0x0000000000000000
gef➤
```

Array의 인자로 넣었던 `0x41414141`, `0x42424242`, `0x43434343`과 `array.length`로 설정했던 `0xdeadbeef`도 보인다.

배열 인자 값들도 객체 포인터와 동일하게 타입 마스킹 처리되어 앞에 `0xfff88`이 붙어있는 것을 확인할 수 있다.

### Understand Type Masking 
> Integer 형식의 변수로 보면 타입 마스킹 처리를 이해하기 편하다.
> 먼저, 다음과 같은 `setInt32()` 함수가 존재한다.
> ```cpp
> void setInt32(int32_t i) {
>  asBits_ = bitsFromTagAndPayload(JSVAL_TAG_INT32, uint32_t(i));
>  MOZ_ASSERT(toInt32() == i);
> }
> ```
> 위 배열의 값을 예로 들면, 배열 인덱스에 값을 채울 때 `setInt32(0x41414141);`가 호출되고, 해당 함수 내에서 `bitsFromTagAndPayload` 함수가 호출된다.
> ```cpp
> static constexpr uint64_t bitsFromTagAndPayload(JSValueTag tag,
>                    PayloadType payload) {
>   return (uint64_t(tag) << JSVAL_TAG_SHIFT) | payload;
> }
> ```
> 
> 결과적으로 `(uint64_t(JSVAL_TAG_INT32) << JSVAL_TAG_SHIFT) | payload` 연산이 이루어지는데, 각 `JSVAL_TAG`는 다음과 같다.
> ```cpp
> #if defined(JS_NUNBOX32)
> #  define JSVAL_TAG_SHIFT 32 // if x86, 32
> #elif defined(JS_PUNBOX64)
> #  define JSVAL_TAG_SHIFT 47 // if x64, 47
> #endif
> ```
>
> ```cpp
> JSVAL_TYPE_INT32 = 0x01
> JSVAL_TAG_MAX_DOUBLE = 0x1FFF0
> JSVAL_TAG_INT32 = JSVAL_TAG_MAX_DOUBLE | JSVAL_TYPE_INT32
> ```
>
> 따라서 위 연산은 다음과 같다.
> ```cpp
> return ( uint64_t(0x1FFF1) << 47 ) | payload
> ```
> 
> 검증을 해보면 다음과 같다.
> ```py
> >>> hex((0x1FFF1 << 47) | 0x41414141)
> '0xfff8800041414141L'
> ```
>
> 타입 마스킹 계산 정복!!

## Shape Debugging
JSObject 관리는 properties와 shapes로 나누어 관리된다. (자세한 내용은 더 공부해봐야함)
여기서, 객체가 가지고 있는 속성을 표현하는 shapes를 디버깅하는 방법을 더듬더듬 짚어나가보자.

먼저 아래와 같은 코드를 작성해봤다.

**object.js**
```js
var o = {
        x: 0x41,
        y: 0x42
};
Math.atan(o);

o.z = 0x43;
Math.atan(o);

o[0] = 0x1337;
o[1] = 0x1338;
Math.atan(o);

o.a = 0x44;
Math.atan(o);
```

이후 아래와 같이 디버깅을 진행한다.

```cpp
nonetype@box:~/hack/browser/firefox/CVE-2019-9791/gecko-release/dist/bin$ lldb js
(lldb) target create "js"
Current executable set to 'js' (x86_64).
(lldb) b js::math_atan
Breakpoint 1: where = js`js::math_atan(JSContext*, unsigned int, JS::Value*) + 9 [inlined] bool math_function<&(js::math_atan_impl(double))>(JSContext*, unsigned int, JS::Value*) at jsmath.cpp:149, address = 0x0000000000bebae9
(lldb) r object.js
Process 5798 launched: '/home/nonetype/hack/browser/firefox/CVE-2019-9791/gecko-release/dist/bin/js' (x86_64)
Process 5798 stopped
* thread #1, name = 'js', stop reason = breakpoint 1.1
    frame #0: 0x000055555613fae9 js`js::math_atan(JSContext*, unsigned int, JS::Value*) [inlined] bool math_function<&(js::math_atan_impl(double))>(cx=0x00007ffff5b16000, argc=1, vp=0x00007ffff56f60a0) at jsmath.cpp:90
   87   template <UnaryMathFunctionType F>
   88   static bool math_function(JSContext* cx, unsigned argc, Value* vp) {
   89     CallArgs args = CallArgsFromVp(argc, vp);
-> 90     if (args.length() == 0) {
   91       args.rval().setNaN();
   92       return true;
   93     }
(lldb) x/3gx vp
0x7ffff56f60a0: 0xfffe00aee62aa4c0 0xfffe00aee6283180
0x7ffff56f60b0: 0xfffe00aee62811c0
(lldb) p *(JSObject*)0x00aee62811c0
(JSObject) $1 = {
  group_ = {
    js::WriteBarrieredBase<js::ObjectGroup *> = {
      js::BarrieredBase<js::ObjectGroup *> = {
        value = 0x000000aee627d2b0
      }
    }
  }
  shapeOrExpando_ = 0x000000aee62ab0b0
}
(lldb) p *(Shape*)$1.shapeOrExpando_
(js::Shape) $2 = {
  base_ = {
    js::WriteBarrieredBase<js::BaseShape *> = {
      js::BarrieredBase<js::BaseShape *> = {
        value = 0x000000aee627e0e0
      }
    }
  }
  propid_ = {
    js::WriteBarrieredBase<JS::PropertyKey> = {
      js::BarrieredBase<JS::PropertyKey> = {
        value = (asBits = 751185170272)
      }
    }
  }
  immutableFlags = 33554433
  attrs = '\x01'
  mutableFlags = '\0'
  parent = {
    js::WriteBarrieredBase<js::Shape *> = {
      js::BarrieredBase<js::Shape *> = {
        value = 0x000000aee62ab088
      }
    }
  }
   = {
    kids = (w = 0)
    listp = 0x0000000000000000
  }
}
(lldb) p (char*) ((JSString*)$2.propid_.value.asBits)->d.inlineStorageLatin1
(char *) $3 = 0x000000aee6200f68 "y"
(lldb)
```

객체의 인라인 캐시에 저장된 변수 y가 `JSString` 형식으로 저장된 것을 확인할 수 있다.
이어서 변수 x도 따라가보자.

```cpp
(lldb) p (char*) ((JSString*)$2.parent.value->propid_.value.asBits)->d.inlineStorageLatin1
(char *) $6 = 0x000000aee6200f48 "x"
(lldb)
```

이렇게 객체의 shape에 존재하는 변수명을 확인할 수 있다.

## What's 0xe5e5e5e5e5e5e5e5 ???

디버깅을 하다가 알 수 없는 값을 볼 수 있었다.

```cpp
(lldb) p *(NativeObject*)0x00aee62811c0
(js::NativeObject) $8 = {
  js::ShapedObject = {
    JSObject = {
      group_ = {
        js::WriteBarrieredBase<js::ObjectGroup *> = {
          js::BarrieredBase<js::ObjectGroup *> = {
            value = 0x000000aee627d2b0
          }
        }
      }
      shapeOrExpando_ = 0x000000aee62ab8a8
    }
  }
  slots_ = 0x00007ffff54db8c0
  elements_ = 0x000055555656c708
}
(lldb) x/4gx 0x00007ffff54db8c0
0x7ffff54db8c0: 0xfff8800000000043 0xe5e5e5e5e5e5e5e5
0x7ffff54db8d0: 0xe5e5e5e5e5e5e5e5 0xe5e5e5e5e5e5e5e5
(lldb)
```

`0x7ffff54db8c0` 위치의 0x43은 내가 넣은 값인데, 그 뒤에 존재하는 알 수 없는 `0xe5e5e5e5e5e5e5e5` 값들이 반복되고 있어서 구글링을 해봤다.

![1](https://i.imgur.com/V94KCvb.jpg)

bugzilla 사이트에선 별 정보를 얻지 못했는데, [첫번째 링크](https://mozilla.logbot.info/jsapi/20171016)에서 정보를 얻었다.

```
03:40 pbone: Good afternoon.
04:31 pbone: 0xe5e5e5e5e5e5e5e5
04:31 pbone: I know there's a bot here I can ask about that.
04:31 pbone: mrgiggles: can you help me?
07:08 evilpie: firebot: literal 0xe5
07:08 mrgiggles: evilpie: 0xe5 is jemalloc freed memory
07:08 mrgiggles: evilpie: if you're seeing a crash with this pattern, you have a use-after-free on your hands
07:08 mrgiggles: evilpie: and you'd better fix it. They tend to be security bugs!
```

`jemalloc`에서 할당 해제된 청크 내의 값은 0xe5 패턴으로 채워지는 것 같다.

## Object storages

object에는 크게 3가지의 저장소가 존재하는 듯 하다.

1. Inline storage(Inline properties)
2. Out-of-line storage(Out-of-line properties)
3. Elements

**object.js**
```js
var o = {
        x: 0x41, // 1. Inline properties
        y: 0x42
};
Math.atan(o);

o.z = 0x43; // 2. Out-of-line properties
Math.atan(o);

o[0] = 0x1337; // 3. Elements
o[1] = 0x1338;
Math.atan(o);

o.a = 0x44;
Math.atan(o);
```

디버깅을 통해 각각의 값들이 어디에 저장되는지 살펴보자.

### Inline properties
```cpp
nonetype@box:~/hack/browser/firefox/CVE-2019-9791/gecko-release/dist/bin$ lldb js
(lldb) target create "js"
Current executable set to 'js' (x86_64).
(lldb) b js::math_atan
Breakpoint 1: where = js`js::math_atan(JSContext*, unsigned int, JS::Value*) + 9 [inlined] bool math_function<&(js::math_atan_impl(double))>(JSContext*, unsigned int, JS::Value*) at jsmath.cpp:149, address = 0x0000000000bebae9
(lldb) r object.js
Process 7025 launched: '/home/nonetype/hack/browser/firefox/CVE-2019-9791/gecko-release/dist/bin/js' (x86_64)
Process 7025 stopped
* thread #1, name = 'js', stop reason = breakpoint 1.1
    frame #0: 0x000055555613fae9 js`js::math_atan(JSContext*, unsigned int, JS::Value*) [inlined] bool math_function<&(js::math_atan_impl(double))>(cx=0x00007ffff5b16000, argc=1, vp=0x00007ffff56f60a0) at jsmath.cpp:90
   87   template <UnaryMathFunctionType F>
   88   static bool math_function(JSContext* cx, unsigned argc, Value* vp) {
   89     CallArgs args = CallArgsFromVp(argc, vp);
-> 90     if (args.length() == 0) {
   91       args.rval().setNaN();
   92       return true;
   93     }
(lldb) x/3gx vp
0x7ffff56f60a0: 0xfffe3d5accfaa4c0 0xfffe3d5accf83180
0x7ffff56f60b0: 0xfffe3d5accf811c0
(lldb) p *(NativeObject*)0x3d5accf811c0
(js::NativeObject) $1 = {
  js::ShapedObject = {
    JSObject = {
      group_ = {
        js::WriteBarrieredBase<js::ObjectGroup *> = {
          js::BarrieredBase<js::ObjectGroup *> = {
            value = 0x00003d5accf7d2b0
          }
        }
      }
      shapeOrExpando_ = 0x00003d5accfab0b0
    }
  }
  slots_ = 0x0000000000000000
  elements_ = 0x000055555656c708
}
(lldb) x/8gx 0x3d5accf811c0
0x3d5accf811c0: 0x00003d5accf7d2b0 0x00003d5accfab0b0
0x3d5accf811d0: 0x0000000000000000 0x000055555656c708
0x3d5accf811e0: 0xfff8800000000041 0xfff8800000000042 // Inline properties HERE!
0x3d5accf811f0: 0x0000000000000000 0x0000000000000000
(lldb)
```
객체 생성과 동시에 선언되기 때문에 객체의 Shape 포인터 밑에 값이 존재한다.

### Out-of-line properties
```cpp
(lldb) c
Process 7025 resuming
Process 7025 stopped
* thread #1, name = 'js', stop reason = breakpoint 1.1
    frame #0: 0x000055555613fae9 js`js::math_atan(JSContext*, unsigned int, JS::Value*) [inlined] bool math_function<&(js::math_atan_impl(double))>(cx=0x00007ffff5b16000, argc=1, vp=0x00007ffff56f60a0) at jsmath.cpp:90
   87   template <UnaryMathFunctionType F>
   88   static bool math_function(JSContext* cx, unsigned argc, Value* vp) {
   89     CallArgs args = CallArgsFromVp(argc, vp);
-> 90     if (args.length() == 0) {
   91       args.rval().setNaN();
   92       return true;
   93     }
(lldb) p *$1.slots_
(js::HeapSlot) $2 = {
  js::WriteBarrieredBase<JS::Value> = {
    js::BarrieredBase<JS::Value> = {
      value = {
        asBits_ = 18444633011384221763
        asDouble_ = NaN
        debugView_ = (payload47_ = 67, tag_ = JSVAL_TAG_INT32)
        s_ = {
          payload_ = (i32_ = 67, u32_ = 67, why_ = 67)
        }
      }
    }
  }
}
(lldb) p $1.slots_
(js::HeapSlot *) $3 = 0x00007ffff54db8c0
(lldb) x/8gx 0x00007ffff54db8c0
0x7ffff54db8c0: 0xfff8800000000043 0xe5e5e5e5e5e5e5e5 // Out-of-line properties HERE!
0x7ffff54db8d0: 0xe5e5e5e5e5e5e5e5 0xe5e5e5e5e5e5e5e5
0x7ffff54db8e0: 0xe5e5e5e5e5e5e5e5 0xe5e5e5e5e5e5e5e5
0x7ffff54db8f0: 0xe5e5e5e5e5e5e5e5 0xe5e5e5e5e5e5e5e5
(lldb)
```

### Elements
```cpp
(lldb) c
Process 7025 resuming
Process 7025 stopped
* thread #1, name = 'js', stop reason = breakpoint 1.1
    frame #0: 0x000055555613fae9 js`js::math_atan(JSContext*, unsigned int, JS::Value*) [inlined] bool math_function<&(js::math_atan_impl(double))>(cx=0x00007ffff5b16000, argc=1, vp=0x00007ffff56f60a0) at jsmath.cpp:90
   87   template <UnaryMathFunctionType F>
   88   static bool math_function(JSContext* cx, unsigned argc, Value* vp) {
   89     CallArgs args = CallArgsFromVp(argc, vp);
-> 90     if (args.length() == 0) {
   91       args.rval().setNaN();
   92       return true;
   93     }
(lldb) p $1.elements_
(js::HeapSlot *) $4 = 0x00007ffff54db910
(lldb) x/8gx 0x00007ffff54db910
0x7ffff54db910: 0xfff8800000001337 0xfff8800000001338 // Elements HERE!
0x7ffff54db920: 0x0000000000000000 0x0000000000000000
0x7ffff54db930: 0x0000000000000000 0x0000000000000000
0x7ffff54db940: 0x0000000000000000 0x0000000000000000
(lldb)
```

### Summary
```cpp
(lldb) x/8gx 0x3d5accf811c0 // It's object's address
0x3d5accf811c0: 0x00003d5accf7d2b0(Group) 0x00003d5accfab8a8(Shape)
0x3d5accf811d0: 0x00007ffff54db8c0(Out-of-line props) 0x00007ffff54db910(Elements)
0x3d5accf811e0: 0xfff8800000000041 0xfff8800000000042(Inline props)
0x3d5accf811f0: 0x0000000000000000 0x0000000000000000
```

# Troubleshooting

## TypeError: cannot create 'NoneType' instances while js/src/configure

```bash
 nonetype@box  ~/hack/browser/firefox/build  ../firefox-72.0/js/src/configure
Reexecuting in the virtualenv
checking for vcs source checkout... no
checking for a shell... /bin/sh
checking for host system type... x86_64-pc-linux-gnu
checking for target system type... x86_64-pc-linux-gnu
checking for Python 3... /usr/bin/python3 (3.6.9)
checking whether cross compiling... no
checking for yasm... not found
checking for the target C compiler... /usr/bin/gcc
checking whether the target C compiler can be used... yes
checking the target C compiler version... 7.5.0
checking the target C compiler works... yes
checking for the target C++ compiler... /usr/bin/g++
checking whether the target C++ compiler can be used... yes
checking the target C++ compiler version... 7.5.0
checking the target C++ compiler works... yes
checking for the host C compiler... /usr/bin/gcc
checking whether the host C compiler can be used... yes
checking the host C compiler version... 7.5.0
checking the host C compiler works... yes
checking for the host C++ compiler... /usr/bin/g++
checking whether the host C++ compiler can be used... yes
checking the host C++ compiler version... 7.5.0
checking the host C++ compiler works... yes
checking for 64-bit OS... yes
checking for nasm... not found
checking for linker... bfd
checking for the assembler... /usr/bin/gcc
checking for ar... /usr/bin/ar
checking for pkg_config... /usr/bin/pkg-config
checking for pkg-config version... 0.29.1
checking for stdint.h... yes
checking for inttypes.h... yes
checking for malloc.h... yes
checking for alloca.h... yes
checking for sys/byteorder.h... no
checking for getopt.h... yes
checking for unistd.h... yes
checking for nl_types.h... yes
checking for cpuid.h... yes
checking for sys/statvfs.h... yes
checking for sys/statfs.h... yes
checking for sys/vfs.h... yes
checking for sys/mount.h... yes
checking for sys/quota.h... yes
checking for linux/quota.h... yes
checking for linux/if_addr.h... yes
checking for linux/rtnetlink.h... yes
checking for sys/queue.h... yes
checking for sys/types.h... yes
checking for netinet/in.h... yes
checking for byteswap.h... yes
checking for linux/perf_event.h... yes
checking for perf_event_open system call... yes
checking whether the C compiler supports -Wbitfield-enum-conversion... no
checking whether the C++ compiler supports -Wbitfield-enum-conversion... no
checking whether the C compiler supports -Wshadow-field-in-constructor-modified... no
checking whether the C++ compiler supports -Wshadow-field-in-constructor-modified... no
checking whether the C compiler supports -Wunreachable-code-return... no
checking whether the C++ compiler supports -Wunreachable-code-return... no
checking whether the C compiler supports -Wclass-varargs... no
checking whether the C++ compiler supports -Wclass-varargs... no
checking whether the C compiler supports -Wfloat-overflow-conversion... no
checking whether the C++ compiler supports -Wfloat-overflow-conversion... no
checking whether the C compiler supports -Wfloat-zero-conversion... no
checking whether the C++ compiler supports -Wfloat-zero-conversion... no
checking whether the C compiler supports -Wloop-analysis... no
checking whether the C++ compiler supports -Wloop-analysis... no
checking whether the C++ compiler supports -Wc++1z-compat... yes
checking whether the C++ compiler supports -Wc++2a-compat... no
checking whether the C++ compiler supports -Wcomma... no
checking whether the C compiler supports -Wduplicated-cond... yes
checking whether the C++ compiler supports -Wduplicated-cond... yes
checking whether the C++ compiler supports -Wimplicit-fallthrough... yes
checking whether the C compiler supports -Wstring-conversion... no
checking whether the C++ compiler supports -Wstring-conversion... no
checking whether the C compiler supports -Wtautological-overlap-compare... no
checking whether the C++ compiler supports -Wtautological-overlap-compare... no
checking whether the C compiler supports -Wtautological-unsigned-enum-zero-compare... no
checking whether the C++ compiler supports -Wtautological-unsigned-enum-zero-compare... no
checking whether the C compiler supports -Wtautological-unsigned-zero-compare... no
checking whether the C++ compiler supports -Wtautological-unsigned-zero-compare... no
checking whether the C compiler supports -Wno-error=tautological-type-limit-compare... no
checking whether the C++ compiler supports -Wno-error=tautological-type-limit-compare... no
checking whether the C++ compiler supports -Wno-inline-new-delete... no
checking whether the C compiler supports -Wno-error=maybe-uninitialized... yes
checking whether the C++ compiler supports -Wno-error=maybe-uninitialized... yes
checking whether the C compiler supports -Wno-error=deprecated-declarations... yes
checking whether the C++ compiler supports -Wno-error=deprecated-declarations... yes
checking whether the C compiler supports -Wno-error=array-bounds... yes
checking whether the C++ compiler supports -Wno-error=array-bounds... yes
checking whether the C compiler supports -Wno-error=coverage-mismatch... yes
checking whether the C++ compiler supports -Wno-error=coverage-mismatch... yes
checking whether the C compiler supports -Wno-error=backend-plugin... no
checking whether the C++ compiler supports -Wno-error=backend-plugin... no
checking whether the C compiler supports -Wno-error=free-nonheap-object... yes
checking whether the C++ compiler supports -Wno-error=free-nonheap-object... yes
checking whether the C compiler supports -Wno-error=multistatement-macros... no
checking whether the C++ compiler supports -Wno-error=multistatement-macros... no
checking whether the C compiler supports -Wno-error=return-std-move... no
checking whether the C++ compiler supports -Wno-error=return-std-move... no
checking whether the C compiler supports -Wno-error=class-memaccess... no
checking whether the C++ compiler supports -Wno-error=class-memaccess... no
checking whether the C compiler supports -Wno-error=atomic-alignment... no
checking whether the C++ compiler supports -Wno-error=atomic-alignment... no
checking whether the C compiler supports -Wno-error=deprecated-copy... no
checking whether the C++ compiler supports -Wno-error=deprecated-copy... no
checking whether the C compiler supports -Wformat... yes
checking whether the C++ compiler supports -Wformat... yes
checking whether the C compiler supports -Wformat-security... yes
checking whether the C++ compiler supports -Wformat-security... yes
checking whether the C compiler supports -Wformat-overflow=2... yes
checking whether the C++ compiler supports -Wformat-overflow=2... yes
checking whether the C compiler supports -Wno-gnu-zero-variadic-macro-arguments... no
checking whether the C++ compiler supports -Wno-gnu-zero-variadic-macro-arguments... no
checking whether the C++ compiler supports -Wno-noexcept-type... yes
checking whether the C++ compiler supports -fno-sized-deallocation... yes
checking whether the C++ compiler supports -fno-aligned-new... yes
checking for rustc... not found
checking for cargo... not found
Traceback (most recent call last):
  File "../firefox-72.0/js/src/../../configure.py", line 170, in <module>
    sys.exit(main(sys.argv))
  File "../firefox-72.0/js/src/../../configure.py", line 46, in main
    sandbox.run(os.path.join(os.path.dirname(__file__), 'moz.configure'))
  File "/home/nonetype/hack/browser/firefox/firefox-72.0/python/mozbuild/mozbuild/configure/__init__.py", line 499, in run
    func(*args)
  File "/home/nonetype/hack/browser/firefox/firefox-72.0/python/mozbuild/mozbuild/configure/__init__.py", line 543, in _value_for
    return self._value_for_depends(obj)
  File "/home/nonetype/hack/browser/firefox/firefox-72.0/python/mozbuild/mozbuild/util.py", line 997, in method_call
    cache[args] = self.func(instance, *args)
  File "/home/nonetype/hack/browser/firefox/firefox-72.0/python/mozbuild/mozbuild/configure/__init__.py", line 552, in _value_for_depends
    value = obj.result()
  File "/home/nonetype/hack/browser/firefox/firefox-72.0/python/mozbuild/mozbuild/util.py", line 997, in method_call
    cache[args] = self.func(instance, *args)
  File "/home/nonetype/hack/browser/firefox/firefox-72.0/python/mozbuild/mozbuild/configure/__init__.py", line 155, in result
    return self._func(*resolved_args)
  File "/home/nonetype/hack/browser/firefox/firefox-72.0/python/mozbuild/mozbuild/configure/__init__.py", line 1152, in wrapped
    return new_func(*args, **kwargs)
  File "/home/nonetype/hack/browser/firefox/firefox-72.0/build/moz.configure/rust.configure", line 59, in unwrap
    (retcode, stdout, stderr) = get_cmd_output(prog, '+stable')
  File "/home/nonetype/hack/browser/firefox/firefox-72.0/python/mozbuild/mozbuild/configure/__init__.py", line 1152, in wrapped
    return new_func(*args, **kwargs)
  File "/home/nonetype/hack/browser/firefox/firefox-72.0/build/moz.configure/util.configure", line 30, in get_cmd_output
    log.debug('Executing: `%s`', quote(*args))
  File "/home/nonetype/hack/browser/firefox/firefox-72.0/python/mozbuild/mozbuild/shellutil.py", line 210, in quote
    return ' '.join(_quote(s) for s in strings)
  File "/home/nonetype/hack/browser/firefox/firefox-72.0/python/mozbuild/mozbuild/shellutil.py", line 210, in <genexpr>
    return ' '.join(_quote(s) for s in strings)
  File "/home/nonetype/hack/browser/firefox/firefox-72.0/python/mozbuild/mozbuild/shellutil.py", line 198, in _quote
    return t("'%s'") % s.replace(t("'"), t("'\\''"))
TypeError: cannot create 'NoneType' instances
```

[reference](https://bugzilla.mozilla.org/show_bug.cgi?id=1598642): rustc and cargo aren't installed

### Solution

```bash
sudo apt install rustc cargo
```

## ERROR: Cannot find llvm-objdump

```bash
 nonetype@box  ~/hack/browser/firefox/build  ../firefox-72.0/js/src/configure
Reexecuting in the virtualenv
checking for vcs source checkout... no
checking for a shell... /bin/sh
checking for host system type... x86_64-pc-linux-gnu
checking for target system type... x86_64-pc-linux-gnu
checking for Python 3... /usr/bin/python3 (3.6.9)
checking whether cross compiling... no
checking for yasm... not found
checking for the target C compiler... /usr/bin/gcc
checking whether the target C compiler can be used... yes
checking the target C compiler version... 7.5.0
checking the target C compiler works... yes
checking for the target C++ compiler... /usr/bin/g++
checking whether the target C++ compiler can be used... yes
checking the target C++ compiler version... 7.5.0
checking the target C++ compiler works... yes
checking for the host C compiler... /usr/bin/gcc
checking whether the host C compiler can be used... yes
checking the host C compiler version... 7.5.0
checking the host C compiler works... yes
checking for the host C++ compiler... /usr/bin/g++
checking whether the host C++ compiler can be used... yes
checking the host C++ compiler version... 7.5.0
checking the host C++ compiler works... yes
checking for 64-bit OS... yes
checking for nasm... not found
checking for linker... bfd
checking for the assembler... /usr/bin/gcc
checking for ar... /usr/bin/ar
checking for pkg_config... /usr/bin/pkg-config
checking for pkg-config version... 0.29.1
checking for stdint.h... yes
checking for inttypes.h... yes
checking for malloc.h... yes
checking for alloca.h... yes
checking for sys/byteorder.h... no
checking for getopt.h... yes
checking for unistd.h... yes
checking for nl_types.h... yes
checking for cpuid.h... yes
checking for sys/statvfs.h... yes
checking for sys/statfs.h... yes
checking for sys/vfs.h... yes
checking for sys/mount.h... yes
checking for sys/quota.h... yes
checking for linux/quota.h... yes
checking for linux/if_addr.h... yes
checking for linux/rtnetlink.h... yes
checking for sys/queue.h... yes
checking for sys/types.h... yes
checking for netinet/in.h... yes
checking for byteswap.h... yes
checking for linux/perf_event.h... yes
checking for perf_event_open system call... yes
checking whether the C compiler supports -Wbitfield-enum-conversion... no
checking whether the C++ compiler supports -Wbitfield-enum-conversion... no
checking whether the C compiler supports -Wshadow-field-in-constructor-modified... no
checking whether the C++ compiler supports -Wshadow-field-in-constructor-modified... no
checking whether the C compiler supports -Wunreachable-code-return... no
checking whether the C++ compiler supports -Wunreachable-code-return... no
checking whether the C compiler supports -Wclass-varargs... no
checking whether the C++ compiler supports -Wclass-varargs... no
checking whether the C compiler supports -Wfloat-overflow-conversion... no
checking whether the C++ compiler supports -Wfloat-overflow-conversion... no
checking whether the C compiler supports -Wfloat-zero-conversion... no
checking whether the C++ compiler supports -Wfloat-zero-conversion... no
checking whether the C compiler supports -Wloop-analysis... no
checking whether the C++ compiler supports -Wloop-analysis... no
checking whether the C++ compiler supports -Wc++1z-compat... yes
checking whether the C++ compiler supports -Wc++2a-compat... no
checking whether the C++ compiler supports -Wcomma... no
checking whether the C compiler supports -Wduplicated-cond... yes
checking whether the C++ compiler supports -Wduplicated-cond... yes
checking whether the C++ compiler supports -Wimplicit-fallthrough... yes
checking whether the C compiler supports -Wstring-conversion... no
checking whether the C++ compiler supports -Wstring-conversion... no
checking whether the C compiler supports -Wtautological-overlap-compare... no
checking whether the C++ compiler supports -Wtautological-overlap-compare... no
checking whether the C compiler supports -Wtautological-unsigned-enum-zero-compare... no
checking whether the C++ compiler supports -Wtautological-unsigned-enum-zero-compare... no
checking whether the C compiler supports -Wtautological-unsigned-zero-compare... no
checking whether the C++ compiler supports -Wtautological-unsigned-zero-compare... no
checking whether the C compiler supports -Wno-error=tautological-type-limit-compare... no
checking whether the C++ compiler supports -Wno-error=tautological-type-limit-compare... no
checking whether the C++ compiler supports -Wno-inline-new-delete... no
checking whether the C compiler supports -Wno-error=maybe-uninitialized... yes
checking whether the C++ compiler supports -Wno-error=maybe-uninitialized... yes
checking whether the C compiler supports -Wno-error=deprecated-declarations... yes
checking whether the C++ compiler supports -Wno-error=deprecated-declarations... yes
checking whether the C compiler supports -Wno-error=array-bounds... yes
checking whether the C++ compiler supports -Wno-error=array-bounds... yes
checking whether the C compiler supports -Wno-error=coverage-mismatch... yes
checking whether the C++ compiler supports -Wno-error=coverage-mismatch... yes
checking whether the C compiler supports -Wno-error=backend-plugin... no
checking whether the C++ compiler supports -Wno-error=backend-plugin... no
checking whether the C compiler supports -Wno-error=free-nonheap-object... yes
checking whether the C++ compiler supports -Wno-error=free-nonheap-object... yes
checking whether the C compiler supports -Wno-error=multistatement-macros... no
checking whether the C++ compiler supports -Wno-error=multistatement-macros... no
checking whether the C compiler supports -Wno-error=return-std-move... no
checking whether the C++ compiler supports -Wno-error=return-std-move... no
checking whether the C compiler supports -Wno-error=class-memaccess... no
checking whether the C++ compiler supports -Wno-error=class-memaccess... no
checking whether the C compiler supports -Wno-error=atomic-alignment... no
checking whether the C++ compiler supports -Wno-error=atomic-alignment... no
checking whether the C compiler supports -Wno-error=deprecated-copy... no
checking whether the C++ compiler supports -Wno-error=deprecated-copy... no
checking whether the C compiler supports -Wformat... yes
checking whether the C++ compiler supports -Wformat... yes
checking whether the C compiler supports -Wformat-security... yes
checking whether the C++ compiler supports -Wformat-security... yes
checking whether the C compiler supports -Wformat-overflow=2... yes
checking whether the C++ compiler supports -Wformat-overflow=2... yes
checking whether the C compiler supports -Wno-gnu-zero-variadic-macro-arguments... no
checking whether the C++ compiler supports -Wno-gnu-zero-variadic-macro-arguments... no
checking whether the C++ compiler supports -Wno-noexcept-type... yes
checking whether the C++ compiler supports -fno-sized-deallocation... yes
checking whether the C++ compiler supports -fno-aligned-new... yes
checking for rustc... /usr/bin/rustc
checking for cargo... /usr/bin/cargo
checking rustc version... 1.39.0
checking cargo version... 1.39.0
checking for rust target triplet... x86_64-unknown-linux-gnu
checking for rust host triplet... x86_64-unknown-linux-gnu
checking for rustdoc... /usr/bin/rustdoc
checking for rustfmt... not found
checking for clang for bindgen... not found
checking for libclang for bindgen... not found
checking bindgen cflags... -x c++ -fno-sized-deallocation -fno-aligned-new -DTRACING=1 -DIMPL_LIBXUL -DMOZILLA_INTERNAL_API -DRUST_BINDGEN -DOS_POSIX=1 -DOS_LINUX=1
checking for llvm_profdata... not found
checking for awk... /usr/bin/gawk
checking for perl... /usr/bin/perl
checking for minimum required perl version >= 5.006... 5.026001
checking for full perl installation... yes
checking for gmake... /usr/bin/make
checking for watchman... not found
checking for xargs... /usr/bin/xargs
checking for rpmbuild... /usr/bin/rpmbuild
checking for llvm-objdump... not found
DEBUG: llvm_objdump: Trying llvm-objdump
ERROR: Cannot find llvm-objdump
```

### Solution

```bash
sudo apt install llvm
```

## ERROR: Could not find autoconf 2.13

```bash
 nonetype@box  ~/hack/browser/firefox/build  ../firefox-72.0/js/src/configure
Reexecuting in the virtualenv
checking for vcs source checkout... no
checking for a shell... /bin/sh
checking for host system type... x86_64-pc-linux-gnu
checking for target system type... x86_64-pc-linux-gnu
checking for Python 3... /usr/bin/python3 (3.6.9)
checking whether cross compiling... no
checking for yasm... not found
checking for the target C compiler... /usr/bin/gcc
checking whether the target C compiler can be used... yes
checking the target C compiler version... 7.5.0
checking the target C compiler works... yes
checking for the target C++ compiler... /usr/bin/g++
checking whether the target C++ compiler can be used... yes
checking the target C++ compiler version... 7.5.0
checking the target C++ compiler works... yes
checking for the host C compiler... /usr/bin/gcc
checking whether the host C compiler can be used... yes
checking the host C compiler version... 7.5.0
checking the host C compiler works... yes
checking for the host C++ compiler... /usr/bin/g++
checking whether the host C++ compiler can be used... yes
checking the host C++ compiler version... 7.5.0
checking the host C++ compiler works... yes
checking for 64-bit OS... yes
checking for nasm... not found
checking for linker... bfd
checking for the assembler... /usr/bin/gcc
checking for ar... /usr/bin/ar
checking for pkg_config... /usr/bin/pkg-config
checking for pkg-config version... 0.29.1
checking for stdint.h... yes
checking for inttypes.h... yes
checking for malloc.h... yes
checking for alloca.h... yes
checking for sys/byteorder.h... no
checking for getopt.h... yes
checking for unistd.h... yes
checking for nl_types.h... yes
checking for cpuid.h... yes
checking for sys/statvfs.h... yes
checking for sys/statfs.h... yes
checking for sys/vfs.h... yes
checking for sys/mount.h... yes
checking for sys/quota.h... yes
checking for linux/quota.h... yes
checking for linux/if_addr.h... yes
checking for linux/rtnetlink.h... yes
checking for sys/queue.h... yes
checking for sys/types.h... yes
checking for netinet/in.h... yes
checking for byteswap.h... yes
checking for linux/perf_event.h... yes
checking for perf_event_open system call... yes
checking whether the C compiler supports -Wbitfield-enum-conversion... no
checking whether the C++ compiler supports -Wbitfield-enum-conversion... no
checking whether the C compiler supports -Wshadow-field-in-constructor-modified... no
checking whether the C++ compiler supports -Wshadow-field-in-constructor-modified... no
checking whether the C compiler supports -Wunreachable-code-return... no
checking whether the C++ compiler supports -Wunreachable-code-return... no
checking whether the C compiler supports -Wclass-varargs... no
checking whether the C++ compiler supports -Wclass-varargs... no
checking whether the C compiler supports -Wfloat-overflow-conversion... no
checking whether the C++ compiler supports -Wfloat-overflow-conversion... no
checking whether the C compiler supports -Wfloat-zero-conversion... no
checking whether the C++ compiler supports -Wfloat-zero-conversion... no
checking whether the C compiler supports -Wloop-analysis... no
checking whether the C++ compiler supports -Wloop-analysis... no
checking whether the C++ compiler supports -Wc++1z-compat... yes
checking whether the C++ compiler supports -Wc++2a-compat... no
checking whether the C++ compiler supports -Wcomma... no
checking whether the C compiler supports -Wduplicated-cond... yes
checking whether the C++ compiler supports -Wduplicated-cond... yes
checking whether the C++ compiler supports -Wimplicit-fallthrough... yes
checking whether the C compiler supports -Wstring-conversion... no
checking whether the C++ compiler supports -Wstring-conversion... no
checking whether the C compiler supports -Wtautological-overlap-compare... no
checking whether the C++ compiler supports -Wtautological-overlap-compare... no
checking whether the C compiler supports -Wtautological-unsigned-enum-zero-compare... no
checking whether the C++ compiler supports -Wtautological-unsigned-enum-zero-compare... no
checking whether the C compiler supports -Wtautological-unsigned-zero-compare... no
checking whether the C++ compiler supports -Wtautological-unsigned-zero-compare... no
checking whether the C compiler supports -Wno-error=tautological-type-limit-compare... no
checking whether the C++ compiler supports -Wno-error=tautological-type-limit-compare... no
checking whether the C++ compiler supports -Wno-inline-new-delete... no
checking whether the C compiler supports -Wno-error=maybe-uninitialized... yes
checking whether the C++ compiler supports -Wno-error=maybe-uninitialized... yes
checking whether the C compiler supports -Wno-error=deprecated-declarations... yes
checking whether the C++ compiler supports -Wno-error=deprecated-declarations... yes
checking whether the C compiler supports -Wno-error=array-bounds... yes
checking whether the C++ compiler supports -Wno-error=array-bounds... yes
checking whether the C compiler supports -Wno-error=coverage-mismatch... yes
checking whether the C++ compiler supports -Wno-error=coverage-mismatch... yes
checking whether the C compiler supports -Wno-error=backend-plugin... no
checking whether the C++ compiler supports -Wno-error=backend-plugin... no
checking whether the C compiler supports -Wno-error=free-nonheap-object... yes
checking whether the C++ compiler supports -Wno-error=free-nonheap-object... yes
checking whether the C compiler supports -Wno-error=multistatement-macros... no
checking whether the C++ compiler supports -Wno-error=multistatement-macros... no
checking whether the C compiler supports -Wno-error=return-std-move... no
checking whether the C++ compiler supports -Wno-error=return-std-move... no
checking whether the C compiler supports -Wno-error=class-memaccess... no
checking whether the C++ compiler supports -Wno-error=class-memaccess... no
checking whether the C compiler supports -Wno-error=atomic-alignment... no
checking whether the C++ compiler supports -Wno-error=atomic-alignment... no
checking whether the C compiler supports -Wno-error=deprecated-copy... no
checking whether the C++ compiler supports -Wno-error=deprecated-copy... no
checking whether the C compiler supports -Wformat... yes
checking whether the C++ compiler supports -Wformat... yes
checking whether the C compiler supports -Wformat-security... yes
checking whether the C++ compiler supports -Wformat-security... yes
checking whether the C compiler supports -Wformat-overflow=2... yes
checking whether the C++ compiler supports -Wformat-overflow=2... yes
checking whether the C compiler supports -Wno-gnu-zero-variadic-macro-arguments... no
checking whether the C++ compiler supports -Wno-gnu-zero-variadic-macro-arguments... no
checking whether the C++ compiler supports -Wno-noexcept-type... yes
checking whether the C++ compiler supports -fno-sized-deallocation... yes
checking whether the C++ compiler supports -fno-aligned-new... yes
checking for rustc... /usr/bin/rustc
checking for cargo... /usr/bin/cargo
checking rustc version... 1.39.0
checking cargo version... 1.39.0
checking for rust target triplet... x86_64-unknown-linux-gnu
checking for rust host triplet... x86_64-unknown-linux-gnu
checking for rustdoc... /usr/bin/rustdoc
checking for rustfmt... not found
checking for clang for bindgen... not found
checking for libclang for bindgen... not found
checking bindgen cflags... -x c++ -fno-sized-deallocation -fno-aligned-new -DTRACING=1 -DIMPL_LIBXUL -DMOZILLA_INTERNAL_API -DRUST_BINDGEN -DOS_POSIX=1 -DOS_LINUX=1
checking for llvm_profdata... /usr/bin/llvm-profdata
checking for awk... /usr/bin/gawk
checking for perl... /usr/bin/perl
checking for minimum required perl version >= 5.006... 5.026001
checking for full perl installation... yes
checking for gmake... /usr/bin/make
checking for watchman... not found
checking for xargs... /usr/bin/xargs
checking for rpmbuild... /usr/bin/rpmbuild
checking for llvm-objdump... /usr/bin/llvm-objdump
checking for autoconf...
ERROR: Could not find autoconf 2.13
```

### Solution

```bash
sudo apt install autoconf2.13
```

## ERROR: Cannot find cbindgen.

```bash
nonetype@box:~/hack/browser/firefox/build-72.0-debug$ ../firefox-72.0/configure --enable-debug
Creating Python environment
New python executable in /home/nonetype/hack/browser/firefox/build-72.0-debug/_virtualenvs/init/bin/python2.7
Also creating executable in /home/nonetype/hack/browser/firefox/build-72.0-debug/_virtualenvs/init/bin/python
Installing setuptools, pip, wheel...done.
running build_ext
copying build/lib.linux-x86_64-2.7/psutil/_psutil_linux.so -> psutil
copying build/lib.linux-x86_64-2.7/psutil/_psutil_posix.so -> psutil

Error processing command. Ignoring because optional. (optional:packages.txt:comm/build/virtualenv_packages.txt)
Reexecuting in the virtualenv
checking for vcs source checkout... no
checking for a shell... /bin/sh
checking for host system type... x86_64-pc-linux-gnu
checking for target system type... x86_64-pc-linux-gnu
checking whether cross compiling... no
checking for Python 3... /usr/bin/python3 (3.6.9)
checking for yasm... not found
checking for the target C compiler... /usr/bin/gcc
checking whether the target C compiler can be used... yes
checking the target C compiler version... 7.5.0
checking the target C compiler works... yes
checking for the target C++ compiler... /usr/bin/g++
checking whether the target C++ compiler can be used... yes
checking the target C++ compiler version... 7.5.0
checking the target C++ compiler works... yes
checking for the host C compiler... /usr/bin/gcc
checking whether the host C compiler can be used... yes
checking the host C compiler version... 7.5.0
checking the host C compiler works... yes
checking for the host C++ compiler... /usr/bin/g++
checking whether the host C++ compiler can be used... yes
checking the host C++ compiler version... 7.5.0
checking the host C++ compiler works... yes
checking for 64-bit OS... yes
checking for nasm... not found
checking for linker... bfd
checking for the assembler... /usr/bin/gcc
checking for ar... /usr/bin/ar
checking for pkg_config... /usr/bin/pkg-config
checking for pkg-config version... 0.29.1
checking for stdint.h... yes
checking for inttypes.h... yes
checking for malloc.h... yes
checking for alloca.h... yes
checking for sys/byteorder.h... no
checking for getopt.h... yes
checking for unistd.h... yes
checking for nl_types.h... yes
checking for cpuid.h... yes
checking for sys/statvfs.h... yes
checking for sys/statfs.h... yes
checking for sys/vfs.h... yes
checking for sys/mount.h... yes
checking for sys/quota.h... yes
checking for linux/quota.h... yes
checking for linux/if_addr.h... yes
checking for linux/rtnetlink.h... yes
checking for sys/queue.h... yes
checking for sys/types.h... yes
checking for netinet/in.h... yes
checking for byteswap.h... yes
checking for linux/perf_event.h... yes
checking for perf_event_open system call... yes
checking whether the C compiler supports -Wbitfield-enum-conversion... no
checking whether the C++ compiler supports -Wbitfield-enum-conversion... no
checking whether the C compiler supports -Wshadow-field-in-constructor-modified... no
checking whether the C++ compiler supports -Wshadow-field-in-constructor-modified... no
checking whether the C compiler supports -Wunreachable-code-return... no
checking whether the C++ compiler supports -Wunreachable-code-return... no
checking whether the C compiler supports -Wclass-varargs... no
checking whether the C++ compiler supports -Wclass-varargs... no
checking whether the C compiler supports -Wfloat-overflow-conversion... no
checking whether the C++ compiler supports -Wfloat-overflow-conversion... no
checking whether the C compiler supports -Wfloat-zero-conversion... no
checking whether the C++ compiler supports -Wfloat-zero-conversion... no
checking whether the C compiler supports -Wloop-analysis... no
checking whether the C++ compiler supports -Wloop-analysis... no
checking whether the C++ compiler supports -Wc++1z-compat... yes
checking whether the C++ compiler supports -Wc++2a-compat... no
checking whether the C++ compiler supports -Wcomma... no
checking whether the C compiler supports -Wduplicated-cond... yes
checking whether the C++ compiler supports -Wduplicated-cond... yes
checking whether the C++ compiler supports -Wimplicit-fallthrough... yes
checking whether the C compiler supports -Wstring-conversion... no
checking whether the C++ compiler supports -Wstring-conversion... no
checking whether the C compiler supports -Wtautological-overlap-compare... no
checking whether the C++ compiler supports -Wtautological-overlap-compare... no
checking whether the C compiler supports -Wtautological-unsigned-enum-zero-compare... no
checking whether the C++ compiler supports -Wtautological-unsigned-enum-zero-compare... no
checking whether the C compiler supports -Wtautological-unsigned-zero-compare... no
checking whether the C++ compiler supports -Wtautological-unsigned-zero-compare... no
checking whether the C compiler supports -Wno-error=tautological-type-limit-compare... no
checking whether the C++ compiler supports -Wno-error=tautological-type-limit-compare... no
checking whether the C++ compiler supports -Wno-inline-new-delete... no
checking whether the C compiler supports -Wno-error=maybe-uninitialized... yes
checking whether the C++ compiler supports -Wno-error=maybe-uninitialized... yes
checking whether the C compiler supports -Wno-error=deprecated-declarations... yes
checking whether the C++ compiler supports -Wno-error=deprecated-declarations... yes
checking whether the C compiler supports -Wno-error=array-bounds... yes
checking whether the C++ compiler supports -Wno-error=array-bounds... yes
checking whether the C compiler supports -Wno-error=coverage-mismatch... yes
checking whether the C++ compiler supports -Wno-error=coverage-mismatch... yes
checking whether the C compiler supports -Wno-error=backend-plugin... no
checking whether the C++ compiler supports -Wno-error=backend-plugin... no
checking whether the C compiler supports -Wno-error=free-nonheap-object... yes
checking whether the C++ compiler supports -Wno-error=free-nonheap-object... yes
checking whether the C compiler supports -Wno-error=multistatement-macros... no
checking whether the C++ compiler supports -Wno-error=multistatement-macros... no
checking whether the C compiler supports -Wno-error=return-std-move... no
checking whether the C++ compiler supports -Wno-error=return-std-move... no
checking whether the C compiler supports -Wno-error=class-memaccess... no
checking whether the C++ compiler supports -Wno-error=class-memaccess... no
checking whether the C compiler supports -Wno-error=atomic-alignment... no
checking whether the C++ compiler supports -Wno-error=atomic-alignment... no
checking whether the C compiler supports -Wno-error=deprecated-copy... no
checking whether the C++ compiler supports -Wno-error=deprecated-copy... no
checking whether the C compiler supports -Wformat... yes
checking whether the C++ compiler supports -Wformat... yes
checking whether the C compiler supports -Wformat-security... yes
checking whether the C++ compiler supports -Wformat-security... yes
checking whether the C compiler supports -Wformat-overflow=2... yes
checking whether the C++ compiler supports -Wformat-overflow=2... yes
checking whether the C compiler supports -Wno-gnu-zero-variadic-macro-arguments... no
checking whether the C++ compiler supports -Wno-gnu-zero-variadic-macro-arguments... no
checking whether the C++ compiler supports -fno-sized-deallocation... yes
checking whether the C++ compiler supports -fno-aligned-new... yes
checking for libpulse... yes
checking MOZ_PULSEAUDIO_CFLAGS... -D_REENTRANT
checking MOZ_PULSEAUDIO_LIBS... -lpulse
checking for rustc... /usr/bin/rustc
checking for cargo... /usr/bin/cargo
checking rustc version... 1.39.0
checking cargo version... 1.39.0
checking for rust target triplet... x86_64-unknown-linux-gnu
checking for rust host triplet... x86_64-unknown-linux-gnu
checking for rustdoc... /usr/bin/rustdoc
checking for cbindgen... no
ERROR: Cannot find cbindgen. Please run `mach bootstrap`,
`cargo install cbindgen`, ensure that `cbindgen` is on your PATH,
or point at an executable with `CBINDGEN`.
```

### Solution

```bash
cargo install cbindgen
```

## ERROR: Could not find clang to generate run bindings for C/C++

```bash
nonetype@box:~/hack/browser/firefox/build-72.0-debug$ ../firefox-72.0/configure --enable-debug
Reexecuting in the virtualenv
checking for vcs source checkout... no
checking for a shell... /bin/sh
checking for host system type... x86_64-pc-linux-gnu
checking for target system type... x86_64-pc-linux-gnu
checking whether cross compiling... no
checking for Python 3... /usr/bin/python3 (3.6.9)
checking for yasm... not found
checking for the target C compiler... /usr/bin/gcc
checking whether the target C compiler can be used... yes
checking the target C compiler version... 7.5.0
checking the target C compiler works... yes
checking for the target C++ compiler... /usr/bin/g++
checking whether the target C++ compiler can be used... yes
checking the target C++ compiler version... 7.5.0
checking the target C++ compiler works... yes
checking for the host C compiler... /usr/bin/gcc
checking whether the host C compiler can be used... yes
checking the host C compiler version... 7.5.0
checking the host C compiler works... yes
checking for the host C++ compiler... /usr/bin/g++
checking whether the host C++ compiler can be used... yes
checking the host C++ compiler version... 7.5.0
checking the host C++ compiler works... yes
checking for 64-bit OS... yes
checking for nasm... not found
checking for linker... bfd
checking for the assembler... /usr/bin/gcc
checking for ar... /usr/bin/ar
checking for pkg_config... /usr/bin/pkg-config
checking for pkg-config version... 0.29.1
checking for stdint.h... yes
checking for inttypes.h... yes
checking for malloc.h... yes
checking for alloca.h... yes
checking for sys/byteorder.h... no
checking for getopt.h... yes
checking for unistd.h... yes
checking for nl_types.h... yes
checking for cpuid.h... yes
checking for sys/statvfs.h... yes
checking for sys/statfs.h... yes
checking for sys/vfs.h... yes
checking for sys/mount.h... yes
checking for sys/quota.h... yes
checking for linux/quota.h... yes
checking for linux/if_addr.h... yes
checking for linux/rtnetlink.h... yes
checking for sys/queue.h... yes
checking for sys/types.h... yes
checking for netinet/in.h... yes
checking for byteswap.h... yes
checking for linux/perf_event.h... yes
checking for perf_event_open system call... yes
checking whether the C compiler supports -Wbitfield-enum-conversion... no
checking whether the C++ compiler supports -Wbitfield-enum-conversion... no
checking whether the C compiler supports -Wshadow-field-in-constructor-modified... no
+ compiler supports -Wclass-varargs... no
checking whether the C compiler supports -Wfloat-overflow-conversion... no
checking whether the C++ compiler supports -Wfloat-overflow-conversion... no
checking whether the C compiler supports -Wfloat-zero-conversion... no
checking whether the C++ compiler supports -Wfloat-zero-conversion... no
checking whether the C compiler supports -Wloop-analysis... no
checking whether the C++ compiler supports -Wloop-analysis... no
checking whether the C++ compiler supports -Wc++1z-compat... yes
checking whether the C++ compiler supports -Wc++2a-compat... no
checking whether the C++ compiler supports -Wcomma... no
checking whether the C compiler supports -Wduplicated-cond... yes
checking whether the C++ compiler supports -Wduplicated-cond... yes
checking whether the C++ compiler supports -Wimplicit-fallthrough... yes
checking whether the C compiler supports -Wstring-conversion... no
checking whether the C++ compiler supports -Wstring-conversion... no
checking whether the C compiler supports -Wtautological-overlap-compare... no
checking whether the C++ compiler supports -Wtautological-overlap-compare... no
checking whether the C compiler supports -Wtautological-unsigned-enum-zero-compare... no
checking whether the C++ compiler supports -Wtautological-unsigned-enum-zero-compare... no
checking whether the C compiler supports -Wtautological-unsigned-zero-compare... no
checking whether the C++ compiler supports -Wtautological-unsigned-zero-compare... no
checking whether the C compiler supports -Wno-error=tautological-type-limit-compare... no
checking whether the C++ compiler supports -Wno-error=tautological-type-limit-compare... no
checking whether the C++ compiler supports -Wno-inline-new-delete... no
checking whether the C compiler supports -Wno-error=maybe-uninitialized... yes
checking whether the C++ compiler supports -Wno-error=maybe-uninitialized... yes
checking whether the C compiler supports -Wno-error=deprecated-declarations... yes
checking whether the C++ compiler supports -Wno-error=deprecated-declarations... yes
checking whether the C compiler supports -Wno-error=array-bounds... yes
checking whether the C++ compiler supports -Wno-error=array-bounds... yes
checking whether the C compiler supports -Wno-error=coverage-mismatch... yes
checking whether the C++ compiler supports -Wno-error=coverage-mismatch... yes
checking whether the C compiler supports -Wno-error=backend-plugin... no
checking whether the C++ compiler supports -Wno-error=backend-plugin... no
checking whether the C compiler supports -Wno-error=free-nonheap-object... yes
checking whether the C++ compiler supports -Wno-error=free-nonheap-object... yes
checking whether the C compiler supports -Wno-error=multistatement-macros... no
checking whether the C++ compiler supports -Wno-error=multistatement-macros... no
checking whether the C compiler supports -Wno-error=return-std-move... no
checking whether the C++ compiler supports -Wno-error=return-std-move... no
checking whether the C compiler supports -Wno-error=class-memaccess... no
checking whether the C++ compiler supports -Wno-error=class-memaccess... no
checking whether the C compiler supports -Wno-error=atomic-alignment... no
checking whether the C++ compiler supports -Wno-error=atomic-alignment... no
checking whether the C compiler supports -Wno-error=deprecated-copy... no
checking whether the C++ compiler supports -Wno-error=deprecated-copy... no
checking whether the C compiler supports -Wformat... yes
checking whether the C++ compiler supports -Wformat... yes
checking whether the C compiler supports -Wformat-security... yes
checking whether the C++ compiler supports -Wformat-security... yes
checking whether the C compiler supports -Wformat-overflow=2... yes
checking whether the C++ compiler supports -Wformat-overflow=2... yes
checking whether the C compiler supports -Wno-gnu-zero-variadic-macro-arguments... no
checking whether the C++ compiler supports -Wno-gnu-zero-variadic-macro-arguments... no
checking whether the C++ compiler supports -fno-sized-deallocation... yes
checking whether the C++ compiler supports -fno-aligned-new... yes
checking for libpulse... yes
checking MOZ_PULSEAUDIO_CFLAGS... -D_REENTRANT
checking MOZ_PULSEAUDIO_LIBS... -lpulse
checking for rustc... /usr/bin/rustc
checking for cargo... /usr/bin/cargo
checking rustc version... 1.39.0
checking cargo version... 1.39.0
checking for rust target triplet... x86_64-unknown-linux-gnu
checking for rust host triplet... x86_64-unknown-linux-gnu
checking for rustdoc... /usr/bin/rustdoc
checking for cbindgen... /home/nonetype/.cargo/bin/cbindgen
checking for rustfmt... not found
checking for clang for bindgen... not found
checking for libclang for bindgen... not found
ERROR: Could not find clang to generate run bindings for C/C++. Please install the necessary packages, run `mach bootstrap`, or use --with-clang-path to give the location of clang.
```

### Solution

```bash
sudo apt install clang
```

## ERROR: could not find Node.js executable later than 8.11

```bash
nonetype@box:~/hack/browser/firefox/build-72.0$ ../firefox-72.0/configure --enable-debug
Reexecuting in the virtualenv
checking for vcs source checkout... no
checking for a shell... /bin/sh
checking for host system type... x86_64-pc-linux-gnu
checking for target system type... x86_64-pc-linux-gnu
checking whether cross compiling... no
checking for Python 3... /usr/bin/python3 (3.6.9)
checking for yasm... not found
checking for the target C compiler... /usr/bin/clang
checking whether the target C compiler can be used... yes
checking the target C compiler version... 6.0.0
checking the target C compiler works... yes
checking for the target C++ compiler... /usr/bin/clang++
checking whether the target C++ compiler can be used... yes
checking the target C++ compiler version... 6.0.0
checking the target C++ compiler works... yes
checking for the host C compiler... /usr/bin/clang
checking whether the host C compiler can be used... yes
checking the host C compiler version... 6.0.0
checking the host C compiler works... yes
checking for the host C++ compiler... /usr/bin/clang++
checking whether the host C++ compiler can be used... yes
checking the host C++ compiler version... 6.0.0
checking the host C++ compiler works... yes
checking for 64-bit OS... yes
checking for nasm... not found
checking for linker... bfd
checking for the assembler... /usr/bin/clang
checking for ar... /usr/bin/ar
checking for pkg_config... /usr/bin/pkg-config
checking for pkg-config version... 0.29.1
checking for stdint.h... yes
checking for inttypes.h... yes
checking for malloc.h... yes
checking for alloca.h... yes
checking for sys/byteorder.h... no
checking for getopt.h... yes
checking for unistd.h... yes
checking for nl_types.h... yes
checking for cpuid.h... yes
checking for sys/statvfs.h... yes
checking for sys/statfs.h... yes
checking for sys/vfs.h... yes
checking for sys/mount.h... yes
checking for sys/quota.h... yes
checking for linux/quota.h... yes
checking for linux/if_addr.h... yes
checking for linux/rtnetlink.h... yes
checking for sys/queue.h... yes
checking for sys/types.h... yes
checking for netinet/in.h... yes
checking for byteswap.h... yes
checking for linux/perf_event.h... yes
checking for perf_event_open system call... yes
checking whether the C compiler supports -Wbitfield-enum-conversion... yes
checking whether the C++ compiler supports -Wbitfield-enum-conversion... yes
checking whether the C compiler supports -Wshadow-field-in-constructor-modified... yes
checking whether the C++ compiler supports -Wshadow-field-in-constructor-modified... yes
checking whether the C compiler supports -Wunreachable-code-return... yes
checking whether the C++ compiler supports -Wunreachable-code-return... yes
checking whether the C compiler supports -Wclass-varargs... yes
checking whether the C++ compiler supports -Wclass-varargs... yes
checking whether the C compiler supports -Wfloat-overflow-conversion... yes
checking whether the C++ compiler supports -Wfloat-overflow-conversion... yes
checking whether the C compiler supports -Wfloat-zero-conversion... yes
checking whether the C++ compiler supports -Wfloat-zero-conversion... yes
checking whether the C compiler supports -Wloop-analysis... yes
checking whether the C++ compiler supports -Wloop-analysis... yes
checking whether the C++ compiler supports -Wc++1z-compat... yes
checking whether the C++ compiler supports -Wc++2a-compat... yes
checking whether the C++ compiler supports -Wcomma... yes
checking whether the C compiler supports -Wduplicated-cond... no
checking whether the C++ compiler supports -Wduplicated-cond... no
checking whether the C++ compiler supports -Wimplicit-fallthrough... yes
checking whether the C compiler supports -Wstring-conversion... yes
checking whether the C++ compiler supports -Wstring-conversion... yes
checking whether the C compiler supports -Wtautological-overlap-compare... yes
checking whether the C++ compiler supports -Wtautological-overlap-compare... yes
checking whether the C compiler supports -Wtautological-unsigned-enum-zero-compare... yes
checking whether the C++ compiler supports -Wtautological-unsigned-enum-zero-compare... yes
checking whether the C compiler supports -Wtautological-unsigned-zero-compare... yes
checking whether the C++ compiler supports -Wtautological-unsigned-zero-compare... yes
checking whether the C compiler supports -Wno-error=tautological-type-limit-compare... yes
checking whether the C++ compiler supports -Wno-error=tautological-type-limit-compare... yes
checking whether the C++ compiler supports -Wno-inline-new-delete... yes
checking whether the C compiler supports -Wno-error=maybe-uninitialized... no
checking whether the C++ compiler supports -Wno-error=maybe-uninitialized... no
checking whether the C compiler supports -Wno-error=deprecated-declarations... yes
checking whether the C++ compiler supports -Wno-error=deprecated-declarations... yes
checking whether the C compiler supports -Wno-error=array-bounds... yes
checking whether the C++ compiler supports -Wno-error=array-bounds... yes
checking whether the C compiler supports -Wno-error=coverage-mismatch... no
checking whether the C++ compiler supports -Wno-error=coverage-mismatch... no
checking whether the C compiler supports -Wno-error=backend-plugin... yes
checking whether the C++ compiler supports -Wno-error=backend-plugin... yes
checking whether the C compiler supports -Wno-error=free-nonheap-object... no
checking whether the C++ compiler supports -Wno-error=free-nonheap-object... no
checking whether the C compiler supports -Wno-error=multistatement-macros... no
checking whether the C++ compiler supports -Wno-error=multistatement-macros... no
checking whether the C compiler supports -Wno-error=return-std-move... no
checking whether the C++ compiler supports -Wno-error=return-std-move... no
checking whether the C compiler supports -Wno-error=class-memaccess... no
checking whether the C++ compiler supports -Wno-error=class-memaccess... no
checking whether the C compiler supports -Wno-error=atomic-alignment... no
checking whether the C++ compiler supports -Wno-error=atomic-alignment... no
checking whether the C compiler supports -Wno-error=deprecated-copy... no
checking whether the C++ compiler supports -Wno-error=deprecated-copy... no
checking whether the C compiler supports -Wformat... yes
checking whether the C++ compiler supports -Wformat... yes
checking whether the C compiler supports -Wformat-security... yes
checking whether the C++ compiler supports -Wformat-security... yes
checking whether the C compiler supports -Wformat-overflow=2... no
checking whether the C++ compiler supports -Wformat-overflow=2... no
checking whether the C compiler supports -Wno-gnu-zero-variadic-macro-arguments... yes
checking whether the C++ compiler supports -Wno-gnu-zero-variadic-macro-arguments... yes
checking whether the C++ compiler supports -fno-sized-deallocation... yes
checking whether the C++ compiler supports -fno-aligned-new... yes
checking for libpulse... yes
checking MOZ_PULSEAUDIO_CFLAGS... -D_REENTRANT
checking MOZ_PULSEAUDIO_LIBS... -lpulse
checking for rustc... /usr/bin/rustc
checking for cargo... /usr/bin/cargo
checking rustc version... 1.39.0
checking cargo version... 1.39.0
checking for rust target triplet... x86_64-unknown-linux-gnu
checking for rust host triplet... x86_64-unknown-linux-gnu
checking for rustdoc... /usr/bin/rustdoc
checking for cbindgen... /home/nonetype/.cargo/bin/cbindgen
checking for rustfmt... not found
checking for clang for bindgen... /usr/bin/clang++
checking for libclang for bindgen... /usr/lib/llvm-6.0/lib/libclang.so.1
checking that libclang is new enough... yes
checking bindgen cflags... -x c++ -fno-sized-deallocation -fno-aligned-new -DTRACING=1 -DIMPL_LIBXUL -DMOZILLA_INTERNAL_API -DRUST_BINDGEN -DOS_POSIX=1 -DOS_LINUX=1
checking for llvm_profdata... /usr/bin/llvm-profdata
checking for nodejs... no
ERROR: could not find Node.js executable later than 8.11; ensure `node` or `nodejs` is in PATH or set NODEJS in environment to point to an executable.

    Executing `mach bootstrap --no-system-changes` should
    install a compatible version in ~/.mozbuild on most platforms.
    If you believe this is a bug, <https://mzl.la/2vLbXAv> is a good way
    to file.  More details: <https://bit.ly/2BbyD1E>
```

### Solution

`nodejs` 버전이 낮아서 생기는 문제다.

Ubuntu18.04 기준, `apt install nodejs` 명령을 통해 받는 버전은 8.10이므로 아래 명령을 통해 최신 버전의 nodejs를 설치하면 된다. (작성일 기준 최신 nodejs 버전: 12.16.1 LTS)

```bash
curl -sL https://deb.nodesource.com/setup_12.x | sudo -E bash -
sudo apt-get install -y nodejs
```

아래와 같이 버전을 확인할 수 있다.

```bash
nonetype@box:~/hack/browser/firefox$ nodejs -v
v12.16.1
nonetype@box:~/hack/browser/firefox$
```

## ERROR: nasm 2.13 or greater is required for AV1 support

```bash
nonetype@box:~/hack/browser/firefox/build-72.0-debug$ ../firefox-72.0/configure --enable-debug
Reexecuting in the virtualenv
checking for vcs source checkout... no
checking for a shell... /bin/sh
checking for host system type... x86_64-pc-linux-gnu
checking for target system type... x86_64-pc-linux-gnu
checking whether cross compiling... no
checking for Python 3... /usr/bin/python3 (3.6.9)
checking for yasm... not found
checking for the target C compiler... /usr/bin/clang
checking whether the target C compiler can be used... yes
checking the target C compiler version... 6.0.0checking the target C compiler works... yeschecking for the target C++ compiler... /usr/bin/clang++
checking whether the target C++ compiler can be used... yes
checking the target C++ compiler version... 6.0.0
checking the target C++ compiler works... yeschecking for the host C compiler... /usr/bin/clang
checking whether the host C compiler can be used... yes
checking the host C compiler version... 6.0.0
checking the host C compiler works... yes
checking for the host C++ compiler... /usr/bin/clang++
checking whether the host C++ compiler can be used... yes
checking the host C++ compiler version... 6.0.0
checking the host C++ compiler works... yes
checking for 64-bit OS... yes
checking for nasm... not found
checking for linker... bfd
checking for the assembler... /usr/bin/clang
checking for ar... /usr/bin/ar
checking for pkg_config... /usr/bin/pkg-config
checking for pkg-config version... 0.29.1
checking for stdint.h... yes
checking for inttypes.h... yes
checking for malloc.h... yes
checking for alloca.h... yes
checking for sys/byteorder.h... no
checking for getopt.h... yes
checking for unistd.h... yes
checking for nl_types.h... yes
checking for cpuid.h... yes
checking for sys/statvfs.h... yes
checking for sys/statfs.h... yes
checking for sys/vfs.h... yes
checking for sys/mount.h... yes
checking for sys/quota.h... yes
checking for linux/quota.h... yes
checking for linux/if_addr.h... yes
checking for linux/rtnetlink.h... yes
checking for sys/queue.h... yes
checking for sys/types.h... yes
checking for netinet/in.h... yes
checking for byteswap.h... yes
checking for linux/perf_event.h... yes
checking for perf_event_open system call... yes
checking whether the C compiler supports -Wbitfield-enum-conversion... yes
checking whether the C++ compiler supports -Wbitfield-enum-conversion... yes
checking whether the C compiler supports -Wshadow-field-in-constructor-modified... yes
checking whether the C++ compiler supports -Wshadow-field-in-constructor-modified... yes
checking whether the C compiler supports -Wunreachable-code-return... yes
checking whether the C++ compiler supports -Wunreachable-code-return... yes
checking whether the C compiler supports -Wclass-varargs... yes
checking whether the C++ compiler supports -Wclass-varargs... yes
checking whether the C compiler supports -Wfloat-overflow-conversion... yes
checking whether the C++ compiler supports -Wfloat-overflow-conversion... yes
checking whether the C compiler supports -Wfloat-zero-conversion... yes
checking whether the C++ compiler supports -Wfloat-zero-conversion... yes
checking whether the C compiler supports -Wloop-analysis... yes
checking whether the C++ compiler supports -Wloop-analysis... yes
checking whether the C++ compiler supports -Wc++1z-compat... yes
checking whether the C++ compiler supports -Wc++2a-compat... yes
checking whether the C++ compiler supports -Wcomma... yes
checking whether the C compiler supports -Wduplicated-cond... no
checking whether the C++ compiler supports -Wduplicated-cond... no
checking whether the C++ compiler supports -Wimplicit-fallthrough... yes
checking whether the C compiler supports -Wstring-conversion... yes
checking whether the C++ compiler supports -Wstring-conversion... yes
checking whether the C compiler supports -Wtautological-overlap-compare... yes
checking whether the C++ compiler supports -Wtautological-overlap-compare... yes
checking whether the C compiler supports -Wtautological-unsigned-enum-zero-compare... yes
checking whether the C++ compiler supports -Wtautological-unsigned-enum-zero-compare... yes
checking whether the C compiler supports -Wtautological-unsigned-zero-compare... yes
checking whether the C++ compiler supports -Wtautological-unsigned-zero-compare... yes
checking whether the C compiler supports -Wno-error=tautological-type-limit-compare... yes
checking whether the C++ compiler supports -Wno-error=tautological-type-limit-compare... yes
checking whether the C++ compiler supports -Wno-inline-new-delete... yes
checking whether the C compiler supports -Wno-error=maybe-uninitialized... no
checking whether the C++ compiler supports -Wno-error=maybe-uninitialized... no
checking whether the C compiler supports -Wno-error=deprecated-declarations... yes
checking whether the C++ compiler supports -Wno-error=deprecated-declarations... yes
checking whether the C compiler supports -Wno-error=array-bounds... yes
checking whether the C++ compiler supports -Wno-error=array-bounds... yes
checking whether the C compiler supports -Wno-error=coverage-mismatch... no
checking whether the C++ compiler supports -Wno-error=coverage-mismatch... no
checking whether the C compiler supports -Wno-error=backend-plugin... yes
checking whether the C++ compiler supports -Wno-error=backend-plugin... yes
checking whether the C compiler supports -Wno-error=free-nonheap-object... no
checking whether the C++ compiler supports -Wno-error=free-nonheap-object... no
checking whether the C compiler supports -Wno-error=multistatement-macros... no
checking whether the C++ compiler supports -Wno-error=multistatement-macros... no
checking whether the C compiler supports -Wno-error=return-std-move... no
checking whether the C++ compiler supports -Wno-error=return-std-move... no
checking whether the C compiler supports -Wno-error=class-memaccess... no
checking whether the C++ compiler supports -Wno-error=class-memaccess... no
checking whether the C compiler supports -Wno-error=atomic-alignment... no
checking whether the C++ compiler supports -Wno-error=atomic-alignment... no
checking whether the C compiler supports -Wno-error=deprecated-copy... no
checking whether the C++ compiler supports -Wno-error=deprecated-copy... no
checking whether the C compiler supports -Wformat... yes
checking whether the C++ compiler supports -Wformat... yes
checking whether the C compiler supports -Wformat-security... yes
checking whether the C++ compiler supports -Wformat-security... yes
checking whether the C compiler supports -Wformat-overflow=2... no
checking whether the C++ compiler supports -Wformat-overflow=2... no
checking whether the C compiler supports -Wno-gnu-zero-variadic-macro-arguments... yes
checking whether the C++ compiler supports -Wno-gnu-zero-variadic-macro-arguments... yes
checking whether the C++ compiler supports -fno-sized-deallocation... yes
checking whether the C++ compiler supports -fno-aligned-new... yes
checking for libpulse... yes
checking MOZ_PULSEAUDIO_CFLAGS... -D_REENTRANT
checking MOZ_PULSEAUDIO_LIBS... -lpulse
checking for rustc... /usr/bin/rustc
checking for cargo... /usr/bin/cargo
checking rustc version... 1.39.0
checking cargo version... 1.39.0
checking for rust target triplet... x86_64-unknown-linux-gnu
checking for rust host triplet... x86_64-unknown-linux-gnu
checking for rustdoc... /usr/bin/rustdoc
checking for cbindgen... /home/nonetype/.cargo/bin/cbindgen
checking for rustfmt... not found
checking for clang for bindgen... /usr/bin/clang++
checking for libclang for bindgen... /usr/lib/llvm-6.0/lib/libclang.so.1
checking that libclang is new enough... yes
checking bindgen cflags... -x c++ -fno-sized-deallocation -fno-aligned-new -DTRACING=1 -DIMPL_LIBXUL -DMOZILLA_INTERNAL_API -DRUST_BINDGEN -DOS_POSIX=1 -DOS_LINUX=1
checking for llvm_profdata... /usr/bin/llvm-profdata
checking for nodejs... /usr/bin/nodejs (12.16.1)
checking for gtk+-wayland-3.0 >= 3.10 xkbcommon >= 0.4.1 libdrm >= 2.4... yes
checking MOZ_WAYLAND_CFLAGS... -pthread -I/usr/include/gtk-3.0 -I/usr/include/at-spi2-atk/2.0 -I/usr/include/at-spi-2.0 -I/usr/include/dbus-1.0 -I/usr/lib/x86_64-linux-gnu/dbus-1.0/include -I/usr/include/gtk-3.0 -I/usr/include/gio-unix-2.
0/ -I/usr/include/cairo -I/usr/include/pango-1.0 -I/usr/include/harfbuzz -I/usr/include/pango-1.0 -I/usr/include/atk-1.0 -I/usr/include/cairo -I/usr/include/pixman-1 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/freetype
2 -I/usr/include/libpng16 -I/usr/include/gdk-pixbuf-2.0 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/libdrm
checking MOZ_WAYLAND_LIBS... -lgtk-3 -lgdk-3 -lpangocairo-1.0 -lpango-1.0 -latk-1.0 -lcairo-gobject -lcairo -lgdk_pixbuf-2.0 -lgio-2.0 -lgobject-2.0 -lglib-2.0 -lxkbcommon -ldrm
checking for pango >= 1.22.0 pangoft2 >= 1.22.0 pangocairo >= 1.22.0... yes
checking MOZ_PANGO_CFLAGS... -I/usr/include/pango-1.0 -I/usr/include/harfbuzz -I/usr/include/pango-1.0 -I/usr/include/cairo -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/pixman-1 -I/usr/include/freety
pe2 -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16
checking MOZ_PANGO_LIBS... -lpangoft2-1.0 -lfontconfig -lfreetype -lpangocairo-1.0 -lpango-1.0 -lgobject-2.0 -lglib-2.0 -lcairo
checking for fontconfig >= 2.7.0... yes
checking _FONTCONFIG_CFLAGS... -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16
checking _FONTCONFIG_LIBS... -lfontconfig -lfreetype
checking for freetype2 >= 6.1.0... yes
checking _FT2_CFLAGS... -I/usr/include/freetype2 -I/usr/include/libpng16
checking _FT2_LIBS... -lfreetype
ERROR: nasm 2.13 or greater is required for AV1 support. Either install nasm or add --disable-av1 to your configure options.
```

### Solution
내 경우에는 `nasm`이 설치되어 있지 않아 발생하는 문제였다.

```bash
nonetype@box:~/hack/browser/firefox/build-72.0-debug$ nasm -v

Command 'nasm' not found, but can be installed with:

sudo apt install nasm
```

아래 명령을 통해 `nasm`을 설치하면 해결된다.

```bash
sudo apt install nasm
```

## ERROR: Yasm is required to build with ffvpx, jpeg, libav and vpx

```bash
nonetype@box:~/hack/browser/firefox/build-72.0-debug$ ../firefox-72.0/configure --enable-debug
Reexecuting in the virtualenv
checking for vcs source checkout... no
checking for a shell... /bin/sh
checking for host system type... x86_64-pc-linux-gnu
checking for target system type... x86_64-pc-linux-gnu
checking whether cross compiling... no
checking for Python 3... /usr/bin/python3 (3.6.9)
checking for yasm... not found
checking for the target C compiler... /usr/bin/clang
checking whether the target C compiler can be used... yes
checking the target C compiler version... 6.0.0
checking the target C compiler works... yes
checking for the target C++ compiler... /usr/bin/clang++
checking whether the target C++ compiler can be used... yes
checking the target C++ compiler version... 6.0.0
checking the target C++ compiler works... yes
checking for the host C compiler... /usr/bin/clang
checking whether the host C compiler can be used... yes
checking the host C compiler version... 6.0.0
checking the host C compiler works... yes
checking for the host C++ compiler... /usr/bin/clang++
checking whether the host C++ compiler can be used... yes
checking the host C++ compiler version... 6.0.0
checking the host C++ compiler works... yes
checking for 64-bit OS... yes
checking for nasm... /usr/bin/nasm
checking nasm version... 2.13.02
checking for linker... bfd
checking for the assembler... /usr/bin/clang
checking for ar... /usr/bin/ar
checking for pkg_config... /usr/bin/pkg-config
checking for pkg-config version... 0.29.1
checking for stdint.h... yes
checking for inttypes.h... yes
checking for malloc.h... yes
checking for alloca.h... yes
checking for sys/byteorder.h... no
checking for getopt.h... yes
checking for unistd.h... yes
checking for nl_types.h... yes
checking for cpuid.h... yes
checking for sys/statvfs.h... yes
checking for sys/statfs.h... yes
checking for sys/vfs.h... yes
checking for sys/mount.h... yes
checking for sys/quota.h... yes
checking for linux/quota.h... yes
checking for linux/if_addr.h... yes
checking for linux/rtnetlink.h... yes
checking for sys/queue.h... yes
checking for sys/types.h... yes
checking for netinet/in.h... yes
checking for byteswap.h... yes
checking for linux/perf_event.h... yes
checking for perf_event_open system call... yes
checking whether the C compiler supports -Wbitfield-enum-conversion... yes
checking whether the C++ compiler supports -Wbitfield-enum-conversion... yes
checking whether the C compiler supports -Wshadow-field-in-constructor-modified... yes
checking whether the C++ compiler supports -Wshadow-field-in-constructor-modified... yes
checking whether the C compiler supports -Wunreachable-code-return... yes
checking whether the C++ compiler supports -Wunreachable-code-return... yes
checking whether the C compiler supports -Wclass-varargs... yes
checking whether the C++ compiler supports -Wclass-varargs... yes
checking whether the C compiler supports -Wfloat-overflow-conversion... yes
checking whether the C++ compiler supports -Wfloat-overflow-conversion... yes
checking whether the C compiler supports -Wfloat-zero-conversion... yes
checking whether the C++ compiler supports -Wfloat-zero-conversion... yes
checking whether the C compiler supports -Wloop-analysis... yes
checking whether the C++ compiler supports -Wloop-analysis... yes
checking whether the C++ compiler supports -Wc++1z-compat... yes
checking whether the C++ compiler supports -Wc++2a-compat... yes
checking whether the C++ compiler supports -Wcomma... yes
checking whether the C compiler supports -Wduplicated-cond... no
checking whether the C++ compiler supports -Wduplicated-cond... no
checking whether the C++ compiler supports -Wimplicit-fallthrough... yes
checking whether the C compiler supports -Wstring-conversion... yes
checking whether the C++ compiler supports -Wstring-conversion... yes
checking whether the C compiler supports -Wtautological-overlap-compare... yes
checking whether the C++ compiler supports -Wtautological-overlap-compare... yes
checking whether the C compiler supports -Wtautological-unsigned-enum-zero-compare... yes
checking whether the C++ compiler supports -Wtautological-unsigned-enum-zero-compare... yes
checking whether the C compiler supports -Wtautological-unsigned-zero-compare... yes
checking whether the C++ compiler supports -Wtautological-unsigned-zero-compare... yes
checking whether the C compiler supports -Wno-error=tautological-type-limit-compare... yes
checking whether the C++ compiler supports -Wno-error=tautological-type-limit-compare... yes
checking whether the C++ compiler supports -Wno-inline-new-delete... yes
checking whether the C compiler supports -Wno-error=maybe-uninitialized... no
checking whether the C++ compiler supports -Wno-error=maybe-uninitialized... no
checking whether the C compiler supports -Wno-error=deprecated-declarations... yes
checking whether the C++ compiler supports -Wno-error=deprecated-declarations... yes
checking whether the C compiler supports -Wno-error=array-bounds... yes
checking whether the C++ compiler supports -Wno-error=array-bounds... yes
checking whether the C compiler supports -Wno-error=coverage-mismatch... no
checking whether the C++ compiler supports -Wno-error=coverage-mismatch... no
checking whether the C compiler supports -Wno-error=backend-plugin... yes
checking whether the C++ compiler supports -Wno-error=backend-plugin... yes
checking whether the C compiler supports -Wno-error=free-nonheap-object... no
checking whether the C++ compiler supports -Wno-error=free-nonheap-object... no
checking whether the C compiler supports -Wno-error=multistatement-macros... no
checking whether the C++ compiler supports -Wno-error=multistatement-macros... no
checking whether the C compiler supports -Wno-error=return-std-move... no
checking whether the C++ compiler supports -Wno-error=return-std-move... no
checking whether the C compiler supports -Wno-error=class-memaccess... no
checking whether the C++ compiler supports -Wno-error=class-memaccess... no
checking whether the C compiler supports -Wno-error=atomic-alignment... no
checking whether the C++ compiler supports -Wno-error=atomic-alignment... no
checking whether the C compiler supports -Wno-error=deprecated-copy... no
checking whether the C++ compiler supports -Wno-error=deprecated-copy... no
checking whether the C compiler supports -Wformat... yes
checking whether the C++ compiler supports -Wformat... yes
checking whether the C compiler supports -Wformat-security... yes
checking whether the C++ compiler supports -Wformat-security... yes
checking whether the C compiler supports -Wformat-overflow=2... no
checking whether the C++ compiler supports -Wformat-overflow=2... no
checking whether the C compiler supports -Wno-gnu-zero-variadic-macro-arguments... yes
checking whether the C++ compiler supports -Wno-gnu-zero-variadic-macro-arguments... yes
checking whether the C++ compiler supports -fno-sized-deallocation... yes
checking whether the C++ compiler supports -fno-aligned-new... yes
checking for libpulse... yes
checking MOZ_PULSEAUDIO_CFLAGS... -D_REENTRANT
checking MOZ_PULSEAUDIO_LIBS... -lpulse
checking for rustc... /usr/bin/rustc
checking for cargo... /usr/bin/cargo
checking rustc version... 1.39.0
checking cargo version... 1.39.0
checking for rust target triplet... x86_64-unknown-linux-gnu
checking for rust host triplet... x86_64-unknown-linux-gnu
checking for rustdoc... /usr/bin/rustdoc
checking for cbindgen... /home/nonetype/.cargo/bin/cbindgen
checking for rustfmt... not found
checking for clang for bindgen... /usr/bin/clang++
checking for libclang for bindgen... /usr/lib/llvm-6.0/lib/libclang.so.1
checking that libclang is new enough... yes
checking bindgen cflags... -x c++ -fno-sized-deallocation -fno-aligned-new -DTRACING=1 -DIMPL_LIBXUL -DMOZILLA_INTERNAL_API -DRUST_BINDGEN -DOS_POSIX=1 -DOS_LINUX=1
checking for llvm_profdata... /usr/bin/llvm-profdata
checking for nodejs... /usr/bin/nodejs (12.16.1)
checking for gtk+-wayland-3.0 >= 3.10 xkbcommon >= 0.4.1 libdrm >= 2.4... yes
checking MOZ_WAYLAND_CFLAGS... -pthread -I/usr/include/gtk-3.0 -I/usr/include/at-spi2-atk/2.0 -I/usr/include/at-spi-2.0 -I/usr/include/dbus-1.0 -I/usr/lib/x86_64-linux-gnu/dbus-1.0/include -I/usr/include/gtk-3.0 -I/usr/include/gio-unix-2.0/ -I/usr/include/cairo -I/usr/include/pango-1.0 -I/usr/include/harfbuzz -I/usr/include/pango-1.0 -I/usr/include/atk-1.0 -I/usr/include/cairo -I/usr/include/pixman-1 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/gdk-pixbuf-2.0 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/libdrm
checking MOZ_WAYLAND_LIBS... -lgtk-3 -lgdk-3 -lpangocairo-1.0 -lpango-1.0 -latk-1.0 -lcairo-gobject -lcairo -lgdk_pixbuf-2.0 -lgio-2.0 -lgobject-2.0 -lglib-2.0 -lxkbcommon -ldrm
checking for pango >= 1.22.0 pangoft2 >= 1.22.0 pangocairo >= 1.22.0... yes
checking MOZ_PANGO_CFLAGS... -I/usr/include/pango-1.0 -I/usr/include/harfbuzz -I/usr/include/pango-1.0 -I/usr/include/cairo -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/pixman-1 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16
checking MOZ_PANGO_LIBS... -lpangoft2-1.0 -lfontconfig -lfreetype -lpangocairo-1.0 -lpango-1.0 -lgobject-2.0 -lglib-2.0 -lcairo
checking for fontconfig >= 2.7.0... yes
checking _FONTCONFIG_CFLAGS... -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16
checking _FONTCONFIG_LIBS... -lfontconfig -lfreetype
checking for freetype2 >= 6.1.0... yes
checking _FT2_CFLAGS... -I/usr/include/freetype2 -I/usr/include/libpng16
checking _FT2_LIBS... -lfreetype
checking for tar... /bin/tar
checking for unzip... /usr/bin/unzip
checking for zip... /usr/bin/zip
checking for gn... not found
checking for the Mozilla API key... no
checking for the Google Location Service API key... no
checking for the Google Safebrowsing API key... no
checking for the Bing API key... no
checking for the Adjust SDK key... no
checking for the Leanplum SDK key... no
checking for the Pocket API key... no
ERROR: Yasm is required to build with ffvpx, jpeg, libav and vpx, but you do not appear to have Yasm installed.
```

### Solution

```bash
sudo apt install yasm
```

## configure: error: Library requirements

```bash
nonetype@box:~/hack/browser/firefox/build-72.0-debug$ ../firefox-72.0/configure --enable-debug
Reexecuting in the virtualenv
checking for vcs source checkout... no
checking for a shell... /bin/sh
...
...
checking MOZ_GTK3_CFLAGS... -pthread -I/usr/include/gtk-3.0/unix-print -I/usr/include/gtk-3.0 -I/usr/include/at-spi2-atk/2.0 -I/usr/include/at-spi-2.0 -I/usr/include/dbus-1.0 -I/usr/lib/x86_64-linux-gnu/dbus-1.0/include -I/usr/include/gtk-3.0 -I/usr/include/cairo -I/usr/include/pango-1.0 -I/usr/include/harfbuzz -I/usr/include/pango-1.0 -I/usr/include/atk-1.0 -I/usr/include/cairo -I/usr/include/pixman-1 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/gdk-pixbuf-2.0 -I/usr/include/libpng16 -I/usr/include/gio-unix-2.0/ -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include
checking MOZ_GTK3_LIBS... -lgtk-3 -lgdk-3 -lpangocairo-1.0 -lpango-1.0 -latk-1.0 -lcairo-gobject -lcairo -lgdk_pixbuf-2.0 -lgio-2.0 -lgobject-2.0 -lglib-2.0
checking for gtk+-2.0 >= 2.18.0 gtk+-unix-print-2.0 glib-2.0 >= 2.22 gobject-2.0 gio-unix-2.0 gdk-x11-2.0... Package gtk+-2.0 was not found in the pkg-config search path. Perhaps you should add the directory containing 'gtk+-2.0.pc' to the PKG_CONFIG_PATH environment variable No package 'gtk+-2.0' found Package gtk+-unix-print-2.0 was not found in the pkg-config search path. Perhaps you should add the directory containing 'gtk+-unix-print-2.0.pc' to the PKG_CONFIG_PATH environment variable No package 'gtk+-unix-print-2.0' found Package gdk-x11-2.0 was not found in the pkg-config search path. Perhaps you should add the directory containing 'gdk-x11-2.0.pc' to the PKG_CONFIG_PATH environment variable No package 'gdk-x11-2.0' found
configure: error: Library requirements (gtk+-2.0 >= 2.18.0 gtk+-unix-print-2.0 glib-2.0 >= 2.22 gobject-2.0 gio-unix-2.0 gdk-x11-2.0) not met; consider adjusting the PKG_CONFIG_PATH environment variable if your libraries are in a nonstandard prefix so pkg-config can find them.
DEBUG: <truncated - see config.log for full output>
DEBUG: 1 error generated.
DEBUG: configure: failed program was:
DEBUG: #line 7699 "configure"
DEBUG: #include "confdefs.h"
DEBUG: #include <malloc.h>
DEBUG:                   #include <stddef.h>
DEBUG:                   size_t malloc_usable_size(const void *ptr);
DEBUG: int main() {
DEBUG: return malloc_usable_size(0);
DEBUG: ; return 0; }
DEBUG: configure:7730: checking for valloc in malloc.h
DEBUG: configure:7755: checking for valloc in unistd.h
DEBUG: configure:7780: checking for _aligned_malloc in malloc.h
DEBUG: configure:7918: checking NSPR selection
DEBUG: configure:8849: checking if app-specific confvars.sh exists
DEBUG: configure:9030: checking for gtk+-3.0 >= 3.4.0 gtk+-unix-print-3.0 glib-2.0 gobject-2.0 gio-unix-2.0
DEBUG: configure:9037: checking MOZ_GTK3_CFLAGS
DEBUG: configure:9042: checking MOZ_GTK3_LIBS
DEBUG: configure:9113: checking for gtk+-2.0 >= 2.18.0 gtk+-unix-print-2.0 glib-2.0 >= 2.22 gobject-2.0 gio-unix-2.0 gdk-x11-2.0
DEBUG: configure: error: Library requirements (gtk+-2.0 >= 2.18.0 gtk+-unix-print-2.0 glib-2.0 >= 2.22 gobject-2.0 gio-unix-2.0 gdk-x11-2.0) not met; consider adjusting the PKG_CONFIG_PATH environment variable if your libraries are in a nonstandard prefix so pkg-config can find them.
ERROR: old-configure failed
```

### Solution

`sudo apt (update/upgrade)` 후에도 이 문제가 해결되지 않는 경우가 있는데, 아래 라이브러리를 설치하면 해결된다.

```bash
apt-get install libgtk2.0-dev
```

# References

| Title | url | tags |
|---|---|---|
| Javascript Engine(Spider Monkey) Array OOB Analyzing | [link](https://bpsecblog.wordpress.com/2017/04/27/javascript_engine_array_oob) | `pwn`, `JIT`, `ctf`, `OOB` |
| Attacking Client-Side JIT Compilers (v2) | [link](https://saelo.github.io/presentations/blackhat_us_18_attacking_client_side_jit_compilers.pdf) | `rev`, `pwn`, `JIT`, `pdf`, `depth` |
| Introduction to SpiderMonkey exploitation | [link](https://doar-e.github.io/blog/2018/11/19/introduction-to-spidermonkey-exploitation/) | `rev`, `pwn`, `basic`, `JIT`, `ctf`, `OOB` |
| GDB to LLDB command map | [link](https://lldb.llvm.org/use/map.html) | `tool`, `rev` |

[^type_masking]: 자세한 코드는 `js/public/Value.h`에서 확인 가능하다. (`constexpr uint64_t ValueGCThingPayloadMask = 0x0000'7FFF'FFFF'FFFF;`)