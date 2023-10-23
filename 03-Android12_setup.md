本文档的编制是按照交付顺序排列的：1~5章的内容。如果是一位新员工介入此领域，其阅读、设置顺序应该是**A,1,2,3,4,5**。最后的B,C两章用于参考（附录C 描述了在最新Master版本上调试art测试用例的情况 - 与android12有较大形式的变化）。



# 1. 编译riscv64

以附录A为基础形成aosp的代码工作目录之后，可以继续如下步骤。

### 1.1 创建链接

为了方便日常开发测试，我们将常用的脚本放置在 art/csky/tools目录了，但是他们需要在AOSP TOP目录下运行，需要创建如下链接 -- 请自行使用命令创建，结果如下：

```
lrwxrwxrwx 1 wendong wendong  30 2月   3 04:55 brv.sh -> art/csky/tools/build_images.sh
lrwxrwxrwx 1 wendong wendong  26 3月  13 09:05 cleandev.sh -> art/csky/tools/cleandev.sh
lrwxrwxrwx 1 wendong wendong  20 2月   4 09:38 em.sh -> art/csky/tools/em.sh
lrwxrwxrwx 1 wendong wendong  22 2月   3 05:11 push.sh -> art/csky/tools/push.sh
lrwxrwxrwx 1 wendong wendong  27 3月  13 09:05 run_gtest.sh -> art/csky/tools/run_gtest.sh
lrwxrwxrwx 1 wendong wendong  26 3月  13 09:17 run_java.sh -> art/csky/tools/run_java.sh
lrwxrwxrwx 1 wendong wendong  23 2月   3 05:14 rvenv.sh -> art/csky/tools/rvenv.sh
lrwxrwxrwx 1 wendong wendong  18 12月  8 06:43 light.sh -> art/tools/light.sh
```



### 1.2 编译image

编译image以及art测试用例

```bash
## 1. 设置环境，1.1与1.2二选一
#1.1 模拟器： lunch sdk_phone64_riscv64-eng
$ source ./rvenv.sh
#1.2 EVB板子： lunch evb_light-eng
$ source ./light.sh 

## 2. 编译
$ . brv.sh 
## ----其中上面的脚本，包含了下面一些步骤：
## 2.1 编译img
#$ m   ## 注意，大概需要1 小时完成编译

## 2.2 编译测试用例（device上的）
#$ art/tools/buildbot-build.sh --target
```



编译开发板的image，必须是github上的源代码，并使用其中平头哥提供的patches

```bash
source build/envsetup.sh
lunch evb_light-eng
./thead-make.sh -j20

#一般的编译，只需要上面三步

make systemimage -j20
./thead-make.sh gpu
make -j20

# -j20 根据实际的cpu内核数来修改
```



### 1.3 编译测试用例

```bash
## 单独编译的步骤 -- 但是上面的brv.sh脚本张总已经包含了，因此已经使用了brv.sh，这些步骤就不用了
# Host上的测试用例
$ art/tools/buildbot-build.sh --host

# Device上的测试用例
$ art/tools/buildbot-build.sh --target
```



**注意**： 关于编译命令的详细描述可以参考官方网站：https://source.android.com/setup/build/building，以及[./android_build_internal.md](./android_build_internal.md) 





# 2. 功能测试

每个命令的详细解释，请参考第5章的内容。

### 2.1 Host上Gtest测试

```bash
# 编译
$ art/tools/buildbot-build.sh --host

# 测试 : 这两个命令是等价的，必须执行一次以下命令，然后对应的目录才有可执行文件
$ art/test.py --host -g --64

## 可以是独立的测试
$ m test-art-host-gtest64
```

测试用例在以下目录：

```
out.rv/host/linux-x86/nativetest64
```

可以直接**执行其中的文件**以测试单个测试用例



#### 2.1.1 测试结果 - 20220320

```bash
$ art/test.py --host -g --64
...
PASSING TESTS
test-art-host-gtest-art_cmdline_tests64
test-art-host-gtest-art_compiler_host_tests64
test-art-host-gtest-art_compiler_tests64
test-art-host-gtest-art_dex2oat_tests64
test-art-host-gtest-art_dexanalyze_tests64
test-art-host-gtest-art_dexdiag_tests64
test-art-host-gtest-art_dexdump_tests64
test-art-host-gtest-art_dexlayout_tests64
test-art-host-gtest-art_dexlist_tests64
test-art-host-gtest-art_dexoptanalyzer_tests64
test-art-host-gtest-art_hiddenapi_tests64
test-art-host-gtest-art_imgdiag_tests64
test-art-host-gtest-art_libartbase_tests64
test-art-host-gtest-art_libartpalette_tests64
test-art-host-gtest-art_libdexfile_external_tests64
test-art-host-gtest-art_libdexfile_support_static_tests64
test-art-host-gtest-art_libdexfile_support_tests64
test-art-host-gtest-art_libdexfile_tests64
test-art-host-gtest-art_libprofile_tests64
test-art-host-gtest-art_oatdump_tests64
test-art-host-gtest-art_odrefresh_tests64
test-art-host-gtest-art_profman_tests64
test-art-host-gtest-art_runtime_compiler_tests64
test-art-host-gtest-art_runtime_tests64
test-art-host-gtest-art_sigchain_tests64
NO TESTS SKIPPED
NO TESTS FAILED
```



### 2.2 Host上Java测试

这个测试其实是测试JVM的，**对于Devices上的验证没有用处**。

测试命令：

```bash
# 编译
$ art/tools/buildbot-build.sh --host

# 测试 
$ art/test.py --host --64 -r  --optimizing                     ## 仅仅是一个模式:optimizing
$ art/test.py --host --64 -v -r -t 001-Main --baseline --jit   ## 测试一个用例的两种模式
```



#### 2.2.1 测试结果 - 20220320

```bash
[^_^aosp.a12]$ art/test.py --host -r --64  
['/data1/wendong/Android/aosp.a12/art/test/testrunner/testrunner.py', '--host', '--64']
Concurrency: 64 (host)
[ 80% 3730/4640 ] test-art-host-run-test-debug-prebuild-optimizing-no-relocate-ntrace-cms-checkjni-picimage-ndebuggable-no-jvmti-cdex-fast-656-checker-simd-opt64 FAIL                       
4492/4640 (96%) tests passed.                                                                                                                                                                
SKIPPED TESTS: 
test-art-host-run-test-debug-prebuild-speed-profile-no-relocate-ntrace-cms-checkjni-picimage-ndebuggable-no-jvmti-cdex-fast-175-alloc-big-bignums64
test-art-host-run-test-debug-prebuild-interpreter-no-relocate-ntrace-cms-checkjni-picimage-ndebuggable-no-jvmti-cdex-fast-175-alloc-big-bignums64
...
 
656-checker-simd-opt files deleted from host 

----------
test-art-host-run-test-debug-prebuild-optimizing-no-relocate-ntrace-cms-checkjni-picimage-ndebuggable-no-jvmti-cdex-fast-656-checker-simd-opt64
```

只有一个SIMD的测试用例是错误的



### 2.3 测试环境同步

在编译完成后，在板子上/模拟器上测试之前，必须将测试环境同步到目标设备中。

```bash
$ push.sh
```

push.sh脚本其实就是几个art/tools/下面命令的组合:

```bash

# Clean up the device:
art/tools/buildbot-cleanup-device.sh

# Setup the device (including setting up mount points and files in the chroot directory):
art/tools/buildbot-setup-device.sh

# Populate the chroot tree on the device (including "activating" APEX packages
art/tools/buildbot-sync.sh

adb shell mkdir -p /data/dalvik-cache/riscv64
```



### 2.4 Target上Gtest测试

```bash
## 全部测试用例
$ . run_gtest.sh

## 单个测试用例
$. run_gtest.sh  "/apex/com.android.art/bin/art/riscv64/art_cmdline_tests --gtest_filter=CmdlineParserTest.TestVerify"
```

上述单个测试用例示例结果如下

<img src="00-Android12_setup.assets/image-20220313174240108.png" alt="image-20220313174240108" style="zoom:50%;" />



如果出现如下错误 offline

```bash
$ adb devices
List of devices attached
emulator-5554	offline
```

则需要执行一次  `adb kill-server` 



### 2.5 Target上Java测试

```bash
## 全部测试用例
$ art/test.py --target --64 -v -r --ndebug
## 可以通过 ‘--runtime-option=-Xmx512m‘ 设置运行时采用512M heap（缺省的是256M）

## 单个测试用例
$ art/test.py --target --64 -v -r -j 1  -t 001-Main

## 单个模式
$ art/test.py --target --64 -v -r -j 1  --interpreter -t 001-Main
## 单个模式一组测试：测试所有以00开头的测试： 001* 002*  等等
$ art/test.py --target --64 -v -r -j 1  --interpreter -t 00


## 其他调试用的重要参数：
--ndebug : 测试release模式  // 如果不加这个参数缺省测试的是debug模式
--run-test-option： --run-test-option='--dev --never-clean'
--runtime-option: '--runtime-option=-Xjitthreshold:0'

## 调试参数使用例子
$ art/test.py --target --64 -v -r -j 1 --run-test-option='--dev --never-clean' --baseline  -t 001-Main

## 其他有用的测试参数
--runtime-option=-Xmx384m
--runtime-option=-Xusejit:{false|true}
--runtime-option=-Xjitthreshold:0
--run-test-option='-O'
```





# 3. 性能测试

性能测试用例位于art/csky/benchmark目录下，含有如下的测试用例（7个）：

```bash
.
├── 3d-morph
├── binarytrees
├── fasta
├── nbody
├── nsieve
├── spectral-norm
└── partial-sums (没有C版本代码，只有Java版本代码)
```

以上每个测试用例目录下有两类测试用例：C/C++，以及Java。例如fasta：

```bash
fasta]$ tree
.
├── Android.bp    // 编译C/C++测试用例的控制文件
├── fasta.cc      // C/C++测试用例
├── Fasta.java    // Java测试用例
└── string-fasta.js
```

本章只是列出如何运行这些测试，对测试结果的分析请参考Benchmark文档。



### 3.1 Clang-Benchmark

通过运行目录`art/csky/benchmark`下的脚本文件`run_c.sh`可以获取C、C++测试数据

```bash
# Under 'art/csky/benchmarks'
$ . run_c.sh [3d-morph|binarytrees|fasta|nbody|nsieve|partial-sums|spectral-norm|simpleJavaTest]
```

其中后面的测试用例参数是可选的，当加上参数时就执行对应参数的测试用例，否则全部执行。其结果类似

```bash
$ . run_c.sh 

Start benchmark test...

Start benchmark test 3d-morph
CXX Time: 16653.572998 ms,  (count=0.000000)

Start benchmark test binarytrees
CXX Time: 26993.313965 ms, (count=30060000)

Start benchmark test fasta
CXX TIME: 3062.61 ms, (cycle=100)

Start benchmark test nbody
CXX Time: 21443.956055 ms, (count=10000)

Start benchmark test nsieve
CXX Time: 13271.365967 ms, clycle=10000, res=143020000

Start benchmark test spectral-norm
CXX Time: 6743.334961 ms,  (count=25433.471157)

Start benchmark test simpleCTest
CXX Time: 2174.833008 ms,  (count=25897538.768883)
/data/home/wendong/aosp/aosp.a12/art/csky/benchmarks
```



### 3.2 CSKY-Benchmarks

```bash
# run java test
. run_java.sh [3d-morph|binarytrees|fasta|nbody|nsieve|partial-sums|spectral-norm|simpleJavaTest]
```

其中后面的测试用例参数是可选的，当加上参数时就执行对应参数的测试用例，否则全部执行。其结果类似

```bash
$ . run_java.sh 

Start benchmark test...

Start benchmark test 3d-morph
+Java Time: 14646.0ms, (count=6.750155989720952E-11)

Start benchmark test binarytrees
+Java Time: 7478.0ms, (count=17247)

Start benchmark test fasta
+Java Time: 9795.0ms,  (cycle=100)

Start benchmark test nbody
+Java Time: 20483.0ms, (cycle=10000)

Start benchmark test nsieve
+Java Time: 18771.0ms, (cycle=10000)res= 143020000

Start benchmark test partial-sums
+Java Time: 11232.0ms, (count=3.57564431E11)

Start benchmark test spectral-norm
+Java Time: 6852.0ms, (count=25433.471156519223)

Start benchmark test simpleJavaTest

ConstStringBenchmark
+Java Time: 6192.0ms

StringIndexOfBenchmark 
+Java Time: 6360.0ms,  (cycle=10000000)
```



### 3.3 ART-Benchmark

还有一类art/benchmark下的测试用例，正常情况下可以使用vogar命令进行测试

一个例子如下

```bash
vogar --mode device --variant x64 --chroot $ART_TEST_CHROOT  --timeout 120 --benchmark art/benchmark/const-class/src/ConstClassBenchmark.java
```

<img src="00-Android12_setup.assets/image-20221208165456433.png" alt="image-20221208165456433" style="zoom:50%;" />



由于这类测试，其结果都是ns级别的，作为性能测试标准其可信度非常低，暂时不做进一步的分析。



# 4. 已知错误

### 4.1 编译问题

#### 4.1.1 disable angle的编译

在external/angle中

```
mv Android.bp Android.bk
```

​    

#### 4.1.2 修改文件Device的文件大小

build/make/target/board/emulator_riscv64/BoardConfig.mk，

```
BOARD_BOOTIMAGE_PARTITION_SIZE := 0x02000000
改大些 比如 0x04000000
```



### 4.2 push错误

![image-20220313140411841](00-Android12_setup.assets/image-20220313140411841.png)

此时原因是没有执行这个命令 `art/tools/buildbot-build.sh --target`

一旦编译完成，这个错误就不存在了

<img src="00-Android12_setup.assets/image-20220313141243845.png" alt="image-20220313141243845" style="zoom:50%;" />



### 4.3 模拟器运行错误

#### 4.3.1 image文件损坏

如果启动模拟器存在如下错误：

<img src="00-Android12_setup.assets/image-20220316194957861.png" alt="image-20220316194957861" style="zoom:50%;" />

请执行：

```bash
rm -rf $ANDROID_PRODUCT_OUT/*.img; rm $ANDROID_PRODUCT_OUT/system -rf; rm -rf $ANDROID_PRODUCT_OUT/apex
```



#### 4.3.2 libvulkan.so路径

```bash
emulator: feeding guest with passive gps data, in headless mode
cannot add library /back/wendong/Android/aosp.qh/prebuilts/android-emulator/linux-x86_64/qemu/linux-x86_64/lib64/vulkan/libvulkan.so: failed
added library /back/wendong/Android/aosp.qh/prebuilts/android-emulator/linux-x86_64/lib64/vulkan/libvulkan.so
```

这个是aosp12版本的统一的错误，原因是这个目录根本不存在 `/back/wendong/Android/aosp.qh/prebuilts/android-emulator/linux-x86_64/qemu/linux-x86_64/lib64/vulkan/libvulkan.so,`需要增加

```bash
$ ln -s /back/wendong/Android/aosp.qh/prebuilts/android-emulator/linux-x86_64/lib64 /back/wendong/Android/aosp.qh/prebuilts/android-emulator/linux-x86_64/qemu/linux-x86_64/
```





# 5. ART测试概述



### 5.1 测试框架简介

art的测试以`art/test.py`为入口的

这个脚本很简单被分为两类测试：gtest，run_test（Java测试）

对于gtest （`-g` 参数），其代码很简单 - 其实就是执行了一个make对应的命令

```bash
if options.gtest or not options.run_test:
  build_size=''
  for arg in sys.argv[1:]:
    if arg == '--64':
      build_size='64'
  build_target = ''
  if options.host or not options.target:
    build_target += ' test-art-host-gtest' + build_size
  if options.target or not options.host:
    build_target += ' test-art-target-gtest'  + build_size

  build_command = ANDROID_BUILD_TOP + '/build/soong/soong_ui.bash --make-mode'
  build_command += ' -j' + str(options.n_threads)
  build_command += ' ' + build_target
  print(build_command)
  if subprocess.call(build_command.split(), cwd=ANDROID_BUILD_TOP):
    sys.exit(1)
```

从这里可以看到（修订后的代码）

```bash
$ m test-art-host-gtest64
$ art/test.py --host -g --64
## 以上两条命令是等价的
```



#### 5.1.1 test.py

对于Java测试，`art/test.py` 则需要`-r`参数，其实际是调用 `art/test/testrunner/testrunner.py进行真正后续参数分析

```bash
if options.run_test or options.help_runner or not options.gtest:
  testrunner = os.path.join('./',
                          ANDROID_BUILD_TOP,
                            'art/test/testrunner/testrunner.py')
  run_test_args = []
  for arg in sys.argv[1:]:
    if arg == '--run-test' or arg == '--gtest' \
    or arg == '-r' or arg == '-g':
      continue
    if arg == '--help-runner':
      run_test_args = ['--help']
      break
    run_test_args.append(arg)

  test_runner_cmd = [testrunner] + run_test_args
  print(test_runner_cmd)
  if subprocess.call(test_runner_cmd) or options.help_runner:  ## 直接调用art/test/testrunner/testrunner.py'
    sys.exit(1)
```

可以通过`art/test.py --help-runner`命令打印testrunner.py可以接收的参数。



#### 5.1.2 testrunner.py

这个脚本位于`art/test/testrunner/testrunner.py`，看似复杂其实细节并不太多。可以使用命令行`./art/test.py --help-runner`显示对应的参数细节。

```python
def main():
  gather_test_info()
  user_requested_tests = parse_option()
  setup_test_env()
  gather_disabled_test_info()
  if build:
    build_targets = ''
    if 'host' in _user_input_variants['target']:
      build_targets += 'test-art-host-run-test-dependencies '
    if 'target' in _user_input_variants['target']:
      build_targets += 'test-art-target-run-test-dependencies '
    if 'jvm' in _user_input_variants['target']:
      build_targets += 'test-art-host-run-test-dependencies '
    build_command = env.ANDROID_BUILD_TOP + '/build/soong/soong_ui.bash --make-mode'
    build_command += ' DX='
    if dist:
      build_command += ' dist'
    build_command += ' ' + build_targets
    print_text('Build command: %s\n' % build_command)
    if subprocess.call(build_command.split()):
      # Debugging for b/62653020
      if env.DIST_DIR:
        shutil.copyfile(env.SOONG_OUT_DIR + '/build.ninja', env.DIST_DIR + '/soong.ninja')
      sys.exit(1)

  if user_requested_tests:
    run_tests(user_requested_tests)
  else:
    run_tests(RUN_TEST_SET)

  print_analysis()
```

在这个脚本中，最重要的参数如下： 

> 1. - -target / --host:  区分是host测试还是target测试
>    - --64: 只进行64位测试
>    - `-j N`  --- 利用多少个线程进行测试
>    - -v：显示更多的信息
>    - --ndebug：以 release模式进行测试： --debug/--ndebug
>    - --dry-run: 打印测试命令
>    - --run-test-option： --run-test-option='--dev --never-clean'  ## 这里的参数是传递给  run-test脚本的，必须用单引号引起来
>    - --runtime-option:   '--runtime-option=-Xjitthreshold:0'
>    - --gdb ： 调试
>    - 可以通过增加 `-b`参数使得脚本自动编译测试对应的依赖：`test-art-host-run-test-dependencies `， `test-art-target-run-test-dependencies`
>    - compiler-type Options:

```bash
# Java测试有以下模式
#compiler-type Options:
#  Options that control the 'compiler' variants.

  --all-compiler        Enable all variants of compiler
  --baseline
  --jit-on-first-use
  --regalloc_gc
  
  ## 以下五种模式是缺省的，不指定模式的时候测试的五种缺省模式
  --speed-profile
  --jit
  --optimizing
  --interpreter
  --interp-ac
  
# 几种模式的具体区别
[interpreter]   -Xint   -Xcompiler-option --compiler-filter=verify           -Xusejit:false 
[interp-ac]     -Xverify:softfail  -Xint   -Xcompiler-option --compiler-filter=extract          -Xusejit:false   
[optimizing]                                                                                    -Xusejit:false 
[jit-on-first-use]  -Xjitthreshold:0       -Xcompiler-option --compiler-filter=verify           -Xusejit:true  
[jit]                                      -Xcompiler-option --compiler-filter=verify           -Xusejit:true  
[regalloc_gc] -Xcompiler-option --register-allocation-strategy=graph-color                      -Xusejit:false 
[speed-profile] --profile-file=/data/run-test/test-3108186/001-HelloWorld.prof                  -Xusejit:false 
```



比如调用： `art/test.py --target -r -j1 --64 -v  --run-test-option='--dev --never-clean' --interpreter -t 001-Main`

表示在target上以j1方式、interpreter模式测试001-Main，且测试临时创建的目录在测试结束时不被删除。



这个脚本完成两件主要的事情：

1. 处理各种输入参数（通过art/test.py传递进来的），设置缺省参数。
2. 最后通过函数`run_tests`中的 `executor.submit(run_test, command, test, variant_set, test_name)`来调用测试用例的。

```python
def run_tests(tests):
  """This method generates variants of the tests to be run and executes them.

  Args:
    tests: The set of tests to be run.
  """
  options_all = '' 
  
  ##  处理各种参数设置
  run_test_sh = env.ANDROID_BUILD_TOP + '/art/test/run-test'
  command = ' '.join((run_test_sh, options_test, ' '.join(extra_arguments[target]), test))
  return executor.submit(run_test, command, test, variant_set, test_name)
```

比如通过执行一个测试并打印上述代码的command可以得到如下信息：

```bash
$ ./art/test.py -j 4 --target -r --64 -v --optimizing -t 001-H

## 等价于
$ ./art/test/run-test  --output-path /tmp/test-art-ne3oxma4/tmpsxy41p6o --always-clean  --chroot /data/local/art-test-chroot --prebuild --compact-dex-level fast --optimizing --no-relocate --runtime-option -Xcheck:jni --64  001-HelloWorld
```

上述脚本中实际调用`./art/test/run-test `的代码在这里

```python
  if gdb: 
    proc = _popen(
      args=command.split(),
      stderr=subprocess.STDOUT,
      universal_newlines=True,
      start_new_session=True
    )    
  else:
    proc = _popen(
      args=command.split(),
      stderr=subprocess.STDOUT,
      stdout = subprocess.PIPE,
      universal_newlines=True,
      start_new_session=True,
    )    
  script_output, return_value = child_process_tracker.wait(proc, timeout)
```



#### 5.1.3 run-test

首先注意的一个问题是：run-test支持的一些参数，可以通过testrunner.py的参数`--run-test-option：`传递进来（比如参数 `--never-clean`,  `--dev`） -- 以下两个命令是等价的：

```bash
##
$ ./art/test.py -j 4 --target -r --64 -v --optimizing -t 001-H

## 等价于
$ art/test/run-test  --output-path /tmp/test-art-p5doya_8/tmpks8cpvnt --always-clean  --chroot /data/local/art-test-chroot --prebuild --compact-dex-level fast --optimizing --no-relocate --runtime-option -Xcheck:jni --64  001-HelloWorld


/back/wendong/Android/aosp.qh/art/test/001-Main: building...
/back/wendong/Android/aosp.qh/art/test/001-Main: running...
/back/wendong/Android/aosp.qh/art/test/001-Main: succeeded!

## 在这个命令中一定注意的是测试用例一定放在命令行的最后，比如
$ art/test/run-test  --output-path /back/wendong/Android/aosp.qh/wwd  --host --prebuild --compact-dex-level fast --baseline --no-relocate --runtime-option -Xcheck:jni --64 --never-clean  001-Main
```

特别注意的是: --dev参数有时候非常不好用。经常显示不出来内容。

这个脚本`art/test/run-test`  首先调用 ` etc/default-build`进行编译（除非特定测试用例目录下有build脚本）。如果想看更多的内容，建议使用如下命令来指定临时目录：

```bash
$ art/test/run-test  --output-path /tmp/wwd --never-clean  --chroot /data/local/art-test-chroot --prebuild --compact-dex-level fast --optimizing --no-relocate --runtime-option -Xcheck:jni --64  001-HelloWorld
```

即使以host方式运行Java测试，也少不了这个文件`out/host/linux-x86/apex/art_boot_images/javalib/x86_64/boot.art`，而这个文件是通过命令`art/tools/buildbot-build.sh --host`生成的。



脚本`art/test/run-test`包括了几部分内容：分析参数，准备环境，编译，运行。下面对这些内容做简单的描述

##### 5.1.3.1 分析参数

这部分内容非常直接，不在赘述。



##### 5.1.3.2 准备环境

主要是创建临时目录并将对应的文件拷贝这个临时目录，随后的执行都在此临时目录

```bash
test_dir="test-$$"
if [ -z "$TMPDIR" ]; then 
  tmp_dir="/tmp/$USER/${test_dir}"
else
  tmp_dir="${TMPDIR}/${test_dir}"
fi


## 当然这个临时目录也可以被命令行参数覆盖
--run-test-option='--output-path /tmp/wwd/test100'
```

在这之后就会将测试目录以及执行文件拷贝到临时目录

```bash
rm -rf "$tmp_dir"
cp -LRp "$test_dir" "$tmp_dir"
cd "$tmp_dir"

if [ '!' -r "$build" ]; then
    cp "${progdir}/etc/default-build" build     ## 如果当前测试用例没有自己的build，则将缺省的build拷贝过来，且改名为build
else
    cp "${progdir}/etc/default-build" .         ## 否则将缺省的build拷贝过来，且不能改名（当前测试用例的build会调用default-build）
fi

if [ '!' -r "$run" ]; then
    cp "${progdir}/etc/default-run" run
else
    cp "${progdir}/etc/default-run" .
fi

if [ '!' -r "$check_cmd" ]; then
    cp "${progdir}/etc/default-check" check
else
    cp "${progdir}/etc/default-check" .
fi

chmod 755 "$build"
chmod 755 "$run"
chmod 755 "$check_cmd"
```



##### 5.1.3.3 Build

编译包括：java->class,  class->dex, dex->oat

其中前两部分是在Host上执行的，包含在临时目录中的build脚本中（与任何Arch无关的步骤）。

第三步包含在run脚本中（需要在目标机上运行，生成特定架构的指令/oat文件）。



##### 5.1.3.4 Run - 1

Run的第一步其实是编译的第三步：在Device上执行，生成oat文件。

整个run脚本被统一调用：

```bash
export RUN="${progdir}/etc/run-test-jar" && ./run --quiet --chroot /data/local/art-test-chroot -O --prebuild --compact-dex-level fast 
  --runtime-option -Xcheck:jni --64 --runtime-option 
  -XX:ThreadSuspendTimeout=500000 
  --lib libart.so --runtime-option 
  -Djava.library.path=/data/nativetest64/art/riscv64 
  --boot /apex/com.android.art/javalib/boot.art 
  --no-relocate --testlib arttest
  
# run中往往就是一句话：调用$RUN
```

这个脚本的分析请参考5.3节。

##### 5.1.3.4 Run - 2

Run的第二步才是真正执行虚拟机运行代码。

```bash
$ art/test/etc/run-test-jar  --quiet --chroot /data/local/art-test-chroot --prebuild --compact-dex-level fast --runtime-option -Xcheck:jni --64 --runtime-option -XX:ThreadSuspendTimeout=500000 --lib libartd.so --runtime-option -Djava.library.path=/data/nativetest64/art/riscv64 --boot /apex/com.android.art/javalib/boot.art --no-relocate --testlib arttestd
```

详细内容请参考5.3节。



### 5.2 Java测试编译

编译包括：java->class,  class->dex, dex->oat

#### 5.2.1 javac生成class文件

以下命令需要在host上的临时目录执行：

```bash
$ javac -g -Xlint:-options -source 1.8 -target 1.8 -bootclasspath ... -encoding utf8 -implicit:none -d classes src/Main.java

$ JAVAC="javac -g -Xlint:-options -source 1.8 -target 1.8"  $ANDROID_BUILD_TOP/art/tools/javac-helper.sh --core-only --mode=target --show-commands -implicit:none  -d classes src/Main.java
```

上述步骤：将Java源代码编译为class文件 `classes/Main.class`



#### 5.2.2 d8生成dex文件

class -> jar -> dex

```bash
$ d8 --min-api 26 --lib ...  --output classes.jar classes/Main.class

$ unzip -p classes.jar classes.dex > classes.dex
```

因为Android的dex文件（Bytecode文件）与任何Arch都是无关的，到这一步不涉及到任何Arch。



#### 5.2.3 生成odex文件-X86

首先在测试目录下创建目录： `dalvik-cache/x86_64 `, `oat/x86_64`

然后执行`/data/home/wendong/aosp/aosp.a12/out.rv/host/linux-x86/bin/dex2oatd64 `:

```bash
$ /data/home/wendong/aosp/aosp.a12/out.rv/host/linux-x86/bin/dex2oatd64 --compile-art-test --compact-dex-level=fast 
--runtime-arg -Xbootclasspath:  ...
--runtime-arg -Xbootclasspath-locations: ...
--compiler-filter=verify --runtime-arg -Xnorelocate 
--boot-image=/data/home/wendong/aosp/aosp.a12/out.rv/host/linux-x86/apex/art_boot_images/javalib/boot.art 
--dex-file=/data/home/wendong/aosp/aosp.a12/wwd//001-Main.jar 
--oat-file=/data/home/wendong/aosp/aosp.a12/wwd//oat/x86_64/001-Main.odex 
--app-image-file=/data/home/wendong/aosp/aosp.a12/wwd//oat/x86_64/001-Main.art 
--resolve-startup-const-strings=true --generate-mini-debug-info --instruction-set=x86_64 
--watchdog-timeout=300000 
```

上述命令生成了两个文件： 001-Main.odex ，001-Main.art 

（art文件是纯粹的Arch代码文件，而odex则是bytecode与指令混合文件）

为了看清这个odex、art文件的具体格式可以用如下命令验证

```bash
$ ../out.rv/host/linux-x86/bin/oatdump --oat-file=/data/home/wendong/aosp/aosp.a12/wwd//oat/x86_64/001-Main.odex 
oatdump W 02-05 14:09:21 1942265 1942265 oatdump.cc:2540] No dex filename provided, oatdump might fail if the oat file does not contain the dex code.
MAGIC:
oat
195

LOCATION:
/data/home/wendong/aosp/aosp.a12/wwd//oat/x86_64/001-Main.odex

CHECKSUM:
0x5f24ac44

INSTRUCTION SET:
X86_64

INSTRUCTION SET FEATURES:
ssse3,sse4.1,sse4.2,-avx,-avx2,popcnt
```



#### 5.2.4 生成odex文件-RV64

在taget上的临时目录进行oat编译以及运行，临时目录是在`$ART_TEST_CHROOT /data/run-test`，比如正常情况下的临时目录位于开发板的`/data/local/art-test-chroot/data/run-test `。比如以001-Main这个为例：

```
evb_light:/data/local/art-test-chroot/data/run-test/test-607052 # 
evb_light:/data/local/art-test-chroot/data/run-test/test-607052 # ls -l
total 16
-rw-rw-rw- 1 root root  576 2023-02-25 14:11 001-Main.jar
-rw-rw-rw- 1 root root 3636 2023-02-25 14:11 cmdline.sh
drwxrwxrwx 3 root root 4096 2023-02-25 14:10 dalvik-cache
drwxrwxrwx 3 root root 4096 2023-02-25 14:10 oat
evb_light:/data/local/art-test-chroot/data/run-test/test-607052 # ls -l oat/riscv64/                                                                                                                       
total 32
-rw-r--r-- 1 root root  8192 2023-02-25 14:10 001-Main.art
-rw-r--r-- 1 root root 17296 2023-02-25 14:10 001-Main.odex
-rw-r--r-- 1 root root   772 2023-02-25 14:10 001-Main.vdex
evb_light:/data/local/art-test-chroot/data/run-test/test-607052 # 
```

具体的命令行如下所示

```bash
/apex/com.android.art/bin/dex2oatd64 --compile-art-test --compact-dex-level=fast --runtime-arg 
  -Xbootclasspath:/apex/com.android.art/javalib/core-oj.jar:/apex/com.android.art/javalib/core-libart.jar:/apex/com.android.art/javalib/okhttp.jar:
    /apex/com.android.art/javalib/bouncycastle.jar:/apex/com.android.art/javalib/apache-xml.jar:
    /apex/com.android.i18n/javalib/core-icu4j.jar:/apex/com.android.conscrypt/javalib/conscrypt.jar 
  --runtime-arg -Xbootclasspath-locations:/apex/com.android.art/javalib/core-oj.jar:/apex/com.android.art/javalib/core-libart.jar:
    /apex/com.android.art/javalib/okhttp.jar:/apex/com.android.art/javalib/bouncycastle.jar:
    /apex/com.android.art/javalib/apache-xml.jar:/apex/com.android.i18n/javalib/core-icu4j.jar:
    /apex/com.android.conscrypt/javalib/conscrypt.jar 
  --compiler-filter=verify --runtime-arg -Xnorelocate -
  -boot-image=/apex/com.android.art/javalib/boot.art 
  --dex-file=/data/run-test/test-607052/001-Main.jar 
  --oat-file=/data/run-test/test-607052/oat/riscv64/001-Main.odex 
  --app-image-file=/data/run-test/test-607052/oat/riscv64/001-Main.art 
  --resolve-startup-const-strings=true --generate-mini-debug-info 
  --instruction-set=riscv64
```



### 5.3 Java测试运行

#### 5.3.1 run脚本

```bash
export RUN="${progdir}/etc/run-test-jar"

## run脚本中一般情况下就是一条语句
exec ${RUN} "$@"
```

因此，run脚本实际上是 art/test/etc/run-test-jarm，以及对应的一堆参数

```bash
art/test/etc/run-test-jar  --quiet --chroot /data/local/art-test-chroot -O --prebuild --compact-dex-level fast 
  --runtime-option -Xcheck:jni --64 --runtime-option 
  -XX:ThreadSuspendTimeout=500000 
  --lib libart.so --runtime-option 
  -Djava.library.path=/data/nativetest64/art/riscv64 
  --boot /apex/com.android.art/javalib/boot.art 
  --no-relocate --testlib arttest
```



实际执行目标机器命令行代码如下：

```bash
    cmdline="cd $DEX_LOCATION && \
             export ASAN_OPTIONS=$RUN_TEST_ASAN_OPTIONS && \
             export ANDROID_DATA=$DEX_LOCATION && \
             export DEX_LOCATION=$DEX_LOCATION && \
             export ANDROID_ROOT=$ANDROID_ROOT && \
             export ANDROID_I18N_ROOT=$ANDROID_I18N_ROOT && \
             export ANDROID_ART_ROOT=$ANDROID_ART_ROOT && \
             export ANDROID_TZDATA_ROOT=$ANDROID_TZDATA_ROOT && \
             export ANDROID_LOG_TAGS=$ANDROID_LOG_TAGS && \
             rm -rf ${DEX_LOCATION}/dalvik-cache/ && \
             mkdir -p ${mkdir_locations} && \
             export LD_LIBRARY_PATH=$LD_LIBRARY_PATH && \
             export NATIVELOADER_DEFAULT_NAMESPACE_LIBS=$NATIVELOADER_DEFAULT_NAMESPACE_LIBS && \
             export PATH=$PREPEND_TARGET_PATH:\$PATH && \
             $profman_cmdline && \
             $dex2oat_cmdline && \
             $dm_cmdline && \
             $vdex_cmdline && \
             $strip_cmdline && \
             $sync_cmdline && \
             $timeout_prefix $dalvikvm_cmdline"

    cmdfile=$(mktemp cmd-XXXX --suffix "-$TEST_NAME")
    echo "$cmdline" >> $cmdfile

    # ....

    exit_status=0
    if [ "$DRY_RUN" != "y" ]; then 
      if [ -n "$CHROOT" ]; then 
        adb shell chroot "$CHROOT" sh $DEX_LOCATION/cmdline.sh   ## 实际执行对应的cmdline.sh
      else 
        adb shell sh $DEX_LOCATION/cmdline.sh
      fi   
      exit_status=$?
    fi 
```



比如。想再加入simpleperf则可以修改`dalvikvm_cmdline`。



#### 5.3.2 host-dalvikvm64

执行`/data/home/wendong/aosp/aosp.a12/out.rv/host/linux-x86/bin/dalvikvm64`

```bash
$ /data/home/wendong/aosp/aosp.a12/out.rv/host/linux-x86/bin/dalvikvm64 -Xcheck:jni 
-XX:ThreadSuspendTimeout=500000 
-Djava.library.path=
  /data/home/wendong/aosp/aosp.a12/out.rv/host/linux-x86/lib64:
  /data/home/wendong/aosp/aosp.a12/out.rv/host/linux-x86/nativetest64 
  
-XX:SlowDebug=true 
-Xcompiler-option --runtime-arg -Xcompiler-option -XX:SlowDebug=true 
-Xcompiler-option --compile-art-test -XjdwpProvider:none 
-Xbootclasspath:  ...
-Xbootclasspath-locations: ..

-Xnorelocate -Xcompiler-option --generate-mini-debug-info -Xcompiler-option 
--generate-mini-debug-info -XXlib:libartd.so -Xjnigreflimit:512 -Xcheck:jni 
-Xint -Xusejit:false -Xcompiler-option --compiler-filter=verify 
-Ximage:/data/home/wendong/aosp/aosp.a12/out.rv/host/linux-x86/apex/art_boot_images/javalib/boot.art 
-XX:DumpNativeStackOnSigQuit:false 

-cp /data/home/wendong/aosp/aosp.a12/wwd//001-Main.jar Main arttestd
```



#### 5.3.2 RV-dalvikvm64

```bash
$ /apex/com.android.art/bin/dalvikvm64 -Xcheck:jni 
  -XX:ThreadSuspendTimeout=500000 
  -Djava.library.path=/data/nativetest64/art/riscv64 -XX:SlowDebug=true -Xcompiler-option 
  --runtime-arg -Xcompiler-option -XX:SlowDebug=true -Xcompiler-option --compile-art-test 
  -XjdwpProvider:none -Xbootclasspath:/apex/com.android.art/javalib/core-oj.jar:
    /apex/com.android.art/javalib/core-libart.jar:/apex/com.android.art/javalib/okhttp.jar:
    /apex/com.android.art/javalib/bouncycastle.jar:/apex/com.android.art/javalib/apache-xml.jar:
    /apex/com.android.i18n/javalib/core-icu4j.jar:/apex/com.android.conscrypt/javalib/conscrypt.jar 
  -Xbootclasspath-locations:/apex/com.android.art/javalib/core-oj.jar:/apex/com.android.art/javalib/core-libart.jar:
    /apex/com.android.art/javalib/okhttp.jar:/apex/com.android.art/javalib/bouncycastle.jar:
    /apex/com.android.art/javalib/apache-xml.jar:/apex/com.android.i18n/javalib/core-icu4j.jar:
    /apex/com.android.conscrypt/javalib/conscrypt.jar -Xnorelocate -Xcompiler-option 
  --generate-mini-debug-info -Xcompiler-option --generate-mini-debug-info -XXlib:libartd.so 
  -Xjnigreflimit:512 -Xcheck:jni -Xint -Xusejit:false -Xcompiler-option 
  --compiler-filter=verify -Ximage:/apex/com.android.art/javalib/boot.art 
  -Djava.io.tmpdir=/data/local/tmp -XX:DumpNativeStackOnSigQuit:false 
  -cp /data/run-test/test-607052/001-Main.jar Main arttestd
```





# A. 环境和源码

关于下载repo以及设置等内容，请参考第7章。

### A.1 建立工作目录

建立工作目录:

```bash
cd
mkdir aosp_ws
cd aosp_ws

export aosp_ws=`pwd`
```

 以下信息是旧信息，已经不用，只是留作参考而已

> **有关源码repo的重要说明：**
>
> 1，目前，平头哥aosp的repo有两个版本：存放在**github上的模拟器版本**，以及存放在**阿里服务器上的硬件开发板版本**。
>
> 2，虽然是两份repo，但是他们所包含的绝大部分git，其实都指向同一个url，其中又分为两类：
>
> ​      a，没有任何修改的android12的原始代码，默认指向googlesource的源。
>
> ​      b，部分平头哥移植修改过的git，url都是指向github。
>
> 3，另外，有很少的几个git，是硬件开发板版本所独有的，放在了阿里服务器上。
>
> **因此，请根据自己的需求选择相应的设置和源码repo。**



### A.2 设置ssh

**注意：ssh设置只有同步aosp代码时才需要，如果使用熵核服务器上打包好的aosp代码，只是更新art代码，则不需要这些配置和操作。**



#### A.2.1 github

按照github上的说明，注册账户，并上传ssh-key。（参考：https://docs.github.com/cn/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account）



#### A.2.2 阿里服务器

由于阿里的39服务器不再使用，这个步骤就不再需要了！



### A.3 平头哥源代码

在服务器192.168.3.100上，解压文件：`/back/comm_back/1-android//aosp.ali.android12.20230606.tar`

按照如下步骤就可以得到2022-12-22日的代码：

```bash
$ cd $ws ## 个人的workspace

# 解压得到的 AOSP 工程目录
$ tar xvf /back/comm_back/1-android//aosp.ali.android12.20230606.tar
$ cd aosp.a12
```



### A.4 github同步源代码 【公司内部不要使用】

如前所述，**两个repo针对的是不同的需求**，请注意按自己的需要来选择github或者阿里服务器。

另外，**需要提前说明的是**，因为googlesource的源，在国内不能访问，即使用代理，仍有少量代码可能无法正确同步，请按照**1.3.5**的说明，更改为清华源。



#### A.4.1 github代码

参考这里：https://github.com/riscv-android-src/riscv-android/blob/main/doc/android12.md

```bash
mkdir ~/riscv-android-src && cd ~/riscv-android-src
repo init -u git@github.com:riscv-android-src/manifest.git -b riscv64-android-12.0.0_dev

## 如果在国内，需要如下步骤：部分原始的git位于google, 国内必须走代理VPN才可以，修改后就直接使用清华的git
cd .repo/manifests
修订文件default.xml，如下面的代码diff所示

## 继续以下命令
repo sync
cd prebuilts/rust/
git lfs pull
cd -
cd cts/
git lfs pull
cd -
rm external/angle/Android.bp
```

default.xml的具体修改如下

```diff
$ git diff .
diff --git a/default.xml b/default.xml
index d999ece..449868a 100644
--- a/default.xml
+++ b/default.xml
@@ -1,29 +1,27 @@
 <?xml version="1.0" encoding="UTF-8"?>
 <manifest>
 
-  <!--- use aosp android repo as code base -->
+  <!--- use aosp android repo as code base 
   <remote  name="aosp"
            fetch="https://android.googlesource.com/"
-           review="https://android-review.googlesource.com/" />
+           review="https://android-review.googlesource.com/" /-->
 
-  <!--- use tsinghua android repo as code base
   <remote  name="tsinghua"
            fetch="https://mirrors.tuna.tsinghua.edu.cn/git/AOSP"
            review="https://android-review.googlesource.com/" />
   <default revision="refs/tags/android-12.0.0_r2"
            remote="tsinghua"
            sync-j="4" />
-  -->
 
   <remote  name="riscv-android"
            fetch=".."
            review="" />
   
   <default revision="refs/tags/android-12.0.0_r2"
-           remote="aosp"
+           remote="tsinghua"
            sync-j="4" />
 
-  <superproject name="platform/superproject" remote="aosp"/>
+  <superproject name="platform/superproject" remote="tsinghua"/>
```

上述方式获得的代码只能运行在模拟器上。



### A.5 代码同步的补充说明--【在优化过程中暂时用不到】

在开发过程中，aosp里的art是替换了熵核自己的git，一般只需要在art目录下，进行git操作就行（在 `xc-android12`分支中），不要repo sync整个目录。

如果确实需要sync整个repo，则可以如下操作，先将art移出aosp根目录，sync之后，再移回来。

```bash
# cd aosp.a10
# 一定确保1.2.3执行了  (执行了后台代理)

mv art ../art.xc
repo sync
## 完成后，要看看是否报错了

rm -rf art
mv ../art.xc ./art   ## 一定使用我们自己的art

## 验证
$ cd art
$ git br
  csky
  thead
* xc-android12
  xc-android12-aliref
  
### 注意：一定要在xc-android12分支下工作！！！
### thead： 这个分支留下了我们android10移植的最后的代码
### xc-android12-aliref： 这个分支是android12上阿里对art的 “粗略” 的改动，只是为了编译，很多改动是错误的
###   我们只能作为参考，我们需要从 ‘xc-android12’ 开始，从头改动
```





### A.6 KVM

在模拟器上运行时，尽量设置kvm，以下示例展示了如何使用 `kvm-ok` 命令。

1. 安装 `cpu-checker` 软件包：

   ```
   $ sudo apt-get install cpu-checker
   $ egrep -c '(vmx|svm)' /proc/cpuinfo
   ```

   输出值 1 或更大值表示支持虚拟化。输出值 0 表示您的 CPU 不支持硬件虚拟化。

2. 运行命令 `kvm-ok`：

   ```
   $ kvm-ok
   ```

   预期输出： `INFO: /dev/kvm exists KVM acceleration can be used`

   如果出现以下错误，则表示您仍然可以运行虚拟机。如果没有 KVM 扩展，您的虚拟机运行速度会减慢。 `INFO: Your CPU does not support KVM extensions KVM acceleration can NOT be used`

Refer: https://developer.android.com/studio/run/emulator-acceleration#vm-linux





# B. 常用命令

### B.1 QEMU编译

```bash
repo init -u git@github.com:riscv-android-src/manifest.git -b emu-31.2.1.0-riscv64
repo sync
cd external/qemu/
./android/rebuild.sh
ls objs/distribution/

./android/rebuild.sh  --enable-debug --enable-debug-info  --enable-debug-stack-usage
```





### B.2 VM属性

查询虚拟机属性

```bash
adb shell getprop|grep "dalvik.vm."
```

比如

要开启 JIT 日志记录，请运行以下命令：

```bash
adb root
adb shell stop
adb shell setprop dalvik.vm.extra-opts -verbose:jit
adb shell start
```



要停用 JIT，请运行以下命令：

```bash
adb root
adb shell stop
adb shell setprop dalvik.vm.usejit false
adb shell start
```





### B.3 DUMP OAT文件

```bash
$ adb shell "oatdump --oat-file=/system/framework/riscv64/boot-framework.oat" > boot-framework.dump
```



### B.4 模拟器常用命令

```bash
$ emulator -help-datadir

  Use '-datadir <dir>' to specify a directory where writable image files
  will be searched. On this system, the default directory is:

      /Users/me/.android

  See '-help-disk-images' for more information about disk image files.
```

更多的命令请参考 `-help` 或者 `https://developer.android.com/studio/run/emulator-commandline

其实emulator是可以自动找对应文件的，这种情况下只需要执行以下命令即可：

```bash
$ emulator -no-qt -port 5580 -no-window -show-kernel -noaudio  -selinux permissive  -qemu -smp 2 -m 8192
```





# C. Android-13/Master

**此部分内容为非正式文档**，只是为了在帮助平头哥upstream过程中，需要用到最新版本的ART，此处列出部分用到的改动（测试脚本）

### C.1 测试用例

在2022年年底的最新版本中，改动Java测试用例，然后直接使用test.py运行，不会执行任何改动的测试代码，因为测试代码被编译到了这个目录`${ANDROID_HOST_OUT}/etc/art/*`，当执行Java测试的时候只会从这个目录拿代码，而不会再从原始的测试目录里面拿代码。

如果想使得测试改动生效（在移植过程中，这是很常见的），需要进行如下操作：

```bash
$ rm ${ANDROID_HOST_OUT}/etc/art/xxx.zip
$ m art-run-test-target-data  #art/tools/buildbot-build.sh --target
$ ./art/test.py -j 1 --target -r --64 --no-prebuild --interpreter  --ndebug -t 107
```

xxx.zip的模式是测试用例三、四位序号的后两位，比如107，表示这个测试用例被压缩在了文件`**art-run-test-target-data-shard07.zip**`中

```bash
unzip art-run-test-target-data-shard07.zip 
$ ll target/
total 44
drwxr-xr-x 4 wendong xcvmg 4096 1月   4 14:49 007-count10
drwxr-xr-x 4 wendong xcvmg 4096 1月   4 14:49 107-int-math2
drwxr-xr-x 4 wendong xcvmg 4096 1月   4 14:49 1907-suspend-list-self-twice
drwxr-xr-x 6 wendong xcvmg 4096 1月   4 14:49 2007-virtual-structural-finalizable
drwxr-xr-x 4 wendong xcvmg 4096 1月   4 14:49 407-arrays
drwxr-xr-x 4 wendong xcvmg 4096 1月   4 14:49 507-boolean-test
drwxr-xr-x 4 wendong xcvmg 4096 1月   4 14:49 507-referrer
drwxr-xr-x 4 wendong xcvmg 4096 1月   4 14:49 607-daemon-stress
drwxr-xr-x 4 wendong xcvmg 4096 1月   4 14:49 707-checker-invalid-profile
drwxr-xr-x 4 wendong xcvmg 4096 1月   4 14:49 807-method-handle-and-mr
drwxr-xr-x 4 wendong xcvmg 4096 1月   4 14:49 907-get-loaded-classe
```
