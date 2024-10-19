# 1. 环境设置

操作系统：Ubuntu 20.04 或 22.04

本节描述的步骤都是一次性的 - 除了`export`环境变量的命令需要放置到一个特定的文件或者`.bashrc`之外。

### 1.1 工作目录

只做一次的操作：

```bash
## Workspace
mkdir xxx; cd xxx ## xxx为自行设置的目录
mkdir loongson
## export 本目录 为 LA_WS

## Clang15 fold
mkdir -p $LA_WS/clang_la

## ndk fold
mkdir -p $LA_WS/ndk23
```

设置环境变量（建议设置到 `.bashrc`中， 或者`$LA_WS`目录下的`myenv.sh`中）

```bash
## 以下几行需要设置为环境变量
export LA_WS=`pwd`
export CLANG_OUT=$LA_WS/clang_la/out   ## 第四章的测试会用到

export XZ_DEFAULTS="-T 0"  ## Only for xz compress
```



### 1.2 repo工具

```bash
## clone repo
cd $LA_WS
git clone https://mirrors.tuna.tsinghua.edu.cn/git/git-repo

## create a directory to hold repo executable
mkdir ~/.bin
## Added this line to latest line of you env: ~/.bashrc
export PATH=$HOME/.bin:$PATH   ## $HOME/.bin MUST be the first one

## link repo
ln -s $LA_WS/git-repo/repo ~/.bin/repo
export REPO_URL='https://mirrors.tuna.tsinghua.edu.cn/git/git-repo'

## link `python` to `python3` ONLY for you
ln -s /usr/bin/python3 ~/.bin/python
```

上面两条`export`语句，也建议设置到`.bashrc`中

```bash
export PATH=$HOME/.bin:$PATH   ## $HOME/.bin MUST be the first one
export REPO_URL='https://mirrors.tuna.tsinghua.edu.cn/git/git-repo'
```



### 1.3 安装其他系统软件包

#### 1.3.1 Ubuntu Linux
```bash
## System packages
sudo apt install git curl python3 rsync software-properties-common build-essential byacc gcc-multilib g++-multilib zip unzip

## for LLVM test suits
sudo apt install tcl-dev tk-de

## For NDK
sudo apt install libncurses5 python3-distutils file

## 如果需要编译Windows版本的clang，必须按照
sudo apt install mingw-w64

```



# 2. Clang15源代码

有2种获取初始源代码的方式 - 其中第二种方式（从清华+github获取）作为目前主要方式。

任何人需要初始源代码，只要执行2.1 、 2.2中的**任何一个步骤**即可！！！



### 2.1 直接解压文件

```bash
## 将 /back/comm_back/1-android/clang15/clang_la-android.src.tar 解压到$ATOOLCHAIN_WS
cd $LA_WS
tar xf /back/comm_back/1-android/loongarch/clang15.src.la.tar

cd clang_la
repo sync -l -c  ## 一定要做这个步骤

## 从github上同步最新的代码 - 需要设置好ssh -- 参考2.2节的前两个命令
repo sync -c
```



### 2.2 下载源码（清华镜像 + github）

如果不存在2.1中的压缩文件，那么就从头下载。

```bash
##!! 必须先设置好ssh
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_rsa  ## 后面的文件只是一个例子，个人需要换成自己在github上的id
# 注意上述命令需要输入密码！

## Clang15 toolchain source fold
cd $LA_WS/clang_la

## initialize repo
## 一定要确认python版本的正确性!!
repo init -u git@github.com:android-la64/manifest.git -b clang-r468909b

## synchronize repo
repo sync -c
```

注意， 提交代码前必须在本地将对应的代码库切换为`a12_larch`分支。

```bash
[^_^llvm-project]$ git br
* (no branch)

(/data1/wendong/loongson/clang_la/toolchain/llvm-project @ xcvm 12:47:28)
[^_^llvm-project]$ git br -r
  android-la64/a12_larch
  m/clang-r468909b -> android-la64/a12_larch
```



# 3. Clang15 编译与测试

## 3.1 编译Linux版本

使用Python3脚本build.py编译源代码，添加选项--no-build lldb跳过lldb的编译。按如下步骤编译源代码：

```bash
cd $LA_WS/clang_la

## 编译整个Clang工具链，不编译lldb
## 在服务器上约70分钟，内存使用不太多。64核，< 20G内存
##
python toolchain/llvm_android/build.py --lto --pgo --bolt --no-build windows,lldb --build-name r468909b

```

上述编译脚本也进行了部分回归测试（参考第四章），如下：

```bash
[0/955] Running libcxx tests
Testing Time: 856.03s
  Unsupported      :  304
  Passed           : 7282
  Expectedly Failed:   41

[952/955] Running clang_tools regression tests
Testing Time: 24.63s
  Unsupported      :    7
  Passed           : 2506
  Expectedly Failed:    2


[953/955] Running the Clang regression tests
Testing Time: 95.58s
  Skipped          :    33
  Unsupported      :   551
  Passed           : 30479
  Expectedly Failed:    26

[954/955] Running the LLVM regression tests
Testing Time: 70.62s
  Skipped          :    11
  Unsupported      : 12530
  Passed           : 36592
  Expectedly Failed:    64
```

从脚本`builders.py`中可以看到，编译过程中的测试其实就是执行了如下测试脚本：

```python
def test(self) -> None:
    with timer.Timer(f'stage2_test'):
        # newer test tools like dexp, clang-query, c-index-test
        # need libedit.so.*, libxml2.so.*, etc. in stage2/lib.
        self._install_dep_libs(self.output_dir / 'lib')
        self._ninja(
            ['check-clang', 'check-llvm', 'check-clang-tools', 'check-cxx'])
    # Known failed tests:
    #   Clang :: CodeGenCXX/builtins.cpp
    #   Clang :: CodeGenCXX/unknown-anytype.cpp
    #   Clang :: Sema/builtin-setjmp.c
    #   LLVM :: Bindings/Go/go.test (disabled by LLVM_INCLUDE_GO_TESTS=OFF)
    #   LLVM :: CodeGen/X86/extractelement-fp.ll
    #   LLVM :: CodeGen/X86/fp-round.ll
```



#### 3.1.1 特别注意

我们当前暂时禁用了部分可选的Clang模块（通过toolchain/llvm_android`目录下的脚本文件），这部分需要龙芯继续。

```diff
diff --git a/do_build.py b/do_build.py
index 3ede79b..3c8bbff 100755
--- a/do_build.py
+++ b/do_build.py
@@ -172,20 +172,20 @@ def build_runtimes(build_lldb_server: bool):
     builders.BuiltinsBuilder().build()
     builders.LibUnwindBuilder().build()
     builders.PlatformLibcxxAbiBuilder().build()
-    builders.CompilerRTBuilder().build()
-    builders.TsanBuilder().build()
+    # builders.CompilerRTBuilder().build()
+    # builders.TsanBuilder().build()
     # Build musl runtimes and 32-bit glibc for Linux
     if hosts.build_host().is_linux:
         builders.CompilerRTHostI386Builder().build()
         add_lib_links('stage2')
         builders.MuslHostRuntimeBuilder().build()
-    builders.LibOMPBuilder().build()
+    # builders.LibOMPBuilder().build()
     if build_lldb_server:
         builders.LldbServerBuilder().build()
     # Bug: http://b/64037266. `strtod_l` is missing in NDK r15. This will break
     # libcxx build.
     # build_libcxx(toolchain, version)
-    builders.SanitizerMapFileBuilder().build()
+    # builders.SanitizerMapFileBuilder().build()
```



## 3.2 打包

编译完成后，Linux对应的安装文件位于`$OUT_DIR/install/linux-x86`。按如下步骤打包：

```bash
# final install files are in $OUT_DIR/install
cd $CLANG_OUT/install/linux-x86

# 打包生成的Linux版本的编译器，这个xz包可以给到aosp那边使用
tar cfJ ../../../clang15.0.3-linux.bin.xz clang-r468909b

# done. go back to previous working directory
cd -
```



# 4. Clang15 测试

Clang的测试分为三类：

1. 回归测试
2. 单元测试
3. 测试集测试

回归测试与单元测试的代码分别位于`llvm/test`   和  `llvm/unittests` ； 而测试集测试代码位于单独的git中： `https://github.com/llvm/llvm-test-suite.git`。

在本章我们描述回归测试与单元测试，在第5章，描测试集测试相关内容。

### 4.01 测试准备

在进行单元测试前，需要先设置环境。由于最终打包的是stage 2 的CLANG，所以单元测试均在**stage 2**的编译目录中完成。由于stage 2中缺乏部分单元测试所需的库文件，测试前需先复制缺少的文件或建立符号链接。

以下操作以编译工作目录即 `$ATOOLCHAIN_WS/clang-15.0.3` 为初始目录，步骤如下：

```bash
cd $LA_WS/clang_la

# copy libxml2 from stage2-install to stage2
cp $CLANG_OUT/stage2-install/lib/libxml2* $CLANG_OUT/stage2/lib/

# copy libc++ from stage2-install to stage2
cp $CLANG_OUT/stage2-install/lib/libc++.* $CLANG_OUT/stage2/lib/

# copy libc++abi from stage2-install to stage2
cp $CLANG_OUT/stage2-install/lib/libc++abi.* $CLANG_OUT/stage2/lib/

# copy llvm-lit from stage1 to stage2
cp $CLANG_OUT/stage1/bin/llvm-lit $CLANG_OUT/stage2/bin/

# prepare crt files
cd prebuilts/gcc/linux-x86/host/x86_64-linux-glibc2.17-4.8/sysroot/usr/lib

# copy crtbegin.o from GCC 4.8.3 to sysroot
cp ../../../lib/gcc/x86_64-linux/4.8.3/crtbegin.o .

# copy crtend.o from GCC 4.8.3 to sysroot
cp ../../../lib/gcc/x86_64-linux/4.8.3/crtend.o .

# copy libgcc from GCC 4.8.3 to sysroot
cp ../../../lib/gcc/x86_64-linux/4.8.3/libgcc.a .

# copy libgcc_s from GCC lib64 to sysroot
cp ../../../x86_64-linux/lib64/libgcc_s.so.1 .

# done, go back to src working directory
cd -
```

所有测试之前，执行测试的终端中必须先执行一次：

```bash
export LD_LIBRARY_PATH=$CLANG_OUT/stage2/lib:$LD_LIBRARY_PATH
```



### 4.02 Clang单元测试

按以下步骤进行Clang单元测试：

```bash
# run following unit test in $OUT_DIR/stage2 directory
cd $CLANG_OUT/stage2

# run clang unit test with ninja
ninja check-clang
```

结果如下（Linux）:

```bash
Testing Time: 191.67s
  Skipped          :    33
  Unsupported      :   551
  Passed           : 30479
  Expectedly Failed:    26
```



### 4.03 LLVM单元测试

按以下步骤进行LLVM单元测试：

```bash
# run llvm unit test with ninja
ninja check-llvm
```

结果如下（Linux）：

```
[0/1] Running the LLVM regression tests

Testing Time: 80.34s
  Skipped          :    11
  Unsupported      : 12530
  Passed           : 36592
  Expectedly Failed:    64
```



### 4.04 codegen测试

按以下步骤进行codegen单元测试：

```bash
# run codegen unit test with ninja
$ ninja check-llvm-codegen
```

结果如下

```bash
[0/1] Running lit suite /data2/wendong/aclang_toolchain/clang-toolchain/clang-15.0.3/out/llvm-project/llvm/test/CodeGen
Testing Time: 34.61s  
  Unsupported      :  9354
  Passed           : 10851
  Expectedly Failed:    20
```



### 4.05 LLD单元测试

按以下步骤进行LLD单元测试：

```bash
# run lld unit test with ninja
ninja check-lld
```

测试完成后，得到如下报告：

```
Testing Time: 9.46s
  Unsupported      :  440
  Passed           : 2251
  Expectedly Failed:    1
```



### 4.06 crt单元测试

按以下步骤进行crt单元测试：

```bash
# run lld unit test with ninja
ninja check-crt
```

测试完成后，得到如下报告：

```bash
[0/1] Running the CRT tests

Testing Time: 0.20s
  Passed: 2
```



### 4.07 Builtins单元测试

按以下步骤进行Builtins单元测试：

```bash
# run lld unit test with ninja
ninja check-builtins
```

测试完成后，得到如下报告：

```
[0/1] Running the Builtins tests

Testing Time: 2.35s
  Unsupported      :  59
  Passed           : 154
  Expectedly Failed:   1
```



### 4.08 check-llvm-tools测试

按以下步骤进行llvm-tools单元测试：

```bash
# run lld unit test with ninja
ninja check-llvm-tools
```

测试完成后，得到如下报告：

```
[0/1] Running lit suite /data2/wendong/aclang_toolchain/clang-toolchain/clang-15.0.3/out/llvm-project/llvm/test/tools

Testing Time: 6.62s
  Unsupported      :  301
  Passed           : 3211
  Expectedly Failed:    5
```



### 4.09 thinlto测试

按以下步骤进行LLD单元测试：

```bash
# run lld unit test with ninja
ninja check-llvm-thinlto
```

测试完成后，得到如下报告：

```
[0/1] Running lit suite /data2/wendong/aclang_toolchain/clang-toolchain/clang-15.0.3/out/llvm-project/llvm/test/ThinLTO

Testing Time: 4.00s
  Unsupported:   7
  Passed     : 132
```



### 4.10 Clang_C测试

按以下步骤进行Clang_C单元测试：

```bash
# run lld unit test with ninja
ninja check-clang-c
```

测试完成后，得到如下报告：

```
[0/1] Running lit suite /data2/wendong/aclang_toolchain/clang-toolchain/clang-15.0.3/out/llvm-project/clang/test/C

Testing Time: 0.24s
  Passed: 14
```



### 4.11 Clang_cxx单元测试

按以下步骤进行cxx单元测试：

```bash
# run lld unit test with ninja
ninja check-cxx
```

测试完成后，得到如下报告（Linux）:

```bash
Testing Time: 505.39s
  Unsupported      :  304
  Passed           : 7282
  Expectedly Failed:   41
```



### 4.12 clang-cxx测试

按以下步骤进行Clang_Cxx单元测试：

```bash
# run lld unit test with ninja
ninja check-clang-cxx
```

测试完成后，得到如下报告：

```
Testing Time: 1.40s
  Passed           : 833
  Expectedly Failed:   1
```



### 4.13 cxxabi单元测试

按以下步骤进行LLD单元测试：

```bash
# run lld unit test with ninja
ninja check-cxxabi
```

测试完成后，得到如下报告：

```bash
Testing Time: 18.72s
  Unsupported: 15
  Passed     : 56
```



### 4.14 CFI测试 - （原始代码也是同样错误）

按以下步骤进行LLD单元测试：

```bash
# run lld unit test with ninja
ninja check-cfi
```

测试完成后，得到如下报告：

```
Failed Tests (4):
  cfi-devirt-lld-thinlto-x86_64 :: mfcall.cpp
  cfi-devirt-lld-x86_64 :: mfcall.cpp
  cfi-standalone-lld-thinlto-x86_64 :: mfcall.cpp
  cfi-standalone-lld-x86_64 :: mfcall.cpp


Testing Time: 1.82s
  Unsupported      : 208
  Passed           :  40
  Expectedly Failed:   4
  Failed           :   4
```

原始代码的信息如下

```bash
Failed Tests (4):
  cfi-devirt-lld-thinlto-x86_64 :: mfcall.cpp
  cfi-devirt-lld-x86_64 :: mfcall.cpp
  cfi-standalone-lld-thinlto-x86_64 :: mfcall.cpp
  cfi-standalone-lld-x86_64 :: mfcall.cpp


Testing Time: 2.24s
  Unsupported      : 208
  Passed           :  40
  Expectedly Failed:   4
  Failed           :   4
```



### 4.15 check-clang-tools测试

按以下步骤进行clang-tools单元测试：

```bash
# run lld unit test with ninja
ninja check-clang-tools
```

测试完成后，得到如下报告（Linux）:

```text
Testing Time: 42.61s
  Unsupported      :    7
  Passed           : 2506
  Expectedly Failed:    2
```





# 5. NDK_r23

通过访问以下链接可得到NDK官方文档所列的编译步骤：https://android.googlesource.com/platform/ndk/+/master/docs/Building.md

### 5.1 源代码

#### 5.1.1 直接解压文件

```bash
## LA_WS 在第一节中已经定义
$ cd $LA_WS

# This command will extract all files to ndk23/
$ tar xfJ /back/comm_back/1-android/loongarch/ndk23.src.la.xz

## checkout code MUST add '-l'
$ cd ndk23
$ repo sync -l -c

## !!! 特别注意，最后要再次sync为最新代码 -- 此时只会从github上下载代码，理论上可以不翻墙
$ repo sync
```



#### 5.1.2 从github下载源代码

```bash
$ cd $LA_WS
$ mkdir ndk23 && cd ndk23

## initialize repo
$ repo init -u git@github.com:android-la64/manifest.git -b ndk-r23-larch

## synchronize repo
$ repo sync -c
```



### 5.2 编译前设置

#### 5.2.1  切换Clang到新编译的15.0.3

```bash
## Linux
cd $LA_WS/ndk23/prebuilts/clang/host/linux-x86
ln -s ../../../../../clang-15.0.3/out/install/linux-x86/clang-r468909b  clang-r468909b  ##!!
## ！！此处一定不能使用绝对路径进行ln -s操作，否则会导致编译失败！！
## 只能使用相对路径ln，或者直接copy

## fix symbol links for builind NDK tests
cd $LA_WS/ndk23/prebuilts/clang/host/linux-x86/clang-r468909b/lib/x86_64-unknown-linux-gnu
ln -s libc++.so.1 libc++.so.1.0
ln -s libc++abi.so.1 libc++abi.so
ln -s libc++abi.so.1 libc++abi.so.1.0
```



#### 5.2.2 切换Python到Clang 15.0.3中的Python3

```bash
cd $LA_WS/ndk23/prebuilts
mv python3 python3-ori  ## Due to this file too OLD
ln -s $LA_WS/clang-15.0.3/prebuilts/python python3
cd python3
ln -s linux-x86 linux

cd $LA_WS/ndk23
```

以上命令可以拷贝到一个脚本文件中，用source的方式执行。



### 5.3 编译、打包以及测试

其中本部分的测试需要进一步修订

```bash
cd $LA_WS/ndk23

## 编译Linux版本
unset OUT_DIR

## 根据机器的性能确定并行 xx 数字，如：2,8,16等
python ndk/checkbuild.py -j2  --package  --no-build-tests  --system linux --build-number 23
# python ndk/checkbuild.py -j2 --package --system linux --build-number 23


## 结果应该显示
build: PASS 892/1000     FAIL 0/1000  SKIP 108/1000
device: PASS 330/355     FAIL 0/355   SKIP 25/355
libc++: PASS 31380/31380 FAIL 0/31380 SKIP 0/31380
PASS 32602/32735         FAIL 0/32735 SKIP 133/32735
```

打包后的NDK文件
- Linux：`out.rv/dist/android-ndk-r23-linux-x86_64.zip`



# 6. Rust

###  6.1 源代码

```shell
$ cd $LA_WS

## 克隆代码
git clone -b a12_larch git@github.com:android-la64/rust.git && cd rust
```



### 6.2 编译前设置

#### 7.2.1 配置工具链

```shell
## 设置Clang路径
export CLANG_PATH="/your/clang/path"
## 例：
export CLANG_PATH=/data5/yuanjian/loongson/clang_la/out/install/linux-x86/clang-r468909b

## 设置NDK路径
export NDK_PATH="/your/ndk/path"
## 例：
export NDK_PATH=/data5/yuanjian/loongson/ndk23/out/linux/android-ndk-r23c
```



### 6.3 编译、打包

```shell
## 进入rust目录
cd $LA_WS/rust

## 编译
./android_build/build.sh
```

编译完成后，会自动完成打包，创建的包放在**build/dist**目录下。AOSP编译主要用到下面两个包：

```shell
## binary
ls -lh build/dist/rust-1.51.0-dev-x86_64-unknown-linux-gnu.tar.xz
-rw-r--r-- 1 user group 128M 12月 24 17:50 build/dist/rust-1.51.0-dev-x86_64-unknown-linux-gnu.tar.xz

## source for android
ls -lh build/dist/rust-src-android.tar.xz
-rw-r--r-- 1 user group 2.4M 12月 26 15:22 build/dist/rust-src-android.tar.xz
```





# A. 参考

1. Clang15 Release notes： https://releases.llvm.org/15.0.0/tools/clang/docs/index.html
2. Android Clang build doc: https://android.googlesource.com/toolchain/llvm_android/
3. Clang 测试集测试： https://www.llvm.org/docs/TestSuiteGuide.html
4. Clang prebuild for AOSP: https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/+/master/README.md
5. Android官方NDK网址： https://developer.android.com/ndk/
6. Android官方NDK编译说明：https://android.googlesource.com/platform/ndk/+/master/docs/Building.md
7. RISCV Android NDK Project仓库地址：https://github.com/riscv-android-src/platform-ndk
8. Android官方NDK说明：https://android.googlesource.com/platform/ndk/+/master/docs

9. NDK测试：https://android.googlesource.com/platform/ndk/+/master/docs/Testing.md
