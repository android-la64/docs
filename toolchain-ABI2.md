# 安卓工具链移植

要在安卓中添加新的架构支持，首先需要移植相关工具链：

1. clang. AOSP的prebuilts/clang/host/linux-x86目录下提供了安卓的clang工具链。
2. ndk. AOSP为开发安卓本地应用提供的开发工具包，是在clang的基础上添加一些相应的脚本等支持文件而来的。
3. rust. AOSP源码中引入了rust程序。

编译工具添加新的架构支持，都有一个bootstrap过程，在编译器源码中添加新架构的支持，然后用已有的未支持新架构的编译器编译得到新编译器。安卓clang工具链bootstrap过程比较麻烦的是，基础C库的支持似乎没有做到其工具链的bootstrap过程中，需要手工迭代。即用编译的半成品clang去编译bionic/art等aosp内容，用得到的libc.so等内容继续bootstrap完成新工具链。而安卓系统的基础库编译又依赖于其复杂的构建系统，要编译aosp就意味着需要先对其构建系统以及bionic/art等基础包进行移植。

## clang移植

关于prebuilts里边的工具链版本情况，谷歌提供了相应的[版本说明](https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/+/refs/heads/main/README.md)。目前最新的发布版本是clang-r530567，发布于2024.7。每年大致会更新3-4个prebuilt版本，每个版本会在上游llvm的基础上添加一些补丁。这个说明里也包括相关源码中如何指定具体编译器版本的方法。例如，对于AOSP大部分地方编译使用的工具链版本，是在build/soong/cc/config/global.go中，通过ClangDefaultVersion变量来指定的。

安卓工具链的源码维护在一个独立的repo manifest里，这里提供了具体的[构建说明](https://android.googlesource.com/toolchain/llvm_android/+/main/BUILD.md)。但其中不少内容都有点过时(例如docker环境应该太老旧了，不能用来编译最新代码)，或者疑似指向谷歌内部环境才能访问的一些链接(例如http://go/android-llvm-update-process)。具体的做法记录在下边。

### 源代码准备

```bash
mkdir llvm-toolchain && cd llvm-toolchain
repo init -u https://android.googlesource.com/platform/manifest -b llvm-toolchain
cp $TOOLCHAIN_DIR/manifest_12085363.xml .repo/manifests  # 设置相应目录，每个自带的clang版本都有对应的manifest文件，用这个文件恢复和prebuilt一样的版本
repo init -m manifest_12085363.xml
repo sync -c
```

这里$TOOLCHAIN_DIR应替换你机器上aosp15源码prebuilts/clang/host/linux-x86/clang-r530567/的相应xml文件。

同步完成之后顶层目录包括如下文件：

bionic  external  prebuilts  toolchain  tools

### 源码修改

1. bionic: 需要添加loongarch支持。 头文件更新到6.6内核，其他地方添加相应的架构支持，具体参考a12的修改。详细过程后续给出。
2. toolchain/llvm_android: 在相关脚本中添加loongarch架构支持。
3. toolchain/llvm-project: 添加loongarch64-linux-android编译target，以及少量其他补丁，如与内核版本有关的补丁(ucontext结构体变化等）
4. toolchain/prebuilts/sysroot/platform目录下，仿照riscv64-linux-android，制作一个loongarch64-linux-android目录，以便编译clang loongarch相关代码。其中新增usr/include/loongarch64-linux-android目录，里边的asm子目录从(修改完之后的）bionic/libc/kernel/uapi/asm-loongarch拷贝。usr/lib/loongarch64-linux-android/目录下需要一些库文件，而其中很多都是暂时不具备的（需要从aosp源码编译，而后者还没有合适的编译器）。这就是工具链添加新架构过程中常见的鸡生蛋、蛋生鸡的问题：编译工具链需要一些库，这些库又依赖新工具链。解决的办法只能是多次迭代：先在没有库的环境下编译最小编译器，然后用它编译部分库，再回头去编译更完整的编译器，直到功能完整。

riscv64 sysroot下对应的库包括：

```bash
10000/  libc.a  libcompiler_rt-extras.a  libdl.a  libm.a  libstdc++.a  libz.a
../../../../riscv64-linux-android/usr/lib/riscv64-linux-android/10000:
crtbegin_dynamic.o  crtend_so.o    libbinder_ndk.so  libEGL.so        libicu.so          libm.so               libOpenMAXAL.so  libvulkan.so
crtbegin_so.o       libaaudio.so   libcamera2ndk.so  libGLESv1_CM.so  libjnigraphics.so  libnativehelper.so    libOpenSLES.so   libz.so
crtbegin_static.o   libamidi.so    libc.so           libGLESv2.so     liblog.so          libnativewindow.so    libstdc++.so
crtend_android.o    libandroid.so  libdl.so          libGLESv3.so     libmediandk.so     libneuralnetworks.so  libsync.so
```

对已经被支持的架构来说，prebuilts提供了完整的库。但是对于还没有被支持的架构来说，prebuilts没有相应的库，需要设法手工加上。在编译clang的早期，真正用到的文件比较有限。一种做法是从对应的gcc/glibc工具链获取相应文件先替代。但是在实际操作中，发现来自gcc/glibc会导致各种链接问题。在缺对应sysroot的情况下，clang编译可以一直进行到stage2编译Android-LoongArch64的device-sysroots target的时候，此时out/stage2-install/下已经有一个半成品的clang，可以用它来编译aosp获得相应的sysroot需要的库文件。

只需要在第一次移植时用这种多次迭代，现在已经有了完整的aosp LoongArch ABI2.0工具链，直接用现成的sysroot结果即可完成工具链编译。

### 半成品clang编译

完成对源码的修改之后，开始编译。

```bash
./toolchain/llvm_android/build.py --no-build windows --skip-tests --build-name r530567
```

如果没有添加sysroot下loongarch64相关的内容，会碰到如下错误：

```bash
INFO:base_builders:Building device-sysroots for Android-LoongArch64 (platform=True static=False None)
Traceback (most recent call last):
  File "/home/foxsen/llvm-toolchain/toolchain/llvm_android/do_build.py", line 1204, in <module>
    main()
  File "/home/foxsen/llvm-toolchain/toolchain/llvm_android/do_build.py", line 1143, in main
    build_runtimes(build_lldb_server=build_lldb,
  File "/home/foxsen/llvm-toolchain/toolchain/llvm_android/do_build.py", line 202, in build_runtimes
    builders.DeviceSysrootsBuilder().build()
  File "/home/foxsen/llvm-toolchain/toolchain/llvm_android/builder_registry.py", line 67, in wrapper
    function(builder, *args, **kwargs)
  File "/home/foxsen/llvm-toolchain/toolchain/llvm_android/base_builders.py", line 176, in build
    self._build_config()
  File "/home/foxsen/llvm-toolchain/toolchain/llvm_android/builders.py", line 1015, in _build_config
    shutil.copytree(src_sysroot / 'usr' / 'include',
  File "internal/python3.11/shutil.py", line 559, in copytree
FileNotFoundError: [Errno 2] No such file or directory: '/home/foxsen/llvm-toolchain/toolchain/prebuilts/sysroot/platform/loongarch64-linux-android/usr/include'
Traceback (most recent call last):
  File "./toolchain/llvm_android/build.py", line 21, in <module>
    py3_utils.run_with_py3('do_build.py')
  File "/home/foxsen/llvm-toolchain/toolchain/llvm_android/py3_utils.py", line 36, in run_with_py3
    [python_bin, os.path.join(THIS_DIR, script_name)] + sys.argv[1:])
  File "/usr/lib64/python3.6/subprocess.py", line 311, in check_call
    raise CalledProcessError(retcode, cmd)
subprocess.CalledProcessError: Command '['/home/foxsen/llvm-toolchain/prebuilts/build-tools/path/linux-x86/python3', '/home/foxsen/llvm-toolchain/toolchain/llvm_android/do_build.py', '--no-build', 'windows', '--skip-tests', '--build-name', 'r530567']' returned non-zero exit status 1.
```

构造sysroot的时候，尽可能让头文件和相应的库更接近最后的状态。usr/include中的头文件可以从移植好的bionic拷贝。得到半成品clang工具链之后，用编译初来的内容(out/stage2-install/覆盖prebuilts.../clang-r530567的相应内容，然后编译移植好aosp(用指定目标的方法跳过一些尚不能编译通过的模块, m libc libm libdl liblog libstdc++ libc++_static libc++abi libz). 编译过程如果报缺clang中的某个loongarch相关库，可以先尝试用空文件替代或者其他已有la库文件，这样可以跳过一些早期的检查，一直到它被实际用于链接才会出错。

TODO: 具体的迭代过程。基本上只需要一次迭代即可，即stage2-install下拷贝的工具链编译出libc.so/libm.so/crtbegin*.o等sysroot需要的文件，拷贝回llvm-toolchain编译所需要的sysroot目录，就可以完成工具链的完整编译了。

### aosp移植

为了提供编译工具链所需要的基础库，我们需要用半成品的clang工具链去编译aosp代码，得到相应的库文件。初步尝试，发现用clang-r530567版本的clang去编译aosp12的bionic等代码会碰到不少由于编译器版本兼容引起的问题。考虑到最终我们也需要升级aosp到最新代码，我们选择了先在aosp15的基础上移植之前在aosp12上做过的loongarch支持。因为已经有aosp的基础，理论上对每个要进行架构支持的代码，我们只需要将aosp12中最新代码和其基础(android-12.0.0.r2)做diff，把补丁应用到aosp15即可。但实践发现，从aosp12到aosp15版本跨度比较大，很多补丁无法简单直接应用，需要根据实际情况重新做移植。


TODO: 补充详细情况。

1. 构建系统. 在安卓的构建系统中添加对loongarch的架构支持。包括各种makefile，脚本和soong等。
2. bionic. aosp15使用6.1或者6.6的linux内核，需要重新导入内核头文件生成bionic相应的头文件，然后根据参考原有的补丁，结合最新版本的情况重新移植相关内容。在aosp12相关补丁不能直接应用的情况下，参考新代码中riscv64的相应处理是一条捷径，由于ABI的相似性，很多时候可以仿照riscv64的实现。
3. art. 安卓编译早期用到art中一些模块，因此也需要先移植。
4. device. 需要添加loongarch的参考设备。aosp15 lunch的目标稍微有些变化，从xxx-eng/userdebug等变为xxx-trunk_staging-eng.


## ndk

有了clang工具链之后，就可以着手编译ndk工具链。

这里是[官方文档](https://android.googlesource.com/platform/ndk/+/master/docs/Building.md)，我们用最新的r27，因此初始化命令是：

```bash
repo init -u https://android.googlesource.com/platform/manifest -b ndk-release-r27 --partial-clone
```

我们目前没有去生成windows和darwin的clang，但是ndk编译过程会去检查相应目前的notice文件以及使用他们生成ndk，repo sync之后，可以用如下处理忽略：

```bash
 cp -a prebuilts/clang/host/linux-x86/clang-r530567b prebuilts/clang/host/darwin-x86/
 cp -a prebuilts/clang/host/linux-x86/clang-r530567b prebuilts/clang/host/windows-x86/
```

update: 2024.10.14, 目前的做法生成ndk时，由于clang-r530567b下边有个.git符号链接会导致最终检查ndk合法性时报错（符号链接指向目录外），更新manifests改为使用含有该目录的linux-x86仓库，然后用linkfile语法在darwin-x86/windows-x86下建立符号链接，可以正常生成。


到aosp的编译环境，lunch loongson_3a5000-trunk_staging-eng，设置好必要的环境变量以便ndk编译找到相应的工具链等。

```bash
cd ndk
../prebuilt/python/linux-x86/bin/python3 ./checkbuild.py --package  --no-build-tests  --system linux --build-number 27
```

如有异常，也可以按照官方的说明安装好python poetry环境，然后在poetry shell中运行相应命令。

```bash
sudo pip3 install --upgrade pip
pip3 install poetry # 注意比较老的python可能会导致问题, poetry版本需要大于1.2.0
~/.local/bin/poetry install # 在ndk子目录下
~/.local/bin/poetry shell
./checkbuild.py --package  --no-build-tests  --system linux --build-number 27
```

正常输出如下：

```bash
./checkbuild.py  --package  --no-build-tests  --system linux --build-number 27
Machine has 24 CPUs
Building modules: black canary-readme changelog clang cpufeatures gtest isort libshaderc make meta mypy native_app_glue ndk-build ndk-build-shortcut ndk-gdb ndk-gdb-shortcut ndk-lldb-shortcut ndk-stack ndk-stack-shortcut ndk-which ndk-which-shortcut ndk_helper pylint pytest python-packages pythonlint readme shader-tools simpleperf source.properties sysroot system-stl toolbox toolchain wrap.sh yasm
Build finished
Packaging NDK...

Installed size: 2070 MiB
Package size: 624 MiB
Finished successfully
Build: 0:03:38
Packaging: 0:04:02
Testing: 0:00:00
Total: 0:07:41
GDBus.Error:org.freedesktop.DBus.Error.ServiceUnknown: The name org.freedesktop.Notifications was not provided by any .service files
```

## rust工具链

rust工具链的编译和llvm-toolchain类似，可以参考[官方文档](https://android.googlesource.com/toolchain/android_rust/+/16b83d21f33d9967fc849ccc5823dbf613581bb6/README.md)。同样的，这个文档也有一些过时和谷歌内部相关的东西。我们的目标是复现安卓使用的rust版本，然后在它基础上添加LA支持。基本流程如下：

1. 初始化一个repo。
2. 让repo用给定版本的android rust manifest
3. 修改源码. 
    a. rustc/bionic/llvm_android/llvm_project/android_rust都需要一定的修改
    b. toolchain/prebuits/sysroot/platform/目录下添加loongarch64-linux-android目录
    c. prebuilts/clang/host/linux-x86/下的clang-r522817目录替换为支持la的clang-r530567版本
4. 编译

```bash
mkdir rust-toolchain; ce rust-toolchain
repo init -u https://android.googlesource.com/platform/manifest -b rust-toolchain
cp $AOSP_ROOT/prebuilts/rust/linux-x86/1.80.1/manifest_12274397.xml .repo/manifests/
repo init -m manifest_12274397.xml
repo sync -c
```

编译安卓prebuilt目录的rust工具可以参考./toolchain/android_rust/artifacts/1.79.0/rust_build_command.linux-x86.12250333.sh，它用了五次迭代来获得最佳性能（含profile feedback优化)。开发阶段可以用简化的两阶段编译：

```
./prebuilts/python/linux-x86/bin/python3 toolchain/android_rust/tools/build.py --build-name linux-12274397 --llvm-linkage shared --lto thin --ndk <path-to-your-android-ndk>
```

ndk应该指向上一节生成的带LA支持的ndk。

用系统的python可能会有不兼容的问题，应使用prebuilts/python目录的版本。

## 相关仓库以及复现的manifests

### 相关仓库

1. sysroot: git@github.com:android-la64/loongarch64-linux-android-sysroot.git, main分支
2. llvm_android: git@github.com:android-la64/llvm_android.git, r530567_larch分支, r1.80.1_larch分支(for rust)
3. bionic: git@github.com:android-la64/bionic.git,r530567_larch分支(for llvm-toolchain), a15_larch分支(for aosp15)
4. llvm-project: git@github.com:android-la64/llvm-project.git, r530567_larch分支(for clang), r1.80.1_larch分支(for rust)
5. art: git@github.com:android-la64/platform-art.git, a15_larch分支
6. build/soong: git@github.com:android-la64/aosp-platform-soong.git
7. build/make: git@github.com:android-la64/aosp-platform-build.git, a15_larch分支
8. device/loongson/loongsonboard: ssh://git@8.140.33.210:2222/android/loongsonboard.git, a15_larch分支
9. rustc: git@github.com:android-la64/rust.git, a15_larch分支
10. prebuilt clang with loongarch64 support: ssh://git@8.140.33.210:2222/android/clang-prebuilt-loongarch64.git, main分支
11. prebuilt rust with loongarch64 support: git@github.com:android-la64/rust-prebuilt-loongarch64.git, main分支
12. ndk: git@github.com:android-la64/platform-ndk.git, ndk-release-r27-larch分支, 这是ndk源码编译环境的ndk子目录
13. android_rust: ssh://git@8.140.33.210:2222/android/android_rust.git, r1.80.1_larch分支
14. kernel-headers: git@github.com:android-la64/kernel-headers.git, a15_larch分支
15. prebuilt-ndk: ssh://git@8.140.33.210:2222/android/prebuilt-ndk.git, a15_larch分支, 这是aosp源码的prebuilts/ndk目录内容
16. prebuilt-clang-linux-x86: ssh://git@8.140.33.210:2222/android/platform-prebuilts-clang-host-linux-x86.git, a15_larch分支
17. prebuilt-rust: ssh://git@8.140.33.210:2222/android/prebuilt-rust.git, a15_larch分支
18. ndk-prebuilts-ndk: ssh://git@8.140.33.210:2222/android/ndk-prebuilts-ndk.git, a15_larch分支。这是ndk源码编译环境的prebuilts/ndk
19. prebuilts-build-tools: ssh://git@8.140.33.210:2222/android/prebuilts-build-tools.git, a15_larch分支。目前只更新了一个create_minidebuginfo文件

### manifests

1. llvm-toolchain. 
 repo init -u ssh://git@github.com/android-la64/manifest -b llvm-toolchain-12085363
2. ndk.
 repo init -u ssh://git@github.com/android-la64/manifest -b ndk-release-r27-larch
3. rust-toolchain.
 repo init -u ssh://git@github.com/android-la64/manifest -b rust-toolchain-larch
4. aosp15.
 repo init -u ssh://git@github.com/android-la64/manifest -b a15-larch
