*本文修改的aapt的源码为Android 6.0.0_r1版本*

[请尊重博主劳动成果，转载请标明原文链接。](http://blog.csdn.net/hwliu51/article/details/76945286)

**本文中的aapt源码查看和修改参照[Android中如何修改编译的资源ID值(默认值是0x7F...可以随意改成0x02~0x7E)](http://blog.csdn.net/jiangwei0910410003/article/details/50820219)和[Android aapt实现资源分区（补充携程aapt源码）](http://blog.csdn.net/sbsujjbcy/article/details/51405207)这两篇博客。本文主要是记录修改过程和编译aapt模块的命令。**

### 源码查看
aapt的源码在所在的目录：Android/frameworks/base/tools/aapt/。

Main.cpp位置：android-6.0.0_r1/frameworks/base/tools/aapt/Main.cpp
首先查看Main.cpp的main方法:

```c++
int main(int argc, char* const argv[])
{
    char *prog = argv[0];
    Bundle bundle;
    bool wantUsage = false;
    int result = 1;    // pessimistically assume an error.
    
    ...
    //进行资源打包的参数p或package
    else if (argv[1][0] == 'p')
        bundle.setCommand(kCommandPackage);
    ...
    
    /*
     * Pull out flags.  We support "-fv" and "-f -v".
     */
    while (argc && argv[0][0] == '-') {
        /* flag(s) found */
        const char* cp = argv[0] +1;

        while (*cp != '\0') {
            ...
            //收集appt命令输入的参数，这些参数以"-"开头
            case '-':
                if (strcmp(cp, "-debug-mode") == 0) {
                    bundle.setDebugMode(true);
                } 
            ...
                break;
            default:
                fprintf(stderr, "ERROR: Unknown flag '-%c'\n", *cp);
                wantUsage = true;
                goto bail;
            }

            cp++;
        }
        argc--;
        argv++;
    }

    /*
     * We're past the flags.  The rest all goes straight in.
     */
    bundle.setFileSpec(argv, argc);
    //根据bundle收集的参数进行资源处理
    result = handleCommand(&bundle);

//输入参数错误时会跳转到此
bail:
    if (wantUsage) {
        usage();
        result = 2;
    }
    
    return result;
}
```
在main函数内，首先创建一个Bundle对象，这个对象用来存储输入的操作类型和相关的参数。argv相当于java中的字符串数组。取该数组的第2个字符串的第一个char值。因为是执行资源打包，所以它是'p'。bundle记录执行类型为kCommandPackage，即资源打包。while循环处理剩余的char数组（即字符串数组），将参数按照类型设置到bundle中。参数解析完毕，则会执行`handleCommand(&bundle)`。如果在解析输入的参数时出现了错误，便使用goto跳转到bail代码块。在bail代码块中可能会执行`usage();`，这个方法会打印出aapt所有的命令类型和相关的参数。

先看看usage()函数，省略了与资源打包无关的信息。
```C++
void usage(void)
{
    ...
    fprintf(stderr,
        " %s p[ackage] [-d][-f][-m][-u][-v][-x][-z][-M AndroidManifest.xml] \\\n"
        "        [-0 extension [-0 extension ...]] [-g tolerance] [-j jarfile] \\\n"
        "        [--debug-mode] [--min-sdk-version VAL] [--target-sdk-version VAL] \\\n"
        "        [--app-version VAL] [--app-version-name TEXT] [--custom-package VAL] \\\n"
        "        [--rename-manifest-package PACKAGE] \\\n"
        "        [--rename-instrumentation-target-package PACKAGE] \\\n"
        "        [--utf16] [--auto-add-overlay] \\\n"
        "        [--max-res-version VAL] \\\n"
        "        [-I base-package [-I base-package ...]] \\\n"
        "        [-A asset-source-dir]  [-G class-list-file] [-P public-definitions-file] \\\n"
        "        [-S resource-sources [-S resource-sources ...]] \\\n"
        "        [-F apk-file] [-J R-file-dir] \\\n"
        "        [--product product1,product2,...] \\\n"
        "        [-c CONFIGS] [--preferred-density DENSITY] \\\n"
        "        [--split CONFIGS [--split CONFIGS]] \\\n"
        "        [--feature-of package [--feature-after package]] \\\n"
        "        [raw-files-dir [raw-files-dir] ...] \\\n"
        "        [--output-text-symbols DIR]\n"
        "        [--apk-module moduleName]\n"
        "\n"
        "   Package the android resources.  It will read assets and resources that are\n"
        "   supplied with the -M -A -S or raw-files-dir arguments.  The -J -P -F and -R\n"
        "   options control which files are output.\n\n"
        , gProgName);
        ...
}
```
打印appt执行的操作类型和对应的参数。

先看看handleCommand()函数，
```C++
int handleCommand(Bundle* bundle)
{
    switch (bundle->getCommand()) {
    case kCommandVersion:      return doVersion(bundle);
    ...
    case kCommandPackage:      return doPackage(bundle);
    ...
    default:
        fprintf(stderr, "%s: requested command not yet supported\n", gProgName);
        return 1;
    }
}
```
在main函数中bundle设置的cmmand类型为kCommandPackage，所以会执行`doPackage(bundle)`代码。使用操作系统自带的搜索，检索aapt目录内的文件内容'doPackage'。会查找到doPackage()是Command.cpp中的函数。

Command.cpp位置：android-6.0.0_r1/frameworks/base/tools/aapt/Command.cpp
doPackage()函数中处理资源相关的代码：
```C++
int doPackage(Bundle* bundle)
{
    ...
    // If they asked for any fileAs that need to be compiled, do so.
    if (bundle->getResourceSourceDirs().size() || bundle->getAndroidManifestFile()) {
        err = buildResources(bundle, assets, builder);
        if (err != 0) {
            goto bail;
        }
    }
    ...
}
```
继续搜索buildResources()函数所在的文件，查找到Resource.cpp。

Resource.cpp位置：android-6.0.0_r1/frameworks/base/tools/aapt/Resource.cpp
buildResources()函数中创建资源表相关的代码：
```C++
status_t buildResources(Bundle* bundle, const sp<AaptAssets>& assets, sp<ApkBuilder>& builder)
{
    ...
    //设置资源类型
    ResourceTable::PackageType packageType = ResourceTable::App;
    if (bundle->getBuildSharedLibrary()) {
        packageType = ResourceTable::SharedLibrary;
    } else if (bundle->getExtending()) {
        packageType = ResourceTable::System;
    } else if (!bundle->getFeatureOfPackage().isEmpty()) {
        packageType = ResourceTable::AppFeature;
    }
    //创建资源表
    ResourceTable table(bundle, String16(assets->getPackage()), packageType);
    ...
}
```
继续查看ResourceTable.cpp的构造函数。

ResourceTable.cpp位置：android-6.0.0_r1/frameworks/base/tools/aapt/ResourceTable.cpp
ResourceTable构造函数
```C++
ResourceTable::ResourceTable(Bundle* bundle, const String16& assetsPackage, ResourceTable::PackageType type)
    : mAssetsPackage(assetsPackage)
    , mPackageType(type)
    , mTypeIdOffset(0)
    , mNumLocal(0)
    , mBundle(bundle)
{
    ssize_t packageId = -1;
    switch (mPackageType) {
        case App:
        case AppFeature:
            packageId = 0x7f;
            break;

        case System:
            packageId = 0x01;
            break;

        case SharedLibrary:
            packageId = 0x00;
            break;

        default:
            assert(0);
            break;
    }


    sp<Package> package = new Package(mAssetsPackage, packageId);
    mPackages.add(assetsPackage, package);
    mOrderedPackages.add(package);

    // Every resource table always has one first entry, the bag attributes.
    const SourcePos unknown(String8("????"), 0);
    getType(mAssetsPackage, String16("attr"), unknown);
}
```
应用的资源id从0x7f开始，系统的资源id从0x01开始，共享类库的从0x00开始。如果我们想要自定义应用的资源id的起始值，则需要在switch结束后重新设置packageId值。这个自定义值可以Main.cpp的main函数中解析获取，并存放到bundle中。


### 修改源码
1，修改Bundle.h文件。
Bundle.h位置：android-6.0.0_r1/frameworks/base/tools/aapt/Bundle.h

添加如下代码：
```C++
class Bundle {
public:
    ...
    //添加的获取和设置自定义id的函数
    const android::String8& getApkModule() const {return mApkModule;}
    void setApkModule(const char* str) { mApkModule=str;}
    ...

private:
    /* commands & modifiers */
    ...
    //自定义id
    android::String8 mApkModule;
    ...
}
```

2，修改Main.cpp的main函数，添加解析自定义id的参数并设置到bundle。
代码如下：
```C++
int main(int argc, char* const argv[])
{
    ...
    while (argc && argv[0][0] == '-') {
        ...
        while (*cp != '\0') {
            ...
            case '-':
                ...
                //添加解析-apk-module参数信息
                } else if(strcmp(cp, "-apk-module") == 0){
                    argc--;
                    argv++;
                    if (!argc) {
                        fprintf(stderr, "ERROR: No argument supplied for '--apk-model' option\n");
                         wantUsage = true;
                         goto bail;
                    }
                    bundle.setApkModule(argv[0]);
                } else if (strcmp(cp, "-feature-of") == 0) {
            	...
                break;
            ...
        }
        ...
    }
    ...
}
```

3，修改ResourceTable.cpp的构造函数，添加判断是否存在自定义id，如果存在，则修改packageId为自定义id。
```C++
ResourceTable::ResourceTable(Bundle* bundle, const String16& assetsPackage, ResourceTable::PackageType type)
    ...
{
    ssize_t packageId = -1;
    switch (mPackageType) {
        ...
    }
    
    //判断和设置自定义id
    if(!bundle->getApkModule().isEmpty()){
        android::String8 apkModuleVal = bundle->getApkModule();
        packageId = apkStringToInt(apkModuleVal);
    }
    
    sp<Package> package = new Package(mAssetsPackage, packageId);
    ...
}

//将字符串转换为ssize_t类型
ssize_t ResourceTable::apkStringToInt(const String8& s){
    size_t i = 0;
    ssize_t val = 0;
    size_t len=s.length();
    if (s[i] < '0' || s[i] > '9') {
        return -1;
    }

    // Decimal or hex?
    if (s[i] == '0' && s[i+1] == 'x') {
        i += 2;
        bool error = false;
        while (i < len && !error) {
            val = (val*16) + apkgetHex(s[i], &error);
            i++;
        }
        if (error) {
            return -1;
        }
    } else {
        while (i < len) {
            if (s[i] < '0' || s[i] > '9') {
                return false;
            }
            val = (val*10) + s[i]-'0';
            i++;
        }
    }

    if (i == len) {
        return val;
    }
    return -1;
}
//转换为16进制
uint32_t ResourceTable::apkgetHex(char c, bool* outError){
    if (c >= '0' && c <= '9') {
        return c - '0';
    } else if (c >= 'a' && c <= 'f') {
        return c - 'a' + 0xa;
    } else if (c >= 'A' && c <= 'F') {
        return c - 'A' + 0xa;
    }
    *outError = true;
    return 0;
}
```

还需要在ResourceTable.h文件中声明apkStringToInt()和apkgetHex()函数。
ResourceTable.h位置：android-6.0.0_r1/frameworks/base/tools/aapt/ResourceTable.h
添加到public或private中。
```C++
    ssize_t apkStringToInt(const String8& s);
    uint32_t apkgetHex(char c, bool* outError);
```

### 编译和修改错误

修改完毕，开启终端（或控制台）进入到android-6.0.0_r1目录，运行`. build/envsetup.sh`命令，配置运行环境。配置命令运行完毕，运行`cd frameworks/base/tools/aapt`，进入到aapt目录。执行`mm`命令，该命令会编译aapt模块中的代码，并生成可执行文件。运行结果如下：
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
host C++: aapt <= frameworks/base/tools/aapt/Main.cpp
host Executable: aapt (out/host/darwin-x86/obj/EXECUTABLES/aapt_intermediates/aapt)
clang: warning: argument unused during compilation: '-pie'
Install: out/host/darwin-x86/bin/aapt

\e[0;32m#### make completed successfully (3 seconds) ####\e[00m
```
生产的文件为aapt，在'android-6.0.0_r1/out/host/darwin-x86/bin/'目录中。
将aapt拷贝到指定目录，进入PluginDemo工程目录，使用aapt对工程进行资源打包，执行的命令如下：
```
../../../devTools/aapt/aapt package -f -m --apk-module 0x8f -J gen -S res -M AndroidManifest.xml -I ../../../devTools/android/android-sdk-macosx/platforms/android-23
/android.jar -F build/out/res.ap_
```
该命令执行失败，某个资源id找不到对应的资源。错误信息：
```
res/layout/xxx.xml:9: error: Error: No resource found that matches the given name (at 'text' with value '@string/xxx').
```

搜索相关的博客，在[区长的专栏](http://blog.csdn.net/sbsujjbcy)的[Android aapt实现资源分区（补充携程aapt源码）](http://blog.csdn.net/sbsujjbcy/article/details/51405207)这篇博客找到了出错的原因和解决方法。

继续看代码，
```C++
bool ResTable::stringToValue(Res_value* outValue, String16* outString,
                             const char16_t* s, size_t len,
                             bool preserveSpaces, bool coerceType,
                             uint32_t attrID,
                             const String16* defType,
                             const String16* defPackage,
                             Accessor* accessor,
                             void* accessorCookie,
                             uint32_t attrType,
                             bool enforcePrivate) const
{
    ...
    if (*s == '@') {
            if (accessor) {
                uint32_t rid = accessor->getCustomResourceWithCreation(package, type, name,
                                                                       createIfNotFound);
                if (rid != 0) {
                    if (kDebugTableNoisy) {
                        ALOGI("Pckg %s:%s/%s: 0x%08x\n",
                                String8(package).string(), String8(type).string(),
                                String8(name).string(), rid);
                    }
                    uint32_t packageId = Res_GETPACKAGE(rid) + 1;
                    if (packageId == 0x00) {
                        outValue->data = rid;
                        outValue->dataType = Res_value::TYPE_DYNAMIC_REFERENCE;
                        return true;
                    } else if (packageId == APP_PACKAGE_ID || packageId == SYS_PACKAGE_ID ) {
                        // We accept packageId's generated as 0x01 in order to support
                        // building the android system resources
                        outValue->data = rid;
                        return true;
                    }
                }
            }
        }

        if (accessor != NULL) {
            accessor->reportError(accessorCookie, "No resource found that matches the given name");
        }
        return false;
    }
    ...
}
```
在if(accessor)代码中对packageId进行验证。如果packageId不是ResourceTable构造函数中设置的3种类型，则出错，无法生成R.java文件和资源包。需要将ResourceTable构造函数中设置的packageId值存储下来，并在`if (packageId == APP_PACKAGE_ID || packageId == SYS_PACKAGE_ID )`添加一个自定义id判断。

按照区长的博客添加头文件和cpp文件，并添加相应的代码。具体的代码请查看[Android aapt实现资源分区（补充携程aapt源码）](http://blog.csdn.net/sbsujjbcy/article/details/51405207)博客，本文不再写出。
1，创建文件
创建Help.h文件，存放在android-6.0.0_r1/frameworks/base/include/androidfw/。
创建Help.cpp文件，存放在android-6.0.0_r1/frameworks/base/libs/androidfw/。并将Help.cpp添加到android-6.0.0_r1/frameworks/base/libs/androidfw/Android.mk文件。
2，修改代码
在ResourceTable::ResourceTable()函数判断和设置自定义id代码后面添加使用Help记录packageId代码。
在ResTable::stringToValue()函数中添加对Help中记录的packageId判断代码。

重新编译aapt模块，获取新生成的aapt可执行文件。再次运行生成R.java和资源打包命令（与之前的命令相同），成功运行。

生成的R.java文件内容
```java
package com.plugin.test;

public final class R {
    public static final class attr {
    }
    public static final class drawable {
        public static final int p_icon_play=0x8f020000;
    }
    public static final class id {
        public static final int activity_main=0x8f070000;
    }
    public static final class layout {
        public static final int plugin_item=0x8f040000;
    }
    public static final class mipmap {
        public static final int p_icon_lock=0x8f030000;
    }
    public static final class string {
        public static final int app_name=0x8f050000;
        public static final int p_str=0x8f050001;
    }
    public static final class style {
        
        public static final int AppBaseTheme=0x8f060000;
        
        public static final int AppTheme=0x8f060001;
    }
}
```
不再是原来系统默认的0x7f，都是自定义的0x8f。

继续执行命令编译代码，将class转换为dex，并将dex和资源包合并生成的apk。对apk签名，运行apk。apk正常运行，截图如下。

![apk](http://img.blog.csdn.net/20170806003808134?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHdsaXU1MQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


