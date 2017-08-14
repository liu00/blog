[请尊重博主劳动成果，转载请标明出处。](http://blog.csdn.net/hwliu51/article/details/74896512)

### 一 下载相关开发工具
#### 1 JDK
已安装了JDK 1.8

#### 2 Android SDK
（Android相关资源整理网站：http://www.androiddevtools.cn/）
这个已有，环境变量也设置好了。
Android SDK	 Paltform-tools 25.0.1
Android SDK Build-tools 25.0.1
Android SDK Build-tools 23.0.1
SDK Platform 25
SDK Platform 23
Android Support Repository 更新到最新

react-native的Android相关的模块中使用到23的编译sdk和build-tools。而25是我用来编译工程用的。将Android Support Repository 更新到最新后，就不会出现缺少第三方包问题。

#### 3 Android Studio 
已安装了2.2版本，用来编译和调试，以及添加自定义的模块。

#### 4 [Python](https://www.python.org/downloads/release/python-2710/)
下载2.7版本的安装包，直接进行安装。这个好像是用来作为运行辅助的python脚本。

#### 5 [Node.js](https://nodejs.org/)
下载6.10.3版本安装。用于安装react-native相关的包，以及当作服务器运行编写的js代

#### 6 [Git](https://git-for-windows.github.io/)
已安装了2.10.1版本，这个用于版本管理。


### 二 使用命令安装React Native

运行node.js工具，配置下npm镜像。（以后的下载就可以直接从淘宝的下载，可以有效避免几kb的国外下载速度。）。命令如下：

```js
npm config set registry https://registry.npm.taobao.org --global
npm config set disturl https://npm.taobao.org/dist --global
```

现在可以安装React Native相关的工具。
Yarn是Facebook提供的替代npm的工具，可以加速node模块的下载。React Native的命令行工具用于执行创建、初始化、更新项目、运行打包服务（packager）等任务。
运行以下命令：

```js
npm install -g yarn react-native-cli
```

继续设置yarn镜像。

```
yarn config set registry https://registry.npm.taobao.org --global
yarn config set disturl https://npm.taobao.org/dist --global
```

使用命令查看安装的ract-native-cli和yarn版本。

```
react-native -v
```
结果：
react-native-cli: 2.0.1
react-native: n/a - not inside a React Native project directory

查看yarn版本。

```
yarn -v
```
结果：
yarn install v0.24.4

安装react-native成功。

### 三创建工程和运行
#### 1 创建工程
将控制台的当前目录切换到工程目录，使用命令创建react-native工程。

```
react-native init RNDemo
```
运行命令后，控制台便不断输出生成过程的日志信息。需要10来分钟的手机。如果网络好，可能会更短些。

#### 2 运行工程
RNDemo工程生成成功，运行Android Studio，将RNDemo中的android项目导入。连接手机编译运行。手机上显示了一个红色的错误页面。

因为没有运行js，所以提示服务器找不到。使用`cd RNDemo`进入工程，继续输入命令：

```
react-native start
```
输出日志信息：

```
Scanning 580 folders for symlinks in /xx/xx/RNDemo/node_modules (8ms)
 ┌────────────────────────────────────────────────────────────────────────────┐ 
 │  Running packager on port 8081.                                            │ 
 │                                                                            │ 
 │  Keep this packager running while developing on any JS projects. Feel      │ 
 │  free to close this tab and run your own packager instance if you          │ 
 │  prefer.                                                                   │ 
 │                                                                            │ 
 │  https://github.com/facebook/react-native                                  │ 
 │                                                                            │ 
 └────────────────────────────────────────────────────────────────────────────┘ 
Looking for JS files in
   /xx/xx/RNDemo 


React packager ready.
```

可以看到“React packager ready”，RNDemo的js代码成功运行。从手机的任务栏退出RNDemo，重新进入。结果为白屏。

再启动一个控制台，输入以下adb命令：
```
adb reverse tcp:8081 tcp:8081
```
如果有连接多个设备，则需要指定设备。首先使用`adb devices`查看设备，然后使用`-s`来指定设备。
例如：查看到的设备号为123-XX-0x1234,则输入命令为`adb -s 123-XX-0x1234 reverse tcp:8081 tcp:8081`。

#### 3 安装其它模块命令
如果js代码有引用到navigation或redux等模块时，由于没有安装这些模块，运行时会报错。可以使用npm命令来安装这些模块，格式：**`npm install --save modelName`**。
例如：
安装react-navigation：`npm install --save react-navigation`
安装react-redux：`npm install --save react-redux`

### 四 推荐的网站

**js6语法学习**
ECMAScript 6 入门：http://es6.ruanyifeng.com/

**API介绍和使用文档**
英文文档：https://facebook.github.io/react-native/docs/getting-started.html
中文文档：https://reactnative.cn/docs/0.46/getting-started.html

**RN导航跳转**
React Navigation：https://reactnavigation.org/docs/navigators/navigation-prop

**状态和数据管理容器**
Redux中文文档：http://cn.redux.js.org/

**官方Demo**
F8 App开发指南：https://f8-app.liaohuqiu.net/#content





