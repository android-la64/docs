# 使用qemu调试安卓系统程序

## 用户态qemu

### 源代码

使用官方qemu master分支（理论上7.2之后的版本即可，但由于龙芯持续在推送功能增强补丁到新的版本，建议用最新版本）。我这里用版本可以参考gitlab中qemu项目的a12_larch分支。

### 编译

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

### 使用

对于静态链接的安卓应用，如bionic-unit-tests-static，直接用qemu-loongarch64运行即可。如有命令行参数，紧跟着可执行程序输入：

```bash
foxsen@fx:~/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64$ ~/software/qemu-up/build/qemu-loongarch64 data/nativetest64/bionic-unit-tests-static/bionic-unit-tests-static -h
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

### 调试

用户级qemu调试应用的详细方法可以参见官方文档。这里给出一个案例：

```bash
foxsen@fx:~/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64$ /opt/gdb/bin/loongarch64-unknown-linux-gnu-gdb ./symbols/data/nativetest64/bionic-unit-tests-static/bionic-unit-tests-static --no_isolate
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

## 系统级qemu

### 源代码和编译安装

同用户级。

### 使用

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



