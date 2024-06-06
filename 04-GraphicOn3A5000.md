---
titile: Android在3A5000上启动图形栈说明
---
# 1.硬件准备
3A5000测试机，Radeon 520OEM显卡,串口用于连接3A5000测试机以及工作机（串口软件minicom或者picocom），HDMI线连接显卡和显示器（VGA线暂时不支持）
```
loongson@loongson-pc:~$ lspci -v -s 06:00.0
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
# 2.硬盘分区及相关配置说明
```
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
这里为了方便调试我采用的是双系统方案，硬盘分区格式是gpt格式，其中分区1-4是loongnix系统，分区5-8分别是安卓的super,data,cache,metadata分区，分区9用来放置内核和ramdisk等一些启动程序。
需要注意的是，goldfish中的fstab.ranchu的分区需要与上述分区一致,由下可见
```
# Android fstab file.
#<dev>  <mnt_point> <type>  <mnt_flags options> <fs_mgr_flags>
system   /system     ext4    ro,barrier=1     wait,logical,first_stage_mount
vendor   /vendor     ext4    ro,barrier=1     wait,logical,first_stage_mount
/dev/block/sda6 /data     ext4      noatime,nosuid,nodev,nomblk_io_submit,errors=panic   wait,check,quota,fileencryption=aes-256-xts:aes-256-cts,reservedsize=128M,latemount
/dev/block/sda7 /cache ext4      noatime,nosuid,nodev,nomblk_io_submit,errors=panic   wait,check,quota,fileencryption=aes-256-xts:aes-256-cts,reservedsize=128M,latemount
/dev/block/sda8 /metadata    ext4    noatime,nosuid,nodev    wait,formattable,latemount
/dev/block/vdb auto   auto      defaults    voldmanaged=sdcard:auto,encryptable=userdata
/dev/block/zram0 none swap  defaults zramsize=75%
```
另外在goldfish的init.ranchu-core.sh中可以设置测试机的ip地址
```
/system/bin/ifconfig eth0 up  
/system/bin/ifconfig eth0 192.168.2.2  
 sleep 10  
/system/bin/ip rule add from all lookup main pref 1                                                     
 ```
# 3.烧录

编译完成后可以使用测试机中的loongnix系统下载源码并烧录
这里提供一下烧录脚本，用于从源码所在的服务器或者工作机上把镜像下载到本地并烧录到对应分区
```
#!/bin/bash
server=172.17.103.55 
password=****
username=dongzhe
remote_dir=/home/dongzhe/loongson/aosp.la/out/target/product/generic_loongarch64
image_files=(
        "cache.img"
        "userdata.img"
#        "ramdisk.img"
        "super.img"
#       "vmlinuz.efi"
        )
for file in "${image_files[@]}"; do
        sshpass -p "${password}" scp  "${username}@${server}:${remote_dir}/${file}" ./
        if [ $? -eq 0 ];then
                echo "copy successed: ${file}"
        else
                echo "copy failed: ${file}"
        fi
done

dd if=super.img of=/dev/sda5 bs=32M
dd if=userdata.img of=/dev/sda6 bs=32M
dd if=cache.img of=/dev/sda7 bs=32M

```

# 4.启动参数
```
menuentry "android"{
        linux (hd0,gpt9)/vmlinuz.efi console=ttyS0,115200 norandmaps earlycon enforcing=0 init=/init rw rootfstype=cpio androidboot.selinux=permissive androidboot.hardware=ranchu loglevel=1 androidboot.boot_devices=pci0000:00/0000:00:08.0 printk.devkmsg=on androidboot.verifiedbootstate=orange
        boot
}
```
boot_devices需与sda所在的pci地址保持一致,采用双系统方案的话可以在装好的loongnix系统中查看硬盘pci地址。
```
loongson@loongson-pc:/sys/block$ realpath sda
/sys/devices/pci0000:00/0000:00:08.0/ata1/host0/target0:0:0/0:0:0:0/block/sda
```

内核可使用android_qemu_env下的vmlinuz.efi