
本文中所使用的工具信息：
系统为Mac 10.12版本。
Android build-tools版本：22.0.1
Android platforms版本：22
JDK版本：1.7
测试的手机系统：Android 5.1

[请尊重博主劳动成果，转载请标明原文链接。](http://blog.csdn.net/hwliu51/article/details/76945265)

### Android应用构建流程

以前使用Eclipse，现在使用Android Studio开发，点击“run”，等待一会儿，便能输出Apk文件。应用构建的基本流程，自己大概明白，但是没有使用过命令行来构建Apk。现在使用命令构建，并记录相关流程。

先看看Android开发文档上的两张图片。

（一）Android工程构建简化图
![build0](http://img.blog.csdn.net/20170804103238900?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHdsaXU1MQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

（二）Android打包流程图
![build progress](http://img.blog.csdn.net/20170804103327668?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHdsaXU1MQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

Apk文件内部可以分为4个部分：
dex文件：有一个或多个，编译所产生的class文件都被转换打包到dex。
资源包resources.arsc：以表的形式记录应用的资源文件信息。
未编译的文件：签名信息相关的文件，以及其它写入信息。
AndroidManifest.xml文件：记录版本信息，应用所需要的权限，四大组件等基本信息。

构建Apk的步骤：
1，通过appt生产R.java和资源包resources.arsc。如果有aidl文件则通过aidl工具生成对应的java文件；
2，通过jdk编译工程中的java文件；
3，将工程编译的class和引入的jar文件，通过dx工具转换生成dex文件；
4，使用ApkBuilder工具将dex文件和资源文件合并为apk文件；
5，使用jarsigner命令和keystore对apk进行签名；
6，使用zipalign命令优化签名后的apk文件。如果为debug包，则不需要进行这一步。

手动创建一个Android工程，使用命令构建一个可安装的apk。

### 一 创建Android工程

手工创建一个简单的PluginDemo工程，目录和文件如下：
```
PluginDemo
|____AndroidManifest.xml
|____build
| |____out
|____gen
|____res
| |____drawable
| | |____p_icon_play.png
| |____layout
| | |____plugin_item.xml
| |____mipmap
| | |____p_icon_lock.png
| |____values
| | |____string.xml
| | |____styles.xml
|____src
| |____com
| | |____plugin
| | | |____test
| | | | |____PluginActivity.java
```

### 二 生产R.java和resoures.ap_

生成R.java和资源文件所使用的aapt基本命令：
-f 如果编译生成的文件存在，则强制覆盖
-m 让生成的文件存放到-J指定的目录
-M 指定AndroidManifest.xml文件的路径
-J 指定R.java文件存放的目录
-A 指定asset source目录
-S 指定res资源目录
-F 指定生成的资源包文件路径
-I 指定编译使用的版本平台的android.jar路径

更多相关命令，可以在控制台执行appt，查看打印出来的Usag信息。

开启终端（Window系统则是控制台），进入到PluginDemo工程目录。使用aapt命令生成R.java文件和生成资源包文件。

```java
../../../devTools/android/android-sdk-macosx/build-tools/22.0.1/aapt package -f -m -J gen -S res -M AndroidManifest.xml -I ../../../devTools/android/android-sdk-macosx/platforms/android-22/android.jar -F build/out/resources.ap_
```

### 三 编译java文件
使用javac编译java文件
```java
javac -encoding GBK -bootclasspath ../../../devTools/android/android-sdk-macosx/platforms/android-22/android.jar -d ./build/out ./gen/com/plugin/test/R.java
```

执行`javac -help`，查看相关命令：
```
用法: javac <options> <source files>
其中, 可能的选项包括:
  -g                         生成所有调试信息
  -g:none                    不生成任何调试信息
  -g:{lines,vars,source}     只生成某些调试信息
  -nowarn                    不生成任何警告
  -verbose                   输出有关编译器正在执行的操作的消息
  -deprecation               输出使用已过时的 API 的源位置
  -classpath <路径>            指定查找用户类文件和注释处理程序的位置
  -cp <路径>                   指定查找用户类文件和注释处理程序的位置
  -sourcepath <路径>           指定查找输入源文件的位置
  -bootclasspath <路径>        覆盖引导类文件的位置
  -extdirs <目录>              覆盖所安装扩展的位置
  -endorseddirs <目录>         覆盖签名的标准路径的位置
  -proc:{none,only}          控制是否执行注释处理和/或编译。
  -processor <class1>[,<class2>,<class3>...] 要运行的注释处理程序的名称; 绕过默认的搜索进程
  -processorpath <路径>        指定查找注释处理程序的位置
  -d <目录>                    指定放置生成的类文件的位置
  -s <目录>                    指定放置生成的源文件的位置
  -implicit:{none,class}     指定是否为隐式引用文件生成类文件
  -encoding <编码>             指定源文件使用的字符编码
  -source <发行版>              提供与指定发行版的源兼容性
  -target <发行版>              生成特定 VM 版本的类文件
  -version                   版本信息
  -help                      输出标准选项的提要
  -A关键字[=值]                  传递给注释处理程序的选项
  -X                         输出非标准选项的提要
  -J<标记>                     直接将 <标记> 传递给运行时系统
  -Werror                    出现警告时终止编译
  @<文件名>                     从文件读取选项和文件名
```

### 四 将class文件转换为dex文件

将编译生成的class文件转换成dex文件
```java
../../../devTools/android/android-sdk-macosx/build-tools/22.0.1/dx --dex --output=build/out/classes.dex build/out/
```
由于系统默认的JDK版本为1.8，所以在使用22.0.1的dx工具生成dex文件时报错了。

```java
UNEXPECTED TOP-LEVEL EXCEPTION:
com.android.dx.cf.iface.ParseException: bad class file magic (cafebabe) or version (0034.0000)
	at com.android.dx.cf.direct.DirectClassFile.parse0(DirectClassFile.java:472)
	at com.android.dx.cf.direct.DirectClassFile.parse(DirectClassFile.java:406)
	at com.android.dx.cf.direct.DirectClassFile.parseToInterfacesIfNecessary(DirectClassFile.java:388)
	at com.android.dx.cf.direct.DirectClassFile.getMagic(DirectClassFile.java:251)
	at com.android.dx.command.dexer.Main.processClass(Main.java:704)
	at com.android.dx.command.dexer.Main.processFileBytes(Main.java:673)
	at com.android.dx.command.dexer.Main.access$300(Main.java:83)
	at com.android.dx.command.dexer.Main$1.processFileBytes(Main.java:602)
	at com.android.dx.cf.direct.ClassPathOpener.processOne(ClassPathOpener.java:170)
	at com.android.dx.cf.direct.ClassPathOpener.processDirectory(ClassPathOpener.java:229)
	at com.android.dx.cf.direct.ClassPathOpener.processOne(ClassPathOpener.java:158)
	at com.android.dx.cf.direct.ClassPathOpener.processDirectory(ClassPathOpener.java:229)
	at com.android.dx.cf.direct.ClassPathOpener.processOne(ClassPathOpener.java:158)
	at com.android.dx.cf.direct.ClassPathOpener.processDirectory(ClassPathOpener.java:229)
	at com.android.dx.cf.direct.ClassPathOpener.processOne(ClassPathOpener.java:158)
	at com.android.dx.cf.direct.ClassPathOpener.processDirectory(ClassPathOpener.java:229)
	at com.android.dx.cf.direct.ClassPathOpener.processOne(ClassPathOpener.java:158)
	at com.android.dx.cf.direct.ClassPathOpener.process(ClassPathOpener.java:144)
	at com.android.dx.command.dexer.Main.processOne(Main.java:632)
	at com.android.dx.command.dexer.Main.processAllFiles(Main.java:510)
	at com.android.dx.command.dexer.Main.runMonoDex(Main.java:280)
	at com.android.dx.command.dexer.Main.run(Main.java:246)
	at com.android.dx.command.dexer.Main.main(Main.java:215)
	at com.android.dx.command.Main.main(Main.java:106)
...while parsing com/plugin/test/PluginActivity.class
```
这是因为22.0.1版本的dx工具无法识别JDK 1.8编译的class文件中的一些内容。将JDK版本切换到1.7版本，重新编译。然后执行dx打包命令，可以正常生产dex文件。

dx基本命令：
--dex 指定生成dex文件
--output=&lt;file&gt; file为生成的dex文件的路径
[&lt;file&gt;.class | &lt;file&gt;.{zip,jar,apk} | &lt;directory&gt;] 指定class或jar文件路径

更多dx命令，可以使用`dx -help`查看。


### 五 合并dex和资源文件为apk

使用ApkBuilder将资源包和dex文件合并为apk文件
```java
java -cp ../../../devTools/android/android-sdk-macosx/tools/lib/sdklib.jar com.android.sdklib.build.ApkBuilderMain build/out/test.apk -u -v -z build/out/resources.ap_ -f build/out/classes.dex
```
`-cp`为java命令，用来指定目录和 zip/jar 文件的类搜索路径。后面的`-u -v -z -f`为ApkBuilder的命令。

ApkBuilder基本命令：
```
A command line tool to package an Android application from various sources.
Usage: apkbuilder <out archive> [-v][-u][-storetype STORE_TYPE] [-z inputzip]
            [-f inputfile] [-rf input-folder] [-rj -input-path]

    -v      Verbose.
    -d      Debug Mode: Includes debug files in the APK file.
    -u      Creates an unsigned package.
    -storetype Forces the KeyStore type. If omitted the default is used.

    -z      Followed by the path to a zip archive.
            Adds the content of the application package.

    -f      Followed by the path to a file.
            Adds the file to the application package.

    -rf     Followed by the path to a source folder.
            Adds the java resources found in that folder to the application
            package, while keeping their path relative to the source folder.

    -rj     Followed by the path to a jar file or a folder containing
            jar files.
            Adds the java resources found in the jar file(s) to the application
            package.

    -nf     Followed by the root folder containing native libraries to
            include in the application package.
```
`-rf `用来绑定源码，`-rj `将指定的jar文件中的资源文件添加到应用包，`-nf `将so文件添加到应用包中。

### 六 对apk签名

如果没有keystore，或者想生成一个，可以使用keytool命令生成keystore文件。
生成一个别名为test，有效期为20000天，密码为123456，文件名称为test.keystore的签名文件。命令和步骤如下：
```html
xxx:PluginDemo liu1359041$ keytool -genkey -alias test -validity 20000 -keystore test.keystore
输入密钥库口令:  
再次输入新口令: 
您的名字与姓氏是什么?
  [Unknown]:  test
您的组织单位名称是什么?
  [Unknown]:  test
您的组织名称是什么?
  [Unknown]:  test
您所在的城市或区域名称是什么?
  [Unknown]:  test
您所在的省/市/自治区名称是什么?
  [Unknown]:  test
该单位的双字母国家/地区代码是什么?
  [Unknown]:  cn
CN=test, OU=test, O=test, L=test, ST=test, C=cn是否正确?
  [否]:  y

输入 <test> 的密钥口令
	(如果和密钥库口令相同, 按回车):  
再次输入新口令: 
```
执行完命令，便会在PluginDemo目录里生成一个test.keystore的签名文件。

使用test.keystore对apk进行签名
```java
jarsigner -verbose -keystore test.keystore -storepass 123456 -keypass 123456 -signedjar build/out/test_s.apk build/out/test.apk test
```
输出的日志信息：
```
   正在添加: META-INF/MANIFEST.MF
   正在添加: META-INF/TEST.SF
   正在添加: META-INF/TEST.DSA
  正在签名: AndroidManifest.xml
  正在签名: res/drawable/p_icon_play.png
  正在签名: res/layout/plugin_item.xml
  正在签名: res/mipmap/p_icon_lock.png
  正在签名: resources.arsc
  正在签名: classes.dex
jar 已签名。

警告: 
未提供 -tsa 或 -tsacert, 此 jar 没有时间戳。如果没有时间戳, 则在签名者证书的到期日期 (2072-05-07) 或以后的任何撤销日期之后, 用户可能无法验证此 jar
```
对apk签名成功。

SHA-256算法在Android 4.2及以版本报错问题。原因是JDK 1.6以前默认采用SHA1算法，而1.6及以后版本则采用了更安全的SHA-256算法，所以在JDK 1.6+环境使用jarsigner签名，默认使用了SHA-256算法。因为未做兼容，在Android 4.2及以下的系统会出现签名验证失败问题。可以在jarsigner命令后追加`-sigalg SHA1withRSA -digestalg SHA1`命令，强制使用SHA1算法。

这个问题在使用Android 4.0的模拟器测试动态加载签名的jar文件时出现过。

### 七 优化apk

使用zipalign对apk进行优化，减少包的体积
```java
../../../devTools/android/android-sdk-macosx/build-tools/22.0.1/zipalign -f 4 build/out/test.apk build/out/test_opt.apk
```

zipalign用法和命令：
```
Usage: zipalign [-f] [-v] [-z] <align> infile.zip outfile.zip
       zipalign -c [-v] <align> infile.zip

  <align>: alignment in bytes, e.g. '4' provides 32-bit alignment
  -c: check alignment only (does not modify file)
  -f: overwrite existing outfile.zip
  -v: verbose output
  -z: recompress using Zopfli
```

执行完以上命令，再查看PluginDemo工程目录和文件如下：
```
PluginDemo
|____AndroidManifest.xml
|____build
| |____out
| | |____classes.dex
| | |____com
| | | |____plugin
| | | | |____test
| | | | | |____PluginActivity.class
| | | | | |____R$attr.class
| | | | | |____R$drawable.class
| | | | | |____R$id.class
| | | | | |____R$layout.class
| | | | | |____R$mipmap.class
| | | | | |____R$string.class
| | | | | |____R$style.class
| | | | | |____R.class
| | |____resources.ap_
| | |____test.apk
| | |____test_opt.apk
| | |____test_s.apk
|____gen
| |____com
| | |____plugin
| | | |____test
| | | | |____R.java
|____res
| |____drawable
| | |____p_icon_play.png
| |____layout
| | |____plugin_item.xml
| |____mipmap
| | |____p_icon_lock.png
| |____values
| | |____string.xml
| | |____styles.xml
|____src
| |____com
| | |____plugin
| | | |____test
| | | | |____PluginActivity.java
|____test.keystore
```
构建过程中生成了不少的文件。最终能够运行的是test_opt.apk（经过优化的apk）和test_s.apk这两个文件。

安装运行test_opt.apk，如下图：
![test](http://img.blog.csdn.net/20170804181355555?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaHdsaXU1MQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
手工创建的工程和命令构建的Apk正常运行。虽然系统是5.1，由于没有配置Activity的主题和继承framework中的Actvity，而不是v4或v7包的，所以显示的像非常老旧2.3系统上的应用。

### 参考文章

Android官方开发文档

手动命令行编译 APK
https://juejin.im/entry/57cb841e2e958a0068dcf679

Android 打包过程
http://www.jianshu.com/p/7c288a17cda8

