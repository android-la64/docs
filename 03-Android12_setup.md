# 1. 环境和源码

### 1.1 建立工作目录

建立工作目录:

```bash
## Workspace
cd $LA_WS

mkdir -p $LA_WS/aosp.la
cd loongson/aosp.la
```

### 1.2 设置ssh

此处不再赘述 - 请注意一定采用类似下面的方法在同步代码前设置好ssh密码

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_rsa  ## 根据自己的情况设置为不同的id_rsa
```



### 1.3 源代码 - github同步源代码

```bash
## initialize repo
$ repo init -u git@github.com:android-la64/manifest.git -b aosp12-larch

## synchronize repo - 大概130G
$ repo sync -c
```



# 2. 编译运行

### 2.1 编译前的准备工作

有两部分代码由于环境不尽相同，在编译前需要根据自己的运行环境确认、修订。

`device/generic/goldfish/init.ranchu-core.sh`以及`device/loongson/loongsonboard/loongson_3a5000/fstab.loongson_3a5000`

```diff
# device/generic/goldfish/init.ranchu-core.sh
# 此处配置的是Android的IP地址，根据自己路由器的设置配置
diff --git a/init.ranchu-core.sh b/init.ranchu-core.sh
index a9e07084..ad34e9e7 100755
--- a/init.ranchu-core.sh
+++ b/init.ranchu-core.sh
@@ -11,6 +11,6 @@ case "$allowsuspend" in
     ;;
 esac
 /system/bin/ifconfig eth0 up
-/system/bin/ifconfig eth0 192.168.2.2
+/system/bin/ifconfig eth0 192.168.3.10
 sleep 10


# device/loongson/loongsonboard/loongson_3a5000/fstab.loongson_3a5000
# 此文件需要根据自己机器的硬盘情况修改

```



### 2.2 编译image

```bash
## 1. 设置环境
## 注意，任何一个运行aosp命令的终端都必须执行以下两个命令

. build/envsetup.sh
# export OUT_DIR=out.3a
lunch loongson_3a5000-eng


## 2. 编译
$ m
```



### 2.3 烧录

编译完成后可以使用测试机中的loongnix系统下载镜像并烧录。
这里提供一下烧录脚本，用于从源码所在的服务器或者工作机上把镜像下载到本地并烧录到对应分区

```bash
#!/bin/bash
server=172.17.103.55 
password=****
username=dongzhe
remote_dir=/home/dongzhe/loongson/aosp.la/out/target/product/generic_loongarch64
image_files=(
        "super.img"
        "userdata.img"
        "ramdisk.img"
        "cache.img"
        )
for file in "${image_files[@]}"; do
        sshpass -p "${password}" scp  "${username}@${server}:${remote_dir}/${file}" ./
        if [ $? -eq 0 ];then
                echo "copy successed: ${file}"
        else
                echo "copy failed: ${file}"
        fi
done
```

烧录命令（根据硬盘配置情况修订脚本）：

```bash
sudo dd if=super.img     of=/dev/sda1 bs=32M
sudo dd if=userdata.img  of=/dev/sda2 bs=32M
sudo dd if=cache.img     of=/dev/sda3 bs=32M

sudo mkfs.ext4 /dev/sda4
```



### 2.4 ramdisk

ramdisk文件一旦确定，后续基本不变，所以，正常情况下，只需要更改一次即可。

将ramdisk.img 放置在 `/boot`目录下 （参考第3节，关于启动参数中描述的ramdisk）。



### 2.5. 设置启动参数

启动3A5000上的 linux系统，编辑/boot/grub/grub.cfg文件，增加如下内容（目前采用的名字是Android12）：

```bash
menuentry 'Android12' {
  echo	'Loading Android ...'
	set root='hd0,gpt2'
	linux  /vmlinuz.efi console=ttyS0,115200 norandmaps earlycon enforcing=0 init=/init  rw rootfstype=cpio androidboot.hardware=loongson_3a5000 androidboot.selinux=permissive loglevel=1 androidboot.boot_devices=pci0000:00/0000:00:08.2 loglevel=8 printk.devkmsg=on androidboot.verifiedbootstate=orange
  echo  'Loading initrd...'
	initrd	/ramdisk.img
}

```

注意：boot_devices需与sda所在的pci地址保持一致

```bash
## 参考A.1 中描述的步骤、内容

androidboot.boot_devices=pci0000:00/0000:00:08.2
```

编辑后，并使用sudo update-grub，启动时可以在grub界面选择进入android12。

特别注意上述参数： `androidboot.hardware=loongson_3a5000`





# 3. 功能测试

在本章，我们列出部分关键库在Android上的测试。

### 3.1 bioinc编译测试

详见[./02-bionic.md](./02-bionic.md)



### 3.2 art host端单元测试

#### 3.2.1 编译

```bash
# 编译
$ art/tools/buildbot-build.sh --host
```



#### 3.2.2 Host上Gtest测试

```bash
# 测试 : 这两个命令是等价的，必须执行一次以下命令，然后对应的目录才有可执行文件
$ art/test.py --host -g --64
## 或
$ m test-art-host-gtest64
```

**详细描述：**

上述编译命令生成的测试用例在以下目录 `out/host/linux-x86/nativetest64`，可以直接**执行其中的文件**以测试单个测试用例。

**测试结果：** - 存在2个测试用例有错误（这是两个ARM的测试用例，不用管）

```bash
$ art/test.py --host -g --64
...
PASSING TESTS
test-art-host-gtest-art_cmdline_tests64
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
FAILING TESTS
test-art-host-gtest-art_compiler_host_tests64
test-art-host-gtest-art_compiler_tests64


[  FAILED  ] ArmVIXLAssemblerTest.VixlJniHelpers
[  FAILED  ] ArmVIXLAssemblerTest.VixlLoadFromOffset
[  FAILED  ] ArmVIXLAssemblerTest.VixlStoreToOffset
[  FAILED  ] AssemblerX86_64Test.Movss
[  FAILED  ] AssemblerX86_64Test.Movsd

[  FAILED  ] 2 tests, listed below:
[  FAILED  ] DwarfTest.DebugFrame
[  FAILED  ] DwarfTest.x86_64_RegisterMapping
======= 以上错误都是由于采用了Clang15造成的  对比的汇编格式有变化造成的  =========

```



#### 3.2.3 Host上Java测试

这个测试其实是测试JVM的，**对于Devices上的验证没有用处**。除非开发新的测试用例，用此种方法验证测试用例本身，

```bash
# 测试
art/test.py --host -r --64  
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

只有一个SIMD的测试用例是错误的。



### 3.3 art device端单元测试

#### 3.3.1 编译与上传

```bash
# 编译
$ m & art/tools/buildbot-build.sh --target
## 经过此次编译后，对应的image文件会发生变化（/apex）,此image文件不再可以启动qemu，必须删除重新编译image文件才行

# 将测试文件push到device上： push.sh 在TOP目录下
$ . push.sh
```



#### 3.3.2 Device上Gtest测试

```bash
## 可以是独立的测试
$ art/tools/run-gtests.sh -j4
```



#### 3.3.3 Device上Java测试

```bash
# 测试所有用例
$ art/test.py -j 4 --target -r --64 --ndebug  --interpreter -v 
## 上述命令大概需要40分钟
$ art/test.py -j 4 --target -r --64 --no-prebuild --ndebug --no-image  --interpreter

# 测试001-Main
art/test.py -r --target --no-prebuild --debug --no-image --64 --interpreter -j 1 -t 001-Main
```



### 3.4 system/core测试

```bash
## 编译（build）
$ . art/xc_tools/system_core_g_b.sh
## 具体内容请参考上述脚本


## 运行（run)
$ . art/xc_tools/system_core_g_r.sh
```



### 3.5 system/unwinding测试

```bash
## 编译（build）
$ . art/xc_tools/system_unwinding_g_b.sh
## 具体内容请参考上述脚本


## 运行（run)
$ . art/xc_tools/system_unwinding_g_r.sh
```



# 4. 键盘操作

由于在某些情况下（无触摸屏），操作不方便，因此，本章简单列出一些常用的键盘操作命令。

### 4.1 键盘命令

在窗口输入如下命令：

```bash
$ adb shell input keyevent xx
```

其中xx常用的值如下

| 功能                                                         | 键值 |
| ------------------------------------------------------------ | ---- |
| Home                                                         | 3    |
| ### <font style="color:rgb(32, 33, 36);">KEYCODE_EXPLORER</font> | 64   |
| ### <font style="color:rgb(32, 33, 36);">KEYCODE_SETTINGS</font> | 176  |
| ### <font style="color:rgb(32, 33, 36);">KEYCODE_APP_SWITCH</font> | 187  |
| ### <font style="color:rgb(32, 33, 36);">KEYCODE_CONTACTS</font> | 207  |
| ### <font style="color:rgb(32, 33, 36);">KEYCODE_CALENDAR</font> | 208  |
| ### <font style="color:rgb(32, 33, 36);">KEYCODE_MUSIC</font> | 209  |
| ### <font style="color:rgb(32, 33, 36);">KEYCODE_CALCULATOR</font> | 210  |
|                                                              |      |
| ### <font style="color:rgb(32, 33, 36);">KEYCODE_ALL_APPS</font> | 284  |
|                                                              |      |
|                                                              |      |
|                                                              |      |

上述键值的完整在参考这里：[https://developer.android.com/reference/android/view/KeyEvent#KEYCODE_CALENDAR](https://developer.android.com/reference/android/view/KeyEvent#KEYCODE_CALENDAR)

比如：

```bash
# Input key event
$ adb shell input keyevent xxx # 3 = HOME

# Run setting app
$ adb shell am start com.android.settings

## List all installed apps
$ adb shell pm list packages -f

# Input
$ adb shell input tap 500 400

# Perform a swipe gesture from the center to top
$ adb shell input swipe 500 1000 500 100
```



### 4.2 不常用键盘命令

```bash
# Simulates a tap/click event at the specified coordinates (x, y) 
# on the Android device’s screen using the mouse input.
$ adb shell input mouse tap x y
$ adb shell input mouse tap 200 300

# Simulates a swipe gesture starting from coordinates (x1, y1) to 
# coordinates (x2, y2) on the Android device’s screen.
$ adb shell input swipe x1 y1 x2 y2
$ adb shell input swipe 300 500 300 300

## Simulates text input by typing the specified ‘string’ on the 
# Android device’s active text input field.
$ adb shell input text ‘string’
$ adb shell input text ‘Hello, World!’

# Input
$ adb shell input tap 500 400


```



参考： [https://medium.com/@samirdubey/mastering-adb-enhancing-android-aosp-development-with-powerful-commands-74c3b10b957c](https://medium.com/@samirdubey/mastering-adb-enhancing-android-aosp-development-with-powerful-commands-74c3b10b957c)  



### 4.3 屏幕capture

```bash
$ adb shell screencap -p /sdcard/screen.jpg
# Capture a screenshot of the Android device’s screen and 
# save it as “screen.jpg” in the /sdcard directory.

$ adb shell screenrecord sdcard/record.mp4
# Record the screen activity of the Android device and save it 
# as “record.mp4” in the /sdcard directory.
```



### 4.4 包管理命令

```bash
$ adb shell pm list package
# Lists all installed packages on the Android device.

$ adb shell pm list package -s
# Lists only system packages (pre-installed apps) on the Android device.

$ adb shell pm list package -3
# Lists third-party packages (user-installed apps) on the Android device.

$ adb shell pm list package -f
# Lists all installed packages along with their associated APK file paths.

$ adb shell pm list package -¡
# Lists all packages with their installer (installer package) on the Android device.

$ adb shell pm path com.example.com
# Shows the APK file path for the package “com.example.com”.

$ adb shell pm dump com.example.com
# Dumps detailed information about the package “com.example.com”, including permissions, activities, and services.

```



### 4.5 控制旋转命令

```bash
$ adb shell settings put system user_rotation 1   ## 0, 1,2,3
```





# A. 硬件描述

本节主要描述机器硬盘以及对应的fstab处理，GPU信息等内容。

### A.1 硬盘信息确认

为了方便调试采用的是双系统方案:

1. 单一硬盘：硬盘分区格式是gpt格式，其中分区1-4是loongnix系统，分区5-8分别是安卓的super,data,cache,metadata分区，<span style="color:blue">**分区9用来放置内核和ramdisk等一些启动程序**</span>。
2. 双硬盘：如果是两个独立硬盘，其信息如下（熵核目前采用的方式）【<span style="color:blue">**注意/dev/sda5-boot分区暂时没有使用** </span>】: 

```bash
# 第一步，获取硬盘信息
$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda      8:0    0 238.5G  0 disk 
├─sda1   8:1    0    16G  0 part 
├─sda2   8:2    0     8G  0 part 
├─sda3   8:3    0     2G  0 part 
├─sda4   8:4    0     1G  0 part 
└─sda5   8:5    0     1G  0 part 
sdb      8:16   0 238.5G  0 disk 
├─sdb1   8:17   0   300M  0 part /boot/efi
├─sdb2   8:18   0   300M  0 part /boot
├─sdb3   8:19   0  41.3G  0 part /
├─sdb4   8:20   0  41.3G  0 part 
├─sdb5   8:21   0 146.5G  0 part /opt
│                                /root
│                                /home
│                                /var
│                                /data
└─sdb6   8:22   0   8.8G  0 part [SWAP]


## 第二步分区，我们将sda硬盘作为Android启动硬盘，并做如下分区（一定采用gdisk工具进行分区）
$ sudo gdisk -l /dev/sda
GPT fdisk (gdisk) version 1.0.3

Partition table scan:
  MBR: protective
  BSD: not present
  APM: not present
  GPT: present

Found valid GPT with protective MBR; using GPT.
Disk /dev/sda: 500118192 sectors, 238.5 GiB
Model: GG46T256S3C27   
Sector size (logical/physical): 512/512 bytes
Disk identifier (GUID): BBF00249-692F-416D-AEE5-FFCE604D1B99
Partition table holds up to 128 entries
Main partition table begins at sector 2 and ends at sector 33
First usable sector is 34, last usable sector is 500118158
Partitions will be aligned on 2048-sector boundaries
Total free space is 441397869 sectors (210.5 GiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048        33556479   16.0 GiB    8300  super
   2        33556480        50333695   8.0 GiB     8300  data
   3        50333696        54527999   2.0 GiB     8300  cache
   4        54528000        56625151   1024.0 MiB  8300  metadata
   5        56625152        58722303   1024.0 MiB  8300  boot

## 注意，这里的 boot分区暂时用不上，如前面的蓝色文字描述的，boot相关的信息都放在了系统盘（loongnix）的boot分区中


## 第三步，获取Android硬盘的设备号
$ udevadm info -q path -n /dev/sda  ## 或者使用 -q all
/devices/pci0000:00/0000:00:08.2/ata3/host2/target2:0:0/2:0:0:0/block/sda
$ lspci -vv -s 0000:00:08.2
# 上述显示的设备信息 pci0000:00/0000:00:08.2 会在配置文件 ‘/boot/grub/grub.cfg’ 中使用

#或者采用命令获取上述信息
$ realpath /sys/block/sda
/sys/devices/pci0000:00/0000:00:08.2/ata3/host2/target2:0:0/2:0:0:0/block/sda

## 第四步

```

上述信息信息`pci0000:00/0000:00:08.2`记录下来要放到安卓内核启动命令行: androidboot.boot_devices=pci0000:00/0000:00:08.2。 安卓的init程序依赖这个参数去找到安卓对应的分区。

注意<span style="color:blue">**分区必须使用GPT分区而不是传统的MBR分区**</span>，建议使用gdisk软件。推荐的硬盘分区如下：

```bash
### TODO
$ sudo echo -e 'n\n\n\n+16G\n\nn\n\n\n+8G\n\nn\n\n\n+8G\n\nn\n\n\n+32M\n\nn\n\n\n+4G\n\nc\n1\nsuper\nc\n2\ndata\nc\n3\ncache\nc\n4\nmetadata\nc\n5\nboot\nwq\nY\n' | sudo gdisk /dev/sdb
# partion 1: super
# partion 2: userdata
# partion 3: cache
# partion 4: metadata
# partion 5: boot

```

具体分区大小可以改变，但不能小于安卓设定的相关分区image大小。



如果是一个硬盘，分区信息大概如下（龙芯采用的方式）

```bash
loongson@loongson-pc:~$ sudo gdisk
GPT fdisk (gdisk) version 1.0.3

Type device filename, or press <Enter> to exit: /dev/sda
Partition table scan:
  MBR: protective
  BSD: not present
  APM: not present
  GPT: present

Found valid GPT with protective MBR; using GPT.

Command (? for help): p
Disk /dev/sda: 500118192 sectors, 238.5 GiB
Model: YMTC SC001-256GB
Sector size (logical/physical): 512/512 bytes
Disk identifier (GUID): D647FC08-8976-4385-A183-BB1F67ACB5DA
Partition table holds up to 128 entries
Main partition table begins at sector 2 and ends at sector 33
First usable sector is 34, last usable sector is 500118158
Partitions will be aligned on 2-sector boundaries
Total free space is 146565775 sectors (69.9 GiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1              34          616447   301.0 MiB   0700  
   2          616448         1230847   300.0 MiB   8300  
   3         1230848       282249215   134.0 GiB   8300  
   4       282249216       315803647   16.0 GiB    8200  
   5       315803648       336775167   10.0 GiB    8300  super
   6       336775168       345163775   4.0 GiB     8300  data
   7       345163776       349358079   2.0 GiB     8300  cache
   8       349358080       351455231   1024.0 MiB  8300  metadata
   9       351455232       353552383   1024.0 MiB  8300  boot
```

单硬盘模式的其他内容不再赘述。




### A.2. 硬盘分区配置

需要注意的是，fstab需要与硬盘分区信息一致。我们设置了两个fstab文件，分别是

文件`fstab.loongson_3a5000.ramdisk`用于ramdisk.

```bash
$ cat loongson_3a5000/fstab.loongson_3a5000.ramdisk 
#<src>              <mnt_point>         <type>    	<mnt_flags and options>               	<fs_mgr_flags>
system    			/system             ext4      	ro,barrier=1                          	wait,logical,first_stage_mount
vendor    			/vendor             ext4      	ro,barrier=1                          	wait,logical,first_stage_mount


$ cat loongson_3a5000/fstab.loongson_3a5000
#<src>              <mnt_point>         <type>    	<mnt_flags and options>                               	<fs_mgr_flags>
system    			/system             ext4      	ro,barrier=1                                          	wait,logical,first_stage_mount
vendor    			/vendor             ext4      	ro,barrier=1                                          	wait,logical,first_stage_mount

/dev/block/sda2 	/data       		ext4      	noatime,nosuid,nodev,nomblk_io_submit,errors=panic   	wait,check,quota,reservedsize=128M,latemount
/dev/block/sda3 	/cache      		ext4      	noatime,nosuid,nodev,nomblk_io_submit,errors=panic  	wait,check,quota,reservedsize=128M,latemount
/dev/block/sda4 	/metadata   		ext4    	noatime,nosuid,nodev    							wait,formattable,latemount

```



### A.3 GPU信息确认

3A5000测试机，Radeon 520OEM显卡,串口用于连接3A5000测试机以及工作机（串口软件minicom或者picocom），HDMI线连接显卡和显示器（VGA线暂时不支持）

```
$ lspci |grep "VGA compatible controller" | awk '{print $1}' |xargs lspci -v -s

06:00.0 VGA compatible controller: Advanced Micro Devices, Inc. [AMD/ATI] Oland [Radeon HD 8570 / R5 430 OEM / R7 240/340 / Radeon 520 OEM] (rev 87) (prog-if 00 [VGA controller])
        Subsystem: Bitland(ShenZhen) Information Technology Co., Ltd. Radeon 520 OEM
        Flags: bus master, fast devsel, latency 0, IRQ 48, NUMA node 0
        Memory at e0030000000 (64-bit, prefetchable) [size=256M]
        Memory at e0041000000 (64-bit, non-prefetchable) [size=256K]
        I/O ports at 4000 [size=256]
        Expansion ROM at e0041040000 [disabled] [size=128K]
        Capabilities: <access denied>
        Kernel driver in use: radeon
        Kernel modules: radeon, amdgpu
        
```









# B. 调试



### B.1 Tombstone

Tombstone是指发生应用程序崩溃或系统崩溃时生成的一种特殊类型的日志文件。Tombstone文件通常包含了导致崩溃的信息，例如堆栈跟踪、寄存器值等。

Tombstone文件通常位于`/data/tombstones`目录下，文件名通常以“tombstone_”开头，后跟ID。

```bash
# 输出tombstone文件列表
$ adb shell ls /data/tombstones
                                                                
tombstone_00     tombstone_08.pb  tombstone_17     tombstone_25.pb  tombstone_34     tombstone_42.pb
tombstone_00.pb  tombstone_09     tombstone_17.pb  tombstone_26     tombstone_34.pb  tombstone_43
...
tombstone_07.pb  tombstone_16     tombstone_24.pb  tombstone_33     tombstone_41.pb
tombstone_08     tombstone_16.pb  tombstone_25     tombstone_33.pb  tombstone_42

# 输出tombstone_00的内容
$ adb shell cat /data/tombstones/tombstone_00
*** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
Build fingerprint: 'Android/aosp_loongarch64/generic_loongarch64:12/SP1A.210812.016/eng.yuanji.20240228.123014:eng/test-keys'
Revision: '0'
ABI: 'loongarch64'
Timestamp: 2024-03-12 08:40:11.012352070+0000
Process uptime: 1s
Cmdline: /apex/com.android.art/bin/art/loongarch64/art_compiler_tests --gtest_filter=LinearizeTest.CFG6
pid: 4555, tid: 4555, name: art_compiler_te  >>> /apex/com.android.art/bin/art/loongarch64/art_compiler_tests <<<
uid: 0
signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0x0
Cause: null pointer dereference
    r2  00007fffe7d90050  r4  0000000000000000  r5  00007fffe79e0f68  r6  00007fffe7964068
    r7  0000000000000000  r8  0000000000000000  r9  0000000000040000  r10 0000000000000000
    r11 0000000000000001  r12 0000000000000000  r13 00007fffd8da3d48  r14 00000000000000c4
    r15 00007fffe726829c  r16 000000000059b284  r17 000000000a000000  r18 000000007fffffff
    r19 00007ffffc100010  r20 0000000000000000  r21 0000000000000000  r22 0000000000000000
    r23 0000000000000000  r24 00007fffe79e0f68  r25 0000000000000000  r26 00007ffffbc9e7b8
    r27 00007ffffbc9e868  r28 0000000000000018  r29 000000000000007f  r30 0000000000000000
    r31 0000000000000005
    pc  0000000000000000  ra  00007fffd8d72030  sp  00007ffffbc9e760

backtrace:
      #00 pc 0000000000000000  <unknown>
      #01 pc 00000000002f402c  /apex/com.android.art/lib64/libartd-compiler.so (art::SsaLivenessAnalysis::NumberInstructions()+584) (BuildId: affcacc21144368505ed70949c62eecb)
      #02 pc 000000000000715c  [anon:scudo:primary]
...
```



### B.2 art device端gdb调试

在测试前一定需要对“3.3 art device端单元测试”章节有充分的了解。

下面以001-Main为例

在【终端1】 启动gdb server端

```bash
# 启动gdb server
$ art/test.py -r --target --no-prebuild --debug --no-image --64 --interpreter -j 1 -t 001-Main --gdb
...
Forward :5039 to local port and connect GDB
Process /apex/com.android.art/bin/dalvikvm64 created; pid = 5668
Listening on port 5039
```

在【终端2】 启动gdb client端

```bash
# 设置端口转发
$ adb forward tcp:5039 tcp:5039

# 启动gdb。该文件位于loongson-gnu-toolchain-8.3-x86_64-loongarch64-linux-gnu-rc1.2
$ loongarch64-linux-gnu-gdb

# 输入gdb命令
(gdb) target extended-remote :5039
Remote debugging using :5039
No executable file now.
warning: Could not load vsyscall page because no executable was specified
0x00007fffd49373a0 in ?? ()

## 暂时因未知原因无法在主程序上设置断点，只能在库函数上设置断点。
## 因此通过执行continue后马上按ctrl + c来绕过无法b main的问题
(gdb) c
Continuing.
^C
Program received signal SIGINT, Interrupt.
0x00007ffd3577a028 in ?? ()

## 加载符号信息
(gdb) set sysroot out.test/target/product/generic_loongarch64/symbols
Reading symbols from out.test/target/product/generic_loongarch64/symbols/system/lib64/liblog.so...done.
...
(gdb) set solib-search-path out.test/target/product/generic_loongarch64/symbols/system/system_ext/apex/com.android.art.testing/lib64
Reading symbols from ...

## 设置断点
(gdb) b art_quick_invoke_static_stub
Breakpoint 1 at 0x7ffd3e53ab00: file art/runtime/arch/loongarch64/quick_entrypoints_loongarch64.S, line 314.

## 运行到断点
(gdb) c
Continuing.

Breakpoint 1, art_quick_invoke_static_stub ()
    at art/runtime/arch/loongarch64/quick_entrypoints_loongarch64.S:314
314	    INVOKE_STUB_CREATE_FRAME
```







# C. 常用命令

### C.1 logcat

logcat是一个用于查看系统日志的命令行工具。它允许你查看应用程序、系统服务以及其他系统组件产生的日志消息。

以下是一些常用的logcat命令：

1. **查看所有日志：**

   ```bash
   $ adb logcat
   ```

   这条命令将输出所有的日志消息，包括应用程序、系统服务等的日志消息。

2. **过滤日志：**

   ```bash
   $ adb logcat <TAG>:<LEVEL>
   ```

   这条命令将输出指定标签（TAG）和级别（LEVEL）的日志消息。例如，要查看特定应用程序的日志消息，可以使用应用程序的标签和级别，例如：

   ```bash
   $ adb logcat MyAppTag:D
   ```

   这将输出MyAppTag标签的调试（Debug）级别的日志消息。

3. **保存日志到文件：**

   ```bash
   $ adb logcat -d > logfile.txt
   ```

   这条命令将所有的日志消息保存到指定的文件中。

4. **清除日志缓冲区：**

   ```bash
   $ adb logcat -c
   ```

   这条命令将清除日志缓冲区中的所有日志消息。

以上是一些常用的logcat命令示例。logcat还有许多其他选项和功能，你可以使用`adb logcat --help`来查看更多用法和选项。



### C.2 VM属性

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





### C.3 DUMP OAT文件

```bash
$ adb shell "oatdump --oat-file=/system/framework/riscv64/boot-framework.oat" > boot-framework.dump
```





### C.4 QEMU编译

```bash
repo init -u git@github.com:riscv-android-src/manifest.git -b emu-31.2.1.0-riscv64
repo sync
cd external/qemu/
./android/rebuild.sh
ls objs/distribution/

./android/rebuild.sh  --enable-debug --enable-debug-info  --enable-debug-stack-usage
```





### C.5 模拟器常用命令

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

