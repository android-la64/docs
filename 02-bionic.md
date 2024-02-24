# 1. 代码

bionic代码位于 `$ANDROID_BUILD_TOP/bionic`





# 2. 移植概述

### 2.1 Kernel

确定kernel的版本，将部分头文件移植到libc目录下，并且需要将libc下的部分头文件内容跟对应的kernel版本匹配。



### 2.2 libc

位于目录`libc/arch-riscv64`，这个名字必须跟其他aosp、clang里面的名字保持一致

```bash
libc/arch-common/bionic/crtbegin.c
libc/arch-riscv64/bionic/__bionic_clone.S
libc/arch-riscv64/bionic/__set_tls.c
libc/arch-riscv64/bionic/_exit_with_stack_teardown.S
libc/arch-riscv64/bionic/renameat.cpp
libc/arch-riscv64/bionic/setjmp.S
libc/arch-riscv64/bionic/syscall.S
libc/arch-riscv64/bionic/vfork.S
libc/arch-riscv64/dynamic_function_dispatch.cpp
libc/arch-riscv64/generic/bionic/memcmp.S
libc/arch-riscv64/generic/bionic/memcpy.S
libc/arch-riscv64/generic/bionic/memset.S
libc/arch-riscv64/generic/bionic/memset_chk.c
libc/arch-riscv64/static_function_dispatch.S
```

这里的大部分内容可以从libc或者其他库中移过来。



### 2.3 linker

这部分是重点，也是容易出错的地方。请参考RISC-V的移植。





### 2.4 原子操作等

这部分也是容易出错的地方。





# 3. 编译与测试

### 3.1 Host

不支持这种模式。

```bash
$ ./tests/run-on-host.sh 64
./tests/run-on-host.sh not supported on TARGET_ARCH=riscv64
```



### 3.2 Device - 功能测试

```bash
## 编译测试用例
$ cd $ANDROID_BUILD_TOP/bionic
$ m bionic-unit-tests bionic-unit-tests-static linker-unit-tests bionic-fortify-runtime-asan-test malloc_debug_unit_tests malloc_debug_system_tests malloc_hooks_system_tests fdtrack_test memunreachable_test  memunreachable_binder_test memunreachable_unit_test


## 测试
## 以下的测试也在脚本scripts/bionic-test.sh
source  bionic_func_tests.sh
```

` bionic_func_tests.sh `脚本的内容如下：

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
log_dir="./bionic-test-log"

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
all_tests+=" bionic-fortify-runtime-asan-test"
all_tests+=" malloc_debug_unit_tests"
all_tests+=" malloc_debug_system_tests"
all_tests+=" malloc_hooks_system_tests"
all_tests+=" fdtrack_test/fdtrack_test"
all_tests+=" memunreachable_test"
all_tests+=" memunreachable_binder_test"
all_tests+=" memunreachable_unit_test"

# 以下这个测试时间非常长，没有放入上面的测试列表
# bionic-stress-tests

time_stamp=`date +%m%d_%H%M%S`
idx=1
for test in $all_tests; do
  adb shell "${BIONIC_TEST_DIR}/$test/$test" | tee "$log_dir/${time_stamp}_gtest_${idx}_$test.log"
  idx=$(( $idx + 1 ))
done
```



### 3.3 测试结果

【20240224】测试，有不少FAILED，需要检查。

```
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] fenv.feenableexcept_fegetexcept (104 ms)
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] setjmp.setjmp_signal_mask (93 ms)
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] setjmp.sigsetjmp_1_signal_mask (90 ms)
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] stack_unwinding.unwind_through_signal_frame (89 ms)
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] stack_unwinding.unwind_through_signal_frame_SA_SIGINFO (89 ms)
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] unistd.exec_argv0_null (281 ms)
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] sys_ptrace.watchpoint_stress (127 ms)
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] sys_ptrace.watchpoint_imprecise (104 ms)
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] sys_ptrace.hardware_breakpoint (102 ms)
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] unistd_nofortify.exec_argv0_null (272 ms)
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] dlfcn.dlopen_invalid_rw_load_segment (95 ms)
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] dlfcn.dlopen_invalid_unaligned_shdr_offset (94 ms)
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] dlfcn.dlopen_invalid_zero_shentsize (93 ms)
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] dlfcn.dlopen_invalid_zero_shstrndx (93 ms)
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] dlfcn.dlopen_invalid_empty_shdr_table (97 ms)
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] dlfcn.dlopen_invalid_zero_shdr_table_offset (94 ms)
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] dlfcn.dlopen_invalid_zero_shdr_table_content (93 ms)
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] dlfcn.dlopen_invalid_textrels (94 ms)
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] dlfcn.dlopen_invalid_textrels2 (94 ms)
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] 19 tests, listed below:
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] fenv.feenableexcept_fegetexcept
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] setjmp.setjmp_signal_mask
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] setjmp.sigsetjmp_1_signal_mask
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] stack_unwinding.unwind_through_signal_frame
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] stack_unwinding.unwind_through_signal_frame_SA_SIGINFO
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] unistd.exec_argv0_null
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] sys_ptrace.watchpoint_stress
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] sys_ptrace.watchpoint_imprecise
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] sys_ptrace.hardware_breakpoint
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] unistd_nofortify.exec_argv0_null
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] dlfcn.dlopen_invalid_rw_load_segment
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] dlfcn.dlopen_invalid_unaligned_shdr_offset
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] dlfcn.dlopen_invalid_zero_shentsize
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] dlfcn.dlopen_invalid_zero_shstrndx
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] dlfcn.dlopen_invalid_empty_shdr_table
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] dlfcn.dlopen_invalid_zero_shdr_table_offset
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] dlfcn.dlopen_invalid_zero_shdr_table_content
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] dlfcn.dlopen_invalid_textrels
0224_042815_gtest_1_bionic-unit-tests.log:[  FAILED  ] dlfcn.dlopen_invalid_textrels2
0224_042815_gtest_1_bionic-unit-tests.log:19 FAILED TESTS
0224_042815_gtest_2_bionic-unit-tests-static.log:[  FAILED  ] fenv.feenableexcept_fegetexcept (89 ms)
0224_042815_gtest_2_bionic-unit-tests-static.log:[  FAILED  ] setjmp.setjmp_signal_mask (75 ms)
0224_042815_gtest_2_bionic-unit-tests-static.log:[  FAILED  ] setjmp.sigsetjmp_1_signal_mask (74 ms)
0224_042815_gtest_2_bionic-unit-tests-static.log:[  FAILED  ] stack_unwinding.unwind_through_signal_frame (73 ms)
0224_042815_gtest_2_bionic-unit-tests-static.log:[  FAILED  ] stack_unwinding.unwind_through_signal_frame_SA_SIGINFO (74 ms)
0224_042815_gtest_2_bionic-unit-tests-static.log:[  FAILED  ] unistd.exec_argv0_null (243 ms)
0224_042815_gtest_2_bionic-unit-tests-static.log:[  FAILED  ] sys_ptrace.watchpoint_stress (109 ms)
0224_042815_gtest_2_bionic-unit-tests-static.log:[  FAILED  ] sys_ptrace.watchpoint_imprecise (87 ms)
0224_042815_gtest_2_bionic-unit-tests-static.log:[  FAILED  ] sys_ptrace.hardware_breakpoint (88 ms)
0224_042815_gtest_2_bionic-unit-tests-static.log:[  FAILED  ] unistd_nofortify.exec_argv0_null (252 ms)
0224_042815_gtest_2_bionic-unit-tests-static.log:[  FAILED  ] 10 tests, listed below:
0224_042815_gtest_2_bionic-unit-tests-static.log:[  FAILED  ] fenv.feenableexcept_fegetexcept
0224_042815_gtest_2_bionic-unit-tests-static.log:[  FAILED  ] setjmp.setjmp_signal_mask
0224_042815_gtest_2_bionic-unit-tests-static.log:[  FAILED  ] setjmp.sigsetjmp_1_signal_mask
0224_042815_gtest_2_bionic-unit-tests-static.log:[  FAILED  ] stack_unwinding.unwind_through_signal_frame
0224_042815_gtest_2_bionic-unit-tests-static.log:[  FAILED  ] stack_unwinding.unwind_through_signal_frame_SA_SIGINFO
0224_042815_gtest_2_bionic-unit-tests-static.log:[  FAILED  ] unistd.exec_argv0_null
0224_042815_gtest_2_bionic-unit-tests-static.log:[  FAILED  ] sys_ptrace.watchpoint_stress
0224_042815_gtest_2_bionic-unit-tests-static.log:[  FAILED  ] sys_ptrace.watchpoint_imprecise
0224_042815_gtest_2_bionic-unit-tests-static.log:[  FAILED  ] sys_ptrace.hardware_breakpoint
0224_042815_gtest_2_bionic-unit-tests-static.log:[  FAILED  ] unistd_nofortify.exec_argv0_null
0224_042815_gtest_2_bionic-unit-tests-static.log:10 FAILED TESTS
0224_042815_gtest_7_malloc_hooks_system_tests.log:[  FAILED  ] MallocHooksTest.DISABLED_aligned_alloc_hook_error (28 ms)
0224_042815_gtest_7_malloc_hooks_system_tests.log:[  FAILED  ] 1 test, listed below:
0224_042815_gtest_7_malloc_hooks_system_tests.log:[  FAILED  ] MallocHooksTest.DISABLED_aligned_alloc_hook_error
0224_042815_gtest_7_malloc_hooks_system_tests.log: 1 FAILED TEST
0224_042815_gtest_7_malloc_hooks_system_tests.log:[  FAILED  ] MallocHooksTest.aligned_alloc_hook_error (635 ms)
0224_042815_gtest_7_malloc_hooks_system_tests.log:[  FAILED  ] 1 test, listed below:
0224_042815_gtest_7_malloc_hooks_system_tests.log:[  FAILED  ] MallocHooksTest.aligned_alloc_hook_error
0224_042815_gtest_7_malloc_hooks_system_tests.log: 1 FAILED TEST

```




### 3.4 libm Clang builtin测试
```bash
cd $ANDROID_BUILD_TOP
m libm-builtin-tests
python $ANDROID_BUILD_TOP/bionic/tests/check_libm_builtin.py
```

测试结果：
```text
PASS: 24 FAIL: 0  ALL: 24
```



# 4. Device - 性能测试

性能测试只能在Device上运行

## 4.1 benchmark测试

```bash
## 编译测试用例
$ cd $ANDROID_BUILD_TOP/bionic
$ m bionic-benchmarks  bionic-benchmarks-static


## 测试例子
$ adb shell "/data/benchmarktest64/bionic-benchmarks-static/bionic-benchmarks-static --benchmark_filter=BM_stdlib_strtoll --benchmark_repetitions=4"
## 2023-07-02T02:21:39+00:00
## Running /data/benchmarktest64/bionic-benchmarks-static/bionic-benchmarks-static
## Run on (4 X 1848 MHz CPU s)
## -------------------------------------------------------------------
## Benchmark                         Time             CPU   Iterations
## -------------------------------------------------------------------
## BM_stdlib_strtoll              60.4 ns         60.4 ns     11527124
## BM_stdlib_strtoll              54.8 ns         54.8 ns     11527124
## BM_stdlib_strtoll              54.2 ns         54.2 ns     11527124
## BM_stdlib_strtoll              54.2 ns         54.2 ns     11527124
## BM_stdlib_strtoll_mean         55.9 ns         55.9 ns            4
## BM_stdlib_strtoll_median       54.5 ns         54.5 ns            4
## BM_stdlib_strtoll_stddev       2.99 ns         3.00 ns            4

```





# 5. 性能对比

请参考对应的Excel表格。



# 6. 移植步骤

### 6.1 确定版本

```bash
## 以这个版本为基础
$ git co android-12.0.0_r1
$ git br a12_larch

```



### 6.2 初始patch

```bash
8a7f8750c : 2023-10-20 10:45:02 +0000 : caiyinyu : caiyinyu@loongson.cn : fix 0161-riscv64-syscall-stub-and-seccomp-filter-generation.patch
c33636ed7 : 2023-10-20 10:45:02 +0000 : caiyinyu : caiyinyu@loongson.cn : fix bug in generate_uapi_headers.sh
6df30ca00 : 2023-10-20 10:44:36 +0000 : caiyinyu : caiyinyu@loongson.cn : 0010-riscv64-increase-jmp_buf-size.patch
49838b75e : 2023-10-20 10:44:36 +0000 : caiyinyu : caiyinyu@loongson.cn : 0016-riscv64-inline-raise.patch
f67c7e460 : 2023-10-20 10:44:36 +0000 : caiyinyu : caiyinyu@loongson.cn : 0024-De-pessimize-SigSetConverter-usage.patch
ffb590b21 : 2023-10-20 10:44:36 +0000 : caiyinyu : caiyinyu@loongson.cn : correct libc/tools/gensyscalls.py
4b2481f2a : 2023-10-20 10:44:36 +0000 : caiyinyu : caiyinyu@loongson.cn : 0067-riscv64-typedef-struct-ucontext-.-ucontext_t.patch
81e072f7f : 2023-10-20 10:44:36 +0000 : caiyinyu : caiyinyu@loongson.cn : 0081-Guard-registers-definitions-for-riscv64.patch
f72a60e94 : 2023-10-20 10:44:36 +0000 : caiyinyu : caiyinyu@loongson.cn : 0089-Remove-obsolete-hacks-for-the-fabs-family.patch
e52d10b80 : 2023-10-20 10:44:36 +0000 : caiyinyu : caiyinyu@loongson.cn : 0101-Update-sys_ptrace_test.cpp-for-riscv64.patch 0098-Add-riscv64-support-to-the-linker-relocation-benchma.patch
835bb3c06 : 2023-10-20 10:44:36 +0000 : caiyinyu : caiyinyu@loongson.cn : tmp-import-riscv/0104-Simplify-the-malloc_debug-unwind.patch
cf1d09fc6 : 2023-10-20 10:44:36 +0000 : caiyinyu : caiyinyu@loongson.cn : tmp-import-riscv/0105-Use-the-same-union-in-riscv64-s-ucontext.patch
d2e97ce7b : 2023-10-20 10:44:36 +0000 : caiyinyu : caiyinyu@loongson.cn : tmp-import-riscv/0124-riscv64-build-the-linker.patch
2c3bed714 : 2023-10-20 10:44:36 +0000 : caiyinyu : caiyinyu@loongson.cn : tmp-import-riscv/0125-Punch-a-hole-for-renameat-2-for-app-compat.patch
911513259 : 2023-10-20 10:44:36 +0000 : caiyinyu : caiyinyu@loongson.cn : tmp-import-riscv/0128-Add-a-zip-package-containing-the-crt-.o-objects.patch
c8f71e76a : 2023-10-20 10:44:36 +0000 : caiyinyu : caiyinyu@loongson.cn : tmp-import-riscv/0130-Add-riscv64-kernel-headers-to-the-ndk-sysroot.patch
a151b4850 : 2023-10-20 10:44:36 +0000 : caiyinyu : caiyinyu@loongson.cn : tmp-import-riscv/0149-Add-riscv64-crtbegin-assembler.patch
67651bc28 : 2023-10-20 10:44:36 +0000 : caiyinyu : caiyinyu@loongson.cn : tmp-import-riscv/0150-riscv64-enable-the-version-scripts.patch
17e1e20a2 : 2023-10-20 10:44:36 +0000 : caiyinyu : caiyinyu@loongson.cn : tmp-import-riscv/0157-riscv64-add-bionic-assembler-and-string-functions.patch
4d5945ece : 2023-10-20 10:44:36 +0000 : caiyinyu : caiyinyu@loongson.cn : tmp-import-riscv/0158-riscv64-fenv-implementation.patch
1d4213607 : 2023-10-20 10:44:36 +0000 : caiyinyu : caiyinyu@loongson.cn : tmp-import-riscv/0161-riscv64-syscall-stub-and-seccomp-filter-generation.patch
8a3d99f61 : 2023-10-20 10:44:36 +0000 : caiyinyu : caiyinyu@loongson.cn : tmp-import-riscv/0169-Build-libdl-for-risc-v.patch
cda00f16a : 2023-10-20 10:44:36 +0000 : caiyinyu : caiyinyu@loongson.cn : tmp-import-riscv/0177-riscv64-more-sys-ucontext.h.patc
6bab83962 : 2023-10-20 10:44:35 +0000 : caiyinyu : caiyinyu@loongson.cn : tmp-import-riscv/0178-riscv64-add-private-bionic_asm.h.patch
4c4f9eb74 : 2023-10-20 10:44:35 +0000 : caiyinyu : caiyinyu@loongson.cn : tmp-import-riscv/0182-Add-riscv64-ucontext.h.patch
45edbc7cd : 2023-10-20 10:44:35 +0000 : caiyinyu : caiyinyu@loongson.cn : tmp-import-riscv/0185-riscv64-TLS-support.patch
9f738d55f : 2023-10-20 10:44:35 +0000 : caiyinyu : caiyinyu@loongson.cn : tmp-import-riscv/0196-riscv64-BIONIC_STOP_UNWIND.patch
04ecf8741 : 2023-10-20 10:44:35 +0000 : caiyinyu : caiyinyu@loongson.cn : 0201-Pull-in-the-riscv64-uapi-headers-for-riscv64-builds.patch
c38deaa2a : 2023-10-20 10:44:35 +0000 : caiyinyu : caiyinyu@loongson.cn : 0202-riscv64-__get_tls.patch
fb1778298 : 2023-10-20 10:44:35 +0000 : caiyinyu : caiyinyu@loongson.cn : 0204-riscv64-fenv.h.patch
87c210660 : 2023-10-20 10:44:35 +0000 : caiyinyu : caiyinyu@loongson.cn : 0205-riscv64-sys-user.h.patch
857c697fe : 2023-10-20 10:44:35 +0000 : caiyinyu : caiyinyu@loongson.cn : 0208-Switch-to-FreeBSD-s-elf_common.h.patch
ea7a9dd45 : 2023-10-20 10:44:35 +0000 : caiyinyu : caiyinyu@loongson.cn : 0214-Add-__tls_get_addr-for-riscv64.patch
d06d7bddd : 2023-10-20 10:44:35 +0000 : caiyinyu : caiyinyu@loongson.cn : 0218-riscv64-s-struct-stat-is-the-same-as-arm64-s.patch
e6081860f : 2023-10-20 10:44:35 +0000 : caiyinyu : caiyinyu@loongson.cn : 0229-s-riscv-riscv64.patch
1f0f540e9 : 2023-10-20 10:44:35 +0000 : caiyinyu : caiyinyu@loongson.cn : 0235-Add-riscv64-to-the-map-files.patch
27d921387 : 2023-10-20 10:44:35 +0000 : caiyinyu : caiyinyu@loongson.cn : 0236-Tell-the-version-script-generation-script-about-risc.patch
604daeee7 : 2023-10-20 10:44:35 +0000 : caiyinyu : caiyinyu@loongson.cn : 0240-Add-risc-v-support-to-versioner.patch
bf6fc59f8 : 2023-10-20 10:44:35 +0000 : caiyinyu : caiyinyu@loongson.cn : fix __bionic_clone.S
5430ff2ce : 2023-10-20 10:44:35 +0000 : caiyinyu : caiyinyu@loongson.cn : add files in libc/bionic/ and modify related file
2abeba135 : 2023-10-20 10:44:35 +0000 : caiyinyu : caiyinyu@loongson.cn : __bionic_clone.S
fa7163f17 : 2023-10-20 10:44:35 +0000 : caiyinyu : caiyinyu@loongson.cn : .bp file and linker_relocs.h
e1c42ccac : 2021-07-24 03:01:23 +0000 : Android Build Coastguard Worker : android-build-coastguard-worker@google.com : Snap for 7579381 
```





# 可能存在的问题



```C++
--- a/libc/include/setjmp.h
+++ b/libc/include/setjmp.h
@@ -45,7 +45,7 @@
 #if defined(__aarch64__)
 #define _JBLEN 32
 #if defined(__loongarch__)
-#define _JBLEN 32
+#define _JBLEN 64
 #elif defined(__arm__)
 #define _JBLEN 64
 #elif defined(__i386__)
```







# A. 参考

1. libm（Bionic的一个库）性能的一次修订（我们关注的就是这个修订）：
   1. https://android.googlesource.com/platform/bionic/+/edf386b1f9839b36cd08148bb61936b94f7e300c
2. 上述修订可以被使用的前提是Clang支持几个Intrinsics，这几个Intrinsics必须先被移植到我们自己编译的Clang15中（因为这些Intrinsics实际是在Clang16中才有的）
   1. https://reviews.llvm.org/D136508
   2. https://releases.llvm.org/16.0.0/tools/clang/docs/ReleaseNotes.html



