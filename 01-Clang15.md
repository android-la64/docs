# 1. 环境设置

操作系统：Ubuntu 20.04 或 22.04

本章描述的步骤都是一次性的 - 除了`export`环境变量的命令需要放置到一个特定的文件或者`.bashrc`

### 1.1 工作目录

只做一次的操作：

```bash
## Workspace
mkdir xxx; cd xxx ## xxx为自行设置的目录
mkdir toolchain_ws
## export 本目录 为 ATOOLCHAIN_WS

## Clang15 fold
mkdir -p $ATOOLCHAIN_WS/clang-15.0.3

## ndk fold
mkdir -p $ATOOLCHAIN_WS/ndk23
```

设置环境变量（建议设置到 `.bashrc`中， 或者`$ATOOLCHAIN_WS`目录下的`myenv.sh`中）

```bash
## 以下几行需要设置为环境变量
export ATOOLCHAIN_WS=xxx/toolchain_ws
#export OUT_DIR=out.rv
export CLANG_OUT=$ATOOLCHAIN_WS/clang-15.0.3/out
export XZ_DEFAULTS="-T 0"  ## Only for xz compress
```



### 1.2 repo工具

```bash
## clone repo
cd $ATOOLCHAIN_WS
git clone https://mirrors.tuna.tsinghua.edu.cn/git/git-repo

## create a directory to hold repo executable
mkdir ~/.bin
## Added this line to latest line of you env: ~/.bashrc
export PATH=$HOME/.bin:$PATH   ## $HOME/.bin MUST be the first one

## link repo
ln -s $ATOOLCHAIN_WS/git-repo/repo ~/.bin/repo
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



### 2.1 初始源代码

有三种获取初始源代码的方式 - 其中第二种方式（从清华获取）作为目前主要方式。

任何人需要初始源代码，只要执行2.1.1 ~ 2.1.3中的**任何一个步骤**即可！！！



 <span style="color:red">如果是在已有正确测试的环境下重新执行代码打补丁、测试，则本步骤不需要做，但是需要做附录A.1 中的步骤！！！</span>

<span style="color:green">如果是仅仅重新测试（代码不变），则直接进入2.2节！</span>



#### 2.1.1 【内部】直接解压文件

```bash
## 将 /back/comm_back/1-android/clang-15.0.3-android.src.qh.orig.xz 解压到$ATOOLCHAIN_WS
cd $ATOOLCHAIN_WS
tar xfJ /back/comm_back/1-android/clang15/clang-15.0.3-android.src.qh.orig.xz

cd clang-15.0.3
repo sync -l -c  ## 一定要做这个步骤
```



#### 2.1.2 下载源码（清华）

```bash
## Clang15 toolchain source fold
cd $ATOOLCHAIN_WS/clang-15.0.3

## initialize repo
## 一定要确认python版本的正确性
repo init -u https://mirrors.tuna.tsinghua.edu.cn/git/AOSP/platform/manifest -b llvm-toolchain

## copy manifest file for Clang 15.0.3
cp ../aosp12-clang15-patch/manifests/manifest_9146769.xml ./.repo/manifests/

## re-initialize repo
repo init -m manifest_9146769.xml

## synchronize repo
repo sync -c
```



#### 2.1.3【内部/外部】下载源代码 - LoongArch



```bash
## 待添加
```





### 2.2 代码打补丁

下述步骤只是临时方式，当代码都提交到thead上之后，只有`repo sync`就可以了，不需要这里描述的打补丁步骤！！





#### 2.2.2 替换NDK23

由于Clang15.0.3自带的NDK是r24，需要替换为r23版本。

```bash
cd $ATOOLCHAIN_WS/clang-15.0.3

tar xfJ ../aosp12-clang15-patch/clang-15.0.3/ndk_r23.xz
```



#### 2.2.3 替换Bionic

<span style="color:red">注意</span>： 因为aosp中采用了clang12.0.3中一致的bionic，但是clang15.0.3中的bioinc是更新的版本，为了减少bionic更多对aosp的影响，我们在clang15中必须采用平头哥Clang12中自带的bioinc。

```bash
cd $ATOOLCHAIN_WS/clang-15.0.3
mv bionic bionic-ori

tar Jxf ../aosp12-clang15-patch/bionic.xz
```





# 3. Clang15 编译与回归测试

## 3.1 编译Linux和Windows版本

使用Python3脚本build.py编译源代码，添加选项--no-build lldb跳过lldb的编译。按如下步骤编译源代码：

```bash
cd $ATOOLCHAIN_WS/clang-15.0.3

## 编译整个Clang工具链，不编译lldb
## 在服务器上约70分钟，内存使用不太多。64核，< 20G内存
##
python toolchain/llvm_android/build.py --lto --pgo --bolt --no-build lldb  --build-name r468909b

## 如果是仅仅需要linux版本，可以简化为
python toolchain/llvm_android/build.py --lto --pgo --bolt --no-build windows,lldb  --build-name r468909b
```



上述编译也进行了部分回归测试（参考第三章），如下：

```bash
[0/955] Running libcxx tests
Testing Time: 856.03s
  Unsupported      :  306
  Passed           : 7262
  Expectedly Failed:   41

[952/955] Running clang_tools regression tests
Testing Time: 24.63s
  Unsupported      :    7
  Passed           : 2498
  Expectedly Failed:    2


[953/955] Running the Clang regression tests
Testing Time: 95.58s
  Skipped          :    33
  Unsupported      :   555
  Passed           : 30622
  Expectedly Failed:    26

[954/955] Running the LLVM regression tests
Testing Time: 70.62s
  Skipped          :    11
  Unsupported      : 12613
  Passed           : 37005
  Expectedly Failed:    80
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



## 3.2 编译macOS版本

```bash
cd $ATOOLCHAIN_WS/clang-15.0.3

python toolchain/llvm_android/build.py --lto --pgo --no-build windows,lldb --build-name r468909b
```



## 3.3 打包

编译完成后，Linux对应的安装文件位于`$OUT_DIR/install/linux-x86`。按如下步骤打包：

```bash
# final install files are in $OUT_DIR/install
cd $CLANG_OUT/install/linux-x86

# 打包生成的Linux版本的编译器，这个xz包可以给到aosp那边使用
tar cfJ ../../../clang15.0.3-linux.bin.xz clang-r468909b

cd $CLANG_OUT/install/windows-x86

# 打包生成的Windows版本的编译器，这个xz包可以给到aosp那边使用
tar cfJ ../../../clang15.0.3-windows.bin.xz clang-r468909b

cd $CLANG_OUT/install/darwin-x86

# 打包生成的macOS版本的编译器，这个xz包可以给到aosp那边使用
tar cfJ ../../../clang15.0.3-darwin.bin.xz clang-r468909b

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
cd $ATOOLCHAIN_WS/clang-15.0.3

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

结果如下

Linux:
```bash
Testing Time: 191.67s
  Skipped          :    33
  Unsupported      :   558
  Passed           : 30656
  Expectedly Failed:    27
```



macOS:

```text
  Skipped          :    34
  Unsupported      :   542
  Passed           : 30635
  Expectedly Failed:    25
```



### 4.03 LLVM单元测试

按以下步骤进行LLVM单元测试：

```bash
# run llvm unit test with ninja
ninja check-llvm
```

结果如下

Linux:
```
[0/1] Running the LLVM regression tests

Testing Time: 80.34s
  Skipped          :    11
  Unsupported      : 12657
  Passed           : 37049
  Expectedly Failed:    79
```



macOS(原始代码也是同样错误）:

```text
Failed Tests (1):
  LLVM :: ExecutionEngine/JITLink/X86/MachO_x86-64_self_relocation_exec.test


Testing Time: 438.31s
  Skipped          :    11
  Unsupported      : 12642
  Passed           : 36970
  Expectedly Failed:    84
  Failed           :     1
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
  Unsupported      :  9433
  Passed           : 11283
  Expectedly Failed:    28
```



### 4.05 LLD单元测试

按以下步骤进行LLD单元测试：

```bash
# run lld unit test with ninja
ninja check-lld
```

测试完成后，得到如下报告：

```
Testing Time: 15.24s
  Unsupported:  440
  Passed     : 2234
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

Testing Time: 17.18s
  Unsupported      :  304
  Passed           : 3205
  Expectedly Failed:   11
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

测试完成后，得到如下报告：

Linux:
```bash
Testing Time: 601.67s
  Unsupported      :  303
  Passed           : 7280
  Expectedly Failed:   41
```

macOS:
```text
  Unsupported      :  245
  Passed           : 7332
  Expectedly Failed:   32
```



### 4.12 Clang_clang-cxx测试

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

测试完成后，得到如下报告：

Linux:
```text
Testing Time: 42.61s
  Unsupported      :    7
  Passed           : 2503
  Expectedly Failed:    2
```

macOS(原始代码也是同样错误）:
```text
Failed Tests (3):
  Clang Tools :: clang-tidy/checkers/performance/trivially-destructible.cpp
  Extra Tools Unit Tests :: clang-change-namespace/./ClangChangeNamespaceTests/ChangeNamespaceTest/NamespaceAliasInAncestorNamespace
  Extra Tools Unit Tests :: clang-change-namespace/./ClangChangeNamespaceTests/ChangeNamespaceTest/NamespaceAliasInOtherNamespace


Testing Time: 47.84s
  Unsupported      :    8
  Passed           : 2495
  Expectedly Failed:    2
  Failed           :    3
```



### 4.16 RVV 0.7.1测试

【暂无单独的测试用例】



### 4.20 熵核标记的测试用例

以下16个应用Clang 15.0.0 THEAD CPU 补丁后新增失败的测试用例被标记为 `Expectedly Failed`:

```bash
- llvm/test/CodeGen/ARM/debug-info-sreg2.ll
- llvm/test/CodeGen/RISCV/THEAD/bswap.ll
- llvm/test/CodeGen/RISCV/THEAD/rvv0p7/vsetvli-insert-crossbb.ll
- llvm/test/CodeGen/RISCV/THEAD/rvv0p7/vsetvli-insert.ll
- llvm/test/CodeGen/RISCV/THEAD/thead-memcpy-expand.ll
- llvm/test/CodeGen/RISCV/vararg-ilp32e.ll
- llvm/test/CodeGen/RISCV/vararg.ll
- llvm/test/CodeGen/X86/conditional-tailcall.ll
- llvm/test/CodeGen/X86/licm-dominance.ll
- llvm/test/CodeGen/X86/x86-repmov-copy-eflags.ll
- llvm/test/DebugInfo/ARM/s-super-register.ll
- llvm/test/tools/llvm-objdump/ELF/ARM/debug-vars-dwarf4-sections.s
- llvm/test/tools/llvm-objdump/ELF/ARM/debug-vars-dwarf4.s
- llvm/test/tools/llvm-objdump/ELF/ARM/debug-vars-dwarf5-sections.s
- llvm/test/tools/llvm-objdump/ELF/ARM/debug-vars-dwarf5.s
- llvm/test/tools/llvm-objdump/ELF/ARM/debug-vars-wide-chars.s
```





# 5. Clang15 测试集测试

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
cd $ATOOLCHAIN_WS

# tar xfJ /back/comm_back/1-android/clang15/ndk-r23.src.github.xz
tar xf /back/comm_back/1-android/clang15/ndk-r23.src.github.tar

## checkout code MUST add '-l'
cd ndk23
repo sync -l -c
```



#### 6.1.2 【外部】 -  从github下载源代码

```bash
## create a directory for building NDK
cd $ATOOLCHAIN_WS
mkdir ndk23 && cd ndk23

## initialize repo
repo init -u https://github.com/riscv-android-src/manifest.git -b riscv-ndk-release-r23

## synchronize repo
repo sync -c
```



### 6.2 编译前设置

#### 6.2.1 打补丁

```bash
cd $ATOOLCHAIN_WS/ndk23

bash $ATOOLCHAIN_WS/aosp12-clang15-patch/apply_patch.sh $ATOOLCHAIN_WS/aosp12-clang15-patch/ndk-r23/patches
```



#### 6.2.2  切换Clang到新编译的15.0.3

##### 6.2.2.1 Linux 和 Windows
```bash
## 在6.2.1节的补丁中，我们已经将 ndk/ndk/toolchains.py 中的 CLANG_VERSION 修订为15.0.3了

## Linux
cd $ATOOLCHAIN_WS/ndk23/prebuilts/clang/host/linux-x86
ln -s ../../../../../clang-15.0.3/out/install/linux-x86/clang-r468909b  clang-r468909b  ##!!
## ！！此处一定不能使用绝对路径进行ln -s操作，否则会导致编译失败！！
## 只有使用相对路径ln，或者直接copy

## Windows
cd $ATOOLCHAIN_WS/ndk23/prebuilts/clang/host/windows-x86
ln -s ../../../../../clang-15.0.3/out/install/windows-x86/clang-r468909b clang-r468909b

## fix symbol links for builind NDK tests
cd $ATOOLCHAIN_WS/ndk23/prebuilts/clang/host/linux-x86/clang-r468909b/lib/x86_64-unknown-linux-gnu
ln -s libc++.so.1 libc++.so.1.0
ln -s libc++abi.so.1 libc++abi.so
ln -s libc++abi.so.1 libc++abi.so.1.0
```



##### 6.2.2.2 macOS

首先将3.3节打包的clang15.0.3-linux.bin.xz从Linux机器拷贝到Mac机器

```bash
cd $ATOOLCHAIN_WS
tar Jxf clang15.0.3-linux.bin.xz

cd $ATOOLCHAIN_WS/ndk23/prebuilts/clang/host/linux-x86
ln -s $ATOOLCHAIN_WS/clang-r468909b clang-r468909b

cd $ATOOLCHAIN_WS/ndk23/prebuilts/clang/host/darwin-x86
mv clang-r468909b clang-r468909b-ori
ln -s $CLANG_OUT/install/darwin-x86/clang-r468909b clang-r468909b

cd $ATOOLCHAIN_WS/ndk23/prebuilts/clang/host/windows-x86
ln -s $ATOOLCHAIN_WS/ndk23/prebuilts/clang/host/darwin-x86/clang-r468909b-ori clang-r468909b
```



#### 6.2.3 切换Python到Clang 15.0.3中的Python3

```bash
cd $ATOOLCHAIN_WS/ndk23/prebuilts
mv python3 python3-ori
ln -s $ATOOLCHAIN_WS/clang-15.0.3/prebuilts/python python3
cd python3
ln -s linux-x86 linux

cd $ATOOLCHAIN_WS/ndk23
```

以上命令可以copy到一个脚本文件中，用source的方式执行。



#### 6.2.4 bionic

将 `bionic`切换为最新版本

```bash
cd $ATOOLCHAIN_WS/ndk23
mv bionic bionic-ori

tar Jxf $ATOOLCHAIN_WS/aosp12-clang15-patch/bionic.xz
```



### 6.3 编译、打包以及测试

```bash
cd $ATOOLCHAIN_WS/ndk23

## 编译Linux版本
unset OUT_DIR
python ndk/checkbuild.py -jxx --package --build-number 23      ## 根据机器的性能确定xx数字：2,8,16等


## 编译Windows版本
python ndk/checkbuild.py -jxx --package --build-number 23 --system windows64


## 编译macOS版本
python ndk/checkbuild.py -jxx --package --build-number 23


## 结果应该显示
build: PASS 892/1000     FAIL 0/1000  SKIP 108/1000
device: PASS 330/355     FAIL 0/355   SKIP 25/355
libc++: PASS 31380/31380 FAIL 0/31380 SKIP 0/31380
PASS 32602/32735         FAIL 0/32735 SKIP 133/32735
```

打包后的NDK文件
- Linux：`out.rv/dist/android-ndk-r23-linux-x86_64.zip`
- Windows：`out.rv/dist/android-ndk-r23-windows-x86_64.zip`
- macOS：`out.rv/dist/android-ndk-r23-darwin-x86_64.zip`，`android-ndk-r23-app-bundle.zip`




### 6.4 Device上测试

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



#### 6.4.1 Device运行结果（20230819）

```bash
## check output
grep PASS test_log.txt
# PASS 5296/5304         FAIL 7/5304 SKIP 1/5304
# libc++: PASS 5202/5206 FAIL 3/5206 SKIP 1/5206
# ndk-build: PASS 64/68  FAIL 4/68 SKIP 0/68
# cmake: PASS 30/30      FAIL 0/30 SKIP 0/30

- libc++ (3)
  - libc++.std/localization/locale.categories/category.numeric/locale.nm.put/facet.num.put.members/put_long_double.pass.cpp
  - libc++.std/input.output/file.streams/fstreams/filebuf.members/close.pass.cpp

- ndk-build (1)
  - test-cpufeatures.test_cpufeatures
```





# A. 内部测试用的额外步骤

本章描述的步骤只用于<span style="color:red">熵核</span>内部测试人员反复进行clang、ndk测试时使用。



### A.1 Clang测试

这个步骤只是为了将跳过  <span style="color:red">2.1</span> 节的步骤 - 节省测试时间（不删除out.rv，节约编译时间）。

```bash
$ cd $ATOOLCHAIN_WS/clang-15.0.3
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



### A.3 lit测试介绍

Clang的lit测试，其实就是运行如下的命令（个人环境）：

```
../../prebuilts/python/linux-x86/bin/python3.10 ./bin/llvm-lit -sv --param USE_Z3_SOLVER=0  ../../out.rv/llvm-project/libcxx/test/std/input.output/filesystems/fs.op.funcs/fs.op.exists
```

所以，最后的目录可以任意子目录，这样可以较大程度的缩小运行的测试用例。



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
