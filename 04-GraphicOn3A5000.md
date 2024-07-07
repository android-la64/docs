# 1.硬件准备



### 1.1 硬盘信息确认

这里为了方便调试我采用的是双系统方案，硬盘分区格式是gpt格式，其中分区1-4是loongnix系统，分区5-8分别是安卓的super,data,cache,metadata分区，<span style="color:blue">**分区9用来放置内核和ramdisk等一些启动程序**</span>。

如果是两个独立硬盘，其信息如下（熵核采用的方式）【<span style="color:blue">**注意/dev/sda5-boot分区暂时没有使用** </span>】: 

```bash
# 第一步，获取硬盘信息
$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda      8:0    0 238.5G  0 disk 
├─sda1   8:1    0    16G  0 part 
├─sda2   8:2    0     8G  0 part 
├─sda3   8:3    0     8G  0 part 
├─sda4   8:4    0     8G  0 part 
├─sda5   8:5    0   256M  0 part 
└─sda6   8:6    0     2G  0 part 
sdb      8:16   0 238.5G  0 disk 
├─sdb1   8:17   0   300M  0 part /boot/efi
├─sdb2   8:18   0   300M  0 part /boot
├─sdb3   8:19   0  41.3G  0 part /
├─sdb4   8:20   0  41.3G  0 part 
├─sdb5   8:21   0 146.5G  0 part /var
│                                /root
│                                /opt
│                                /home
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
Total free space is 411513453 sectors (196.2 GiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048        33556479   16.0 GiB    8300  system
   2        33556480        50333695   8.0 GiB     8300  vendor
   3        50333696        67110911   8.0 GiB     8300  data
   4        67110912        83888127   8.0 GiB     8300  cache
   5        83888128        84412415   256.0 MiB   8300  metadata
   6        84412416        88606719   2.0 GiB     8300  boot
## 注意，这里的 boot分区暂时用不上，如前面的蓝色文字描述的，boot相关的信息都放在了系统盘（loongnix）的boot分区中


## 第三步，获取Android硬盘的设备号
$ udevadm info -q path -n /dev/sda  ## 或者使用 -q all
/devices/pci0000:00/0000:00:08.2/ata3/host2/target2:0:0/2:0:0:0/block/sda
$ lspci -vv -s 0000:00:08.2
# 上述显示的设备号， pci0000:00/0000:00:08.2 会在配置文件 ‘/boot/grub/grub.cfg’ 中使用

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



### 1.2 GPU硬件信息确认

3A5000测试机，Radeon 520OEM显卡,串口用于连接3A5000测试机以及工作机（串口软件minicom或者picocom），HDMI线连接显卡和显示器（VGA线暂时不支持）

```
loongson@loongson-pc:~$ lspci |grep "VGA compatible controller" | awk '{print $1}' |xargs lspc -v -s

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




# 2. 硬盘分区配置

需要注意的是，fstab需要与硬盘分区信息一致。我们设置了两个fstab文件，分别是

文件`fstab.loongson_3a5000.ramdisk`用于ramdisk： ``



```bash
$ cat fstab.loongson_3a5000
#<src>                  <mnt_point>         <type>    	<mnt_flags and options>                               	<fs_mgr_flags>

/dev/sda1    			/system             ext4      	ro,barrier=1                                          	wait,first_stage_mount
/dev/sda2    			/vendor             ext4      	ro,barrier=1                                          	wait,first_stage_mount

/dev/sda3    			/data               ext4      	noatime,nosuid,nodev,nomblk_io_submit,errors=panic    	wait
/dev/sda4 				/cache 				ext4      	noatime,nosuid,nodev,nomblk_io_submit,errors=panic   	wait,formattable,latemount
# XC-TODO: wait,check,quota,fileencryption=aes-256-xts:aes-256-cts,reservedsize=128M,latemount

/dev/sda5 				/metadata    		ext4    	noatime,nosuid,nodev    								wait,formattable,latemount


```
另外在goldfish的init.ranchu-core.sh中可以设置测试机的ip地址  
```
/system/bin/ifconfig eth0 up  
/system/bin/ifconfig eth0 192.168.2.2  
 sleep 10  
/system/bin/ip rule add from all lookup main pref 1                                                     
```


# 3.烧录

编译完成后可以使用测试机中的loongnix系统下载镜像并烧录。
这里提供一下烧录脚本，用于从源码所在的服务器或者工作机上把镜像下载到本地并烧录到对应分区

```bash
#!/bin/bash
server=172.17.103.55 
password=****
username=dongzhe
remote_dir=/home/dongzhe/loongson/aosp.la/out/target/product/generic_loongarch64
image_files=(
        "system.img"
        "userdata.img"
        "ramdisk.img"
        "vendor.img"
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



烧录

```bash
$ sudo dd if=system.img 	of=/dev/sda1 bs=50M
$ sudo dd if=vendor.img 	of=/dev/sda2 bs=50M
$ sudo dd if=userdat.img 	of=/dev/sda3 bs=50M

# Copy firmware & ramdisk to boot partition
device/arv/jh7110/boot/* to p3:/
out/target/product/jh7110/ramdisk.img to p3:/
```





# 4.启动参数

启动3A5000 linux系统，编辑/boot/grub/grub.cfg文件，增加如下内容（目前采用的是Android12）：

```bash
menuentry "android"{
   linux (hd0,gpt9)/vmlinuz.efi console=ttyS0,115200 norandmaps earlycon enforcing=0 init=/init rw rootfstype=cpio androidboot.selinux=permissive androidboot.hardware=loongson_3a5000 loglevel=1 androidboot.boot_devices=pci0000:00/0000:00:08.0 printk.devkmsg=on androidboot.verifiedbootstate=orange
   boot    
}

menuentry 'Android12' {
  echo	'Loading Android ...'
	set root='hd0,gpt2'
	linux  /vmlinuz.efi console=ttyS0,115200 norandmaps earlycon enforcing=0 init=/init  rw rootfstype=cpio  androidboot.selinux=permissive androidboot.hardware=loongson_3a5000 loglevel=1 androidboot.boot_devices=pci0000:00/0000:00:08.2 loglevel=8 printk.devkmsg=on androidboot.verifiedbootstate=orange
  echo  'Loading initrd...'
	initrd	/ramdisk.img
}

```
注意：boot_devices需与sda所在的pci地址保持一致
```bash
## 参考1.1 中描述的步骤、内容

androidboot.boot_devices=pci0000:00/0000:00:08.2
```
编辑后，并使用sudo update-grub，启动时可以在grub界面选择进入android12。

【内核可使用android_qemu_env下的`vmlinuz.efi`：  将安卓内核和ramdisk.img拷贝到/boot，分别为vmlinuz.efi和ramdisk.img 】





## 注意： 3A5000安卓内核

目前模拟器使用的安卓内核理论上可以直接用于3A5000真机，但是实际有两个问题需要处理：

1. 部分真机的BIOS尚未支持这个内核需要的initrd传递方式，initrd /ramdisk.img不起作用，建议将安卓编译出的ramdisk.img内置到内核

```bash
cp <android-out>/ramdisk.img <kernel source>/android-ramdisk.cpio.gz # (具体看选择了什么压缩，目前是gzip压缩算法）
cd <kernel source>/
gunzip android-ramdisk.cpio.gz
cp android-ramdisk.cpio ramdisk.cpio
```

make menuconfig，选择内置initrd(general setup/Initramfs source files)，配置后.config应该包含如下内容：

```bash
CONFIG_INITRAMFS_SOURCE="./android-ramdisk.cpio"
```



2. 真机的radeon显卡驱动需要加载一些固件，由于android.img没有内置这些文件，需要处理。一种办法是内置到内核，配置内核(Device Drivers/Generic Driver Options/Firmware Loader/Firmware blobs root directory和firmware loading facility),参考配置如下（根据显卡型号不同内置文件也不同）。另一种办法是将linux /lib/firmware目录下相关文件加入ramdisk.img. 

```bash
CONFIG_EXTRA_FIRMWARE="radeon/oland_pfp.bin radeon/oland_me.bin radeon/oland_smc.bin radeon/oland_ce.bin radeon/oland_rlc.bin radeon/oland_mc.bin radeon/oland_k_smc.bin"
CONFIG_EXTRA_FIRMWARE_DIR="/lib/firmware"
```



# 5. 当前步骤

1. 最新repo代码编译

2. 配置文件在`device/loongson/loongsonboard/loongson_3a5000`中。

3. `m`之后，生成如下image

   ```bash
   $ ll *.img
   -rw-rw-r-- 1 lihf lihf    1683054 Jul  7 17:49 ramdisk-debug.img
   -rw-rw-r-- 1 lihf lihf    1517023 Jul  7 17:49 ramdisk.img
   -rw-rw-r-- 1 lihf lihf    1683104 Jul  7 17:49 ramdisk-test-harness.img
   -rw-rw-r-- 1 lihf lihf 2147483648 Jul  7 03:38 system.img
   -rw-rw-r-- 1 lihf lihf  536870912 Jul  7 03:38 userdata.img
   -rw-rw-r-- 1 lihf lihf  268435456 Jul  7 17:49 vendor.img
   ```

4. 将system，userdata，vendor分别按照第三节的步骤用dd命令烧录。

5. 将`ramdisk.img`拷贝到 龙芯机器的 /boot目录下

6. 修订文件`/boot/grub/grub.cfg`如下

   ```
   menuentry 'Android12' {
     	echo	'Loading Android ...'
   	set root='hd0,gpt2'
   	linux  /vmlinuz.efi console=ttyS0,115200 norandmaps earlycon enforcing=0 init=/init  rw rootfstype=cpio  androidboot.selinux=permissive androidboot.hardware=loongson_3a5000 loglevel=1 androidboot.boot_devices=pci0000:00/0000:00:08.2 loglevel=8 printk.devkmsg=on androidboot.verifiedbootstate=orange
       echo  'Loading initrd...'
   	initrd	/ramdisk.img
   }
   ```

7. 启动后，找不到 /system/bin/init，错误信息如下： `init: execv("/system/bin/init") failed: No such file or directory`

   ```
   [    6.855961][    T1] This architecture does not have kernel memory protection.
   [    6.863058][    T1] Run /init as init process
   [    6.867383][    T1]   with arguments:
   [    6.871016][    T1]     /init
   [    6.873957][    T1]   with environment:
   [    6.877764][    T1]     HOME=/
   [    6.880792][    T1]     TERM=linux
   [    6.884167][    T1]     BOOT_IMAGE=/vmlinuz.efi
   [    6.889779][    T1] init: init first stage started!
   [    6.894698][    T1] init: Unable to open /lib/modules, skipping module loading.
   [    6.902093][    T1] init: Copied ramdisk prop to /second_stage_resources/system/etc/ramdisk/build.prop
   [    6.911462][    T1] init: [libfs_mgr]ReadFstabFromDt(): failed to read fstab from dt
   [    6.919400][    T1] init: Using Android DT directory /proc/device-tree/firmware/android/
   [    6.955276][    T1] init: bool android::init::BlockDevInitializer::InitDevices(std::set<std::string>): partition(s) not found in /sys, waiting for their uevent(s): super
   [   16.980391][    T1] init: Wait for partitions returned after 10010ms
   [   16.986715][    T1] init: bool android::init::BlockDevInitializer::InitDevices(std::set<std::string>): partition(s) not found after polling timeout: super
   [   17.000495][    T1] init: Failed to mount required partitions early ...
   [   17.007085][    T1] init: Skipped setting INIT_AVB_VERSION (not in recovery mode)
   [   17.014584][    T1] init: execv("/system/bin/init") failed: No such file or directory
   [   17.022611][    T1] init: InitFatalReboot: signal 6
   [   17.031244][    T1] init: #00 pc 00000001201e5ab8  /init (UnwindStackCurrent::UnwindFromContext(unsigned long, void*)+124)
   [   17.042242][    T1] init: Reboot ending, jumping to kernel
   [   17.047708][    T1] kvm: exiting hardware virtualization
   [   17.053039][    T1] sd 3:0:0:0: [sdb] Synchronizing SCSI cache
   [   17.059425][    T1] sd 2:0:0:0: [sda] Synchronizing SCSI cache                                           
   [   17.386172][    T0] NOHZ tick-stop error: Non-RCU local softirq work is pending, handler #80!!!          
   [   18.162937][    T1] reboot: Restarting system with command 'bootloader'    
   
   ```

   

8. 这里存在的问题可能有两个

   1. ramdisk.img并未编译到kernel里面（存在新的fstab）
   2. 文件`fstab.loongson_3a5000.ramdisk`内容存在问题 - 这个可能性不大（因为我们在其他平台也是这样处理的）

9. 



