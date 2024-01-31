# 1. 环境和源码

### 1.1 建立工作目录

建立工作目录:

```bash
cd
mkdir -p loongson/aosp.la
cd loongson/aosp.la

export aosp_ws=`pwd`
```



### 1.2 设置ssh

此处不再赘述 - 请注意一定采用类似下面的方法在同步代码前设置好ssh密码

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_rsa  ## 根据自己的情况设置为不同的id_rsa
```



### 1.3 源代码 - 熵核内部

在服务器192.168.3.100上，解压文件：`/back/comm_back/1-android//aosp.ali.android12.20230606.tar`

按照如下步骤就可以得到2022-12-22日的代码：

```bash
$ cd $ws ## 个人的workspace

# 解压得到的 AOSP 工程目录 - aosp.la
$ tar xf /back/comm_back/1-android/loongarch/aosp.src.la.tar
$ cd aosp.la
$ repo sync -l -c

## 同步最新代码 # --force-remove-dirty --force-sync
$ repo sync -c
```



### 1.4 源代码 - github同步源代码

```bash
## initialize repo
$ repo init -u git@github.com:android-la64/manifest.git -b aosp12-larch

## synchronize repo - 大概130G
$ repo sync -c

```



### 1.5 环境设置

#### 1.5.1 设置Clang

由于Clang编译器所在的git太大（超过10G），不被github接受，所以，采用自行编译的Clang方式。

```bash
## 以为本地的目录为例
$ tree -L 1
.
├── aosp.la
├── clang_la
├── myenv.sh
├── ndk23
├── proxy.sh
└── sync.github.sh


## 设置编译器
## 编译好的编译器在： clang_la/out/install/linux-x86/clang-r468909b
$ cd aosp.la/
$ cd prebuilts/clang/host/linux-x86/
$ ln -s ../../../../../clang_la/out/install/linux-x86/clang-r468909b ./clang-r468909b
## 注意上述软链接要使用相对路径，不要用绝对路径。当然，直接copy到 目录prebuilts/clang/host/linux-x86/
##  下也行

```



#### 1.5.2 Clang-runtime

由于目前Clang的runtime部分还存在错误，不分库不能生成，但是编译aosp的时候又需要检查这些库，所以，暂时通过下面的命令生成一些假的。

```bash
## 以下命令可以拷贝到一个shell脚本中执行
$ cd prebuilts/clang/host/linux-x86/clang-r468909b/lib64/clang/15.0.3/lib/linux/

touch libclang_rt.hwasan-loongarch64-android.so 

cd -
```

# 2. 编译loongarch64

### 2.1 编译image

编译image以及art测试用例

```bash
## 设置环境
## 注意，任何一个运行aosp命令的终端都必须执行以下两个命令
. build/envsetup.sh
lunch aosp_loongarch64-eng


## 2. 编译
$ m
```



### 2.2 bioinc的编译测试

```bash
## 编译 - 目前这些编译都存在错误，需要bioinic模块负责人修订
$ m libBionicTests
$ m bionic-unit-tests-static
$ m bionic-fortify-runtime-asan-test 
$ m heap_tagging_static_sync_helper


## 以下可以正常编译，但是暂时无机器环境执行
$ m malloc_debug_system_tests linker-unit-tests exec_linker_helper
$ m cfi_test_helper cfi_test_helper2
```



### 2.3 art编译测试用例

```bash

```

# 3. 功能测试 - 【待整理】

每个命令的详细解释，请参考第5章的内容。

### 3.1 Host上Gtest测试

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



#### 3.1.1 测试结果 - 20220320

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



### 3.2 Host上Java测试

这个测试其实是测试JVM的，**对于Devices上的验证没有用处**。

测试命令：

```bash
# 编译
$ art/tools/buildbot-build.sh --host

# 测试 
$ art/test.py --host --64 -r  --optimizing                     ## 仅仅是一个模式:optimizing
$ art/test.py --host --64 -v -r -t 001-Main --baseline --jit   ## 测试一个用例的两种模式
```



#### 3.2.1 测试结果 - 20220320

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



### 3.3 测试环境同步

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





### 3.5 Target上Java测试







# 4. 性能测试





# 5. ART测试概述





# 6. 在qemu上运行

### 6.1 编译image for qemu

```bash
## 设置环境
## 注意，任何一个运行aosp命令的终端都必须执行以下两个命令
. build/envsetup.sh
lunch sdk_phone64_loongarch64


## 2. 编译
$ m
```

编译完成后，`$ANDROID_PRODUCT_OUT`目录下会产生xxx-qemu.img文件

```shell
ls $ANDROID_PRODUCT_OUT/*-qemu.img
xxx/aosp.la64/out/target/product/emulator_loongarch64/product-qemu.img
xxx/aosp.la64/out/target/product/emulator_loongarch64/ramdisk-qemu.img
xxx/aosp.la64/out/target/product/emulator_loongarch64/system-qemu.img
xxx/aosp.la64/out/target/product/emulator_loongarch64/system_ext-qemu.img
xxx/aosp.la64/out/target/product/emulator_loongarch64/vendor-qemu.img
```

### 6.2 运行qemu

下载qemu运行环境

```shell
git clone ssh://git@8.140.33.210:2222/android/android_qemu_env.git -b a12_larch
```

运行编译好的image

```shell
cd android_qemu_env
./start-from-out-dir.sh
```

注意，运行这个脚本前需要先运行lunch命令`lunch sdk_phone64_loongarch64`

脚本内容如下：

```shell
$ cat start-from-out-dir.sh

#!/bin/bash

IMG_DIR=$ANDROID_PRODUCT_OUT
#IMG_DIR=$(pwd)

./qemu-system-loongarch64 \
        -M virt \
        -bios ./QEMU_EFI.fd -kernel ./vmlinuz.efi \
        -append "rootwait root=/dev/vda rw init=/ramdisk/init security=selinux androidboot.selinux=permissive printk.devkmsg=on androidboot.hardware=ranchu console=ttyS0,115200" \
        -initrd $IMG_DIR/ramdisk-qemu.img \
        -drive index=0,id=vendor,file=$IMG_DIR/vendor-qemu.img,read-only=on,if=none \
        -device virtio-blk-pci,drive=vendor \
        -drive index=1,id=userdata,file=$IMG_DIR/userdata.img,cache=unsafe,if=none \
        -device virtio-blk-pci,drive=userdata \
        -drive index=2,id=cache,file=$IMG_DIR/cache.img,cache=unsafe,if=none \
        -device virtio-blk-pci,drive=cache \
        -drive index=3,id=system,file=$IMG_DIR/system-qemu.img,read-only=on,if=none \
        -device virtio-blk-pci,drive=system \
        -drive index=4,id=product,file=$IMG_DIR/product-qemu.img,read-only=on,if=none \
        -device virtio-blk-pci,drive=product \
        -drive index=5,id=system_ext,file=$IMG_DIR/system_ext-qemu.img,read-only=on,if=none \
        -device virtio-blk-pci,drive=system_ext \
        -nographic \
        -smp 1 -m 3584M \
```





# A. uboot分区烧录方式

常规的Android采用Uboot + fastboot 方式进行。



### A.1 生成分区

启动板子进入uboot环境，设置`partitions`环境变量并保存

```bash
$ setenv partitions "xxx"
$ gpt write mmc 0 ${partitions}  ## 或者其他保存命令
```

上述命令中的xxx代表的信息如下（注意一定没有回车换行，下面加了回车换行只是为了阅读方便）：

```bash
name=sparse,size=2031kb;
name=bootpart_a,size=16MiB;name=bootpart_b,size=16MiB;
name=boot_a,size=32MiB;name=boot_b,size=32MiB;
name=vendor_boot_a,size=32MiB;name=vendor_boot_b,size=32MiB;
name=tee_a,size=32MiB;name=tee_b,size=32MiB;
name=dtbo_a,size=8MiB;name=dtbo_b,size=8MiB;
name=super,size=4096MiB;
name=vbmeta_a,size=1MiB;name=vbmeta_b,size=1MiB;
name=vbmeta_system_a,size=1MiB;name=vbmeta_system_b,size=1MiB;
name=misc,size=2MiB;name=metadata,size=16MiB;
name=userdata,size=-
```

在`$OUT`目录下生成的super.img包含如下四个分区文件

```bash
$ ll
total 1456272
-rw-r--r-- 1 wendong xcvmg   73031680 1月  11 14:30 product_a.img
-rw-r--r-- 1 wendong xcvmg          0 1月  11 14:05 product_b.img
-rw-r--r-- 1 wendong xcvmg 1097191424 1月  11 14:23 system_a.img
-rw-r--r-- 1 wendong xcvmg          0 1月  11 14:05 system_b.img
-rw-r--r-- 1 wendong xcvmg  140648448 1月  12 03:02 system_ext_a.img
-rw-r--r-- 1 wendong xcvmg          0 1月  11 14:05 system_ext_b.img
-rw-r--r-- 1 wendong xcvmg  185692160 1月  12 03:01 vendor_a.img
-rw-r--r-- 1 wendong xcvmg          0 1月  11 14:05 vendor_b.img
```



### A.2 烧写镜像

步骤如下

1. 板子上电后，在窗口A启动minicom（`sudo minicom -D /dev/ttyUSB2` ）。
2. 在窗口B1（image所在目录，比如 `/back/wendong/aosp/aosp_light_imags/20230810`）
3. 依次执行执行如下命令（其中部分命令在另外的窗口B2执行）：

```bash
## 窗口A  - minicom （任意目录都行）： sudo minicom -D /dev/ttyUSB2
## 窗口B1 - images所在目录（比如 /back/wendong/aosp/aosp_light_imags/20230810）
## 窗口B2 - rules所在目录（/back/wendong/aosp/aosp_light_imags/rules）

## 第一步 - 窗口B2
lsusb  ## 此时应该可以看到2345:7654 USB设备
sudo cp 51-android.rules.2345  /etc/udev/rules.d/51-android.rules
sudo cp 51-android.rules.2345  /lib/udev/rules.d/51-android.rules
sudo service udev restart
## 此时重新插拔板子上的USB线（大口的USB）

## 第二步 - 窗口B1
## fastboot可以采用当前目录下的fastboot
./fastboot flash ram u-boot-with-spl.bin
./fastboot reboot

## 第三步 - 窗口B2
lsusb  ## 此时应该可以看到 1234:8888 USB设备
sudo cp 51-android.rules.1234  /etc/udev/rules.d/51-android.rules
sudo cp 51-android.rules.1234  /lib/udev/rules.d/51-android.rules
sudo service udev restart
## 此时重新插拔板子上的USB线（大口的USB）


## 第四步 - 窗口B1
./fastboot flash uboot u-boot-with-spl.bin

## 烧录各个分区
./fastboot flash bootpart bootpart.ext4  ## 从20230905开始，这个文件名字被改为 bootpart.img ！！
## 也就是说，烧录老日期的image，这个文件名字是 bootpart.ext4 ；20230905日期之后的image，文件名字变为 bootpart.img 

./fastboot flash boot boot.img     ## 如果此处被分成了多于3个包下载，说明上述步骤没有作对
./fastboot flash vendor_boot vendor_boot.img
./fastboot flash super super.img   ## 如果此处被分成了多于13个包下载，说明上述步骤没有作对
./fastboot flash userdata userdata.img
./fastboot flash vbmeta vbmeta.img
./fastboot flash vbmeta_system vbmeta_system.img

## 重新划分分区后，需执行此命令，否则可能导致Android无法启动
./fastboot erase metadata 
./fastboot erase misc


### 完成！
## 第五步 - 窗口B2
sudo cp 51-android.rules.18d1  /etc/udev/rules.d/51-android.rules
sudo cp 51-android.rules.18d1  /lib/udev/rules.d/51-android.rules
sudo service udev restart
# 板子关电，拨回跳线，重启板子

## 重启后大约2分钟
lsusb        ## 此时应该可以看到 18d1:4ee7 USB设备
adb devices  ## 此时应该可以看到设备

## root
adb root
```







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
