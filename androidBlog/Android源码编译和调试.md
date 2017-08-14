本文记录使用Mac编译Android 6.0源码，利用编译的rom启动模拟器，将源码导入Android Studio和配置工程，以及debug源码的过程。做个笔记，以备忘记之后查阅。使用的系统为10.12，Xcode版本为7.2，CPU为双核，内存为8G。

[请尊重博主劳动成果，转载请标明出处。](http://blog.csdn.net/hwliu51/article/details/75949060)

## 设置Xcode
Mac安装有两个版本的Xcode，默认为8.0，以及7.2版本。之前有使用7.2版本xcode编译Android 4.4.4和5.0的源码，均出现**`i686-apple-darwin11-g++-4.2.1: command not found`**错误。发现是该版本的xcode的C函数的库被删除了。换为Android 6.0的源码，继续编译，在修改几个小问题后，终于编译成功。为防止Xcode的版本过高，还是将默认的改为7.2版本。

查看现有的Xcode：
```
xcode-select --print-path
```

显示
> /Applications/Xcode.app/Contents/Developer

设置默认使用Xcode 7.2

```
sudo xcode-select -switch /Applications/Xcode7.2.app
```

输入密码，成功

使用`xcode-select --print-path`查看
显示结果
> /Applications/Xcode7.2.app/Contents/Developer

修改默认的Xcode为7.2成功。

## 编译源码
不修改默认的Xcode，即使用Xcode 8.0时，输入`make clobber`命令，会出现一下错误：

```
build/core/combo/mac_version.mk:38: *****************************************************
build/core/combo/mac_version.mk:39: * Can not find SDK 10.6 at /Developer/SDKs/MacOSX10.6.sdk
build/core/combo/mac_version.mk:40: *****************************************************
build/core/combo/mac_version.mk:41: *** Stop..  Stop
```
在编译前期会去检测系统的Xcode SDK的版本，并会判断当前Android源码中的相关内容是否支持该版本。当Xcode的版本比较高时，而Android源码中配置文件又没有包含该版本的版本号，就会报错并终止编译。这时需要去查看当前Xcode的版本。进入应用目录，选择Xcode，点击右键，然后选择“选择包内容”。具体目录路径为/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs。8.0的SDK版本为MacOSX10.12.sdk。7.2的为MacOSX10.11.sdk。需要在build/core/combo/mac_version.mk文件中添加10.12或10.11。因为设置7.2位默认，所以只需要添加10.11即可。

```
mac_sdk_versions_supported :=  10.6 10.7 10.8 10.9 10.11
```
在文件的这一行，添加10.11。


清理之前的编译文件，运行`make clobber`。
默认的JDK位1.8版本，因为版本太高会报以下错：

```
Checking build tools versions...
************************************************************
You are attempting to build with the incorrect version
of java.
 
Your version is: java version "1.8.0_05" Java(TM) SE Runtime Environment (build 1.8.0_05-b13) Java HotSpot(TM) 64-Bit Server VM (build 25.5-b02, mixed mode).
The required version is: "1.7.x"
 
Please follow the machine setup instructions at
    https://source.android.com/source/initializing.html
************************************************************
build/core/main.mk:171: *** stop.  Stop.
```

使用`java -version`查看，显示结果：
```
java version "1.8.0_05"
Java(TM) SE Runtime Environment (build 1.8.0_05-b13)
Java HotSpot(TM) 64-Bit Server VM (build 25.5-b02, mixed mode)
```

这是JDK的版本过高导致的问题，切换到1.7即可解决问题。
先执行
```
export JAVA_7_HOME=/Library/Java/JavaVirtualMachines/jdk1.7.0_80.jdk/Contents/Home
```
然后执行
```
export JAVA_HOME=$JAVA_7_HOME
```
便将当前的Java环境切换到JDK 1.7版本。

使用`java -version`查看，结果显示设置成功
```
java version "1.7.0_80"
Java(TM) SE Runtime Environment (build 1.7.0_80-b15)
Java HotSpot(TM) 64-Bit Server VM (build 24.80-b11, mixed mode)
```

重新运行`make clobber`，结果正常。
```
============================================
PLATFORM_VERSION_CODENAME=REL
PLATFORM_VERSION=6.0
TARGET_PRODUCT=full
TARGET_BUILD_VARIANT=eng
TARGET_BUILD_TYPE=release
TARGET_BUILD_APPS=
TARGET_ARCH=arm
TARGET_ARCH_VARIANT=armv7-a
TARGET_CPU_VARIANT=generic
TARGET_2ND_ARCH=
TARGET_2ND_ARCH_VARIANT=
TARGET_2ND_CPU_VARIANT=
HOST_ARCH=x86_64
HOST_OS=darwin
HOST_OS_EXTRA=Darwin-16.0.0-x86_64-i386-64bit
HOST_BUILD_TYPE=release
BUILD_ID=MRA58K
OUT_DIR=out
============================================
Checking build tools versions...
Entire build directory removed.
```

遍历mk文件，运行`source build/envsetup.sh`，结果正常
```
including device/asus/deb/vendorsetup.sh
including device/asus/flo/vendorsetup.sh
including device/asus/fugu/vendorsetup.sh
including device/generic/mini-emulator-arm64/vendorsetup.sh
including device/generic/mini-emulator-armv7-a-neon/vendorsetup.sh
including device/generic/mini-emulator-mips/vendorsetup.sh
including device/generic/mini-emulator-x86/vendorsetup.sh
including device/generic/mini-emulator-x86_64/vendorsetup.sh
including device/htc/flounder/vendorsetup.sh
including device/lge/hammerhead/vendorsetup.sh
including device/moto/shamu/vendorsetup.sh
including sdk/bash_completion/adb.bash
```

指定编译生成的文件类型，运行`lunch aosp_x86-eng`。因为x86模拟器的比arm的运行速度要快，在debug源码时响应速度还能够接受，所以指定生成x86的rom。之前编译生成的arm模拟器启动和运行速度实在是太慢了。
```
============================================
PLATFORM_VERSION_CODENAME=REL
PLATFORM_VERSION=6.0
TARGET_PRODUCT=aosp_x86
TARGET_BUILD_VARIANT=eng
TARGET_BUILD_TYPE=release
TARGET_BUILD_APPS=
TARGET_ARCH=arm
TARGET_ARCH_VARIANT=x86
TARGET_CPU_VARIANT=generic
TARGET_2ND_ARCH=
TARGET_2ND_ARCH_VARIANT=
TARGET_2ND_CPU_VARIANT=
HOST_ARCH=x86_64
HOST_OS=darwin
HOST_OS_EXTRA=Darwin-16.0.0-x86_64-i386-64bit
HOST_BUILD_TYPE=release
BUILD_ID=MRA58K
OUT_DIR=out
============================================
```

开始编译，执行4线程编译，运行`make -j4`

显示运行完成：
```
\e[0;32m#### make completed successfully (02:45:14 (hh:mm:ss)) ####\e[00m
```
编译花了2小时45分钟。线程数量需要根据CPU核数和内存配置来决定。

## 启动模拟器

使用命令`cd out/target/product/generic`进入generic目录，使用`which emulator`查看当前开启的是否为编译目录的模拟器。
```
/xxx/Android/android-6.0.0_r1/prebuilts/android-emulator/darwin-x86_64/emulator
```
运行`emulator`，等待模拟器开启

![a v d](http://img.blog.csdn.net/20170723202159590?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHdsaXU1MQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## 绑定和调试源码

### 生成工程文件

运行一下命令：
```
mmm development/tools/idegen
development/tools/idegen/idegen.sh
```
进入到android-6.0.0_r1/development/tools/idegen/，将该目录中的android.ipr,android.iml和android.iws这三个文件拷贝到android-6.0.0_r1目录中。

使用`<excludeFolder url="file://$MODULE_DIR$/xxx" />`，将不需要查看的目录排除。这样可以加快导入和查看文件的速度。如果全部倒入，时间会比较长而且会比较卡，可能会出现Android Studio假死情况。为了调式源码方便，我选择只保留frameworks和packages。

### 导入工程
从Android Studio导入已存在的工程，选择android-6.0.0_r1，然后耐心等待。导入之后，生成index又会要占用一段时间。

修改工程设置：
1，修改SDKs设置

![sdk](http://img.blog.csdn.net/20170723211807931?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHdsaXU1MQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
将图中红线框内默认1.8版本的JDK删除，引入1.7版本。只保留23的Android API Platform，其它版本全部删除。

2，修改Modules设置

![modules](http://img.blog.csdn.net/20170723211831740?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHdsaXU1MQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
修改Module SDK为23版本，将“Export”中其它依赖全部删除，并引入23版本Platform。点击“+”，从“JARs or directories”引入frameworks。

3，修改Project设置

![project](http://img.blog.csdn.net/20170723211911888?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHdsaXU1MQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
将Project SDK改为23，Project language level改为7。


### 调试源码

完成以上设置，就可以debug了。点击AS上的虫子图标，会弹出“Choose Process”。因为我需要查看Activity的启动过程，所以选择“system_process”。具体如下图：

![debug](http://img.blog.csdn.net/20170723211943251?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHdsaXU1MQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


点击模拟器中设置图标，进入debuging
![debug AMS](http://img.blog.csdn.net/20170723212428760?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHdsaXU1MQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

可以看到已进入在ActivityManagerService中的断点处，可以愉快的查看Activity启动经过的方法。


----------


参考博客：

Mac 10.10 编译android 4.4.4 for nexus
http://liball.me/mac-10-10-build-android-4-4-4-for-nexus/


自己动手调试Android源码
http://blog.csdn.net/dd864140130/article/details/51815253





