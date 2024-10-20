# 1. QEMU调试

在移植或者某个版本初期，往往真机不可用，Qemu是最好的选择。



## 1. 1. qemu



### 1.1.1 源代码

使用官方qemu master分支（理论上7.2之后的版本即可，但由于龙芯持续在推送功能增强补丁到新的版本，建议用最新版本）。可以参考gitlab中qemu项目的a12_larch分支。

当前测试用的版本信息如下

```bash
## git 信息
ssh://git@8.140.33.210:2222/android/qemu.git
```



### 1.1.2 编译

参考qemu官方文档，安装必要的依赖包之后configure/make/make install。

参考的命令序列：

```bash
cd <qemu-src-dir>
mkdir build
cd build
../configure --target-list=loongarch64-softmmu,loongarch64-linux-user --disable-werror --enable-capstone
make
sudo make install
```



### 1.1.3 编译测试用例

本文档将使用如下测试用例为例进行说明，因此，需要编译出来对应的测试用例。

```bash
$ m bionic-unit-tests bionic-unit-tests-static
$ ll $OUT/testcases/bionic-unit-tests-static/loongarch64/bionic-unit-tests-static
$ ll $OUT/testcases/bionic-unit-tests/loongarch64/bionic-unit-tests
```



## 1. 2. 用户态qemu



### 1.2.1 使用

对于静态链接的安卓应用，如bionic-unit-tests-static，直接用qemu-loongarch64运行即可。如有命令行参数，紧跟着可执行程序输入：

```bash
$ qemu-loongarch64 data/nativetest64/bionic-unit-tests-static/bionic-unit-tests-static -h
Unit Test Options:
  -j [JOB_COUNT] or -j[JOB_COUNT]
      Run up to JOB_COUNT tests in parallel.
      Use isolation mode, Run each test in a separate process.
      If JOB_COUNT is not given, it is set to the count of available processors.
  --no_isolate
      Don't use isolation mode, run all tests in a single process.
      If the test seems to be running in a debugger (based on the parent's name) this will
      be automatically set. If this behavior is not desired use the '--force_isolate' flag
      below.
   ...
```

对于动态链接的程序，需要让qemu-loongarch64知道哪里可以找到其需要链接器以及依赖的动态库。例如，直接运行编译后生成的system/bin/sh文件，会报这样的错误：

```bash
foxsen@fx:~/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64$ ~/software/qemu-up/build/qemu-loongarch64 system/bin/sh
qemu-loongarch64: Could not open '/system/bin/linker64': No such file or directory
foxsen@fx:~/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64$
```

可以直接在host上建立/system目录，将生成的system目录的内容拷贝到其中，使得qemu-loongarch64可以从/system/bin/linker64找到安卓用的链接器。注意，最新的安卓系统（好像从10.0之后）引入了一个apex技术，将常用的系统程序包括linker/libc等runtime都独立打成aapex格式的软件包，然后在启动过程中挂载到必要的位置。因此，编译生成的system/bin/linker64只是一个链接，指向/apex/com.android.runtime/bin/linker64，要将apex目录下的内容也拷贝到/apex/下来解决问题(如果./apex目录没有相关内容，可以看看symbols/apex/目录)。

```bash
foxsen@fx:~/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64$ sudo cp -a apex/* /apex/
foxsen@fx:~/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64$ sudo cp -a system/* /system/
```

用户态qemu仿真有很多不足的地方，比如不支持一些安卓特有的系统调用（binder/ashmem等），建议只用它来跑bionic的单元测试，其他情况可以用系统级qemu。



### 1.2.2 调试

用户级qemu调试应用的详细方法可以参见官方文档。这里给出一个案例：

```bash
$ loongarch64-unknown-linux-gnu-gdb ./symbols/data/nativetest64/bionic-unit-tests-static/bionic-unit-tests-static --no_isolate
GNU gdb (GDB) 14.0.50.20230620-git
...
Reading symbols from ./symbols/data/nativetest64/bionic-unit-tests-static/bionic-unit-tests-static...
(gdb) b main
Breakpoint 1 at 0x1200f2770: main. (2 locations)
(gdb) target remote :1234
Remote debugging using :1234
0x00000001200f2650 in _start ()
(gdb) c
Continuing.

Breakpoint 1.1, main (argc=1, argv=0x555555d55bb8, envp=0x555555d55bc8) at bionic/tests/gtest_main.cpp:40
40	bionic/tests/gtest_main.cpp: 没有那个文件或目录.
(gdb) set directories ~/android-la64/loongson/aosp.la  //设定源代码目录
(gdb) l
35	  return g_envp;
36	}
37	
38	int main(int argc, char** argv, char** envp) {
39	  g_argc = argc;
40	  g_argv = argv;
41	  g_envp = envp;
42	
43	  return IsolateMain(argc, argv, envp);
44	}
(gdb) n
39	  g_argc = argc;
(gdb) n
40	  g_argv = argv;
(gdb) s
41	  g_envp = envp;
(gdb) s
43	  return IsolateMain(argc, argv, envp);
(gdb) s
IsolateMain (argc=1, argv=0x555555d55bb8) at system/testing/gtest_extras/IsolateMain.cpp:129
129	  std::vector<const char*> args{argv[0]};
(gdb) 
```

用户级仿真调试程序的优点是它不会受到目标操作系统的干扰，意外的情况比较少，比较易于复现问题。

上述的案例，如果输入c命令让它继续运行，就会复现这样的问题：

```bash
[WARNING] external/googletest/googletest/src/gtest-death-test.cc:1121:: Death tests use fork(), which is unsafe particularly in a threaded context. For this test, Google Test couldn't detect the number of threads. See https://github.com/google/googletest/blob/master/docs/advanced.md#death-tests-and-threads for more explanation and suggested solutions, especially if this is the last message you see before your test times out.
bionic/tests/pthread_test.cpp:359: Failure
Death test: TestBug37410::main()
    Result: died but not with expected exit code:
            Terminated by signal 11 (core dumped)
Actual msg:
[  DEATH   ] qemu: uncaught target signal 11 (Segmentation fault) - core dumped
[  DEATH   ]
[  FAILED  ] pthread_DeathTest.pthread_bug_37410 (70 ms)
[ RUN      ] pthread_DeathTest.pthread_setname_np__no_such_thread
```

此时在gdb中观察，可以看到出错的代码位置：

```bash
(gdb) where
#0  __set_stack_and_tls_vma_name (is_main_thread=false) at bionic/libc/bionic/pthread_create.cpp:324
#1  0x0000000120757584 in __pthread_start (arg=0x555765e6fcd0) at bionic/libc/bionic/pthread_create.cpp:350
#2  0x00000001206e280c in __start_thread (fn=0x120757554 <__pthread_start(void*)>, arg=0x555765e6fcd0) at bionic/libc/bionic/clone.cpp:53
(gdb) 
```

这样，我们就可以和调试普通程序类似去修复相关的问题了。记得每次m编译后，如果有涉及链接器和库的修改，要更新下/system和/apex的内容。

除了用gdb，用户级模拟器还有一个有用的选项：-strace可以打印程序运行中的syscall情况。



## 1.3. 系统级qemu



一个参考的命令行如下：

```bash
#!/bin/bash

./qemu-system-loongarch64 -machine virt -m 1024 -cpu la464-loongarch-cpu \
      -smp 1 \
      -bios ./QEMU_EFI.fd \
      -kernel ./vmlinuz.efi \
      -drive id=disk0,file=./androidroot.img,if=none,format=raw -device virtio-blk-pci,drive=disk0 \
      -append "root=/dev/vda init=/data/nativetest64/bionic-unit-tests-static/bionic-unit-tests-static --no_isolate" \
      -nographic $*
```

其中，-bios选项指定启动用的uefi bios固件，-kernel指定内核efi文件，-drive设置一个virtio pci 磁盘, -append选项指定内核命令行选项。磁盘对应的内容在androidboot.img文件中，该文件可以用如下方式创建和使用：

```bash
dd if=/dev/zero of=androidroot.img bs=1M count=16384
sudo losetup /dev/loop18 ./androidroot.img
sudo mkfs.ext4 /dev/loop18
sudo mount /dev/loop18 /mnt/
sudo cp -a /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/system/* /mnt/
sudo cp -a /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/data/* /mnt/data/
sudo umount /mnt
sudo losetup -d /dev/loop18
```

如果需要修改内容，只需要重新losetup/mount/cp/umount/losetup -d即可。

系统级qemu同样支持gdb调试，案例如下：

```bash
foxsen@fx:~/software/qemu-up/build$ ./run.sh -s -S # 在终端1启动模拟器，等待调试器连接
# 切换到另一个终端
foxsen@fx:~/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64$ /opt/gdb/bin/loongarch64-unknown-linux-gnu-gdb ./symbols/data/nativetest64/bionic-unit-tests-static/bionic-unit-tests-static 
GNU gdb (GDB) 14.0.50.20230620-git
Copyright (C) 2023 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "--host=x86_64-pc-linux-gnu --target=loongarch64-unknown-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<https://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from ./symbols/data/nativetest64/bionic-unit-tests-static/bionic-unit-tests-static...
(gdb) target remote :1234
Remote debugging using :1234
0x000000001c000000 in ?? ()
(gdb) b main
Breakpoint 1 at 0x1200f2770: main. (2 locations)
(gdb) c
Continuing.

Breakpoint 1.1, main (argc=2, argv=0x7ffffbf5f3a8, envp=0x7ffffbf5f3c0) at bionic/tests/gtest_main.cpp:40
40	bionic/tests/gtest_main.cpp: 没有那个文件或目录.
(gdb) set directories /home/foxsen/android-la64/loongson/aosp.la/
(gdb) l
35	  return g_envp;
36	}
37	
38	int main(int argc, char** argv, char** envp) {
39	  g_argc = argc;
40	  g_argv = argv;
41	  g_envp = envp;
42	
43	  return IsolateMain(argc, argv, envp);
44	}
(gdb) 
```







# 2. 真机调试

在移植或者某个版本初期，往往真机不可用，Qemu是最好的选择。以下是龙芯同事整理的一个问题（请参考git库：``ssh://git@8.140.33.210:2222/android/android_qemu_env.git` ）

1. run create_androidimg.sh to build a virtual disk with generated images.
2. adapt r.sh to run qemu. Use the updated kernel provided here(the old kernel is not tested). You can build from the latest source using defconfig.
3. in android command line, type su to become root, then wait till the output stablized like this:

```bash
console:/ # [   87.667805][    T1] init: Control message: Could not find 'android.hardware.media.c2@1.0::IComponentStore/default' for ctl.interface_start from pid: 111 (/system/bin/hwservicemanager)
s[   88.186496][    T1] init: Control message: Could not find 'android.hardware.gatekeeper@1.0::IGatekeeper/default' for ctl.interface_start from pid: 111 (/system/bin/hwservicemanager)
top[   88.671658][    T1] init: Control message: Could not find 'android.hardware.media.c2@1.0::IComponentStore/default' for ctl.interface_start from pid: 111 (/system/bin/hwservicemanager)
 [   89.192700][    T1] init: Control message: Could not find 'android.hardware.gatekeeper@1.0::IGatekeeper/default' for ctl.interface_start from pid: 111 (/system/bin/hwservicemanager)
[   89.685854][    T1] init: Control message: Could not find 'android.hardware.media.c2@1.0::IComponentStore/default' for ctl.interface_start from pid: 111 (/system/bin/hwservicemanager)
hws[   90.204347][    T1] init: Control message: Could not find 'android.hardware.gatekeeper@1.0::IGatekeeper/default' for ctl.interface_start from pid: 111 (/system/bin/hwservicemanager)
erv[   90.690269][    T1] init: Control message: Could not find 'android.hardware.media.c2@1.0::IComponentStore/default' for ctl.interface_start from pid: 111 (/system/bin/hwservicemanager)
ice[   91.213950][    T1] init: Control message: Could not find 'android.hardware.gatekeeper@1.0::IGatekeeper/default' for ctl.interface_start from pid: 111 (/system/bin/hwservicemanager)
mana[   91.703156][    T1] init: Control message: Could not find 'android.hardware.media.c2@1.0::IComponentStore/default' for ctl.interface_start from pid: 111 (/system/bin/hwservicemanager)
ger
[   92.225024][    T1] init: Control message: Could not find 'android.hardware.gatekeeper@1.0::IGatekeeper/default' for ctl.interface_start from pid: 111 (/system/bin/hwservicemanager)
[   92.454221][    T1] init: Sending signal 9 to service 'hwservicemanager' (pid 111) process group...
[   92.498516][    T1] libprocessgroup: Successfully killed process cgroup uid 1000 pid 111 in 43ms
[   92.504971][    T1] init: Control message: Processed ctl.stop for 'hwservicemanager' from pid: 430 (stop hwservicemanager)
[   92.506166][    T1] init: Service 'hwservicemanager' (pid 111) received signal 9
console:/ # [   92.707756][  T310] binder: 310:310 transaction failed 29189/-22, size 204-32 line 3143
[   93.218844][  T320] binder: 320:320 transaction failed 29189/-22, size 204-32 line 3143

console:/ # start hwservicemanager
[   98.535803][    T1] init: starting service 'hwservicemanager'...
[   98.574409][    T1] init: Control message: Processed ctl.start for 'hwservicemanager' from pid: 433 (start hwservicemanager)
console:/ # [   98.604817][  T434] libprocessgroup: Failed to open /dev/stune/foreground/tasks: No such file or directory: No such file or directory
[   98.605536][  T434] libprocessgroup: Failed to apply HighPerformance task profile: No such file or directory
```

type "stop hwservicemanager" and then "start hwservicemanager" to eliminate the init output. 

5. network setup

We use qemu user network, you can enable the network like this:

```bash
console:/ # su
console:/ # ifconfig eth0 up
[  105.170256][   T16] e1000: eth0 NIC Link is Up 1000 Mbps Full Duplex, Flow Control: RX
console:/ # [  105.197286][   T16] IPv6: ADDRCONF(NETDEV_CHANGE): eth0: link becomes ready

console:/ # ifconfig eth0 10.0.2.15
console:/ # ip rule add from all lookup main pref 1
console:/ # ping 10.0.2.2
PING 10.0.2.2 (10.0.2.2) 56(84) bytes of data.
64 bytes from 10.0.2.2: icmp_seq=1 ttl=255 time=8.91 ms
64 bytes from 10.0.2.2: icmp_seq=2 ttl=255 time=1.82 ms
^C
--- 10.0.2.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1005ms
rtt min/avg/max/mdev = 1.826/5.370/8.915/3.545 ms
```

The 'ip rule' command is necessary because the fault policy route rule will prevent connections by default.

6. Now you can use adb shell to connect to this android instance.

```bash
foxsen@fx:~/android-la64/loongson/aosp.la$ adb shell
adb: no devices/emulators found
foxsen@fx:~/android-la64/loongson/aosp.la$ adb kill-server
foxsen@fx:~/android-la64/loongson/aosp.la$ adb shell
* daemon not running; starting now at tcp:5037
* daemon started successfully
generic_loongarch64:/ # ls
acct        cache   data_mirror    init             metadata  oem          sdcard                  system
apex        config  debug_ramdisk  init.environ.rc  mnt       postinstall  second_stage_resources  system_ext
bin         d       dev            linkerconfig     odm       proc         storage                 vendor
bugreports  data    etc            lost+found       odm_dlkm  product      sys                     vendor_dlkm
generic_loongarch64:/ #
```





# 3. GDB调试surfaceflinger技巧

本部分内容有龙芯贡献。

前一阵在qemu和真机上用gdbserver64和x86主机的cross gdb调试surfaceflinger，遇到几个困难：

1. 不加载符号表。这个问题主要是没设置好路径，gdb不知道在哪里可以找到带调试信息的相应二进制（可执行文件和库），只需要设置适当的solib-search-path就可以了
2. 不加载用dlopen后续打开的动态库。解决了1的问题之后，gdb仍然无法自动加载程序运行过程中动态加载的新动态库，导致那些库无法设置断点或者进行源代码级调试。而surfaceflinger和mesa/hwcomposer等有密切交互，会动态加载不少库。可以在相应的库函数被加载后运行sharedLibrary命令迫使gdb加载相应的符合。不带参数的shareLibrary会试图去找目前已经加载的所有动态库，找到相应文件就会加载其符号表，加载之后就可以对其进行源码级调试了。而定位这种动态库加载的时机，可以通过在dlopen, android_dlopen_ext等函数设置断点来截获。
3. 在模拟器上（真机可能也会，没验证），调试surfaceflinger经常会碰到进程被SIG35中断，导致程序退出无法继续调试。估计是因为某些地方有超时设置引起的，可以通过设置gdb忽略该信号。

解决这些问题之后，就可以对surfaceflinger整个过程进行完整的源码级调试了。



### 3.1 参考的.gdbinit文件内容

```
set auto-solib-add on
handle SIG35 noprint nostop nopass
set solib-search-path ./out/target/product/generic_loongarch64/symbols/system/lib64/:./out/target/product/generic_loongarch64/symbols/vendor/lib64/egl/:./out/target/product/generic_loongarch64/symbols/vendor/lib64/:./out/target/product/generic_loongarch64/symbols/vendor/lib64/dri/:./out/target/product/generic_loongarch64/symbols/system/lib64/egl:./out/target/product/generic_loongarch64/symbols/apex/com.android.runtime/lib64/bionic/:./out/target/product/generic_loongarch64/symbols/apex/com.android.runtime/lib64/:./out/target/product/generic_loongarch64/symbols/vendor/lib64/hw
b main


```



###3.2 初始调试命令

target remote 192.168.2.2:1234 # 地址根据实际情况修改

```
C

sharedlibrary

b eglInitialize

b eglGetDisplayImpl

c

sharedlibrary

sharedlibrary /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/egl/libdrm.so

b dri2_initialize_android

b eglCreateImageKHR

c

b 1150

sharedlibrary




```



### 3.3 参考调试过程

服务器端运行gdbserver64 :1234 /system/bin/surfaceflinger

客户端运行cross gdb(官方提供的8.0rc2), 当前目录设置以上的.gdbinit

```
$> /opt/loongson-gnu-toolchain-8.3-x86_64-loongarch64-linux-gnu-rc1.2/bin/loongarch64-linux-gnu-gdb ./out/target/product/generic_loongarch64/symbols/system/bin/surfaceflinger

Type "apropos word" to search for commands related to "word"...
Reading symbols from ./out/target/product/generic_loongarch64/symbols/system/bin/surfaceflinger...done.
Breakpoint 1 at 0x221a48: file frameworks/native/services/surfaceflinger/main_surfaceflinger.cpp, line 81.
warning: Unable to find dynamic linker breakpoint function.
GDB will be unable to debug shared library initializers
and track explicitly loaded dynamic code.
0x00007fffc24aa4c0 in ?? ()
warning: Could not load shared library symbols for /system/bin/linker64.
Do you need "set solib-search-path" or "set sysroot"?
b
Breakpoint 1, main () at frameworks/native/services/surfaceflinger/main_surfaceflinger.cpp:81
81	    signal(SIGPIPE, SIG_IGN);
(gdb) b frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp
Function "frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp" not defined.
Make breakpoint pending on future shared library load? (y or [n]) n
(gdb) b frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp:162
Breakpoint 2 at 0x55555e5978bc: file frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp, line 162.
(gdb) c
Continuing.
[New Thread 4581.4623]
[New Thread 4581.4624]
[New Thread 4581.4625]

Thread 1 "surfaceflinger" hit Breakpoint 2, android::impl::HWComposer::setCallback (this=<optimized out>, callback=0x7fff2568d040)
    at frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp:162
162	    mComposer->registerCallback(
(gdb) b dlopen
dlopen                    dlopen(char const, int)  dlopen@plt
(gdb) b dlopen
Breakpoint 3 at 0x7fffb1f57cb0: file bionic/libdl/libdl.cpp, line 84.
(gdb) b android
Display all 200 possibilities? (y or n)
(gdb) b android_dl
android_dlopen_ext                                              android_dlwarning
android_dlopen_ext(char const, int, android_dlextinfo const)  android_dlwarning(void, void ()(void, char const))
android_dlopen_ext@plt
(gdb) b android_dl
android_dlopen_ext                                              android_dlwarning
android_dlopen_ext(char const, int, android_dlextinfo const)  android_dlwarning(void, void ()(void, char const*))
android_dlopen_ext@plt
(gdb) b android_dlopen_ext
Breakpoint 4 at 0x7fffb1f57d8c: file bionic/libdl/libdl.cpp, line 133.
(gdb) sharedlibrary
warning: .dynamic section for "/home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/libcutils.so" is not at the expected address (wrong library or version mismatch?)
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/vendor/lib64/egl/libGLES_mesa.so...done.
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/libdrm.so...done.
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/vendor/lib64/libglapi.so...done.
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/libcutils.so...done.
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/libbase.so...done.
(gdb) c
Continuing.
[New Thread 4581.4673]
[New Thread 4581.4674]
[New Thread 4581.4675]
[New Thread 4581.4676]
[New Thread 4581.4677]
[New Thread 4581.4678]

Thread 1 "surfaceflinger" hit Breakpoint 4, android_dlopen_ext (filename=0x7ffde5684170 "/vendor/lib64/hw/android.hardware.graphics.mapper@4.0-impl.minigbm.so", flag=1,
    extinfo=0x7ffffbe86e98) at bionic/libdl/libdl.cpp:133
133	  return __loader_android_dlopen_ext(filename, flag, extinfo, caller_addr);
(gdb) where

```

显示

```
0  android_dlopen_ext (filename=0x7ffde5684170 "/vendor/lib64/hw/android.hardware.graphics.mapper@4.0-impl.minigbm.so", flag=1, extinfo=0x7ffffbe86e98)

    at bionic/libdl/libdl.cpp:133

1  0x00007fffbba18b48 in android_load_sphal_library (name=0x7ffde5684170 "/vendor/lib64/hw/android.hardware.graphics.mapper@4.0-impl.minigbm.so", flag=<optimized out>)

    at system/core/libvndksupport/linker.cpp:71

2  0x00007fffb3ee73e8 in android::hardware::PassthroughServiceManager::openLibs(std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&, std::1::function<bool (void*, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&)> const&) (fqName=..., eachLib=...) at system/libhidl/transport/ServiceManagement.cpp:392

3  0x00007fffb3eeb544 in android::hardware::PassthroughServiceManager::get (this=<optimized out>, fqName=..., name=...)

    at system/libhidl/transport/ServiceManagement.cpp:414

4  0x00007fffb3ee8ad8 in android::hardware::details::getRawServiceInternal (descriptor=..., instance=..., retry=<optimized out>, getStub=false)

    at system/libhidl/transport/ServiceManagement.cpp:825

5  0x00007fffb73751b0 in android::hardware::details::getServiceInternal<android::hardware::graphics::mapper::V4_0::BpHwMapper, android::hardware::graphics::mapper::V4_0::IMapper, void, void> (instance=..., retry=true, getStub=false) at system/libhidl/transport/include/hidl/HidlTransportSupport.h:165

6  0x00007fffb73753d4 in android::hardware::graphics::mapper::V4_0::IMapper::getService (serviceName=..., getStub=true)

    at out/soong/.intermediates/hardware/interfaces/graphics/mapper/4.0/android.hardware.graphics.mapper@4.0_genc++/gen/android/hardware/graphics/mapper/4.0/MapperAll.cpp:3631

7  0x00007fffb75b5d90 in android::Gralloc4Mapper::Gralloc4Mapper (this=0x7ffda568b9b0) at frameworks/native/libs/ui/Gralloc4.cpp:91

8  0x00007fffb75c3ec8 in std::__1::make_unique<android::Gralloc4Mapper const> () at external/libcxx/include/memory:3132

9  android::GraphicBufferMapper::GraphicBufferMapper (this=0x7ffda5689930) at frameworks/native/libs/ui/GraphicBufferMapper.cpp:55

10 0x00007fffb75c00b4 in android::Singletonandroid::GraphicBufferMapper::getInstance () at system/core/libutils/include/utils/Singleton.h:55

11 android::GraphicBufferMapper::get () at frameworks/native/libs/ui/include/ui/GraphicBufferMapper.h:50

12 android::GraphicBuffer::GraphicBuffer (this=0x7ffe45690130) at frameworks/native/libs/ui/GraphicBuffer.cpp:64

13 0x00007fffb75c02d0 in android::GraphicBuffer::GraphicBuffer (this=0x7ffe45690130, inWidth=3840, inHeight=2160, inFormat=1, inLayerCount=1, inUsage=6912,

    requestorName=...) at frameworks/native/libs/ui/GraphicBuffer.cpp:86

14 0x00007fffb1cdab04 in android::BufferQueueProducer::allocateBuffers (this=0x7ffe5568cdd0, width=<optimized out>, height=<optimized out>, format=<optimized out>,

    usage=768) at frameworks/native/libs/gui/BufferQueueProducer.cpp:1443

15 0x00007fffb1d42d10 in android::Surface::allocateBuffers (this=<optimized out>) at frameworks/native/libs/gui/Surface.cpp:143

16 0x000055555e5ccc14 in android::surfaceflinger::impl::createNativeWindowSurface(android::spandroid::IGraphicBufferProducer const&)::NativeWindowSurface::preallocateBuffers() (this=<optimized out>) at frameworks/native/services/surfaceflinger/NativeWindowSurface.cpp:39

17 0x000055555e60ff60 in android::SurfaceFlinger::setupNewDisplayDeviceInternal (this=0x7fff2568d010, displayToken=..., compositionDisplay=..., state=...,

    displaySurface=..., producer=...) at frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:2706

18 0x000055555e610834 in android::SurfaceFlinger::processDisplayAdded (this=0x7fff2568d010, displayToken=..., state=...)

    at frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:2798

--Type <RET> for more, q to quit, c to continue without paging--

19 0x000055555e6056cc in android::SurfaceFlinger::processDisplayChangesLocked (this=0x7fff2568d010) at frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:2948

20 0x000055555e60312c in android::SurfaceFlinger::processDisplayHotplugEventsLocked (this=0x7fff2568d010)

    at frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:2634

21 0x000055555e608ce0 in android::SurfaceFlinger::onComposerHalHotplug (this=0x7fff2568d010, hwcDisplayId=0, connection=<optimized out>)

    at frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:1788

22 0x000055555e608d60 in non-virtual thunk to android::SurfaceFlinger::onComposerHalHotplug(unsigned long, android::hardware::graphics::composer::V2_1::IComposerCallback::Connection) () at frameworks/native/services/surfaceflinger/DisplayIdGenerator.h:55

23 0x000055555e5a6d48 in android::(anonymous namespace)::ComposerCallbackBridge::onHotplug (this=<optimized out>, display=1, connection=(unknown: -68653416))

    at frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp:89

24 0x00007fffa9f08118 in android::hardware::graphics::composer::V2_1::BnHwComposerCallback::hidl_onHotplug(android::hidl::base::V1_0::BnHwBase, android::hardware::Parcel const&, android::hardware::Parcel, std::__1::function<void (android::hardware::Parcel&)>) (hidl_this=0x7ffe1568c2a0, _hidl_data=..., _hidl_reply=0x7ffffbe88090,

    _hidl_cb=...)
    at out/soong/.intermediates/hardware/interfaces/graphics/composer/2.1/android.hardware.graphics.composer@2.1_genc++/gen/android/hardware/graphics/composer/2.1/ComposerCallbackAll.cpp:448

25 0x00007fffb7480878 in android::hardware::graphics::composer::V2_4::BnHwComposerCallback::onTransact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>) (this=0x7ffe1568c2a0, _hidl_code=<optimized out>, _hidl_data=..., _hidl_reply=0x7ffffbe88090,

    _hidl_flags=<optimized out>, _hidl_cb=...)
    at out/soong/.intermediates/hardware/interfaces/graphics/composer/2.4/android.hardware.graphics.composer@2.4_genc++/gen/android/hardware/graphics/composer/2.4/ComposerCallbackAll.cpp:653

26 0x00007fffb3f3b830 in android::hardware::BHwBinder::transact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>) (this=0x7ffe1568c2a0, code=<optimized out>, data=..., reply=0x7ffffbe88090, flags=<optimized out>, callback=...)

    at system/libhwbinder/Binder.cpp:135

27 0x00007fffb3f41a6c in android::hardware::IPCThreadState::executeCommand (this=0x7ffe7568a950, cmd=<optimized out>) at system/libhwbinder/IPCThreadState.cpp:1202

28 0x00007fffb3f42bc4 in android::hardware::IPCThreadState::waitForResponse (this=0x7ffe7568a950, reply=0x7ffffbe88490, acquireResult=0x0)

    at system/libhwbinder/IPCThreadState.cpp:868

29 0x00007fffb3f4262c in android::hardware::IPCThreadState::transact (this=0x7ffe7568a950, handle=<optimized out>, code=1, data=..., reply=0x7ffffbe88490,

    flags=<optimized out>) at system/libhwbinder/IPCThreadState.cpp:652

30 0x00007fffb3f3cbd4 in android::hardware::BpHwBinder::transact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>) (this=0x7ffe1568b7a0, code=<optimized out>, data=..., reply=0x7ffffbe88490, flags=<optimized out>, callback=...)

    at system/libhwbinder/BpHwBinder.cpp:107

31 0x00007fffa9f0efbc in android::hardware::graphics::composer::V2_1::BpHwComposerClient::hidl_registerCallback (hidl_this=<optimized out>, _hidl_this_instrumentor=

    0x7ffe45690940, callback=...)
    at out/soong/.intermediates/hardware/interfaces/graphics/composer/2.1/android.hardware.graphics.composer@2.1_genc++/gen/android/hardware/graphics/composer/2.1/ComposerCl--Type <RET> for more, q to quit, c to continue without paging--

ientAll.cpp:194

32 0x00007fffa9f150b0 in android::hardware::graphics::composer::V2_1::BpHwComposerClient::registerCallback (this=<optimized out>, callback=...)

    at out/soong/.intermediates/hardware/interfaces/graphics/composer/2.1/android.hardware.graphics.composer@2.1_genc++/gen/android/hardware/graphics/composer/2.1/ComposerClientAll.cpp:1949

33 0x000055555e5806b4 in android::Hwc2::impl::Composer::registerCallback(android::spandroid::hardware::graphics::composer::V2_4::IComposerCallback const&)::$_34::operator()() const (this=<optimized out>) at frameworks/native/services/surfaceflinger/DisplayHardware/ComposerHal.cpp:193

34 android::Hwc2::impl::Composer::registerCallback (this=<optimized out>, callback=...) at frameworks/native/services/surfaceflinger/DisplayHardware/ComposerHal.cpp:189

35 0x000055555e5978cc in android::impl::HWComposer::setCallback (this=<optimized out>, callback=0x7fff2568d040)

    at frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp:162

36 0x000055555e60206c in android::SurfaceFlinger::init (this=0x7fff2568d010) at frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:803

37 0x000055555e651d00 in main () at frameworks/native/services/surfaceflinger/main_surfaceflinger.cpp:149

```

进一步

```
(gdb) l
128	}
129
130	attribute((weak))
131	void* android_dlopen_ext(const char* filename, int flag, const android_dlextinfo* extinfo) {
132	  const void* caller_addr = builtin_return_address(0);
133	  return loader_android_dlopen_ext(filename, flag, extinfo, caller_addr);
134	}
135
136	attribute((weak))
137	int android_get_application_target_sdk_version() {
(gdb) finish
Run till exit from #0  android_dlopen_ext (filename=0x7ffde5684170 "/vendor/lib64/hw/android.hardware.graphics.mapper@4.0-impl.minigbm.so", flag=1, extinfo=0x7ffffbe86e98)
    at bionic/libdl/libdl.cpp:133
0x00007fffbba18b48 in android_load_sphal_library (name=0x7ffde5684170 "/vendor/lib64/hw/android.hardware.graphics.mapper@4.0-impl.minigbm.so", flag=<optimized out>)
    at system/core/libvndksupport/linker.cpp:71
71	        void* handle = android_dlopen_ext(name, flag, &dlextinfo);
Value returned is $1 = (void *) 0xea6bcd040c5e3979
(gdb) sharedlibrary
warning: .dynamic section for "/home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/libhardware.so" is not at the expected address (wrong library or version mismatch?)
warning: .dynamic section for "/home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/libhidlbase.so" is not at the expected address (wrong library or version mismatch?)
warning: .dynamic section for "/home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/libprocessgroup.so" is not at the expected address (wrong library or version mismatch?)
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/libc++.so...done.
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/libhardware.so...done.
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/android.hardware.graphics.mapper@4.0.so...done.
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/android.hardware.graphics.common@1.0.so...done.
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/android.hardware.graphics.common@1.1.so...done.
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/android.hardware.graphics.common@1.2.so...done.
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/libhidlbase.so...done.
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/libutils.so...done.
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/libprocessgroup.so...done.
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/libgralloctypes.so...done.
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/android.hardware.graphics.common-V2-ndk_platform.so...done.
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/android.hardware.common-V2-ndk_platform.so...done.
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/vendor/lib64/hw/android.hardware.graphics.mapper@4.0-impl.minigbm.so...done.
(gdb) b GetAndReferenceDriver
Breakpoint 5 at 0x7ffd0ac6c43c: file external/minigbm/cros_gralloc/gralloc4/CrosGralloc4Mapper.cc, line 47.
(gdb) c
Continuing.

Thread 1 "surfaceflinger" hit Breakpoint 5, (anonymous namespace)::DriverProvider::GetAndReferenceDriver (this=0x7ffdd568f580)
    at external/minigbm/cros_gralloc/gralloc4/CrosGralloc4Mapper.cc:47
47	        std::lock_guardstd::mutex lock(mMutex);
(gdb) where

```

显示

```
0  (anonymous namespace)::DriverProvider::GetAndReferenceDriver (this=0x7ffdd568f580) at external/minigbm/cros_gralloc/gralloc4/CrosGralloc4Mapper.cc:47

1  0x00007ffd0ac71354 in CrosGralloc4Mapper::CrosGralloc4Mapper (this=0x7ffdb5690470) at external/minigbm/cros_gralloc/gralloc4/CrosGralloc4Mapper.cc:82

2  HIDL_FETCH_IMapper () at external/minigbm/cros_gralloc/gralloc4/CrosGralloc4Mapper.cc:1056

3  0x00007fffb3eed724 in android::hardware::PassthroughServiceManager::get(android::hardware::hidl_string const&, android::hardware::hidl_string const&)::{lambda(void, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&)#1}::operator()(void, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&) const (this=0x7ffffbe870f8, handle=<optimized out>, lib=..., sym=...) at system/libhidl/transport/ServiceManagement.cpp:429

4  0x00007fffb3eed684 in std::1::invoke<android::hardware::PassthroughServiceManager::get(android::hardware::hidl_string const&, android::hardware::hidl_string const&)::{lambda(void, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&)#1}&, void, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&> (f=..., args=..., args=..., args=...) at external/libcxx/include/type_traits:4353

5  std::1::invoke_void_return_wrapper<bool>::call<android::hardware::PassthroughServiceManager::get(android::hardware::hidl_string const&, android::hardware::hidl_string const&)::{lambda(void*, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&)#1}&, void*, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&>(android::hardware::PassthroughServiceManager::get(android::hardware::hidl_string const&, android::hardware::hidl_string const&)::{lambda(void*, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&)#1}&, void*&&, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&) (args=..., args=..., args=..., __args=...)

    at external/libcxx/include/__functional_base:318

6  std::1::function::alloc_func<android::hardware::PassthroughServiceManager::get(android::hardware::hidl_string const&, android::hardware::hidl_string const&)::{lambda(void*, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&)#1}, std::1::allocator<{lambda(void, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&)#1}>, bool (void, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&)>::operator()(void*&&, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&) (this=0x7ffdd568f580,

    __arg=..., __arg=..., __arg=...) at external/libcxx/include/functional:1527

7  std::1::function::func<android::hardware::PassthroughServiceManager::get(android::hardware::hidl_string const&, android::hardware::hidl_string const&)::{lambda(void*, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&)#1}, std::1::allocator<{lambda(void, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&)#1}>, bool (void, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&)>::operator()(void*&&, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&) (this=<optimized out>,

    __arg=..., __arg=..., __arg=...) at external/libcxx/include/functional:1651

--Type <RET> for more, q to quit, c to continue without paging--

8  0x00007fffb3ee7414 in std::1::function::value_func<bool (void*, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&)>::operator()(void*&&, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&) const (this=0x7ffffbe870f0, args=...,

    __args=..., __args=...) at external/libcxx/include/functional:1799

9  std::1::function<bool (void*, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&)>::operator()(void*, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&) const (this=0x7ffffbe870f0, arg=..., arg=..., arg=...)

    at external/libcxx/include/functional:2347

10 android::hardware::PassthroughServiceManager::openLibs(std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&, std::1::function<bool (void*, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&, std::1::basic_string<char, std::1::char_traits<char>, std::1::allocator<char> > const&)> const&) (fqName=..., eachLib=...) at system/libhidl/transport/ServiceManagement.cpp:403

11 0x00007fffb3eeb544 in android::hardware::PassthroughServiceManager::get (this=<optimized out>, fqName=..., name=...)

    at system/libhidl/transport/ServiceManagement.cpp:414

12 0x00007fffb3ee8ad8 in android::hardware::details::getRawServiceInternal (descriptor=..., instance=..., retry=<optimized out>, getStub=false)

    at system/libhidl/transport/ServiceManagement.cpp:825

13 0x00007fffb73751b0 in android::hardware::details::getServiceInternal<android::hardware::graphics::mapper::V4_0::BpHwMapper, android::hardware::graphics::mapper::V4_0::IMapper, void, void> (instance=..., retry=true, getStub=false) at system/libhidl/transport/include/hidl/HidlTransportSupport.h:165

14 0x00007fffb73753d4 in android::hardware::graphics::mapper::V4_0::IMapper::getService (serviceName=..., getStub=false)

    at out/soong/.intermediates/hardware/interfaces/graphics/mapper/4.0/android.hardware.graphics.mapper@4.0_genc++/gen/android/hardware/graphics/mapper/4.0/MapperAll.cpp:3631

15 0x00007fffb75b5d90 in android::Gralloc4Mapper::Gralloc4Mapper (this=0x7ffda568b9b0) at frameworks/native/libs/ui/Gralloc4.cpp:91

16 0x00007fffb75c3ec8 in std::__1::make_unique<android::Gralloc4Mapper const> () at external/libcxx/include/memory:3132

17 android::GraphicBufferMapper::GraphicBufferMapper (this=0x7ffda5689930) at frameworks/native/libs/ui/GraphicBufferMapper.cpp:55

18 0x00007fffb75c00b4 in android::Singletonandroid::GraphicBufferMapper::getInstance () at system/core/libutils/include/utils/Singleton.h:55

19 android::GraphicBufferMapper::get () at frameworks/native/libs/ui/include/ui/GraphicBufferMapper.h:50

20 android::GraphicBuffer::GraphicBuffer (this=0x7ffe45690130) at frameworks/native/libs/ui/GraphicBuffer.cpp:64

21 0x00007fffb75c02d0 in android::GraphicBuffer::GraphicBuffer (this=0x7ffe45690130, inWidth=3840, inHeight=2160, inFormat=1, inLayerCount=1, inUsage=6912,

    requestorName=...) at frameworks/native/libs/ui/GraphicBuffer.cpp:86

22 0x00007fffb1cdab04 in android::BufferQueueProducer::allocateBuffers (this=0x7ffe5568cdd0, width=<optimized out>, height=<optimized out>, format=<optimized out>,

    usage=768) at frameworks/native/libs/gui/BufferQueueProducer.cpp:1443

23 0x00007fffb1d42d10 in android::Surface::allocateBuffers (this=<optimized out>) at frameworks/native/libs/gui/Surface.cpp:143

24 0x000055555e5ccc14 in android::surfaceflinger::impl::createNativeWindowSurface(android::spandroid::IGraphicBufferProducer const&)::NativeWindowSurface::preallocateBuffers() (this=<optimized out>) at frameworks/native/services/surfaceflinger/NativeWindowSurface.cpp:39

--Type <RET> for more, q to quit, c to continue without paging--

25 0x000055555e60ff60 in android::SurfaceFlinger::setupNewDisplayDeviceInternal (this=0x7fff2568d010, displayToken=..., compositionDisplay=..., state=...,

    displaySurface=..., producer=...) at frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:2706

26 0x000055555e610834 in android::SurfaceFlinger::processDisplayAdded (this=0x7fff2568d010, displayToken=..., state=...)

    at frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:2798

27 0x000055555e6056cc in android::SurfaceFlinger::processDisplayChangesLocked (this=0x7fff2568d010) at frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:2948

28 0x000055555e60312c in android::SurfaceFlinger::processDisplayHotplugEventsLocked (this=0x7fff2568d010)

    at frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:2634

29 0x000055555e608ce0 in android::SurfaceFlinger::onComposerHalHotplug (this=0x7fff2568d010, hwcDisplayId=0, connection=<optimized out>)

    at frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:1788

30 0x000055555e608d60 in non-virtual thunk to android::SurfaceFlinger::onComposerHalHotplug(unsigned long, android::hardware::graphics::composer::V2_1::IComposerCallback::Connection) () at frameworks/native/services/surfaceflinger/DisplayIdGenerator.h:55

31 0x000055555e5a6d48 in android::(anonymous namespace)::ComposerCallbackBridge::onHotplug (this=<optimized out>, display=1,

    connection=android::hardware::graphics::composer::V2_1::IComposerCallback::Connection::DISCONNECTED)
    at frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp:89

32 0x00007fffa9f08118 in android::hardware::graphics::composer::V2_1::BnHwComposerCallback::hidl_onHotplug(android::hidl::base::V1_0::BnHwBase, android::hardware::Parcel const&, android::hardware::Parcel, std::__1::function<void (android::hardware::Parcel&)>) (hidl_this=0x7ffe1568c2a0, _hidl_data=..., _hidl_reply=0x7ffffbe88090,

    _hidl_cb=...)
    at out/soong/.intermediates/hardware/interfaces/graphics/composer/2.1/android.hardware.graphics.composer@2.1_genc++/gen/android/hardware/graphics/composer/2.1/ComposerCallbackAll.cpp:448

33 0x00007fffb7480878 in android::hardware::graphics::composer::V2_4::BnHwComposerCallback::onTransact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>) (this=0x7ffe1568c2a0, _hidl_code=<optimized out>, _hidl_data=..., _hidl_reply=0x7ffffbe88090,

    _hidl_flags=<optimized out>, _hidl_cb=...)
    at out/soong/.intermediates/hardware/interfaces/graphics/composer/2.4/android.hardware.graphics.composer@2.4_genc++/gen/android/hardware/graphics/composer/2.4/ComposerCallbackAll.cpp:653

34 0x00007fffb3f3b830 in android::hardware::BHwBinder::transact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>) (this=0x7ffe1568c2a0, code=<optimized out>, data=..., reply=0x7ffffbe88090, flags=<optimized out>, callback=...)

    at system/libhwbinder/Binder.cpp:135

35 0x00007fffb3f41a6c in android::hardware::IPCThreadState::executeCommand (this=0x7ffe7568a950, cmd=<optimized out>) at system/libhwbinder/IPCThreadState.cpp:1202

36 0x00007fffb3f42bc4 in android::hardware::IPCThreadState::waitForResponse (this=0x7ffe7568a950, reply=0x7ffffbe88490, acquireResult=0x0)

    at system/libhwbinder/IPCThreadState.cpp:868

37 0x00007fffb3f4262c in android::hardware::IPCThreadState::transact (this=0x7ffe7568a950, handle=<optimized out>, code=1, data=..., reply=0x7ffffbe88490,

    flags=<optimized out>) at system/libhwbinder/IPCThreadState.cpp:652

38 0x00007fffb3f3cbd4 in android::hardware::BpHwBinder::transact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::functio--Type <RET> for more, q to quit, c to continue without paging--

n<void (android::hardware::Parcel&)>) (this=0x7ffe1568b7a0, code=<optimized out>, data=..., reply=0x7ffffbe88490, flags=<optimized out>, callback=...)
    at system/libhwbinder/BpHwBinder.cpp:107

39 0x00007fffa9f0efbc in android::hardware::graphics::composer::V2_1::BpHwComposerClient::hidl_registerCallback (hidl_this=<optimized out>,

    _hidl_this_instrumentor=0x7ffe45690940, callback=...)
    at out/soong/.intermediates/hardware/interfaces/graphics/composer/2.1/android.hardware.graphics.composer@2.1_genc++/gen/android/hardware/graphics/composer/2.1/ComposerClientAll.cpp:194

40 0x00007fffa9f150b0 in android::hardware::graphics::composer::V2_1::BpHwComposerClient::registerCallback (this=<optimized out>, callback=...)

    at out/soong/.intermediates/hardware/interfaces/graphics/composer/2.1/android.hardware.graphics.composer@2.1_genc++/gen/android/hardware/graphics/composer/2.1/ComposerClientAll.cpp:1949

41 0x000055555e5806b4 in android::Hwc2::impl::Composer::registerCallback(android::spandroid::hardware::graphics::composer::V2_4::IComposerCallback const&)::$_34::operator()() const (this=<optimized out>) at frameworks/native/services/surfaceflinger/DisplayHardware/ComposerHal.cpp:193

42 android::Hwc2::impl::Composer::registerCallback (this=<optimized out>, callback=...) at frameworks/native/services/surfaceflinger/DisplayHardware/ComposerHal.cpp:189

43 0x000055555e5978cc in android::impl::HWComposer::setCallback (this=<optimized out>, callback=0x7fff2568d040)

    at frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp:162

44 0x000055555e60206c in android::SurfaceFlinger::init (this=0x7fff2568d010) at frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:803

45 0x000055555e651d00 in main () at frameworks/native/services/surfaceflinger/main_surfaceflinger.cpp:149

```

进一步

```
(gdb) l
42	        static DriverProvider* instance = new DriverProvider();
43	        return instance;
44	    }
45
46	    cros_gralloc_driver* GetAndReferenceDriver() {
47	        std::lock_guardstd::mutex lock(mMutex);
48	        if (!mDriver) {
49	            mDriver = std::make_unique<cros_gralloc_driver>();
50	            if (mDriver->init()) {
51	                drv_log("Failed to initialize driver.\n");
(gdb) n
48	        if (!mDriver) {
(gdb) n
49	            mDriver = std::make_unique<cros_gralloc_driver>();
(gdb) n
50	            if (mDriver->init()) {
(gdb) s
cros_gralloc_driver::init (this=0x7ffe05687110) at external/minigbm/cros_gralloc/cros_gralloc_driver.cc:106
106			drv_ = init_try_node(i, render_nodes_fmt);
(gdb) s
105		for (uint32_t i = min_render_node; i < max_render_node; i++) {
(gdb) p i
2 = 128
(gdb) s
106			drv_ = init_try_node(i, render_nodes_fmt);
(gdb) s
init_try_node (idx=129, str=<optimized out>) at external/minigbm/cros_gralloc/cros_gralloc_driver.cc:68
68		char *node;
(gdb) s
71		if (asprintf(&node, str, DRM_DIR_NAME, idx) < 0)
(gdb) p i
No symbol "i" in current context.
(gdb) n
74		fd = open(node, O_RDWR, 0);
(gdb) p node
3 = 0x7ffe05686540 "/dev/dri/renderD129"
(gdb) q
A debugging session is active.

    Inferior 1 [process 4581] will be killed.

Quit anyway? (y or n) y

```

