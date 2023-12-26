# 1. 环境设置

操作系统：Ubuntu 20.04 或 22.04

本章描述的步骤都是一次性的 - 除了`export`环境变量的命令需要放置到一个特定的文件或者`.bashrc`

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
export LA_WS=xxx/loongson
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

任何人需要初始源代码，只要执行2.1.1 ~ 2.1.2中的**任何一个步骤**即可！！！



### 2.1 【熵核内部】直接解压文件

```bash
## 将 /back/comm_back/1-android/clang15/clang_la-android.src.tar 解压到$ATOOLCHAIN_WS
cd $LA_WS
tar xf /back/comm_back/1-android/loongarch/clang15.src.la.tar

cd clang_la
repo sync -l -c  ## 一定要做这个步骤

## 从github上同步最新的代码 - 需要设置好ssh -- 参考2.2节的前两个命令
repo sync -c
```



### 2.2 下载源码（清华 + github）

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

Tips - 提交代码：必须在本地切换为`a12_larch`分支。

```bash
[^_^llvm-project]$ git br
* (no branch)

(/data1/wendong/loongson/clang_la/toolchain/llvm-project @ xcvm 12:47:28)
[^_^llvm-project]$ git br -r
  android-la64/a12_larch
  m/clang-r468909b -> android-la64/a12_larch
```



# 3. Clang15 编译与回归测试

## 3.1 编译Linux版本

使用Python3脚本build.py编译源代码，添加选项--no-build lldb跳过lldb的编译。按如下步骤编译源代码：

```bash
cd $LA_WS/clang_la

## 编译整个Clang工具链，不编译lldb
## 在服务器上约70分钟，内存使用不太多。64核，< 20G内存
##
python toolchain/llvm_android/build.py --lto --pgo --bolt --no-build windows,lldb --build-name r468909b

```

上述编译也进行了部分回归测试（参考第三章），如下：

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

从脚本`builders.py`中可以看到，回归测试其实就是执行了如下脚本

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

我们当前暂时禁用了部分Clang模块的编译，这部分需要龙芯继续，最终版本还是需要这些模块。

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



# 4. Clang15 单元测试

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

# done, go back to riscv-llvm-src working directory
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



### 4.20 新标记失败的测试用例

```bash

```





# 5. Clang15 测试集测试 - TODO(这部分还没有准备好)

LLVM测试集即LLVM test-suite，源代码独立于llvm-project外，代码仓库位于：https://github.com/llvm/llvm-test-suite

 由于编译的Android CLANG大版本号为15.0.3，因此我们采用对应的测试集为：

​	https://github.com/llvm/llvm-project/releases/download/llvmorg-15.0.3/test-suite-15.0.3.src.tar.xz



### 5.1 下载测试集

```bash
cd $ATOOLCHAIN_WS

## 解压测试集
tar xfJ test-suite-15.0.3.src.tar.xz
```



<span style="color:red">注意：以下测试host/qemu: 请确保执行了4.01节的脚本。</span>



### 5.2 Host 测试

#### 5.2.1 测试步骤

```bash
$ cd $ATOOLCHAIN_WS/

$ mkdir test-suite-build-host
$ cd test-suite-build-host

## 将以下内容copy为一个文件，比如 testsuit.env.host.sh
TEST_C=$CLANG_OUT/stage2/bin/clang
TEST_CXX=$CLANG_OUT/stage2/bin/clang++
TEST_FLAGS="-lstdc++ -lm"
TEST_CONFIG="../test-suite-15.0.3.src/cmake/caches/Debug.cmake"
TEST_LIT=$CLANG_OUT/stage2/bin/llvm-lit

cmake -DCMAKE_C_COMPILER="$TEST_C" \
        -DCMAKE_CXX_COMPILER="$TEST_CXX" \
        -DCMAKE_C_FLAGS="$TEST_FLAGS" \
        -DCMAKE_CXX_FLAGS="$TEST_FLAGS" \
        -DTEST_SUITE_LIT="$TEST_LIT" \
        -C$TEST_CONFIG \
        -G Ninja \
        ../test-suite-15.0.3.src
```

在`test-suite-build-host`目录下执行

```bash
## 生成ninja编译文件
source testsuit.env.host.sh
```



#### 5.2.2 测试

```bash
## 运行测试集
$ ninja check
```



#### 5.2.3 测试结果

| 测试配置             | 编译（Pass） | 编译（Fail） | 运行（Pass） | 运行（Fail） |
| -------------------- | :----------- | ------------ | ------------ | ------------ |
| CodeSize.cmake（Os） | 2555         | 0            | 2498         | 0            |
| Debug.cmake  (O0-g)  | 2555         | 0            | 2498         | 0            |
| MinSize.cmake（Oz）  | 2555         | 0            | 2498         | 0            |
| O0.cmake             | 2555         | 0            | 2498         | 0            |
| O3.cmake             | 2555         | 0            | 2498         | 0            |
| Os-g.cmake           | 2555         | 0            | 2498         | 0            |
| OsLTO.cmake          | 2555         | 0            | 2498         | 0            |
| Release.cmake        | 2555         | 0            | 2498         | 0            |
| ReleaseLTO.cmake     | 2555         | 0            | 2498         | 0            |
| ReleaseLTO-g.cmake   | 2555         | 0            | 2498         | 0            |
| ReleaseNoLTO.cmake   | 2555         | 0            | 2498         | 0            |
| ReleaseThinLTO.cmake | 2555         | 0            | 2498         | 0            |



### 5.3 Qemu测试

#### 5.3.1 环境准备

环境准备的工作在某个机器上只做一次即可。

```bash
# install riscv cross compiler and binutils
$ sudo apt install binutils-riscv64-linux-gnu cpp-riscv64-linux-gnu g++-riscv64-linux-gnu gcc-riscv64-linux-gnu

# install riscv cross glibc
$ sudo apt install libatomic1-riscv64-cross libc6-dev-riscv64-cross libc6-riscv64-cross linux-libc-dev-riscv64-cross

# install riscv cross libgcc according to cross gcc version
$ sudo apt install libgcc-8-dev-riscv64-cross libgcc1-riscv64-cross

# install riscv cross libstdc++ according to cross g++ version
$ sudo apt install libstdc++-8-dev-riscv64-cross libstdc++6-riscv64-cross

# install qemu user-mode emulator
$ sudo apt install qemu-user
```

平头哥的qemu：

```bash
## 检查 d-linux-riscv64v0p7_xthead-lp64d.so.1链接
## /lib/ld-linux-riscv64v0p7_xthead-lp64d.so.1
## 应该类似：
## /lib/ld-linux-riscv64v0p7_xthead-lp64d.so.1 -> /usr/riscv64-linux-gnu/lib/ld-linux-riscv64-lp64d.so.1*
## 当链接不存在时执行
$ sudo ln -s /usr/riscv64-linux-gnu/lib/ld-linux-riscv64-lp64d.so.1 /lib/ld-linux-riscv64v0p7_xthead-lp64d.so.1

## 检查qemu-riscv64-thead链接
## 公司内部这个文件位于 /back/comm_back/1-android/qemu-riscv64-thead
##    将上述文件拷贝到自己的PATH路径目录即可
## 公司外部：这个文件就是平头哥自己编译的qemu-riscv64的软连接
$ /data/home/wendong/.bin/qemu-riscv64-thead --version
qemu-riscv64 version 6.0.94
Copyright (c) 2003-2021 Fabrice Bellard and the QEMU Project developers
```



#### 5.3.2 编译

```bash
$ cd $ATOOLCHAIN_WS/

$ mkdir test-suite-build-qemu
$ cd test-suite-build-qemu

## 将以下内容copy为一个文件，比如 testsuit.env.qemu.sh
#!/bin/bash

TEST_C=$CLANG_OUT/stage2-install/bin/clang
TEST_CXX=$CLANG_OUT/stage2-install/bin/clang++
TEST_LIT=$CLANG_OUT/stage2/bin/llvm-lit    ## Note: stage2, not stage2-install
TEST_STRIP=$CLANG/stage2-install/bin/llvm-strip

## 特别注意下面这一行需要改动： qemu-riscv64 路径
TEST_RUN_UNDER="<path to thead qemu>/qemu-riscv64-thead -L /usr/riscv64-linux-gnu"

## 此处在后续需要增加参数  -mcpu=c920
TEST_FLAGS="-target riscv64-unknown-linux -I ./include -I/usr/riscv64-linux-gnu/include -I/usr/riscv64-linux-gnu/include/c++/9/riscv64-linux-gnu -L/usr/riscv64-linux-gnu/lib"

TEST_SUITE_SRC=../test-suite-15.0.3.src
TEST_CONFIG="$TEST_SUITE_SRC/cmake/caches/CodeSize.cmake"

cmake   -DCMAKE_C_COMPILER="$TEST_C" \
        -DCMAKE_CXX_COMPILER="$TEST_CXX" \
        -DCMAKE_C_FLAGS="$TEST_FLAGS" \
        -DCMAKE_CXX_FLAGS="$TEST_FLAGS" \
        -DCMAKE_STRIP="$TEST_STRIP" \
        -DTEST_SUITE_LIT="$TEST_LIT" \
        -DTEST_SUITE_RUN_UNDER="$TEST_RUN_UNDER" \
        -DTEST_SUITE_USER_MODE_EMULATION=True \
        -C$TEST_CONFIG \
        -G Ninja \
        $TEST_SUITE_SRC
```



在`test-suite-build-qemu`目录下执行

```bash
## 生成ninja编译文件
$ source testsuit.env.qemu.sh
```



#### 5.3.3 测试

```bash
## 运行测试集
$ ninja check   ## 大概10-15分钟
```



#### 5.3.4 测试结果

| 测试配置             | 编译（Pass） | 编译（Fail） | 运行（Pass） | 运行（Fail） |
| -------------------- | ------------ | ------------ | ------------ | ------------ |
| CodeSize.cmake（Os） | Y            | 0            | 2026         | 0            |
| Debug.cmake  (O0-g)  | Y            | 0            | 2026         | 0            |
| MinSize.cmake（Oz）  | Y            | 0            | 2026         | 0            |
| O0.cmake             | Y            | 0            | 2026         | 0            |
| O3.cmake             | Y            | 0            | 2026         | 0            |
| Os-g.cmake           | Y            | 0            | 2026         | 0            |
| OsLTO.cmake          | Y            | 0            | 2026         | 0            |
| Release.cmake        | Y            | 0            | 2026         | 0            |
| ReleaseLTO.cmake     | Y            | 0            | 2026         | 0            |
| ReleaseLTO-g.cmake   | Y            | 0            | 2026         | 0            |
| ReleaseNoLTO.cmake   | Y            | 0            | 2026         | 0            |
| ReleaseThinLTO.cmake | Y            | 0            | 2026         | 0            |





# 6. NDK_r23

通过访问以下链接可得到NDK官方文档所列的编译步骤：https://android.googlesource.com/platform/ndk/+/master/docs/Building.md

### 6.1 源代码

 <span style="color:red">如果是在已有正确测试的环境下重新执行代码打补丁、测试，则本步骤不需要做，但是需要做附录A.2 中的步骤！！！</span>

<span style="color:green">如果是仅仅重新测试（代码不变），则直接进入6.2节！</span>



#### 6.1.1 【内部】- 直接解压文件

```bash
## create a directory for building NDK
$ cd $LA_WS

# This command will extract all files to ndk23/
$ tar xfJ /back/comm_back/1-android/loongarch/ndk23.src.la.xz

## checkout code MUST add '-l'
$ cd ndk23
$ repo sync -l -c

## !!! 特别注意，最后要再次sync为最新代码 -- 此时只会从github上下载代码，理论上可以不翻墙
$ repo sync
```



#### 6.1.2 从github下载源代码

```bash
## create a directory for building NDK
$ cd $LA_WS
$ mkdir ndk23 && cd ndk23

## initialize repo
$ repo init -u git@github.com:android-la64/manifest.git -b ndk-r23-larch

## synchronize repo
$ repo sync -c
```



### 6.2 编译前设置

#### 6.2.2  切换Clang到新编译的15.0.3

```bash
## Linux
cd $LA_WS/ndk23/prebuilts/clang/host/linux-x86
ln -s ../../../../../clang-15.0.3/out/install/linux-x86/clang-r468909b  clang-r468909b  ##!!
## ！！此处一定不能使用绝对路径进行ln -s操作，否则会导致编译失败！！
## 只有使用相对路径ln，或者直接copy

## fix symbol links for builind NDK tests
cd $LA_WS/ndk23/prebuilts/clang/host/linux-x86/clang-r468909b/lib/x86_64-unknown-linux-gnu
ln -s libc++.so.1 libc++.so.1.0
ln -s libc++abi.so.1 libc++abi.so
ln -s libc++abi.so.1 libc++abi.so.1.0
```



#### 6.2.3 切换Python到Clang 15.0.3中的Python3

```bash
cd $LA_WS/ndk23/prebuilts
mv python3 python3-ori  ## Due to this file too OLD
ln -s $LA_WS/clang-15.0.3/prebuilts/python python3
cd python3
ln -s linux-x86 linux

cd $LA_WS/ndk23
```

以上命令可以copy到一个脚本文件中，用source的方式执行。



### 6.3 编译、打包以及测试

其中本部分的测试需要进一步修订

```bash
cd $LA_WS/ndk23

## 编译Linux版本
unset OUT_DIR

## 根据机器的性能确定xx数字：2,8,16等
python ndk/checkbuild.py -j2  --package  --no-build-tests  --system linux --build-number 23
# python ndk/checkbuild.py -j2 --package --system linux --build-number 23


## 结果应该显示 -- TODO
build: PASS 892/1000     FAIL 0/1000  SKIP 108/1000
device: PASS 330/355     FAIL 0/355   SKIP 25/355
libc++: PASS 31380/31380 FAIL 0/31380 SKIP 0/31380
PASS 32602/32735         FAIL 0/32735 SKIP 133/32735
```

打包后的NDK文件
- Linux：`out.rv/dist/android-ndk-r23-linux-x86_64.zip`




### 6.4 Device上测试 - TODO

NDK 测试作为NDK正常构建的一部分（使用 checkbuild.py）并通过 run_tests.py 运行。

```bash
# Build the NDK and tests
# ndk/checkbuild.py

cd $ATOOLCHAIN_WS/ndk23

export ANDROID_SERIAL=1234567890

##strip exe files
cp $ATOOLCHAIN_WS/aosp12-clang15-patch/ndk-r23/strip_exe.sh .
bash strip_exe.sh out

## copy test config file
cd out
cp $ATOOLCHAIN_WS/aosp12-clang15-patch/ndk-r23/qa_config_thead.json .

unset OUT_DIR
python ../ndk/run_tests.py --abi riscv64 --clean-device --config qa_config_thead.json | tee test_log.txt


```



# 7. Rust

###  7.1 源代码

#### 7.1.1 【内部】- 直接解压文件

#### 7.1.2 从github下载源代码

```shell
$ cd $LA_WS

## 克隆代码
git clone -b a12_larch git@github.com:android-la64/rust.git && cd rust
```



### 7.2 编译前设置

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



### 7.3 编译、打包

```shell
## 进入rust目录
cd $LA_WS/rust

## 编译
./android_build/build.sh
```

编译完成后，会自动完成打包，创建的包放在**build/dist**目录下。AOSP编译主要用到下面两个包：

```shell
## binary
ls -lh build/dist/rust-dev-1.51.0-dev-x86_64-unknown-linux-gnu.tar.xz
-rw-r--r-- 1 user group 38M 12月 24 17:46 build/dist/rust-dev-1.51.0-dev-x86_64-unknown-linux-gnu.tar.xz

## source for android
ls -lh build/dist/rust-src-android.tar.xz
-rw-r--r-- 1 user group 2.4M 12月 26 15:22 build/dist/rust-src-android.tar.xz
```



# A. 内部测试用的额外步骤

本章描述的步骤只用于<span style="color:red">熵核</span>内部测试人员反复进行clang、ndk测试时使用。



### A.1 Clang测试

这个步骤只是为了将跳过  <span style="color:red">2.1</span> 节的步骤 - 节省测试时间（不删除out.rv，节约编译时间）。

```bash
$ cd $ATOOLCHAIN_WS/clang_la
$ rm -rf bionic*

## 恢复到2.1节之后的初始状态
$ repo sync -l -c

    toolchain/llvm-project/: discarding 32 commits
    toolchain/llvm_android/: discarding 2 commits
    Checking out: 100% (27/27), done in 55.002s
    repo sync has finished successfully.

## 之后就可以从2.2节开始
```



### A.2 NDK测试

这个步骤只是为了将跳过  <span style="color:red">6.1</span> 节的步骤 - 节省测试时间。

```bash
cd $ATOOLCHAIN_WS/ndk23

## 恢复到6.1节之后的初始状态
rm -rf bionic*

cd $ATOOLCHAIN_WS/ndk23/prebuilts
rm -rf python3
mv python3-ori python3

cd -
repo sync -l -c

## 之后就可以从6.2节开始
```



# B. 参考

1. Clang15 Release notes： https://releases.llvm.org/15.0.0/tools/clang/docs/index.html
2. Android Clang build doc: https://android.googlesource.com/toolchain/llvm_android/
3. Clang 测试集测试： https://www.llvm.org/docs/TestSuiteGuide.html
4. Clang prebuild for AOSP: https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/+/master/README.md
5. Android官方NDK网址： https://developer.android.com/ndk/
6. Android官方NDK编译说明：https://android.googlesource.com/platform/ndk/+/master/docs/Building.md
7. RISCV Android NDK Project仓库地址：https://github.com/riscv-android-src/platform-ndk
8. Android官方NDK说明：https://android.googlesource.com/platform/ndk/+/master/docs

9. NDK测试：https://android.googlesource.com/platform/ndk/+/master/docs/Testing.md
