---
layout: post
title: "Janus Porting"
author: "nonetype"
---


JANUS LKL 최신 버전으로 업데이팅

#### Porting Logs

기본적으로 지효형이 올린 자료
```sh
sudo apt-get update
sudo apt-get upgrade -y
sudo apt install git-core
sudo apt install ssh
sudo apt install libboost-all-dev build-essential binutils libc6 libc6-dev libclang-common-3.5-dev libclang1-3.5 libedit2 libgcc-5-dev libgcc1 libllvm3.5v5 libobjc-5-dev libstdc++-5-dev libstdc++6 libtinfo5 llvm-3.5-dev bison flex g++ subversion cmake -y
sudo apt install llvm clang
sudop mkdir /swap/
sudo dd if=/dev/zero of=/swap/swapfile bs=1024 count=10485760
sudo mkswap swapfile
sudo swapon swapfile
swapon -s
sudo apt install lld
sudo apt install lldb
sudo apt install libarchive-dev
sudo apt install python-pip
pip install xattr
pip install pathlib2
apt-get install software-properties-common
add-apt-repository ppa:git-core/ppa
curl -s https://packagecloud.io/install/repositories/github/git-lfs/script.deb.sh | sudo bash
apt-get install git-lfs
git lfs pull
```


우선 git에 올라온 janus를 clone한 다음, lkl 디렉토리 내 파일들을 최신 lkl 내 파일들로 덮어썼다.
이후 `make -C tools/lkl` 명령시 `error: ‘dma_direct_ops’ undeclared (first use in this function)` 에러가 나타난다.

<details>
<summary>Full Log</summary>
<div markdown="1">

```sh
root@system:~/porting/janus-master/lkl# make -C tools/lkl
make: Entering directory '/home/fuzzer/porting/janus-master/lkl/tools/lkl'
scripts/kconfig/conf  --defconfig Kconfig
#
# No change to .config
#
scripts/Makefile.asm-generic:25: redundant generic-y found in arch/lkl/include/asm/Kbuild: compat.h dma-mapping.h page.h
  CALL    scripts/checksyscalls.sh
  CALL    scripts/atomic/check-atomics.sh
  CHK     include/generated/compile.h
  CC      init/do_mounts.o
In file included from ./include/linux/dma-mapping.h:266:0,
                 from ./include/linux/skbuff.h:30,
                 from ./include/net/net_namespace.h:37,
                 from ./include/linux/inet.h:42,
                 from ./include/linux/sunrpc/msg_prot.h:204,
                 from ./include/linux/sunrpc/auth.h:16,
                 from ./include/linux/nfs_fs.h:31,
                 from init/do_mounts.c:23:
./arch/lkl/include/asm/dma-mapping.h: In function ‘get_arch_dma_ops’:
./arch/lkl/include/asm/dma-mapping.h:7:11: error: ‘dma_direct_ops’ undeclared (first use in this function)
   return &dma_direct_ops;
           ^
./arch/lkl/include/asm/dma-mapping.h:7:11: note: each undeclared identifier is reported only once for each function it appears in
scripts/Makefile.build:278: recipe for target 'init/do_mounts.o' failed
make[2]: *** [init/do_mounts.o] Error 1
Makefile:1072: recipe for target 'init' failed
make[1]: *** [init] Error 2
Makefile:65: recipe for target '/home/fuzzer/porting/janus-master/lkl/tools/lkl/lib/lkl.o' failed
make: *** [/home/fuzzer/porting/janus-master/lkl/tools/lkl/lib/lkl.o] Error 2
make: Leaving directory '/home/fuzzer/porting/janus-master/lkl/tools/lkl'
root@system:~/porting/janus-master/lkl#
```

</div>
</details>

이후 `include/linux/dma-mapping.h` 파일 138번 라인에 `extern const struct dma_map_ops dma_direct_ops;`를 추가하고 컴파일하니 정상적으로 컴파일되는줄 알았는데..

```sh
/home/fuzzer/porting/janus-master/lkl/tools/lkl/liblkl.a(lkl.o): In function `get_dma_ops':
/home/fuzzer/porting/janus-master/lkl/./include/linux/dma-mapping.h:273: undefined reference to 'dma_direct_ops'
/home/fuzzer/porting/janus-master/lkl/./include/linux/dma-mapping.h:273: undefined reference to 'dma_direct_ops'
/home/fuzzer/porting/janus-master/lkl/./include/linux/dma-mapping.h:273: undefined reference to 'dma_direct_ops'
/home/fuzzer/porting/janus-master/lkl/./include/linux/dma-mapping.h:273: undefined reference to 'dma_direct_ops'
/home/fuzzer/porting/janus-master/lkl/./include/linux/dma-mapping.h:273: undefined reference to 'dma_direct_ops'
/home/fuzzer/porting/janus-master/lkl/tools/lkl/liblkl.a(lkl.o):/home/fuzzer/porting/janus-master/lkl/./include/linux/dma-mapping.h:273: more undefined references to 'dma_direct_ops' follow
collect2: error: ld returned 1 exit status
Makefile:82: recipe for target '/home/fuzzer/porting/janus-master/lkl/tools/lkl/lklfuse' failed
make: *** [/home/fuzzer/porting/janus-master/lkl/tools/lkl/lklfuse] Error 1
make: Leaving directory '/home/fuzzer/porting/janus-master/lkl/tools/lkl'
root@system:~/porting/janus-master/lkl#
```

위와 같이 함수 내에서 호출할 때의 undefined가 뜬다.

더 찾아보니 최신 LKL에서는 dma_direct_ops 구조체를 사용하는 부분이 없어지고, `return NULL`로 대체되어 있어서 그냥 아래에 명시된 부분을 통으로 주석처리했다.

```c
lib/dma-direct.c:196:const struct dma_map_ops dma_direct_ops = {
lib/dma-direct.c:204:EXPORT_SYMBOL(dma_direct_ops);
```

이후 `make -C tools/lkl` 명령이 성공적으로 실행된다.
이어 `./compile -t btrfs -c` 명령을 실행하면 아래와 같은 afl 관련 오류가 나타난다.

```sh
...

/home/fuzzer/porting/janus-master/lkl/fs/btrfs/tree-checker.c:522: undefined reference to '__afl_prev_loc'
/home/fuzzer/porting/janus-master/lkl/fs/btrfs/tree-checker.c:522: undefined reference to '__afl_area_ptr'
/home/fuzzer/porting/janus-master/lkl/fs/btrfs/tree-checker.c:985: undefined reference to '__afl_in_trace'

...

collect2: error: ld returned 1 exit status
Makefile:82: recipe for target '/home/fuzzer/porting/janus-master/lkl/tools/lkl/lklfuse' failed
make: *** [/home/fuzzer/porting/janus-master/lkl/tools/lkl/lklfuse] Error 1
make: Leaving directory '/home/fuzzer/porting/janus-master/lkl/tools/lkl'
cp: cannot stat 'tools/lkl/fsfuzz': No such file or directory
cp: cannot stat 'tools/lkl/executor': No such file or directory
cp: cannot stat 'tools/lkl/combined': No such file or directory
root@system:~/porting/janus-master/lkl#
```

이제 afl 내의 참조만 건드려주면 되는건가...?

우선 `core/afl-*/llvm_mode`에서 make 명령을 실행하고 다시 컴파일해봤다.
안된다//
느낌상 `ff-gcc/ff-gcc`로 컴파일 할 때 afl 관련 라이브러리 파일 내용이 미스나서 그런 것 같은데...
`core/` 경로에서 make clean하고 make 해봤다.
안된다아..
`ff-gcc/ff-gcc.cc` 파일 내의 llvm_pass.so 파일 경로를 afl-image-syscall 경로로 잡아줬다 (해당 경로의 llvm_mode 폴더에서 make를 해서 .so파일을 생성한 후)
안댐
`ff-gcc.cc`에서 무조건 clang을 사용하게 + llvm-pass.so 파일 경로를 절대 경로로 수정해줌
응 안대~
lkl 경로에서 grep -rni "lklfuse" 명령을 원본 퍼저, 포팅 퍼저 양쪽에서 돌려봤다.
안댐

`tools/lkl/include/lkl.h` 파일 lkl_disk 구조체 내에 buffer, capacity 추가 (원래 파일에는 added by wen 이라고 적혀있음)

이제 lkl_disk 에러는 안난다!!!!
근데 이제 vitio_rings~ 에러가 뜸....

```sh
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl/linux/virtio_ring.h: Infunction ‘void vring_init(lkl_vring*, unsigned int, void*, long unsigned int)’:
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl/linux/virtio_ring.h:169:11: error: invalid conversion from ‘void*’ to ‘lkl_vring_desc*’ [-fpermissive]
  vr->desc = p;
           ^
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl/linux/virtio_ring.h:170:50: warning: pointer of type ‘void *’ used in arithmetic [-Wpointer-arith]
  vr->avail = p + num*sizeof(struct lkl_vring_desc);
                                                  ^
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl/linux/virtio_ring.h:170:16: error: invalid conversion from ‘void*’ to ‘lkl_vring_avail*’ [-fpermissive]
  vr->avail = p + num*sizeof(struct lkl_vring_desc);
                ^
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl/linux/virtio_ring.h:171:11: error: invalid conversion from ‘void*’ to ‘lkl_vring_used*’ [-fpermissive]
  vr->used = (void *)(((lkl_uintptr_t)&vr->avail->ring[num] + sizeof(__lkl__vi
           ^
In file included from executor.hpp:4:0,
                 from executor.cpp:25:
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h: At global scope:
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h:166:67: error: expected ‘,’ or ‘...’ before ‘new’
 static inline long lkl_sys_link(const char *existing, const char *new)
                                                                   ^
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h: In function ‘long int lkl_sys_link(const char*, const char*)’:
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h:168:65: error: expected type-specifier before ‘,’ token
  return lkl_sys_linkat(LKL_AT_FDCWD, existing, LKL_AT_FDCWD, new, 0);
                                                                 ^
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h: At global scope:
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h:186:70: error: expected ‘,’ or ‘...’ before ‘new’
 static inline long lkl_sys_symlink(const char *existing, const char *new)
                                                                      ^
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h: In function ‘long int lkl_sys_symlink(const char*, const char*)’:
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h:188:54: error: expected type-specifier before ‘)’ token
  return lkl_sys_symlinkat(existing, LKL_AT_FDCWD, new);
                                                      ^
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h: At global scope:
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h:206:64: error: expected ‘,’ or ‘...’ before ‘new’
                 from executor.cpp:25:
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h: At global scope:
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h:166:67: error: expected ‘,’ or ‘...’ before ‘new’
 static inline long lkl_sys_link(const char *existing, const char *new)                                                                   ^
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h: In function ‘long int lkl_sys_link(const char*, const char*)’:
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h:168:65: error: expected type-specifier before ‘,’ token
  return lkl_sys_linkat(LKL_AT_FDCWD, existing, LKL_AT_FDCWD, new, 0);                                                                 ^
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h: At global scope:
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h:186:70: error: expected ‘,’ or ‘...’ before ‘new’ static inline long lkl_sys_symlink(const char *existing, const char *new)
                                                                      ^/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h: In function ‘long int lkl_sys_symlink(const char*, const char*)’:
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h:188:54: error: expected type-specifier before ‘)’ token
  return lkl_sys_symlinkat(existing, LKL_AT_FDCWD, new);
                                                      ^
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h: At global scope:
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h:206:64: error: expected ‘,’ or ‘...’ before ‘new’
 static inline long lkl_sys_rename(const char *old, const char *new)
                                                                ^
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h: In function ‘long int lkl_sys_rename(const char*, const char*)’:
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h:208:62: error: expected type-specifier before ‘)’ token
  return lkl_sys_renameat(LKL_AT_FDCWD, old, LKL_AT_FDCWD, new);
                                                              ^
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h: In function ‘long int lkl_sys_poll(lkl_pollfd*, int, int)’:
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h:307:41: error: takingaddress of temporary [-fpermissive]
        .tv_nsec = timeout%1000*1000000 }) : 0,
                                         ^
cat: /home/fuzzer/porting/janus/lkl/tools/lkl/.executor.o.d: No such file or directory
/home/fuzzer/porting/janus/lkl/tools/build/Makefile.build:100: recipe for target '/home/fuzzer/porting/janus/lkl/tools/lkl/executor.o' failed
make[1]: *** [/home/fuzzer/porting/janus/lkl/tools/lkl/executor.o] Error 1
Makefile:122: recipe for target '/home/fuzzer/porting/janus/lkl/tools/lkl/executor-in.o' failed
make: *** [/home/fuzzer/porting/janus/lkl/tools/lkl/executor-in.o] Error 2
make: Leaving directory '/home/fuzzer/porting/janus/lkl/tools/lkl'
cp: cannot stat 'tools/lkl/executor': No such file or directory
cp: cannot stat 'tools/lkl/combined': No such file or directory
root@system:~/porting/janus/lkl#
```
쮸발...........
원본 파일 봤더니 캐스팅이나 기타 등등 추가되어있음...
하나하나 수정 들어갑니다 ^^..

수정 했는데 이번엔

```sh
# /home/fuzzer/porting/janus/lkl
  CC       /home/fuzzer/porting/janus/lkl/tools/lkl/fsfuzz.o
  LD       /home/fuzzer/porting/janus/lkl/tools/lkl/fsfuzz-in.o
  LINK     /home/fuzzer/porting/janus/lkl/tools/lkl/fsfuzz
# @echo '  LINK     '/home/fuzzer/porting/janus/lkl/tools/lkl/fsfuzz;g++ -pie -o /home/fuzzer/porting/janus/lkl/tools/lkl/fsfuzz /home/fuzzer/porting/janus/lkl/tools/lkl/fsfuzz-in.o /home/fuzzer/porting/janus/lkl/tools/lkl/liblkl.a -lrt -pthread -larchive
# /home/fuzzer/porting/janus/lkl
  CXX      /home/fuzzer/porting/janus/lkl/tools/lkl/executor.o
In file included from executor.hpp:4:0,
                 from executor.cpp:25:
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h: In function ‘long int lkl_sys_poll(lkl_pollfd*, int, int)’:
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h:308:41: error: takingaddress of temporary [-fpermissive]
        .tv_nsec = timeout%1000*1000000 }) : 0,
                                         ^
cat: /home/fuzzer/porting/janus/lkl/tools/lkl/.executor.o.d: No such file or directory
/home/fuzzer/porting/janus/lkl/tools/build/Makefile.build:100: recipe for target '/home/fuzzer/porting/janus/lkl/tools/lkl/executor.o' failed
make[1]: *** [/home/fuzzer/porting/janus/lkl/tools/lkl/executor.o] Error 1
Makefile:122: recipe for target '/home/fuzzer/porting/janus/lkl/tools/lkl/executor-in.o' failed
make: *** [/home/fuzzer/porting/janus/lkl/tools/lkl/executor-in.o] Error 2
make: Leaving directory '/home/fuzzer/porting/janus/lkl/tools/lkl'
cp: cannot stat 'tools/lkl/executor': No such file or directory
cp: cannot stat 'tools/lkl/combined': No such file or directory
root@system:~/pl#
```
아니 이건 또 어떻게고쳐야되는거야대체 안ㅇㅁㄴ암이ㅏㄴ묾

ㅎㅎ 컴파일이 되는데 런타임에서 터진다.
lkl_add_disk 함수에서 펑펑펑펑펑ㅍ어펑펑퍼엎어퍼엎어펑



# 처음부터 다시하자!


![nu](https://nonetype.github.io/assets/nu.jpg)

***

# 0906 JANUS Porting

`lkl/tools/lkl/Build` 세줄 추가
```sh
fsfuzz-y += fsfuzz.o
executor-y += executor.o
combined-y += combined.o
```
AFTER
```sh
CFLAGS_lklfuse.o += -D_FILE_OFFSET_BITS=64

fsfuzz-y += fsfuzz.o
executor-y += executor.o
combined-y += combined.o
cptofs-$(LKL_HOST_CONFIG_ARCHIVE) += cptofs.o
fs2tar-$(LKL_HOST_CONFIG_ARCHIVE) += fs2tar.o
lklfuse-$(LKL_HOST_CONFIG_FUSE) += lklfuse.o
```

dma_direct_ops 고쳐야지?

```sh
./arch/lkl/include/asm/dma-mapping.h:7:11: note: each undeclared identifier is reported only once for each function it appears in
In file included from ./include/linux/dma-mapping.h:266:0,
                 from ./include/linux/skbuff.h:30,
                 from ./include/net/net_namespace.h:37,
                 from ./include/linux/inet.h:42,
                 from ./include/linux/sunrpc/msg_prot.h:204,
                 from ./include/linux/sunrpc/auth.h:16,
                 from ./include/linux/nfs_fs.h:31,
                 from kernel/sysctl.c:54:
./arch/lkl/include/asm/dma-mapping.h: In function ‘get_arch_dma_ops’:
./arch/lkl/include/asm/dma-mapping.h:7:11: error: ‘dma_direct_ops’ undeclared (first use in this function)
   return &dma_direct_ops;
           ^
```

위에서 했던것처럼 그냥 주석처리 해도 되나...?

AFTER
```c
static inline const struct dma_map_ops *get_arch_dma_ops(struct bus_type *bus)
{
        // return &dma_noop_ops;
//  return &dma_direct_ops; nonetype
return NULL;
}
```

이제는 afl에러가 난다.

```
/home/fuzzer/porting/janus/lkl/fs/btrfs/acl.c:97: undefined reference to '__afl_prev_loc'
/home/fuzzer/porting/janus/lkl/fs/btrfs/acl.c:97: undefined reference to '__afl_area_ptr'
```

Makefile을 diff해보니 차이가 있다.

`tools/lkl/Makefile`
janus lkl
```sh
# rules to link libs
$(OUTPUT)%$(SOSUF): LDFLAGS += -shared
$(OUTPUT)%$(SOSUF): $(OUTPUT)%-in.o $(OUTPUT)liblkl.a
        $(QUIET_LINK)$(CC) $(LDFLAGS) $(LDFLAGS_$*-y) -o $@ $^ $(LDLIBS) $(LDLIBS_$*-y)

# Compile runtime for fuzzing
$(OUTPUT)kafl-llvm-rt.o: ../../../core/afl-image/llvm_mode/kafl-llvm-rt.o.c
        (cd ../../../core/afl-image/llvm_mode && env -u CPP -u CC -u MAKEFLAGS -u LDFLAGS LLVM_CONFIG=llvm-config make)
        cp -f ../../../core/afl-image/kafl-llvm-rt.o $@

$(OUTPUT)kcov-llvm-rt.o: ../../../core/afl-image/llvm_mode/kcov-llvm-rt.o.cc
        (cd ../../../core/afl-image/llvm_mode && env -u CPP -u CC -u MAKEFLAGS -u LDFLAGS LLVM_CONFIG=llvm-config make)
        cp -f ../../../core/afl-image/kcov-llvm-rt.o $@

# liblkl is special
$(OUTPUT)liblkl$(SOSUF): $(OUTPUT)%-in.o $(OUTPUT)lib/lkl.o
$(OUTPUT)liblkl.a: $(OUTPUT)lib/liblkl-in.o $(OUTPUT)lib/lkl.o $(OUTPUT)kafl-llvm-rt.o
```

original lkl (latest)
```sh
# rules to link libs
$(OUTPUT)%$(SOSUF): LDFLAGS += -shared
$(OUTPUT)%$(SOSUF): $(OUTPUT)%-in.o $(OUTPUT)liblkl.a
        $(QUIET_LINK)$(CC) $(LDFLAGS) $(LDFLAGS_$*-y) -o $@ $^ $(LDLIBS) $(LDLIBS_$*-y)

# liblkl is special
$(OUTPUT)liblkl$(SOSUF): $(OUTPUT)%-in.o $(OUTPUT)lib/lkl.o
$(OUTPUT)liblkl.a: $(OUTPUT)lib/liblkl-in.o $(OUTPUT)lib/lkl.o
        $(QUIET_AR)$(AR) -rc $@ $^
```

가운데의 `'#Compile runtime for fuzzing'` 부분을 추가했다.
추가적으로 살펴보니 추가한 부분의 아래 부분에도 executer의 dependency가 명시되어 있었다.

janus lkl
```c
# liblkl is special
$(OUTPUT)liblkl$(SOSUF): $(OUTPUT)%-in.o $(OUTPUT)lib/lkl.o
$(OUTPUT)liblkl.a: $(OUTPUT)lib/liblkl-in.o $(OUTPUT)lib/lkl.o $(OUTPUT)kafl-llvm-rt.o
# $(OUTPUT)liblkl.a: $(OUTPUT)lib/liblkl-in.o $(OUTPUT)lib/lkl.o $(OUTPUT)kcov-llvm-rt.o
        $(QUIET_AR)$(AR) -rc $@ $^

# executer's dependencies

$(OUTPUT)Image.o:
        cp -f ../../../core/Image.o $@

$(OUTPUT)Program.o:
        cp -f ../../../core/Program.o $@

$(OUTPUT)Utils.o:
        cp -f ../../../core/Utils.o $@

$(OUTPUT)Constants.o:
        cp -f ../../../core/Constants.o $@

# executor (CPP)
$(OUTPUT)executor: $(OUTPUT)executor-in.o $(OUTPUT)Image.o $(OUTPUT)Program.o$(OUTPUT)Utils.o $(OUTPUT)Constants.o $(OUTPUT)liblkl.a
        $(QUIET_LINK)$(CXX) $(LDFLAGS) $(LDFLAGS_$*-y) -o $@ $^ $(LDLIBS) $(LDLIBS_$*-y)

# combined (CPP)
$(OUTPUT)combined: $(OUTPUT)combined-in.o $(OUTPUT)Image.o $(OUTPUT)Program.o$(OUTPUT)Utils.o $(OUTPUT)Constants.o $(OUTPUT)liblkl.a
        $(QUIET_LINK)$(CXX) $(LDFLAGS) $(LDFLAGS_$*-y) -o $@ $^ $(LDLIBS) $(LDLIBS_$*-y)

# rule to link programs
$(OUTPUT)%$(EXESUF): $(OUTPUT)%-in.o $(OUTPUT)liblkl.a
        $(QUIET_LINK)$(CC) $(LDFLAGS) $(LDFLAGS_$*-y) -o $@ $^ $(LDLIBS) $(LDLIBS_$*-y)
        # $(QUIET_LINK)$(CXX) $(LDFLAGS) $(LDFLAGS_$*-y) -o $@ $^ $(LDLIBS) $(LDLIBS_$*-y)
        # $(srctree)
```

original lkl (latest)
```sh
# liblkl is special
$(OUTPUT)liblkl$(SOSUF): $(OUTPUT)%-in.o $(OUTPUT)lib/lkl.o
$(OUTPUT)liblkl.a: $(OUTPUT)lib/liblkl-in.o $(OUTPUT)lib/lkl.o
        $(QUIET_AR)$(AR) -rc $@ $^

# rule to link programs
$(OUTPUT)%$(EXESUF): $(OUTPUT)%-in.o $(OUTPUT)liblkl.a
        $(QUIET_LINK)$(CC) $(LDFLAGS) $(LDFLAGS_$*-y) -o $@ $^ $(LDLIBS) $(LDLIBS_$*-y)
```

그리고 위쪽에 `export CXXFLAGS` 부분도 없어서 추가해줬다.

```sh
export CFLAGS += -I$(OUTPUT)/include -Iinclude -Wall -g -O2 -Wextra \
         -Wno-unused-parameter \
         -Wno-missing-field-initializers -fno-strict-aliasing

export CXXFLAGS += -std=c++11 -fPIC -I$(OUTPUT)/include -Iinclude -I$(OUTPUT)../../../core \
         -Wall -g -O2 -Wextra \
         -Wno-unused-parameter \
         -Wno-missing-field-initializers -fno-strict-aliasing

-include Targets
```

이제 다시 `./compile -t btrfs -c` !!!

또 다시 오류..
```sh
/home/fuzzer/porting/janus/lkl/./include/asm-generic/atomic.h:118: undefined reference to `__afl_prev_loc'
/home/fuzzer/porting/janus/lkl/./include/asm-generic/atomic.h:118: undefined reference to `__afl_area_ptr'
```


Makefile에 `$(OUTPUT)kafl-llvm-rt.o` 추가
AFTER
```
# liblkl is special
$(OUTPUT)liblkl$(SOSUF): $(OUTPUT)%-in.o $(OUTPUT)lib/lkl.o
$(OUTPUT)liblkl.a: $(OUTPUT)lib/liblkl-in.o $(OUTPUT)lib/lkl.o $(OUTPUT)kafl-l
lvm-rt.o
        $(QUIET_AR)$(AR) -rc $@ $^
```

다시 `./compile -t btrfs -c`

```sh
make[1]: Entering directory '/home/fuzzer/porting/janus/core/afl-image/llvm_mode'
[*] Checking for working 'llvm-config'...
[*] Checking for working 'clang'...
[*] Checking for '../afl-showmap'...
[+] All set and ready to build.
make[1]: Leaving directory '/home/fuzzer/porting/janus/core/afl-image/llvm_mode'
cp -f ../../../core/afl-image/kafl-llvm-rt.o /home/fuzzer/porting/janus/lkl/tools/lkl/kafl-llvm-rt.o
  AR       /home/fuzzer/porting/janus/lkl/tools/lkl/liblkl.a
  LINK     /home/fuzzer/porting/janus/lkl/tools/lkl/lklfuse
  CC       /home/fuzzer/porting/janus/lkl/tools/lkl/fs2tar.o
  LD       /home/fuzzer/porting/janus/lkl/tools/lkl/fs2tar-in.o
  LINK     /home/fuzzer/porting/janus/lkl/tools/lkl/fs2tar
  CC       /home/fuzzer/porting/janus/lkl/tools/lkl/cptofs.o
  LD       /home/fuzzer/porting/janus/lkl/tools/lkl/cptofs-in.o
  LINK     /home/fuzzer/porting/janus/lkl/tools/lkl/cptofs
  CC       /home/fuzzer/porting/janus/lkl/tools/lkl/tests/boot.o
  CC       /home/fuzzer/porting/janus/lkl/tools/lkl/tests/test.o
  LD       /home/fuzzer/porting/janus/lkl/tools/lkl/tests/boot-in.o
  LINK     /home/fuzzer/porting/janus/lkl/tools/lkl/tests/boot
  CC       /home/fuzzer/porting/janus/lkl/tools/lkl/tests/disk.o
  CC       /home/fuzzer/porting/janus/lkl/tools/lkl/tests/cla.o
  LD       /home/fuzzer/porting/janus/lkl/tools/lkl/tests/disk-in.o
  LINK     /home/fuzzer/porting/janus/lkl/tools/lkl/tests/disk
  CC       /home/fuzzer/porting/janus/lkl/tools/lkl/tests/net-test.o
  LD       /home/fuzzer/porting/janus/lkl/tools/lkl/tests/net-test-in.o
  LINK     /home/fuzzer/porting/janus/lkl/tools/lkl/tests/net-test
  LINK     /home/fuzzer/porting/janus/lkl/tools/lkl/lib/liblkl.so
  CC       /home/fuzzer/porting/janus/lkl/tools/lkl/lib/hijack/hijack.o
  CC       /home/fuzzer/porting/janus/lkl/tools/lkl/lib/hijack/init.o
  CC       /home/fuzzer/porting/janus/lkl/tools/lkl/lib/hijack/xlate.o
  LD       /home/fuzzer/porting/janus/lkl/tools/lkl/lib/hijack/liblkl-hijack-in.o
  LINK     /home/fuzzer/porting/janus/lkl/tools/lkl/lib/hijack/liblkl-hijack.so
make: Leaving directory '/home/fuzzer/porting/janus/lkl/tools/lkl'
cp: cannot stat 'tools/lkl/fsfuzz': No such file or directory
cp: cannot stat 'tools/lkl/executor': No such file or directory
cp: cannot stat 'tools/lkl/combined': No such file or directory
root@system:~/pl#
```

빌드는 정상적으로 됬는데, fsfuzz, executor, combined 바이너리가 생성되지 않았다.
grep으로 긁어보니 `tools/lkl/Target`에 추가되지 않았다.

BEFORE
```sh
progs-$(LKL_HOST_CONFIG_FUSE) += lklfuse
LDLIBS_lklfuse-y := -lfuse

progs-$(LKL_HOST_CONFIG_ARCHIVE) += fs2tar
LDLIBS_fs2tar-y := -larchive
LDLIBS_fs2tar-$(LKL_HOST_CONFIG_NEEDS_LARGP) += -largp


progs-$(LKL_HOST_CONFIG_ARCHIVE) += cptofs
LDLIBS_cptofs-y := -larchive
LDLIBS_cptofs-$(LKL_HOST_CONFIG_NEEDS_LARGP) += -largp
```

AFTER
```sh
progs-$(LKL_HOST_CONFIG_FUSE) += lklfuse
LDLIBS_lklfuse-y := -lfuse

progs-$(LKL_HOST_CONFIG_ARCHIVE) += fs2tar
LDLIBS_fs2tar-y := -larchive
LDLIBS_fs2tar-$(LKL_HOST_CONFIG_NEEDS_LARGP) += -largp

progs-y += fsfuzz
LDLIBS_fsfuzz-y := -larchive
LDLIBS_fsfuzz-$(LKL_HOST_CONFIG_NEEDS_LARGP) += -largp

progs-y += executor
LDLIBS_executor-y := -larchive
LDLIBS_executor-$(LKL_HOST_CONFIG_NEEDS_LARGP) += -largp

progs-y += combined
LDLIBS_combined-y := -larchive
LDLIBS_combined-$(LKL_HOST_CONFIG_NEEDS_LARGP) += -largp

progs-$(LKL_HOST_CONFIG_ARCHIVE) += cptofs
LDLIBS_cptofs-y := -larchive
LDLIBS_cptofs-$(LKL_HOST_CONFIG_NEEDS_LARGP) += -largp
```

lkl_disk 에러!
```sh
fsfuzz.c: In function ‘main’:
fsfuzz.c:307:7: error: ‘struct lkl_disk’ has no member named ‘buffer’
   disk.buffer = userfault_init(image_buffer, size);
       ^
fsfuzz.c:308:7: error: ‘struct lkl_disk’ has no member named ‘capacity’
   disk.capacity = size;
       ^
```

Wen su가 내가 된다! 'added by wen'를 grep!!!

BEFORE
```c
/**
 * lkl_disk - host disk handle
 *
 * @dev - a pointer to 'virtio_blk_dev' structure for this disk
 * @fd - a POSIX file descriptor that can be used by preadv/pwritev
 * @handle - an NT file handle that can be used by ReadFile/WriteFile
 */
struct lkl_disk {
        void *dev;
        union {
                int fd;
                void *handle;
        };
        struct lkl_dev_blk_ops *ops;
};
```
AFTER
```c
/**
 * lkl_disk - host disk handle
 *
 * @dev - a pointer to 'virtio_blk_dev' structure for this disk
 * @fd - a POSIX file descriptor that can be used by preadv/pwritev
 * @handle - an NT file handle that can be used by ReadFile/WriteFile
 */
struct lkl_disk {
        void *dev;
        union {
                int fd;
                void *handle;
        };
        struct lkl_dev_blk_ops *ops;
        // (added by nonetype)
        void *buffer;
        unsigned long long capacity;
};
```

다시~ 컴파일~

```sh
In file included from /home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl/asm/syscalls.h:161:0,
                 from /home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h:23,
                 from executor.hpp:4,
                 from executor.cpp:25:
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl/linux/virtio_ring.h: Infunction ‘void vring_init(lkl_vring*, unsigned int, void*, long unsigned int)’:
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl/linux/virtio_ring.h:169:11: error: invalid conversion from ‘void*’ to ‘lkl_vring_desc*’ [-fpermissive]
  vr->desc = p;
           ^
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl/linux/virtio_ring.h:170:50: warning: pointer of type ‘void *’ used in arithmetic [-Wpointer-arith]
  vr->avail = p + num*sizeof(struct lkl_vring_desc);
                                                  ^
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl/linux/virtio_ring.h:170:16: error: invalid conversion from ‘void*’ to ‘lkl_vring_avail*’ [-fpermissive]
  vr->avail = p + num*sizeof(struct lkl_vring_desc);
                ^
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl/linux/virtio_ring.h:171:11: error: invalid conversion from ‘void*’ to ‘lkl_vring_used*’ [-fpermissive]
  vr->used = (void *)(((lkl_uintptr_t)&vr->avail->ring[num] + sizeof(__lkl__vi
           ^
In file included from executor.hpp:4:0,
                 from executor.cpp:25:
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h: At global scope:
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h:166:67: error: expected ‘,’ or ‘...’ before ‘new’
 static inline long lkl_sys_link(const char *existing, const char *new)
                                                                   ^
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h: In function ‘long int lkl_sys_link(const char*, const char*)’:
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h:168:65: error: expected type-specifier before ‘,’ token
  return lkl_sys_linkat(LKL_AT_FDCWD, existing, LKL_AT_FDCWD, new, 0);
                                                                 ^
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h: At global scope:
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h:186:70: error: expected ‘,’ or ‘...’ before ‘new’
 static inline long lkl_sys_symlink(const char *existing, const char *new)
                                                                      ^
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h: In function ‘long int lkl_sys_symlink(const char*, const char*)’:
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h:188:54: error: expected type-specifier before ‘)’ token
  return lkl_sys_symlinkat(existing, LKL_AT_FDCWD, new);
                                                      ^
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h: At global scope:
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h:206:64: error: expected ‘,’ or ‘...’ before ‘new’
 static inline long lkl_sys_rename(const char *old, const char *new)
                                                                ^
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h: In function ‘long int lkl_sys_rename(const char*, const char*)’:
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h:208:62: error: expected type-specifier before ‘)’ token
  return lkl_sys_renameat(LKL_AT_FDCWD, old, LKL_AT_FDCWD, new);
                                                              ^
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h: In function ‘long int lkl_sys_poll(lkl_pollfd*, int, int)’:
/home/fuzzer/porting/janus/lkl/tools/lkl//include/lkl.h:307:41: error: takingaddress of temporary [-fpermissive]
        .tv_nsec = timeout%1000*1000000 }) : 0,
                                         ^
```
아니 언눔시끼가 변수명을 new로 써....
일단 `lkl.h` 파일부터 수정한다. (변수명 new를 _new로 변경!)

그리고 sys_poll 함수는.....

BEFORE
```c
/**
 * lkl_sys_poll - wrapper for lkl_sys_ppoll
 */
static inline long lkl_sys_poll(struct lkl_pollfd *fds, int n, int timeout)
{
        return lkl_sys_ppoll(fds, n, timeout >= 0 ?
                             (struct __lkl__kernel_timespec *)
                             &((struct lkl_timespec){ .tv_sec = timeout/1000,
                                   .tv_nsec = timeout%1000*1000000 }) : 0,
                             0, _LKL_NSIG/8);
}
```

AFTER
```c
/**
 * lkl_sys_poll - wrapper for lkl_sys_ppoll
 */
static inline long lkl_sys_poll(struct lkl_pollfd *fds, int n, int timeout)
{
        struct lkl_timespec fs;
        fs.tv_sec  = timeout/1000;
        fs.tv_nsec = timeout%1000*1000000;
        return lkl_sys_ppoll(fds, n, timeout >= 0 ? (struct __lkl__kernel_timespec *)&(fs) : 0, 0, _LKL_NSIG/8);
}
```

그리고 `virtio_ring.h` 파일을 수정해야 하는데, 에러나서 나타나는 파일 경로는 카피본의 파일 경로이기 때문에 어제는 아무리 고쳐도 적용이 안됬었다 \^\^..

```sh
root@system:~/pl# grep -rni "vr->desc = p"
tools/lkl/include/lkl/linux/virtio_ring.h:169:	vr->desc = p;
include/uapi/linux/virtio_ring.h:171:	vr->desc = p;
root@system:~/pl#
```
짜잔~

BEFORE
```c
static inline void vring_init(struct vring *vr, unsigned int num, void *p,
                              unsigned long align)
{
        vr->num = num;
        vr->desc = p;
        vr->avail = p + num*sizeof(struct vring_desc);
        vr->used = (void *)(((uintptr_t)&vr->avail->ring[num] + sizeof(__virtio16)
                + align-1) & ~(align - 1));
}
```

AFTER
```c
static inline void vring_init(struct vring *vr, unsigned int num, void *p,
                              unsigned long align)
{
        vr->num = num;
        vr->desc = (struct vring_desc*)p;
        vr->avail = (struct vring_avail*)((char *)p + num*sizeof(struct vring_desc));
        vr->used = (struct vring_used*)(((uintptr_t)&vr->avail->ring[num] + sizeof(__virtio16)
                + align-1) & ~(align - 1));
}
```

다시 컴파일~~

성공!
하지만 이젠 afl을 돌리면 에러가 터진다..

```sh
root@system:~/porting/janus# ./core/afl-image-syscall/afl-fuzz -b btrfs -s fs/btrfs/btrfs_wrapper.so -e ./samples/evaluation/btrfs-00.image -S btrfs -y prog -i input -o output -m none -u 2 -- ./lkl/tools/lkl/btrfs-combined -t btrfs -p @@
afl-fuzz 2.52b by <lcamtuf@google.com>
[+] [fs-fuzz] shm name to store image buffer: btrfs
[+] [fs-fuzz] target wrapper (.so) path: fs/btrfs/btrfs_wrapper.so
[+] [fs-fuzz] seed image path: ./samples/evaluation/btrfs-00.image
[+] [fs-fuzz] syscall input directory: prog
[+] You have 8 CPU cores and 1 runnable tasks (utilization: 12%).
[+] Try parallel jobs - see docs/parallel_fuzzing.txt.
[+] Found a free CPU core, binding to #2.
[*] Checking core_pattern...
[+] [+] Open shm btrfs success.
[+] [+] Map shm btrfs at 0x7fb2f127e000 size: 0x8000000.

[*] Scanning 'prog'...
[*] Setting up output directories...
[+] Output directory exists but deemed OK to reuse.
[*] Deleting old session data...
[+] Output dir cleanup successful.
[*] Scanning 'input'...
[+] No auto-generated dictionary tokens to reuse.
[*] Creating hard links for all input files...
[*] Validating target binary...
[*] Attempting dry run with 'id:000000,orig:open_read0'...
[*] Spinning up the fork server...
[+] All right - fork server is up.
[!] WARNING: Test case results in a crash (skipping)
[*] Attempting dry run with 'id:000001,orig:open_read1'...
[!] WARNING: Test case results in a crash (skipping)
[*] Attempting dry run with 'id:000002,orig:open_read2'...
[!] WARNING: Test case results in a crash (skipping)
[*] Attempting dry run with 'id:000003,orig:open_read3'...
[!] WARNING: Test case results in a crash (skipping)
[*] Attempting dry run with 'id:000004,orig:open_read4'...
[!] WARNING: Test case results in a crash (skipping)
[*] Attempting dry run with 'id:000005,orig:open_read5'...
[!] WARNING: Test case results in a crash (skipping)
[*] Attempting dry run with 'id:000006,orig:open_read6'...
[!] WARNING: Test case results in a crash (skipping)
[*] Attempting dry run with 'id:000007,orig:open_read7'...
[!] WARNING: Test case results in a crash (skipping)
[*] Attempting dry run with 'id:000008,orig:open_read8'...
[!] WARNING: Test case results in a crash (skipping)

[-] PROGRAM ABORT : All test cases time out or crash, giving up!
         Location : perform_dry_run(), afl-fuzz.c:3042
```

***

#### janus lkl ~~ https://github.com/lkl/linux/commit/7cfde0af731c14664e3882c7ba77ace1059f2c5e
```sh
root@system:~/porting/janus# gdb ./lkl/tools/lkl/btrfs-combined
GNU gdb (Ubuntu 7.11.1-0ubuntu1~16.5) 7.11.1
Copyright (C) 2016 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
pwndbg: loaded 175 commands. Type pwndbg [filter] for a list.
pwndbg: created $rebase, $ida gdb functions (can be used with print/break)
Reading symbols from ./lkl/tools/lkl/btrfs-combined...done.
pwndbg> r -t btrfs -p prog/open_read0 -i samples/evaluation/btrfs-00.image
Starting program: /home/fuzzer/porting/janus/lkl/tools/lkl/btrfs-combined -t btrfs -p prog/open_read0 -i samples/evaluation/btrfs-00.image
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
[New Thread 0x7fffe6d46700 (LWP 38017)]
can't add disk: Out of memory

Thread 1 "btrfs-combined" received signal SIGSEGV, Segmentation fault.
0x00005555555bc334 in __cpu_try_get_lock (n=0) at arch/lkl/kernel/cpu.c:63
63		lkl_ops->mutex_lock(cpu.lock);
LEGEND: STACK | HEAP | CODE | DATA | RWX | RODATA
────────────────────────────────────────────────────[ REGISTERS ]─────────────────────────────────────────────────────
 RAX  0x0
 RBX  0x555555f09420 (cpu) ◂— 0x0
 RCX  0x0
 RDX  0x7ffff7416770 (_IO_stdfile_2_lock) ◂— 0
 RDI  0x0
 RSI  0x7fffffffe310 ◂— 0xfee1dead
 R8   0x7ffff7fe1740 ◂— 0x7ffff7fe1740
 R9   0x1e
 R10  0xd
 R11  0x0
 R12  0x555555f00188 (lkl_ops) ◂— 0x0
 R13  0x0
 R14  0x0
 R15  0x0
 RBP  0x7fffffffe2a0 —▸ 0x7fffffffe2d0 —▸ 0x7fffffffe300 —▸ 0x7fffffffe340 —▸ 0x555555aecfb8 ◂— ...
 RSP  0x7fffffffe280 ◂— 0x6e0000005b /* '[' */
 RIP  0x5555555bc334 (__cpu_try_get_lock+52) ◂— call   qword ptr [rax + 0x48]
──────────────────────────────────────────────────────[ DISASM ]──────────────────────────────────────────────────────
 ► 0x5555555bc334 <__cpu_try_get_lock+52>    call   qword ptr [rax + 0x48]

   0x5555555bc337 <__cpu_try_get_lock+55>    cmp    dword ptr [rbx + 8], 0xf423f
   0x5555555bc33e <__cpu_try_get_lock+62>    mov    eax, 0xffffffff
   0x5555555bc343 <__cpu_try_get_lock+67>    ja     __cpu_try_get_lock+101 <0x5555555bc365>

   0x5555555bc345 <__cpu_try_get_lock+69>    mov    rax, qword ptr [r12]
   0x5555555bc349 <__cpu_try_get_lock+73>    call   qword ptr [rax + 0x78]

   0x5555555bc34c <__cpu_try_get_lock+76>    mov    rdi, qword ptr [rbx + 0x18]
   0x5555555bc350 <__cpu_try_get_lock+80>    mov    r13, rax
   0x5555555bc353 <__cpu_try_get_lock+83>    test   rdi, rdi
   0x5555555bc356 <__cpu_try_get_lock+86>    jne    __cpu_try_get_lock+112 <0x5555555bc370>

   0x5555555bc358 <__cpu_try_get_lock+88>    add    dword ptr [rbx + 0x14], 1
──────────────────────────────────────────────────[ SOURCE (CODE) ]───────────────────────────────────────────────────
In file: /home/fuzzer/porting/janus/lkl/arch/lkl/kernel/cpu.c
   58 	lkl_thread_t self;
   59
   60 	if (__sync_fetch_and_add(&cpu.shutdown_gate, n) >= MAX_THREADS)
   61 		return -2;
   62
 ► 63 	lkl_ops->mutex_lock(cpu.lock);
   64
   65 	if (cpu.shutdown_gate >= MAX_THREADS)
   66 		return -1;
   67
   68 	self = lkl_ops->thread_self();
──────────────────────────────────────────────────────[ STACK ]───────────────────────────────────────────────────────
00:0000│ rsp  0x7fffffffe280 ◂— 0x6e0000005b /* '[' */
01:0008│      0x7fffffffe288 —▸ 0x7fffffffe310 ◂— 0xfee1dead
02:0010│      0x7fffffffe290 ◂— 0x8e
03:0018│      0x7fffffffe298 ◂— 0x0
04:0020│ rbp  0x7fffffffe2a0 —▸ 0x7fffffffe2d0 —▸ 0x7fffffffe300 —▸ 0x7fffffffe340 —▸ 0x555555aecfb8 ◂— ...
05:0028│      0x7fffffffe2a8 —▸ 0x5555555bc4b5 (lkl_cpu_get+21) ◂— test   eax, eax
06:0030│      0x7fffffffe2b0 —▸ 0x7fffffffe5c0 ◂— 0x7
07:0038│      0x7fffffffe2b8 ◂— 0x0
────────────────────────────────────────────────────[ BACKTRACE ]─────────────────────────────────────────────────────
 ► f 0     5555555bc334 __cpu_try_get_lock+52
   f 1     5555555bc4b5 lkl_cpu_get+21
   f 2                0
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
Program received signal SIGSEGV (fault address 0x48)
pwndbg> r -t btrfs -p prog/open_read0 -i samples/evaluation/btrfs-00.image
```

tools/lkl/lib/virtio_blk.c:
```c
/* commented by nonetype
        ret = dev->ops->get_capacity(*disk, &capacity);
        if (ret) {
                ret = -LKL_ENOMEM;
                goto out_free;
        }
*/
```

`tools/lkl/lib/posix-host.c`


BEFORE
```c
static int do_rw(ssize_t (*fn)(), struct lkl_disk disk, struct lkl_blk_req *req)
{
        off_t off = req->sector * 512;
        void *addr;
        int len;
        int i;
        int ret = 0;

        for (i = 0; i < req->count; i++) {

                addr = req->buf[i].iov_base;
                len = req->buf[i].iov_len;

                do {
                        ret = fn(disk.fd, addr, len, off);

                        if (ret <= 0) {
                                ret = -1;
                                goto out;
                        }

                        addr += ret;
                        len -= ret;
                        off += ret;

                } while (len);
        }

out:
        return ret;
}
```
AFTER
```c
// static int do_rw(ssize_t (*fn)(), struct lkl_disk disk, struct lkl_blk_req *req)
static int do_rw(struct lkl_disk disk, struct lkl_blk_req *req)
{
        off_t off = req->sector * 512;
        void *addr;
        int len;
        int i;
        int ret = 0;

        for (i = 0; i < req->count; i++) {

                addr = req->buf[i].iov_base;
                len = req->buf[i].iov_len;

                do {

          // ret = fn(disk.fd, addr, len, off);
      if (req->type == LKL_DEV_BLK_TYPE_READ) {
        // printf("read at off 0x%lx len 0x%lx\n", off, len);
        memcpy(addr, disk.buffer + off, len);
      } else {
        // printf("write at off 0x%lx len 0x%lx\n", off, len);
        memcpy(disk.buffer + off, addr, len);
      }
      ret = len;

                              if (ret <= 0) {
                                      ret = -1;
                                      goto out;
                              }

                              addr += ret;
                              len -= ret;
                              off += ret;

                      } while (len);
              }

      out:
              return ret;
      }

```

BEFORE
```c
static int blk_request(struct lkl_disk disk, struct lkl_blk_req *req)
{
        int err = 0;

        switch (req->type) {
        case LKL_DEV_BLK_TYPE_READ:
                err = do_rw(pread, disk, req);
                break;
        case LKL_DEV_BLK_TYPE_WRITE:
                err = do_rw(pwrite, disk, req);
                break;
        case LKL_DEV_BLK_TYPE_FLUSH:
        case LKL_DEV_BLK_TYPE_FLUSH_OUT:
#ifdef __linux__
                err = fdatasync(disk.fd);
#else
                err = fsync(disk.fd);
#endif
                break;
        default:
                return LKL_DEV_BLK_STATUS_UNSUP;
        }

        if (err < 0)
                return LKL_DEV_BLK_STATUS_IOERR;

        return LKL_DEV_BLK_STATUS_OK;
}
```

AFTER
```c
static int blk_request(struct lkl_disk disk, struct lkl_blk_req *req)
{
        int err = 0;

        switch (req->type) {
        case LKL_DEV_BLK_TYPE_READ:
                // err = do_rw(pread, disk, req);
                err = do_rw(disk, req);
                break;
        case LKL_DEV_BLK_TYPE_WRITE:
                // err = do_rw(pwrite, disk, req);
                err = do_rw(disk, req);
                break;
        case LKL_DEV_BLK_TYPE_FLUSH:
        case LKL_DEV_BLK_TYPE_FLUSH_OUT:
#ifdef __linux__
                // err = fdatasync(disk.fd);
#else
                // err = fsync(disk.fd);
#endif
                break;
        default:
                return LKL_DEV_BLK_STATUS_UNSUP;
        }

        if (err < 0)
        return LKL_DEV_BLK_STATUS_IOERR;

return LKL_DEV_BLK_STATUS_OK;
}
```

이제 실행은 되는데 중간에 `can't mount disk: Invalid argument`가 뜬다.
아까 do_rw 수정해서 고쳐진 줄 알았는데 뭐가 또 있나..

```sh
root@system:~/porting/janus# ./lkl/tools/lkl/btrfs-fsfuzz -t btrfs -p -i ./samples/evaluation/btrfs-00.image
[    0.000000] Linux version 5.2.0 (root@system) (gcc version 5.4.0 20160609 (Ubuntu 5.4.0-6ubuntu1~16.04.11)) #10 Fri Sep 6 19:02:54 KST 2019
[    0.000000] memblock address range: 0x7f99e8000000 - 0x7f99effff000
[    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 32319
[    0.000000] Kernel command line: mem=128M virtio_mmio.device=316@0x1000000:1
[    0.000000] Dentry cache hash table entries: 16384 (order: 5, 131072 bytes)
[    0.000000] Inode-cache hash table entries: 8192 (order: 4, 65536 bytes)
[    0.000000] Memory available: 129044k/131068k RAM
[    0.000000] SLUB: HWalign=32, Order=0-3, MinObjects=0, CPUs=1, Nodes=1
[    0.000000] NR_IRQS: 4096
[    0.000000] lkl: irqs initialized
[    0.000000] clocksource: lkl: mask: 0xffffffffffffffff max_cycles: 0x1cd42e4dffb, max_idle_ns: 881590591483 ns
[    0.000000] lkl: time and timers initialized (irq2)
[    0.000007] pid_max: default: 4096 minimum: 301
[    0.000024] Mount-cache hash table entries: 512 (order: 0, 4096 bytes)
[    0.000027] Mountpoint-cache hash table entries: 512 (order: 0, 4096 bytes)
[    0.004619] random: get_random_bytes called from _etext+0xbca1/0x147c8 with crng_init=0
[    0.004756] printk: console [lkl_console0] enabled
[    0.004772] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 19112604462750000 ns
[    0.004775] xor: automatically using best checksumming function   8regs
[    0.004843] NET: Registered protocol family 16
[    0.197565] raid6: int64x8  gen()  5900 MB/s
[    0.384020] raid6: int64x8  xor()  3757 MB/s
[    0.570527] raid6: int64x4  gen()  6581 MB/s
[    0.756991] raid6: int64x4  xor()  4379 MB/s
[    0.943536] raid6: int64x2  gen()  6856 MB/s
[    1.130058] raid6: int64x2  xor()  4409 MB/s
[    1.315511] raid6: int64x1  gen()  6044 MB/s
[    1.502006] raid6: int64x1  xor()  3380 MB/s
[    1.502024] raid6: using algorithm int64x2 gen() 6856 MB/s
[    1.502026] raid6: .... xor() 4409 MB/s, rmw enabled
[    1.502032] raid6: using intx1 recovery algorithm
[    1.502147] clocksource: Switched to clocksource lkl
[    1.502248] NET: Registered protocol family 2
[    1.502430] tcp_listen_portaddr_hash hash table entries: 256 (order: 0, 4096 bytes)
[    1.502449] TCP established hash table entries: 1024 (order: 1, 8192 bytes)
[    1.502455] TCP bind hash table entries: 1024 (order: 1, 8192 bytes)
[    1.502459] TCP: Hash tables configured (established 1024 bind 1024)
[    1.502502] UDP hash table entries: 128 (order: 0, 4096 bytes)
[    1.502518] UDP-Lite hash table entries: 128 (order: 0, 4096 bytes)
[    1.502584] virtio-mmio: Registering device virtio-mmio.0 at 0x1000000-0x100013b, IRQ 1.
[    1.503307] workingset: timestamp_bits=62 max_order=16 bucket_order=0
[    1.504627] io scheduler mq-deadline registered
[    1.504644] io scheduler kyber registered
[    1.504667] virtio-mmio virtio-mmio.0: Failed to enable 64-bit or 32-bit DMA.  Trying to continue, but this might not work.
[    1.507052] virtio_blk virtio0: [vda] 0 512-byte logical blocks (0 B/0 B)
[    1.507275] NET: Registered protocol family 10
[    1.507813] Segment Routing with IPv6
[    1.507840] sit: IPv6, IPv4 and MPLS over IPv4 tunneling driver
[    1.508176] Btrfs loaded, crc32c=crc32c-generic
[    1.508373] Warning: unable to open an initial console.
[    1.508398] This architecture does not have kernel memory protection.
[    1.508411] Run /init as init process
can't mount disk: Invalid argument
[    1.509550] reboot: Restarting system
root@system:~/porting/janus#
```


수정 테스트
`vim pt/lib/fs.c`
```
if (err == -LKL_EBUSY) {
//                      lkl_sys_nanosleep((struct __lkl__kernel_timespec *)&ts,
//                                        NULL);
        lkl_sys_nanosleep(&ts, NULL); // edited by nonetype
        timeout_ms -= incr / 1000000;
}
```

여전히 오류

`tools/lkl/lib/Build` 파일 수정
```sh
diff -r ot/lib/Build pt/lib/Build
7c7
< # liblkl-y += net.o
---
> liblkl-y += net.o
16,23c16,23
< # liblkl-$(LKL_HOST_CONFIG_VIRTIO_NET) += virtio_net.o
< # liblkl-$(LKL_HOST_CONFIG_VIRTIO_NET) += virtio_net_fd.o
< # liblkl-$(LKL_HOST_CONFIG_VIRTIO_NET) += virtio_net_tap.o
< # liblkl-$(LKL_HOST_CONFIG_VIRTIO_NET) += virtio_net_raw.o
< # liblkl-$(LKL_HOST_CONFIG_VIRTIO_NET_MACVTAP) += virtio_net_macvtap.o
< # liblkl-$(LKL_HOST_CONFIG_VIRTIO_NET_DPDK) += virtio_net_dpdk.o
< # liblkl-$(LKL_HOST_CONFIG_VIRTIO_NET_VDE) += virtio_net_vde.o
< # liblkl-$(LKL_HOST_CONFIG_VIRTIO_NET) += virtio_net_pipe.o
---
> liblkl-$(LKL_HOST_CONFIG_VIRTIO_NET) += virtio_net.o
> liblkl-$(LKL_HOST_CONFIG_VIRTIO_NET) += virtio_net_fd.o
> liblkl-$(LKL_HOST_CONFIG_VIRTIO_NET) += virtio_net_tap.o
> liblkl-$(LKL_HOST_CONFIG_VIRTIO_NET) += virtio_net_raw.o
> liblkl-$(LKL_HOST_CONFIG_VIRTIO_NET_MACVTAP) += virtio_net_macvtap.o
> liblkl-$(LKL_HOST_CONFIG_VIRTIO_NET_DPDK) += virtio_net_dpdk.o
> liblkl-$(LKL_HOST_CONFIG_VIRTIO_NET_VDE) += virtio_net_vde.o
> liblkl-$(LKL_HOST_CONFIG_VIRTIO_NET) += virtio_net_pipe.o
```

새로운 에러 발생
```sh
/home/fuzzer/porting/janus/lkl/tools/lkl/liblkl.a(liblkl-in.o): In function 'lkl_config_netdev_create':
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:475: undefined reference to 'lkl_netdev_tap_create'
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:524: undefined reference to 'lkl_netdev_add'
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:502: undefined reference to 'lkl_netdev_raw_create'
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:480: undefined reference to 'lkl_netdev_tap_create'
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:482: undefined reference to 'lkl_netdev_macvtap_create'
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:488: undefined reference to 'lkl_netdev_pipe_create'
/home/fuzzer/porting/janus/lkl/tools/lkl/liblkl.a(liblkl-in.o): In function 'lkl_config_netdev_configure':
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:549: undefined reference to 'lkl_netdev_get_ifindex'
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:551: undefined reference to 'lkl_if_up'
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:561: undefined reference to 'lkl_if_set_mtu'
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:591: undefined reference to 'lkl_if_set_ipv4_gateway'
/home/fuzzer/porting/janus/lkl/tools/lkl/liblkl.a(liblkl-in.o): In function 'add_neighbor':
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:400: undefined reference to 'lkl_add_neighbor'
/home/fuzzer/porting/janus/lkl/tools/lkl/liblkl.a(liblkl-in.o): In function 'lkl_config_netdev_configure':
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:638: undefined reference to 'lkl_qdisc_parse_add'
/home/fuzzer/porting/janus/lkl/tools/lkl/liblkl.a(liblkl-in.o): In function 'lkl_load_config_post':
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:733: undefined reference to 'lkl_set_ipv4_gateway'
/home/fuzzer/porting/janus/lkl/tools/lkl/liblkl.a(liblkl-in.o): In function 'lkl_config_netdev_configure':
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:611: undefined reference to 'lkl_if_set_ipv6'
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:577: undefined reference to 'lkl_if_set_ipv4'
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:624: undefined reference to 'lkl_if_set_ipv6_gateway'
/home/fuzzer/porting/janus/lkl/tools/lkl/liblkl.a(liblkl-in.o): In function 'lkl_load_config_post':
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:746: undefined reference to 'lkl_set_ipv6_gateway'
/home/fuzzer/porting/janus/lkl/tools/lkl/liblkl.a(liblkl-in.o): In function 'lkl_unload_config':
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:783: undefined reference to 'lkl_netdev_remove'
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:784: undefined reference to 'lkl_netdev_free'
collect2: error: ld returned 1 exit status
Makefile:117: recipe for target '/home/fuzzer/porting/janus/lkl/tools/lkl/lklfuse' failed
make: *** [/home/fuzzer/porting/janus/lkl/tools/lkl/lklfuse] Error 1
make: Leaving directory '/home/fuzzer/porting/janus/lkl/tools/lkl'
cp: cannot stat 'tools/lkl/fsfuzz': No such file or directory
cp: cannot stat 'tools/lkl/executor': No such file or directory
cp: cannot stat 'tools/lkl/combined': No such file or directory
```

tools/lkl/Makefile 수정
```sh
#libraries_install: $(libs-y:%=$(OUTPUT)%$(SOSUF)) $(OUTPUT)liblkl.a #edited by nonetype
libraries_install: $(libs-y:%=$(OUTPUT)%$(SOSUF))
        $(call QUIET_INSTALL, libraries) \
            install -d $(DESTDIR)$(PREFIX)/lib ; \
            install -m 644 $^ $(DESTDIR)$(PREFIX)/lib
```

tools/lkl/Targets 수정
```sh
# progs-y += tests/net-test
```

tools/lkl/Targets 수정
```
LDLIBS_lib/hijack/liblkl-hijack-$(LKL_HOST_CONFIG_ANDROID) += -lgcc -lc
#LDLIBS_lib/hijack/liblkl-hijack-$(LKL_HOST_CONFIG_ARM) += -lgcc -lc
#LDLIBS_lib/hijack/liblkl-hijack-$(LKL_HOST_CONFIG_AARCH64) += -lc
#LDLIBS_lib/hijack/liblkl-hijack-$(LKL_HOST_CONFIG_I386) += -lc_nonshared
```

tools/lkl/Makefile 수정
```sh
headers_install: $(TARGETS)
        $(call QUIET_INSTALL, headers) \
            install -d $(DESTDIR)$(PREFIX)/include ; \
            install -m 644 include/lkl.h include/lkl_host.h $(OUTPUT)include/lkl_autoconf.h \
              include/lkl_config.h $(DESTDIR)$(PREFIX)/include ; \
            cp -r $(OUTPUT)include/lkl $(DESTDIR)$(PREFIX)/include
```

grep떠보니 *hijack* 파일이 없어서 그런가..? 싶음

`tools/lkl/Targets`에서 마지막줄(tests/net-test) 주석 해제해봄

컴파일...

안댐

`tools/lkl/lib/Build` 파일 원상복구 (주석 해제)

zjavkdlf..

Error

```sh
root@system:~/porting/janus# ./lkl/tools/lkl/btrfs-fsfuzz -t btrfs -p -i ./samples/evaluation/btrfs-00.image
[    0.000000] Linux version 5.2.0 (root@system) (gcc version 5.4.0 20160609 (Ubuntu 5.4.0-6ubuntu1~16.04.11)) #22 Fri Sep 6 21:47:00 KST 2019
[    0.000000] memblock address range: 0x7f1cf8000000 - 0x7f1cfffff000
[    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 32319
[    0.000000] Kernel command line: mem=128M virtio_mmio.device=316@0x1000000:1
[    0.000000] Dentry cache hash table entries: 16384 (order: 5, 131072 bytes)
[    0.000000] Inode-cache hash table entries: 8192 (order: 4, 65536 bytes)
[    0.000000] Memory available: 129044k/131068k RAM
[    0.000000] SLUB: HWalign=32, Order=0-3, MinObjects=0, CPUs=1, Nodes=1
[    0.000000] NR_IRQS: 4096
[    0.000000] lkl: irqs initialized
[    0.000000] clocksource: lkl: mask: 0xffffffffffffffff max_cycles: 0x1cd42e4dffb, max_idle_ns: 881590591483 ns
[    0.000005] lkl: time and timers initialized (irq2)
[    0.000023] pid_max: default: 4096 minimum: 301
[    0.000110] Mount-cache hash table entries: 512 (order: 0, 4096 bytes)
[    0.000127] Mountpoint-cache hash table entries: 512 (order: 0, 4096 bytes)
[    0.032490] random: get_random_bytes called from _etext+0xbca4/0x147cb with crng_init=0
[    0.035327] printk: console [lkl_console0] enabled
[    0.035471] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff,max_idle_ns: 19112604462750000 ns
[    0.035544] xor: automatically using best checksumming function   8regs
[    0.038392] NET: Registered protocol family 16
[    0.241403] raid6: int64x8  gen()  5333 MB/s
[    0.427856] raid6: int64x8  xor()  3732 MB/s
[    0.613397] raid6: int64x4  gen()  6510 MB/s
[    0.799846] raid6: int64x4  xor()  4372 MB/s
[    0.985395] raid6: int64x2  gen()  6768 MB/s
[    1.171953] raid6: int64x2  xor()  4411 MB/s
[    1.358456] raid6: int64x1  gen()  6105 MB/s
[    1.544949] raid6: int64x1  xor()  3383 MB/s
[    1.544967] raid6: using algorithm int64x2 gen() 6768 MB/s
[    1.544969] raid6: .... xor() 4411 MB/s, rmw enabled
[    1.544978] raid6: using intx1 recovery algorithm
[    1.545078] clocksource: Switched to clocksource lkl
[    1.545179] NET: Registered protocol family 2
[    1.549328] tcp_listen_portaddr_hash hash table entries: 256 (order: 0, 4096 bytes)
[    1.549359] TCP established hash table entries: 1024 (order: 1, 8192 bytes)
[    1.549364] TCP bind hash table entries: 1024 (order: 1, 8192 bytes)
[    1.549369] TCP: Hash tables configured (established 1024 bind 1024)
[    1.550155] UDP hash table entries: 128 (order: 0, 4096 bytes)
[    1.550175] UDP-Lite hash table entries: 128 (order: 0, 4096 bytes)
[    1.550257] virtio-mmio: Registering device virtio-mmio.0 at 0x1000000-0x100013b, IRQ 1.
[    1.550881] workingset: timestamp_bits=62 max_order=16 bucket_order=0
[    1.552311] io scheduler mq-deadline registered
[    1.552328] io scheduler kyber registered
[    1.552353] virtio-mmio virtio-mmio.0: Failed to enable 64-bit or 32-bit DMA.  Trying to continue, but this might not work.
[    1.554634] virtio_blk virtio0: [vda] 0 512-byte logical blocks (0 B/0 B)
[    1.555980] NET: Registered protocol family 10
[    1.558811] Segment Routing with IPv6
[    1.558837] sit: IPv6, IPv4 and MPLS over IPv4 tunneling driver
[    1.562452] Btrfs loaded, crc32c=crc32c-generic
[    1.562770] Warning: unable to open an initial console.
[    1.562795] This architecture does not have kernel memory protection.
[    1.562797] Run /init as init process
can't mount disk: Invalid argument
[    1.565443] reboot: Restarting system
root@system:~/porting/janus#
```

TEST SCRIPT
`./lkl/tools/lkl/btrfs-fsfuzz -t btrfs -p -i ./samples/evaluation/btrfs-00.image`

<!--![nu](https://nonetype.github.io/assets/nu.jpg)-->
# 0908 JANUS Porting

swapon /swap/swapfile 안해주면 gcc internal error 뜬다. 꼭 swapfile 등록해주자.

JANUS 테스트 스크립트 실행 결과
```sh
root@system:~/fuzzer/janus# ./lkl/tools/lkl/btrfs-fsfuzz -t btrfs -p -i ./samples/evaluation/btrfs-00.image
[    0.000000] Linux version 4.18.0-rc1 (root@system) (gcc version 5.4.0 20160609 (Ubuntu 5.4.0-6ubuntu1~16.04.11)) #2 Fri Aug 23 19:59:15 KST 2019
[    0.000000] bootmem address range: 0x7f5090000000 - 0x7f5097fff000
[    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 32319
[    0.000000] Kernel command line: mem=128M virtio_mmio.device=292@0x1000000:1
[    0.000000] Dentry cache hash table entries: 16384 (order: 5, 131072 bytes)
[    0.000000] Inode-cache hash table entries: 8192 (order: 4, 65536 bytes)
[    0.000000] Memory available: 129040k/131068k RAM
[    0.000000] SLUB: HWalign=32, Order=0-3, MinObjects=0, CPUs=1, Nodes=1
[    0.000000] NR_IRQS: 4096
[    0.000000] lkl: irqs initialized
[    0.000000] clocksource: lkl: mask: 0xffffffffffffffff max_cycles: 0x1cd42e4dffb, max_idle_ns: 881590591483 ns
[    0.000003] lkl: time and timers initialized (irq2)
[    0.000016] pid_max: default: 4096 minimum: 301
[    0.000106] Mount-cache hash table entries: 512 (order: 0, 4096 bytes)
[    0.000120] Mountpoint-cache hash table entries: 512 (order: 0, 4096 bytes)
[    0.006688] console [lkl_console0] enabled
[    0.006744] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff,max_idle_ns: 19112604462750000 ns
[    0.006765] xor: automatically using best checksumming function   8regs
[    0.008913] raid6: using algorithm int64x2 gen() 0 MB/s
[    0.008966] raid6: .... xor() 0 MB/s, rmw enabled
[    0.008979] raid6: using intx1 recovery algorithm
[    0.009015] clocksource: Switched to clocksource lkl
[    0.009256] virtio-mmio: Registering device virtio-mmio.0 at 0x1000000-0x1000123, IRQ 1.
[    0.010137] workingset: timestamp_bits=62 max_order=15 bucket_order=0
[    0.012157] io scheduler noop registered
[    0.012284] io scheduler cfq registered (default)
[    0.012349] virtio-mmio virtio-mmio.0: Failed to enable 64-bit or 32-bit DMA.  Trying to continue, but this might not work.
[    0.012684] virtio_blk virtio0: [vda] 262144 512-byte logical blocks (134 MB/128 MiB)
[    0.014960] random: get_random_bytes called from init_oops_id+0x35/0x40 with crng_init=0
[    0.015723] Btrfs loaded, crc32c=crc32c-generic
[    0.016171] Warning: unable to open an initial console.
[    0.016251] This architecture does not have kernel memory protection.
[    0.018428] BTRFS: device fsid a62e00e8-e94e-4200-8217-12444de93c2e devid 1 transid 8 /dev/0000fe00
[    0.019802] BTRFS info (device vda): disk space caching is enabled
[    0.019854] BTRFS info (device vda): has skinny extents
[    0.079222] reboot: Restarting system
root@system:~/fuzzer/janus#
```
original lkl 로그는 NET, raid6 등등이 추가되어 있음.
삭제해야 timeout이 안뜰듯

삭제
```sh
include/lkl/autoconf.h:74:#define LKL_CONFIG_RAID6_PQ 1
include/lkl/autoconf.h:93:#define LKL_CONFIG_RAID6_PQ_BENCHMARK 1
```

수정
```sh
root@system:~/fuzzer/janus/lkl/tools/lkl/lib/hijack# diff ~/ol/tools/build/Makefile ~/pl/tools/build/Makefile
46c46
< 	$(QUIET_LINK)$(HOSTCC) $(LDFLAGS) -o $@ $<
---
> 	$(QUIET_LINK)$(HOSTCC) $(KBUILD_HOSTLDFLAGS) -o $@ $<
root@system:~/fuzzer/janus/lkl/tools/lkl/lib/hijack#
```

BACKUP
```sh
lib/config.c: In function ‘add_neighbor’:
lib/config.c:385:9: warning: implicit declaration of function ‘inet_pton’ [-Wimplicit-function-declaration]
   ret = inet_pton(LKL_AF_INET, ip, ip_addr);
         ^
  LD       /home/fuzzer/porting/janus/lkl/tools/lkl/lib/liblkl-in.o
(cd ../../../core/afl-image/llvm_mode && env -u CPP -u CC -u MAKEFLAGS -u LDFLAGS LLVM_CONFIG=llvm-config make)
make[1]: Entering directory '/home/fuzzer/porting/janus/core/afl-image/llvm_mode'
[*] Checking for working 'llvm-config'...
[*] Checking for working 'clang'...
[*] Checking for '../afl-showmap'...
[+] All set and ready to build.
make[1]: Leaving directory '/home/fuzzer/porting/janus/core/afl-image/llvm_mode'
cp -f ../../../core/afl-image/kafl-llvm-rt.o /home/fuzzer/porting/janus/lkl/tools/lkl/kafl-llvm-rt.o
  AR       /home/fuzzer/porting/janus/lkl/tools/lkl/liblkl.a
  LINK     /home/fuzzer/porting/janus/lkl/tools/lkl/lklfuse
/home/fuzzer/porting/janus/lkl/tools/lkl/liblkl.a(liblkl-in.o): In function `lkl_config_netdev_create':
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:475: undefined reference to `lkl_netdev_tap_create'
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:524: undefined reference to `lkl_netdev_add'
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:502: undefined reference to `lkl_netdev_raw_create'
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:480: undefined reference to `lkl_netdev_tap_create'
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:482: undefined reference to `lkl_netdev_macvtap_create'
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:488: undefined reference to `lkl_netdev_pipe_create'
/home/fuzzer/porting/janus/lkl/tools/lkl/liblkl.a(liblkl-in.o): In function `lkl_config_netdev_configure':
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:549: undefined reference to `lkl_netdev_get_ifindex'
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:551: undefined reference to `lkl_if_up'
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:561: undefined reference to `lkl_if_set_mtu'
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:591: undefined reference to `lkl_if_set_ipv4_gateway'
/home/fuzzer/porting/janus/lkl/tools/lkl/liblkl.a(liblkl-in.o): In function `add_neighbor':
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:400: undefined reference to `lkl_add_neighbor'
/home/fuzzer/porting/janus/lkl/tools/lkl/liblkl.a(liblkl-in.o): In function `lkl_config_netdev_configure':
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:638: undefined reference to `lkl_qdisc_parse_add'
/home/fuzzer/porting/janus/lkl/tools/lkl/liblkl.a(liblkl-in.o): In function `lkl_load_config_post':
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:733: undefined reference to `lkl_set_ipv4_gateway'
/home/fuzzer/porting/janus/lkl/tools/lkl/liblkl.a(liblkl-in.o): In function `lkl_config_netdev_configure':
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:611: undefined reference to `lkl_if_set_ipv6'
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:577: undefined reference to `lkl_if_set_ipv4'
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:624: undefined reference to `lkl_if_set_ipv6_gateway'
/home/fuzzer/porting/janus/lkl/tools/lkl/liblkl.a(liblkl-in.o): In function `lkl_load_config_post':
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:746: undefined reference to `lkl_set_ipv6_gateway'
/home/fuzzer/porting/janus/lkl/tools/lkl/liblkl.a(liblkl-in.o): In function `lkl_unload_config':
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:783: undefined reference to `lkl_netdev_remove'
/home/fuzzer/porting/janus/lkl/tools/lkl/lib/config.c:784: undefined reference to `lkl_netdev_free'
collect2: error: ld returned 1 exit status
Makefile:116: recipe for target '/home/fuzzer/porting/janus/lkl/tools/lkl/lklfuse' failed
make: *** [/home/fuzzer/porting/janus/lkl/tools/lkl/lklfuse] Error 1
make: Leaving directory '/home/fuzzer/porting/janus/lkl/tools/lkl'
cp: cannot stat 'tools/lkl/fsfuzz': No such file or directory
cp: cannot stat 'tools/lkl/executor': No such file or directory
cp: cannot stat 'tools/lkl/combined': No such file or directory
root@system:~/porting/janus/lkl#
```
