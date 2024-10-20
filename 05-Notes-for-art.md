

# 1. 注意事项

1. 安装应用：大部分带有so文件的APK安装文件不能安装在本系统中（so文件往往会带有架构相关的代码）

2. 如果采用触摸屏，需要在编译kernel的时候enable部分选项。

   

3. 提交的AOSP中，采用了手势导航方式:

   ```diff
   diff --git a/core/res/res/values/config.xml b/core/res/res/values/config.xml
   index db43b5b3..d120d4a6 100644
   --- a/core/res/res/values/config.xml
   +++ b/core/res/res/values/config.xml
   @@ -3434,7 +3434,7 @@
             0: 3 button mode (back, home, overview buttons)
             1: 2 button mode (back, home buttons + swipe up for overview)
             2: gestures only for back, home and overview -->
   -    <integer name="config_navBarInteractionMode">0</integer>
   +    <integer name="config_navBarInteractionMode">2</integer>
   ```

4. 缺省的显示是中文方式

   ```bash
   # common/common_la.mk
   PRODUCT_LOCALES := zh_CN
   ```

5. 系统时会有Keystore错误，这个错误不影响后续使用，只是在启动时会存在，原因是没有生成对应的密码Provider（缺省的是TEE提供）





# 2. ART的特别注意事项

当前版本的ART虚拟机还是以解释方式运行的。需要进一步移植为AOT、JIT方式。

在移植JIT、AOT方式时需要注意如下事项：

1. 步骤 - 正常情况下是优先移植JIT、然后再AOT。

2. 测试 - 缺省情况下，ART提供了5中测试模式

   ```bash
     ## 以下五种模式是缺省的，不指定模式的时候测试的五种缺省模式
      --interpreter
      --jit   【测试的时候还可以强制使用： --jit-on-first-use 选项，保证必须采用JIT进行测试】
      --optimizing
      --speed-profile
      --interp-ac
   ```

3. 因为ART本身不支持纯Interpreter执行（缺省情况下，AOT是一定支持的），因此，当前代码中做了一些HACK，当JIT Enable后，必须将其修订回来

   ```bash
   ## ART git
   checkin: 874fab84f6954cad49d549176ba75966d145cda7
   checkin: cbc74adb95652af85a05ef7989afcdc54ed470a5
   
   ## device/loongson/loongsonboard/
   dalvik.vm.usejit = false ## 需要修改为true
   WITH_DEXPREOPT := false ## 需要修改为true
   
   ## build/make
   checkin: 64e29b4aa7
   
   ## frameworks/base
   checkin: 7da02639
   ```

   



## 注意!!!： 3A5000安卓内核

目前模拟器使用的安卓内核理论上可以直接用于3A5000真机，但是实际有两个问题需要处理：

2. 真机的radeon显卡驱动需要加载一些固件，由于android.img没有内置这些文件，需要处理。一种办法是内置到内核，配置内核(Device Drivers/Generic Driver Options/Firmware Loader/Firmware blobs root directory和firmware loading facility),参考配置如下（根据显卡型号不同内置文件也不同）。另一种办法是将linux /lib/firmware目录下相关文件加入ramdisk.img. 

```bash
CONFIG_EXTRA_FIRMWARE="radeon/oland_pfp.bin radeon/oland_me.bin radeon/oland_smc.bin radeon/oland_ce.bin radeon/oland_rlc.bin radeon/oland_mc.bin radeon/oland_k_smc.bin"
CONFIG_EXTRA_FIRMWARE_DIR="/lib/firmware"
```

