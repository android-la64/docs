# 1. 代码

bionic代码位于 `$ANDROID_BUILD_TOP/bionic` 目录下。



# 2. 移植概述

此部分主要有龙芯的同事完成，熵核辅助。bionic的移植主要有5部分内容，分别是libc/kernel， libc/arch， libc，libm，linker。

### 2.1 Kernel

确定kernel的版本，将部分头文件移植到libc目录下，并且需要将libc下的部分头文件内容跟对应的kernel版本匹配。



### 2.2 libc/arch

位于目录`libc/arch-loongarch64`，这个名字必须跟其他aosp、clang里面的名字保持一致。主要涉及到的问题如下：

```bash
 libc/arch-loongarch64/bionic/__bionic_clone.S
 libc/arch-loongarch64/bionic/__set_tls.c
 libc/arch-loongarch64/bionic/_exit_with_stack_teardown.S
 libc/arch-loongarch64/bionic/renameat.cpp
 libc/arch-loongarch64/bionic/setjmp.S
 libc/arch-loongarch64/bionic/syscall.S
 libc/arch-loongarch64/bionic/vfork.S
 libc/arch-loongarch64/string/memset_chk.c
```

这里的大部分内容可以从libc或者其他库中移植过来。



### 2.3 libc部分关键实现

这部分移植主要是在在公共文件中增加 loongson分支。主要待移植的文件有：

```
 libc/arch-common/bionic/crtbegin.c
 libc/bionic/android_profiling_dynamic.cpp
 libc/bionic/android_unsafe_frame_pointer_chase.cpp
 libc/bionic/bionic_elf_tls.cpp
 libc/bionic/clone.cpp
 libc/bionic/libc_init_static.cpp
 libc/bionic/poll.cpp
 libc/bionic/signal.cpp
 libc/bionic/sigprocmask.cpp
 libc/bionic/spawn.cpp
 libc/bionic/sys_epoll.cpp
 libc/bionic/sys_signalfd.cpp
 libc/include/bits/elf_loongarch64.h
 libc/include/bits/fenv_loongarch64.h
 libc/include/bits/fortify/stdio.h
 libc/include/bits/fortify/string.h
 libc/include/bits/glibc-syscalls.h
 libc/include/elf.h
 libc/include/fenv.h
 libc/include/setjmp.h
 libc/include/sys/cdefs.h
 libc/include/sys/stat.h
 libc/include/sys/ucontext.h
 libc/include/sys/user.h
```

这部分的关键有两点：

1. 正确性
2. 可移植 - 也就是说不能破坏其他Arch的代码，否则进一步移植最新版本会比较费事。



### 2.4 libm

这部分移植的代码位于libm目录。这部分初期移植可以仅仅考虑功能，后续再进一步移植各个Intrinsic

```
libm/builtins.cpp
libm/loongarch64/fenv.c
```



### 2.5 linker

这部分移植的代码位于linker目录，主要涉及到如下文件：

```
linker/Android.bp
linker/arch/loongarch64/begin.S
linker/linker.cpp
linker/linker_main.cpp
linker/linker_phdr.cpp
linker/linker_relocate.cpp
linker/linker_relocs.h
```



# 3. 编译与测试

### 3.1 测试用例编译

```bash
## 编译测试用例
$ . art/xc_tools/bionic_g_b.sh 


## 测试
$ . art/xc_tools/bionic_g_r.sh 
```

其中`bionic_g_b.sh `内容如下

```bash
m bionic-unit-tests bionic-unit-tests-static linker-unit-tests malloc_debug_unit_tests malloc_debug_system_tests malloc_hooks_system_tests fdtrack_test memunreachable_test  memunreachable_binder_test memunreachable_unit_test
```



其中`bionic_g_r.sh `内容如下

```bash
#!/bin/bash

########################################
# Run Bionic functional tests
########################################

## Makesure you in adb root status
# adb root

## create test directory
export BIONIC_TEST_DIR="/data/local/bionic-tests/data"

# out_dir="${OUT_DIR:=out}"
log_dir="./native-test-log"

## upload tests
adb shell "mkdir -p $BIONIC_TEST_DIR"
adb push ${OUT}/data/nativetest64 "$BIONIC_TEST_DIR"

if [[ ! -d $log_dir ]]; then
  mkdir -p $log_dir
fi

## run tests
all_tests="bionic-unit-tests"
all_tests+=" bionic-unit-tests-static"
all_tests+=" linker-unit-tests"
#all_tests+=" bionic-fortify-runtime-asan-test"
all_tests+=" malloc_debug_unit_tests"
all_tests+=" malloc_debug_system_tests"
all_tests+=" malloc_hooks_system_tests"
all_tests+=" fdtrack_test/fdtrack_test"
all_tests+=" memunreachable_test"
all_tests+=" memunreachable_binder_test"
all_tests+=" memunreachable_unit_test"

time_stamp=`date +%m%d_%H%M%S`
idx=1
for test in $all_tests; do
  adb shell "${BIONIC_TEST_DIR}/nativetest64/$test/$test" | tee "$log_dir/${time_stamp}_gtest_${idx}_$test.log"
  idx=$(( $idx + 1 ))
done
```



### 3.3 测试结果

```bash
[  SLOW    ] time.clock (5007 ms, exceeded 2000 ms)
[  FAILED  ] 4 tests, listed below:
[  FAILED  ] fenv.feenableexcept_fegetexcept
[  FAILED  ] sys_ptrace.watchpoint_stress
[  FAILED  ] sys_ptrace.watchpoint_imprecise
[  FAILED  ] sys_ptrace.hardware_breakpoint

14 SLOW TESTS
 4 FAILED TESTS
 
 ## 除了上述4个测试用例FAIL之外，其他的都正确
```


