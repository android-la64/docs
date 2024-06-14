# GDB调试surfaceflinger的几个小技巧

前一阵在qemu和真机上用gdbserver64和x86主机的cross gdb调试surfaceflinger，遇到几个困难：

1. 不加载符号表。这个问题主要是没设置好路径，gdb不知道在哪里可以找到带调试信息的相应二进制（可执行文件和库），只需要设置适当的solib-search-path就可以了
2. 不加载用dlopen后续打开的动态库。解决了1的问题之后，gdb仍然无法自动加载程序运行过程中动态加载的新动态库，导致那些库无法设置断点或者进行源代码级调试。而surfaceflinger和mesa/hwcomposer等有密切交互，会动态加载不少库。可以在相应的库函数被加载后运行sharedLibrary命令迫使gdb加载相应的符合。不带参数的shareLibrary会试图去找目前已经加载的所有动态库，找到相应文件就会加载其符号表，加载之后就可以对其进行源码级调试了。而定位这种动态库加载的时机，可以通过在dlopen, android_dlopen_ext等函数设置断点来截获。
3. 在模拟器上（真机可能也会，没验证），调试surfaceflinger经常会碰到进程被SIG35中断，导致程序退出无法继续调试。估计是因为某些地方有超时设置引起的，可以通过设置gdb忽略该信号。

解决这些问题之后，就可以对surfaceflinger整个过程进行完整的源码级调试了。

## 一个参考的.gdbinit文件内容

set auto-solib-add on
handle SIG35 noprint nostop nopass
set solib-search-path ./out/target/product/generic_loongarch64/symbols/system/lib64/:./out/target/product/generic_loongarch64/symbols/vendor/lib64/egl/:./out/target/product/generic_loongarch64/symbols/vendor/lib64/:./out/target/product/generic_loongarch64/symbols/vendor/lib64/dri/:./out/target/product/generic_loongarch64/symbols/system/lib64/egl:./out/target/product/generic_loongarch64/symbols/apex/com.android.runtime/lib64/bionic/:./out/target/product/generic_loongarch64/symbols/apex/com.android.runtime/lib64/:./out/target/product/generic_loongarch64/symbols/vendor/lib64/hw
b main
#target remote 127.0.0.1:1234
target remote 192.168.2.2:1234 # 地址根据实际情况修改
c
#sharedlibrary
#b eglInitialize
#b eglGetDisplayImpl
#c
#sharedlibrary
#sharedlibrary /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/egl/libdrm.so
#b dri2_initialize_android
#b eglCreateImageKHR
#c
#b 1150
#sharedlibrary

## 一个参考的调试过程

服务器端运行gdbserver64 :1234 /system/bin/surfaceflinger

客户端运行cross gdb(官方提供的8.0rc2), 当前目录设置以上的.gdbinit
$> /opt/loongson-gnu-toolchain-8.3-x86_64-loongarch64-linux-gnu-rc1.2/bin/loongarch64-linux-gnu-gdb ./out/target/product/generic_loongarch64/symbols/system/bin/surfaceflinger

Type "apropos word" to search for commands related to "word"...
Reading symbols from ./out/target/product/generic_loongarch64/symbols/system/bin/surfaceflinger...done.
Breakpoint 1 at 0x221a48: file frameworks/native/services/surfaceflinger/main_surfaceflinger.cpp, line 81.
warning: Unable to find dynamic linker breakpoint function.
GDB will be unable to debug shared library initializers
and track explicitly loaded dynamic code.
0x00007fffc24aa4c0 in ?? ()
warning: Could not load shared library symbols for /system/bin/linker64.
Do you need "set solib-search-path" or "set sysroot"?
b
Breakpoint 1, main () at frameworks/native/services/surfaceflinger/main_surfaceflinger.cpp:81
81	    signal(SIGPIPE, SIG_IGN);
(gdb) b frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp
Function "frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp" not defined.
Make breakpoint pending on future shared library load? (y or [n]) n
(gdb) b frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp:162
Breakpoint 2 at 0x55555e5978bc: file frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp, line 162.
(gdb) c
Continuing.
[New Thread 4581.4623]
[New Thread 4581.4624]
[New Thread 4581.4625]

Thread 1 "surfaceflinger" hit Breakpoint 2, android::impl::HWComposer::setCallback (this=<optimized out>, callback=0x7fff2568d040)
    at frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp:162
162	    mComposer->registerCallback(
(gdb) b dlopen
dlopen                    dlopen(char const*, int)  dlopen@plt
(gdb) b dlopen
Breakpoint 3 at 0x7fffb1f57cb0: file bionic/libdl/libdl.cpp, line 84.
(gdb) b android
Display all 200 possibilities? (y or n)
(gdb) b android_dl
android_dlopen_ext                                              android_dlwarning
android_dlopen_ext(char const*, int, android_dlextinfo const*)  android_dlwarning(void*, void (*)(void*, char const*))
android_dlopen_ext@plt
(gdb) b android_dl
android_dlopen_ext                                              android_dlwarning
android_dlopen_ext(char const*, int, android_dlextinfo const*)  android_dlwarning(void*, void (*)(void*, char const*))
android_dlopen_ext@plt
(gdb) b android_dlopen_ext
Breakpoint 4 at 0x7fffb1f57d8c: file bionic/libdl/libdl.cpp, line 133.
(gdb) sharedlibrary
warning: .dynamic section for "/home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/libcutils.so" is not at the expected address (wrong library or version mismatch?)
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/vendor/lib64/egl/libGLES_mesa.so...done.
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/libdrm.so...done.
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/vendor/lib64/libglapi.so...done.
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/libcutils.so...done.
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/libbase.so...done.
(gdb) c
Continuing.
[New Thread 4581.4673]
[New Thread 4581.4674]
[New Thread 4581.4675]
[New Thread 4581.4676]
[New Thread 4581.4677]
[New Thread 4581.4678]

Thread 1 "surfaceflinger" hit Breakpoint 4, android_dlopen_ext (filename=0x7ffde5684170 "/vendor/lib64/hw/android.hardware.graphics.mapper@4.0-impl.minigbm.so", flag=1,
    extinfo=0x7ffffbe86e98) at bionic/libdl/libdl.cpp:133
133	  return __loader_android_dlopen_ext(filename, flag, extinfo, caller_addr);
(gdb) where
#0  android_dlopen_ext (filename=0x7ffde5684170 "/vendor/lib64/hw/android.hardware.graphics.mapper@4.0-impl.minigbm.so", flag=1, extinfo=0x7ffffbe86e98)
    at bionic/libdl/libdl.cpp:133
#1  0x00007fffbba18b48 in android_load_sphal_library (name=0x7ffde5684170 "/vendor/lib64/hw/android.hardware.graphics.mapper@4.0-impl.minigbm.so", flag=<optimized out>)
    at system/core/libvndksupport/linker.cpp:71
#2  0x00007fffb3ee73e8 in android::hardware::PassthroughServiceManager::openLibs(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::function<bool (void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)> const&) (fqName=..., eachLib=...) at system/libhidl/transport/ServiceManagement.cpp:392
#3  0x00007fffb3eeb544 in android::hardware::PassthroughServiceManager::get (this=<optimized out>, fqName=..., name=...)
    at system/libhidl/transport/ServiceManagement.cpp:414
#4  0x00007fffb3ee8ad8 in android::hardware::details::getRawServiceInternal (descriptor=..., instance=..., retry=<optimized out>, getStub=false)
    at system/libhidl/transport/ServiceManagement.cpp:825
#5  0x00007fffb73751b0 in android::hardware::details::getServiceInternal<android::hardware::graphics::mapper::V4_0::BpHwMapper, android::hardware::graphics::mapper::V4_0::IMapper, void, void> (instance=..., retry=true, getStub=false) at system/libhidl/transport/include/hidl/HidlTransportSupport.h:165
#6  0x00007fffb73753d4 in android::hardware::graphics::mapper::V4_0::IMapper::getService (serviceName=..., getStub=true)
    at out/soong/.intermediates/hardware/interfaces/graphics/mapper/4.0/android.hardware.graphics.mapper@4.0_genc++/gen/android/hardware/graphics/mapper/4.0/MapperAll.cpp:3631
#7  0x00007fffb75b5d90 in android::Gralloc4Mapper::Gralloc4Mapper (this=0x7ffda568b9b0) at frameworks/native/libs/ui/Gralloc4.cpp:91
#8  0x00007fffb75c3ec8 in std::__1::make_unique<android::Gralloc4Mapper const> () at external/libcxx/include/memory:3132
#9  android::GraphicBufferMapper::GraphicBufferMapper (this=0x7ffda5689930) at frameworks/native/libs/ui/GraphicBufferMapper.cpp:55
#10 0x00007fffb75c00b4 in android::Singleton<android::GraphicBufferMapper>::getInstance () at system/core/libutils/include/utils/Singleton.h:55
#11 android::GraphicBufferMapper::get () at frameworks/native/libs/ui/include/ui/GraphicBufferMapper.h:50
#12 android::GraphicBuffer::GraphicBuffer (this=0x7ffe45690130) at frameworks/native/libs/ui/GraphicBuffer.cpp:64
#13 0x00007fffb75c02d0 in android::GraphicBuffer::GraphicBuffer (this=0x7ffe45690130, inWidth=3840, inHeight=2160, inFormat=1, inLayerCount=1, inUsage=6912,
    requestorName=...) at frameworks/native/libs/ui/GraphicBuffer.cpp:86
#14 0x00007fffb1cdab04 in android::BufferQueueProducer::allocateBuffers (this=0x7ffe5568cdd0, width=<optimized out>, height=<optimized out>, format=<optimized out>,
    usage=768) at frameworks/native/libs/gui/BufferQueueProducer.cpp:1443
#15 0x00007fffb1d42d10 in android::Surface::allocateBuffers (this=<optimized out>) at frameworks/native/libs/gui/Surface.cpp:143
#16 0x000055555e5ccc14 in android::surfaceflinger::impl::createNativeWindowSurface(android::sp<android::IGraphicBufferProducer> const&)::NativeWindowSurface::preallocateBuffers() (this=<optimized out>) at frameworks/native/services/surfaceflinger/NativeWindowSurface.cpp:39
#17 0x000055555e60ff60 in android::SurfaceFlinger::setupNewDisplayDeviceInternal (this=0x7fff2568d010, displayToken=..., compositionDisplay=..., state=...,
    displaySurface=..., producer=...) at frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:2706
#18 0x000055555e610834 in android::SurfaceFlinger::processDisplayAdded (this=0x7fff2568d010, displayToken=..., state=...)
    at frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:2798
--Type <RET> for more, q to quit, c to continue without paging--
#19 0x000055555e6056cc in android::SurfaceFlinger::processDisplayChangesLocked (this=0x7fff2568d010) at frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:2948
#20 0x000055555e60312c in android::SurfaceFlinger::processDisplayHotplugEventsLocked (this=0x7fff2568d010)
    at frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:2634
#21 0x000055555e608ce0 in android::SurfaceFlinger::onComposerHalHotplug (this=0x7fff2568d010, hwcDisplayId=0, connection=<optimized out>)
    at frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:1788
#22 0x000055555e608d60 in non-virtual thunk to android::SurfaceFlinger::onComposerHalHotplug(unsigned long, android::hardware::graphics::composer::V2_1::IComposerCallback::Connection) () at frameworks/native/services/surfaceflinger/DisplayIdGenerator.h:55
#23 0x000055555e5a6d48 in android::(anonymous namespace)::ComposerCallbackBridge::onHotplug (this=<optimized out>, display=1, connection=(unknown: -68653416))
    at frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp:89
#24 0x00007fffa9f08118 in android::hardware::graphics::composer::V2_1::BnHwComposerCallback::_hidl_onHotplug(android::hidl::base::V1_0::BnHwBase*, android::hardware::Parcel const&, android::hardware::Parcel*, std::__1::function<void (android::hardware::Parcel&)>) (_hidl_this=0x7ffe1568c2a0, _hidl_data=..., _hidl_reply=0x7ffffbe88090,
    _hidl_cb=...)
    at out/soong/.intermediates/hardware/interfaces/graphics/composer/2.1/android.hardware.graphics.composer@2.1_genc++/gen/android/hardware/graphics/composer/2.1/ComposerCallbackAll.cpp:448
#25 0x00007fffb7480878 in android::hardware::graphics::composer::V2_4::BnHwComposerCallback::onTransact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>) (this=0x7ffe1568c2a0, _hidl_code=<optimized out>, _hidl_data=..., _hidl_reply=0x7ffffbe88090,
    _hidl_flags=<optimized out>, _hidl_cb=...)
    at out/soong/.intermediates/hardware/interfaces/graphics/composer/2.4/android.hardware.graphics.composer@2.4_genc++/gen/android/hardware/graphics/composer/2.4/ComposerCallbackAll.cpp:653
#26 0x00007fffb3f3b830 in android::hardware::BHwBinder::transact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>) (this=0x7ffe1568c2a0, code=<optimized out>, data=..., reply=0x7ffffbe88090, flags=<optimized out>, callback=...)
    at system/libhwbinder/Binder.cpp:135
#27 0x00007fffb3f41a6c in android::hardware::IPCThreadState::executeCommand (this=0x7ffe7568a950, cmd=<optimized out>) at system/libhwbinder/IPCThreadState.cpp:1202
#28 0x00007fffb3f42bc4 in android::hardware::IPCThreadState::waitForResponse (this=0x7ffe7568a950, reply=0x7ffffbe88490, acquireResult=0x0)
    at system/libhwbinder/IPCThreadState.cpp:868
#29 0x00007fffb3f4262c in android::hardware::IPCThreadState::transact (this=0x7ffe7568a950, handle=<optimized out>, code=1, data=..., reply=0x7ffffbe88490,
    flags=<optimized out>) at system/libhwbinder/IPCThreadState.cpp:652
#30 0x00007fffb3f3cbd4 in android::hardware::BpHwBinder::transact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>) (this=0x7ffe1568b7a0, code=<optimized out>, data=..., reply=0x7ffffbe88490, flags=<optimized out>, callback=...)
    at system/libhwbinder/BpHwBinder.cpp:107
#31 0x00007fffa9f0efbc in android::hardware::graphics::composer::V2_1::BpHwComposerClient::_hidl_registerCallback (_hidl_this=<optimized out>, _hidl_this_instrumentor=
    0x7ffe45690940, callback=...)
    at out/soong/.intermediates/hardware/interfaces/graphics/composer/2.1/android.hardware.graphics.composer@2.1_genc++/gen/android/hardware/graphics/composer/2.1/ComposerCl--Type <RET> for more, q to quit, c to continue without paging--
ientAll.cpp:194
#32 0x00007fffa9f150b0 in android::hardware::graphics::composer::V2_1::BpHwComposerClient::registerCallback (this=<optimized out>, callback=...)
    at out/soong/.intermediates/hardware/interfaces/graphics/composer/2.1/android.hardware.graphics.composer@2.1_genc++/gen/android/hardware/graphics/composer/2.1/ComposerClientAll.cpp:1949
#33 0x000055555e5806b4 in android::Hwc2::impl::Composer::registerCallback(android::sp<android::hardware::graphics::composer::V2_4::IComposerCallback> const&)::$_34::operator()() const (this=<optimized out>) at frameworks/native/services/surfaceflinger/DisplayHardware/ComposerHal.cpp:193
#34 android::Hwc2::impl::Composer::registerCallback (this=<optimized out>, callback=...) at frameworks/native/services/surfaceflinger/DisplayHardware/ComposerHal.cpp:189
#35 0x000055555e5978cc in android::impl::HWComposer::setCallback (this=<optimized out>, callback=0x7fff2568d040)
    at frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp:162
#36 0x000055555e60206c in android::SurfaceFlinger::init (this=0x7fff2568d010) at frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:803
#37 0x000055555e651d00 in main () at frameworks/native/services/surfaceflinger/main_surfaceflinger.cpp:149
(gdb) l
128	}
129
130	__attribute__((__weak__))
131	void* android_dlopen_ext(const char* filename, int flag, const android_dlextinfo* extinfo) {
132	  const void* caller_addr = __builtin_return_address(0);
133	  return __loader_android_dlopen_ext(filename, flag, extinfo, caller_addr);
134	}
135
136	__attribute__((__weak__))
137	int android_get_application_target_sdk_version() {
(gdb) finish
Run till exit from #0  android_dlopen_ext (filename=0x7ffde5684170 "/vendor/lib64/hw/android.hardware.graphics.mapper@4.0-impl.minigbm.so", flag=1, extinfo=0x7ffffbe86e98)
    at bionic/libdl/libdl.cpp:133
0x00007fffbba18b48 in android_load_sphal_library (name=0x7ffde5684170 "/vendor/lib64/hw/android.hardware.graphics.mapper@4.0-impl.minigbm.so", flag=<optimized out>)
    at system/core/libvndksupport/linker.cpp:71
71	        void* handle = android_dlopen_ext(name, flag, &dlextinfo);
Value returned is $1 = (void *) 0xea6bcd040c5e3979
(gdb) sharedlibrary
warning: .dynamic section for "/home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/libhardware.so" is not at the expected address (wrong library or version mismatch?)
warning: .dynamic section for "/home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/libhidlbase.so" is not at the expected address (wrong library or version mismatch?)
warning: .dynamic section for "/home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/libprocessgroup.so" is not at the expected address (wrong library or version mismatch?)
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/libc++.so...done.
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/libhardware.so...done.
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/android.hardware.graphics.mapper@4.0.so...done.
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/android.hardware.graphics.common@1.0.so...done.
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/android.hardware.graphics.common@1.1.so...done.
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/android.hardware.graphics.common@1.2.so...done.
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/libhidlbase.so...done.
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/libutils.so...done.
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/libprocessgroup.so...done.
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/libgralloctypes.so...done.
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/android.hardware.graphics.common-V2-ndk_platform.so...done.
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/system/lib64/android.hardware.common-V2-ndk_platform.so...done.
Reading symbols from /home/foxsen/android-la64/loongson/aosp.la/out/target/product/generic_loongarch64/symbols/vendor/lib64/hw/android.hardware.graphics.mapper@4.0-impl.minigbm.so...done.
(gdb) b GetAndReferenceDriver
Breakpoint 5 at 0x7ffd0ac6c43c: file external/minigbm/cros_gralloc/gralloc4/CrosGralloc4Mapper.cc, line 47.
(gdb) c
Continuing.

Thread 1 "surfaceflinger" hit Breakpoint 5, (anonymous namespace)::DriverProvider::GetAndReferenceDriver (this=0x7ffdd568f580)
    at external/minigbm/cros_gralloc/gralloc4/CrosGralloc4Mapper.cc:47
47	        std::lock_guard<std::mutex> lock(mMutex);
(gdb) where
#0  (anonymous namespace)::DriverProvider::GetAndReferenceDriver (this=0x7ffdd568f580) at external/minigbm/cros_gralloc/gralloc4/CrosGralloc4Mapper.cc:47
#1  0x00007ffd0ac71354 in CrosGralloc4Mapper::CrosGralloc4Mapper (this=0x7ffdb5690470) at external/minigbm/cros_gralloc/gralloc4/CrosGralloc4Mapper.cc:82
#2  HIDL_FETCH_IMapper () at external/minigbm/cros_gralloc/gralloc4/CrosGralloc4Mapper.cc:1056
#3  0x00007fffb3eed724 in android::hardware::PassthroughServiceManager::get(android::hardware::hidl_string const&, android::hardware::hidl_string const&)::{lambda(void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)#1}::operator()(void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&) const (this=0x7ffffbe870f8, handle=<optimized out>, lib=..., sym=...) at system/libhidl/transport/ServiceManagement.cpp:429
#4  0x00007fffb3eed684 in std::__1::__invoke<android::hardware::PassthroughServiceManager::get(android::hardware::hidl_string const&, android::hardware::hidl_string const&)::{lambda(void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)#1}&, void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&> (__f=..., __args=..., __args=..., __args=...) at external/libcxx/include/type_traits:4353
#5  std::__1::__invoke_void_return_wrapper<bool>::__call<android::hardware::PassthroughServiceManager::get(android::hardware::hidl_string const&, android::hardware::hidl_string const&)::{lambda(void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)#1}&, void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&>(android::hardware::PassthroughServiceManager::get(android::hardware::hidl_string const&, android::hardware::hidl_string const&)::{lambda(void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)#1}&, void*&&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&) (__args=..., __args=..., __args=..., __args=...)
    at external/libcxx/include/__functional_base:318
#6  std::__1::__function::__alloc_func<android::hardware::PassthroughServiceManager::get(android::hardware::hidl_string const&, android::hardware::hidl_string const&)::{lambda(void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)#1}, std::__1::allocator<{lambda(void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)#1}>, bool (void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)>::operator()(void*&&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&) (this=0x7ffdd568f580,
    __arg=..., __arg=..., __arg=...) at external/libcxx/include/functional:1527
#7  std::__1::__function::__func<android::hardware::PassthroughServiceManager::get(android::hardware::hidl_string const&, android::hardware::hidl_string const&)::{lambda(void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)#1}, std::__1::allocator<{lambda(void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)#1}>, bool (void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)>::operator()(void*&&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&) (this=<optimized out>,
    __arg=..., __arg=..., __arg=...) at external/libcxx/include/functional:1651
--Type <RET> for more, q to quit, c to continue without paging--
#8  0x00007fffb3ee7414 in std::__1::__function::__value_func<bool (void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)>::operator()(void*&&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&) const (this=0x7ffffbe870f0, __args=...,
    __args=..., __args=...) at external/libcxx/include/functional:1799
#9  std::__1::function<bool (void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)>::operator()(void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&) const (this=0x7ffffbe870f0, __arg=..., __arg=..., __arg=...)
    at external/libcxx/include/functional:2347
#10 android::hardware::PassthroughServiceManager::openLibs(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::function<bool (void*, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&)> const&) (fqName=..., eachLib=...) at system/libhidl/transport/ServiceManagement.cpp:403
#11 0x00007fffb3eeb544 in android::hardware::PassthroughServiceManager::get (this=<optimized out>, fqName=..., name=...)
    at system/libhidl/transport/ServiceManagement.cpp:414
#12 0x00007fffb3ee8ad8 in android::hardware::details::getRawServiceInternal (descriptor=..., instance=..., retry=<optimized out>, getStub=false)
    at system/libhidl/transport/ServiceManagement.cpp:825
#13 0x00007fffb73751b0 in android::hardware::details::getServiceInternal<android::hardware::graphics::mapper::V4_0::BpHwMapper, android::hardware::graphics::mapper::V4_0::IMapper, void, void> (instance=..., retry=true, getStub=false) at system/libhidl/transport/include/hidl/HidlTransportSupport.h:165
#14 0x00007fffb73753d4 in android::hardware::graphics::mapper::V4_0::IMapper::getService (serviceName=..., getStub=false)
    at out/soong/.intermediates/hardware/interfaces/graphics/mapper/4.0/android.hardware.graphics.mapper@4.0_genc++/gen/android/hardware/graphics/mapper/4.0/MapperAll.cpp:3631
#15 0x00007fffb75b5d90 in android::Gralloc4Mapper::Gralloc4Mapper (this=0x7ffda568b9b0) at frameworks/native/libs/ui/Gralloc4.cpp:91
#16 0x00007fffb75c3ec8 in std::__1::make_unique<android::Gralloc4Mapper const> () at external/libcxx/include/memory:3132
#17 android::GraphicBufferMapper::GraphicBufferMapper (this=0x7ffda5689930) at frameworks/native/libs/ui/GraphicBufferMapper.cpp:55
#18 0x00007fffb75c00b4 in android::Singleton<android::GraphicBufferMapper>::getInstance () at system/core/libutils/include/utils/Singleton.h:55
#19 android::GraphicBufferMapper::get () at frameworks/native/libs/ui/include/ui/GraphicBufferMapper.h:50
#20 android::GraphicBuffer::GraphicBuffer (this=0x7ffe45690130) at frameworks/native/libs/ui/GraphicBuffer.cpp:64
#21 0x00007fffb75c02d0 in android::GraphicBuffer::GraphicBuffer (this=0x7ffe45690130, inWidth=3840, inHeight=2160, inFormat=1, inLayerCount=1, inUsage=6912,
    requestorName=...) at frameworks/native/libs/ui/GraphicBuffer.cpp:86
#22 0x00007fffb1cdab04 in android::BufferQueueProducer::allocateBuffers (this=0x7ffe5568cdd0, width=<optimized out>, height=<optimized out>, format=<optimized out>,
    usage=768) at frameworks/native/libs/gui/BufferQueueProducer.cpp:1443
#23 0x00007fffb1d42d10 in android::Surface::allocateBuffers (this=<optimized out>) at frameworks/native/libs/gui/Surface.cpp:143
#24 0x000055555e5ccc14 in android::surfaceflinger::impl::createNativeWindowSurface(android::sp<android::IGraphicBufferProducer> const&)::NativeWindowSurface::preallocateBuffers() (this=<optimized out>) at frameworks/native/services/surfaceflinger/NativeWindowSurface.cpp:39
--Type <RET> for more, q to quit, c to continue without paging--
#25 0x000055555e60ff60 in android::SurfaceFlinger::setupNewDisplayDeviceInternal (this=0x7fff2568d010, displayToken=..., compositionDisplay=..., state=...,
    displaySurface=..., producer=...) at frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:2706
#26 0x000055555e610834 in android::SurfaceFlinger::processDisplayAdded (this=0x7fff2568d010, displayToken=..., state=...)
    at frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:2798
#27 0x000055555e6056cc in android::SurfaceFlinger::processDisplayChangesLocked (this=0x7fff2568d010) at frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:2948
#28 0x000055555e60312c in android::SurfaceFlinger::processDisplayHotplugEventsLocked (this=0x7fff2568d010)
    at frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:2634
#29 0x000055555e608ce0 in android::SurfaceFlinger::onComposerHalHotplug (this=0x7fff2568d010, hwcDisplayId=0, connection=<optimized out>)
    at frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:1788
#30 0x000055555e608d60 in non-virtual thunk to android::SurfaceFlinger::onComposerHalHotplug(unsigned long, android::hardware::graphics::composer::V2_1::IComposerCallback::Connection) () at frameworks/native/services/surfaceflinger/DisplayIdGenerator.h:55
#31 0x000055555e5a6d48 in android::(anonymous namespace)::ComposerCallbackBridge::onHotplug (this=<optimized out>, display=1,
    connection=android::hardware::graphics::composer::V2_1::IComposerCallback::Connection::DISCONNECTED)
    at frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp:89
#32 0x00007fffa9f08118 in android::hardware::graphics::composer::V2_1::BnHwComposerCallback::_hidl_onHotplug(android::hidl::base::V1_0::BnHwBase*, android::hardware::Parcel const&, android::hardware::Parcel*, std::__1::function<void (android::hardware::Parcel&)>) (_hidl_this=0x7ffe1568c2a0, _hidl_data=..., _hidl_reply=0x7ffffbe88090,
    _hidl_cb=...)
    at out/soong/.intermediates/hardware/interfaces/graphics/composer/2.1/android.hardware.graphics.composer@2.1_genc++/gen/android/hardware/graphics/composer/2.1/ComposerCallbackAll.cpp:448
#33 0x00007fffb7480878 in android::hardware::graphics::composer::V2_4::BnHwComposerCallback::onTransact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>) (this=0x7ffe1568c2a0, _hidl_code=<optimized out>, _hidl_data=..., _hidl_reply=0x7ffffbe88090,
    _hidl_flags=<optimized out>, _hidl_cb=...)
    at out/soong/.intermediates/hardware/interfaces/graphics/composer/2.4/android.hardware.graphics.composer@2.4_genc++/gen/android/hardware/graphics/composer/2.4/ComposerCallbackAll.cpp:653
#34 0x00007fffb3f3b830 in android::hardware::BHwBinder::transact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::function<void (android::hardware::Parcel&)>) (this=0x7ffe1568c2a0, code=<optimized out>, data=..., reply=0x7ffffbe88090, flags=<optimized out>, callback=...)
    at system/libhwbinder/Binder.cpp:135
#35 0x00007fffb3f41a6c in android::hardware::IPCThreadState::executeCommand (this=0x7ffe7568a950, cmd=<optimized out>) at system/libhwbinder/IPCThreadState.cpp:1202
#36 0x00007fffb3f42bc4 in android::hardware::IPCThreadState::waitForResponse (this=0x7ffe7568a950, reply=0x7ffffbe88490, acquireResult=0x0)
    at system/libhwbinder/IPCThreadState.cpp:868
#37 0x00007fffb3f4262c in android::hardware::IPCThreadState::transact (this=0x7ffe7568a950, handle=<optimized out>, code=1, data=..., reply=0x7ffffbe88490,
    flags=<optimized out>) at system/libhwbinder/IPCThreadState.cpp:652
#38 0x00007fffb3f3cbd4 in android::hardware::BpHwBinder::transact(unsigned int, android::hardware::Parcel const&, android::hardware::Parcel*, unsigned int, std::__1::functio--Type <RET> for more, q to quit, c to continue without paging--
n<void (android::hardware::Parcel&)>) (this=0x7ffe1568b7a0, code=<optimized out>, data=..., reply=0x7ffffbe88490, flags=<optimized out>, callback=...)
    at system/libhwbinder/BpHwBinder.cpp:107
#39 0x00007fffa9f0efbc in android::hardware::graphics::composer::V2_1::BpHwComposerClient::_hidl_registerCallback (_hidl_this=<optimized out>,
    _hidl_this_instrumentor=0x7ffe45690940, callback=...)
    at out/soong/.intermediates/hardware/interfaces/graphics/composer/2.1/android.hardware.graphics.composer@2.1_genc++/gen/android/hardware/graphics/composer/2.1/ComposerClientAll.cpp:194
#40 0x00007fffa9f150b0 in android::hardware::graphics::composer::V2_1::BpHwComposerClient::registerCallback (this=<optimized out>, callback=...)
    at out/soong/.intermediates/hardware/interfaces/graphics/composer/2.1/android.hardware.graphics.composer@2.1_genc++/gen/android/hardware/graphics/composer/2.1/ComposerClientAll.cpp:1949
#41 0x000055555e5806b4 in android::Hwc2::impl::Composer::registerCallback(android::sp<android::hardware::graphics::composer::V2_4::IComposerCallback> const&)::$_34::operator()() const (this=<optimized out>) at frameworks/native/services/surfaceflinger/DisplayHardware/ComposerHal.cpp:193
#42 android::Hwc2::impl::Composer::registerCallback (this=<optimized out>, callback=...) at frameworks/native/services/surfaceflinger/DisplayHardware/ComposerHal.cpp:189
#43 0x000055555e5978cc in android::impl::HWComposer::setCallback (this=<optimized out>, callback=0x7fff2568d040)
    at frameworks/native/services/surfaceflinger/DisplayHardware/HWComposer.cpp:162
#44 0x000055555e60206c in android::SurfaceFlinger::init (this=0x7fff2568d010) at frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp:803
#45 0x000055555e651d00 in main () at frameworks/native/services/surfaceflinger/main_surfaceflinger.cpp:149
(gdb) l
42	        static DriverProvider* instance = new DriverProvider();
43	        return instance;
44	    }
45
46	    cros_gralloc_driver* GetAndReferenceDriver() {
47	        std::lock_guard<std::mutex> lock(mMutex);
48	        if (!mDriver) {
49	            mDriver = std::make_unique<cros_gralloc_driver>();
50	            if (mDriver->init()) {
51	                drv_log("Failed to initialize driver.\n");
(gdb) n
48	        if (!mDriver) {
(gdb) n
49	            mDriver = std::make_unique<cros_gralloc_driver>();
(gdb) n
50	            if (mDriver->init()) {
(gdb) s
cros_gralloc_driver::init (this=0x7ffe05687110) at external/minigbm/cros_gralloc/cros_gralloc_driver.cc:106
106			drv_ = init_try_node(i, render_nodes_fmt);
(gdb) s
105		for (uint32_t i = min_render_node; i < max_render_node; i++) {
(gdb) p i
$2 = 128
(gdb) s
106			drv_ = init_try_node(i, render_nodes_fmt);
(gdb) s
init_try_node (idx=129, str=<optimized out>) at external/minigbm/cros_gralloc/cros_gralloc_driver.cc:68
68		char *node;
(gdb) s
71		if (asprintf(&node, str, DRM_DIR_NAME, idx) < 0)
(gdb) p i
No symbol "i" in current context.
(gdb) n
74		fd = open(node, O_RDWR, 0);
(gdb) p node
$3 = 0x7ffe05686540 "/dev/dri/renderD129"
(gdb) q
A debugging session is active.

	Inferior 1 [process 4581] will be killed.

Quit anyway? (y or n) y

