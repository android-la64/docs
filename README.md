# Android/LoongArch64

本项目记录安卓操作系统移植到龙芯平台的过程中各种重要工作的文档。

截止2025年1月，龙芯已经完成AOSP12（ABI 1/旧世界）和 AOSP15(ABI 2/新世界)的初步移植工作。目前，AOSP12可以在模拟器和真机上启动到桌面，AOSP15运行到开机动画出现。已经做的工作包括：

1. 移植工具链。针对旧世界和新世界ABI分别做了一套工具链，前者基于llvm15，后者基于llvm19。
2. 各种安卓软件包添加LoongArch架构支持，包括构建系统以及运行一个基本系统所需要的大部分软件包。其中工作量相对较大的包括Bionic库和Art虚拟机等。

## 文档结构

```bash

├── 01-toolchain.md						： 工具链编译文档
├── 02-bionic.md							： bionic移植、编译、测试文档
├── 03-Android12_setup.md			： AOSP整体设置、编译、烧录、运行步骤
├── 04-debug.md								： qemu运行以及各种移植过程中的调试技巧
├── 05-Notes-for-art.md				； 进一步移植JIT、OAT的注意事项
├── toolchain-ABI2.md					:  build toolchain for ABI version 2
```

其中01-05开头的文档针对AOSP12编写， AOSP15可以参考。toolchain-ABI2.md针对AOSP15.

## 编译运行

安卓12系统编译运行可以参考03-Android12_setup.md和安卓官方文档。由于绝大部分开源社区软件的LoongArch支持是面向新世界的，我们鼓励大家重点玩AOSP15.

安卓15的编译运行过程如下：

### 源代码 - github同步源代码

```bash
## initialize repo
$ repo init -u git@github.com:android-la64/manifest.git -b a15_larch

## synchronize repo
$ repo sync -c
```

### 配置

```bash
. build/envsetup.sh
lunch aosp_cf_x86_64_phone-trunk_staging-eng # 官方脚本似乎有点bug，直接lunch无法显示选择菜单
build_build_var_cache
lunch loongson_3a5000-trunk_staging-eng
```

### 编译

m

### 运行

安卓的运行可以参考[这个文档](http://8.140.33.210:2280/android/android_qemu_env)。其中aosp15目录下的内容是运行aosp15所需要的一些脚本、内核二进制等。

## 内核

内核源码位于[内核仓库](git@github.com:android-la64/kernel_common.git)，缺省配置文件参见上述仓库。

## TODO

安卓是一个巨大的项目，目前的工作还十分初步，有大量需要进一步完善的地方。以下是一些下一步可能着手的方向：

* 安卓15系统启动到桌面
* 虚拟机的JIT和AOT模式。龙芯实验室一个同学已经完成了JIT模式支持所需要的大部分工作，能够通过几乎所有测试了。AOT还在进行中。这个对性能至关重要。
* 大量软件包的测试和进一步优化。由于时间关系，目前很多支持还没有得到充分测试，代码和文档也比较粗糙。
* 安卓系统在PC类硬件的配置完善和优化。安卓最适合的安装对象是移动设备，不过目前龙芯暂时没有手机类的产品线，我们是在龙芯PC上进行测试。现在的版本还缺很多东西，比如声音支持、有线网络的配置界面等。之前有一个开源项目android-x86做了很好的工作，让安卓在PC上能得到比较好的适配。可惜这个项目没有在继续维护。不过还有[BlissOS](https://blissos.org/)等衍生项目可以参考。BTW, 安卓15的一个卖点是向桌面支持迈进了一大步：支持大屏幕多任务处理。可以预见，未来安卓在桌面上会越来越不违和。
* 安卓与Linux桌面的集成。龙芯实验室有同学在尝试移植waydroid，已经能够初步启动到界面。也许不久的将来，我们就可以在龙芯PC兼容运行很多安卓应用了。
* 安卓应用二进制翻译支持。要完整支持安卓生态，还需要二进制翻译，因为很多安卓软件会以ARM动态库形式提供部分功能。谷歌最近开源berberis软件，以便在x86上兼容运行riscv版安卓应用。龙芯实验室有位同学致力于在其基础上实现ARM安卓应用在LoongArch的运行。
* ...

同时，安卓也是一个设计精细和久经考验的产品级操作系统，有非常多值得学习和研究之处。安卓向一个全新架构的移植非常锻炼能力，欢迎大家来参与。

Happy Hacking!

