
*本所引用的代码为Android 6.0版本*

[请尊重博主劳动成果，转载请标明出处。](http://blog.csdn.net/hwliu51/article/details/75579130)

## 一 ZygoteInit进程

本文从Java的文件来分析ZygotInit创建流程。如果想从linux和底层C/C++来分析，可以阅读底部的参考博客。

/android-6.0.0_r1/frameworks/base/core/java/com/android/internal/os/ZygoteInit.java

1，首先调用main方法：

```java
    public static void main(String argv[]) {
        try {
            ...
            boolean startSystemServer = false;
            String socketName = "zygote";
            String abiList = null;
            ...

            //创建LocalServerSocket，用来接收创建进程的请求和通信
            registerZygoteSocket(socketName);
            ...
            //执行预加载
            preload();

            ...

            if (startSystemServer) {//创建系统服务进程
                startSystemServer(abiList, socketName);
            }

            Log.i(TAG, "Accepting command socket connections");
            //接收Socket连接并处理相应请求
            runSelectLoop(abiList);

            //关闭ServerSocket，即ZygoteInit进程退出
            closeServerSocket();
        } catch (MethodAndArgsCaller caller) {
            //调用该异常的run方法
            caller.run();
        } catch (RuntimeException ex) {
            Log.e(TAG, "Zygote died with exception", ex);
            closeServerSocket();
            throw ex;
        }
    }
```
获取系统abi（Android Binary Interface）列表，创建ServerSocket。接着创建系统服务进程，完成之后便无限循环接收Socket连接和处理请求信息。于是Zygote进程便创建了。Zygote进程主要是用来fork创建VM。


2，创建LocalServerSocket的registerZygoteSocket()方法代码：

```java
    private static void registerZygoteSocket(String socketName) {
        if (sServerSocket == null) {
            int fileDesc;
            //ANDROID_SOCKET_PREFIX为"ANDROID_SOCKET_"，socketName为“zygote”
            final String fullSocketName = ANDROID_SOCKET_PREFIX + socketName;
            try {
                String env = System.getenv(fullSocketName);
                fileDesc = Integer.parseInt(env);
            } catch (RuntimeException ex) {
                throw new RuntimeException(fullSocketName + " unset or invalid", ex);
            }

            try {
                FileDescriptor fd = new FileDescriptor();
                fd.setInt$(fileDesc);
                sServerSocket = new LocalServerSocket(fd);
            } catch (IOException ex) {
                throw new RuntimeException(
                        "Error binding to local socket '" + fileDesc + "'", ex);
            }
        }
    }
```
创建LocalServerSocket服务，它会一直接收Socket链接，并对Socket发送过来的信息进行处理，即创建相应的进程。

3，preload()方法代码：

```java
    static void preload() {
        Log.d(TAG, "begin preload");
        //加载class
        preloadClasses();
        //加载资源
        preloadResources();
        //加载OpenGL相关的资源
        preloadOpenGL();
        //加载公共的so文件
        preloadSharedLibraries();
        //加载文字相关的资源
        preloadTextResources();
        // Ask the WebViewFactory to do any initialization that must run in the zygote process,
        // for memory sharing purposes.
        //创建WebView进程，即Chromium进程
        WebViewFactory.prepareWebViewInZygote();
        Log.d(TAG, "end preload");
    }

```
preload()方法会加载系统运行所需的基本资源和文件。

4，runSelectLoop()方法代码：

```java
    private static void runSelectLoop(String abiList) throws MethodAndArgsCaller {
        ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
        ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();

        fds.add(sServerSocket.getFileDescriptor());
        peers.add(null);

        //无限循环接收连接
        while (true) {
            StructPollfd[] pollFds = new StructPollfd[fds.size()];
            for (int i = 0; i < pollFds.length; ++i) {
                pollFds[i] = new StructPollfd();
                pollFds[i].fd = fds.get(i);
                pollFds[i].events = (short) POLLIN;
            }
            try {
                Os.poll(pollFds, -1);
            } catch (ErrnoException ex) {
                throw new RuntimeException("poll failed", ex);
            }
            for (int i = pollFds.length - 1; i >= 0; --i) {
                if ((pollFds[i].revents & POLLIN) == 0) {
                    continue;
                }
                if (i == 0) {
                    ZygoteConnection newPeer = acceptCommandPeer(abiList);
                    peers.add(newPeer);
                    fds.add(newPeer.getFileDesciptor());
                } else {
                    //执行ZygoteConnection的runOnce()方法
                    boolean done = peers.get(i).runOnce();
                    //如果为true，则将ZygoteConnection对象从peers移除，
                    if (done) {
                        peers.remove(i);
                        fds.remove(i);
                    }
                }
            }
        }
    }
```
无限循环等等和接收Socket连接，并将获取的连接封装为ZygoteConnection对象，并将对象存入peers列表，只有执行结果为true的对象才将其从peers集合中移除。这种处理方式能够让执行失败的ZygoteConnection对象被循环处理。ZygoteConnection对象带有需要创建的进程信息和执行创建的方法。ZygoteConnection类将在下一节进行分析。

5，acceptCommandPeer()方法代码：
```java
    private static ZygoteConnection acceptCommandPeer(String abiList) {
        try {
            return new ZygoteConnection(sServerSocket.accept(), abiList);
        } catch (IOException ex) {
            throw new RuntimeException(
                    "IOException during accept()", ex);
        }
    }
```
这个方法很简单，接收Socket链接，将收到的连接封装为ZygoteConnection对象。

Zygote进程创建成功，之后所有的应用进程都是从它fork而来。

## 二 创建应用进程

### ActivityManagerService启动应用创建
/android-6.0.0_r1/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java

应用进程由ActivityManagerService创建：
以点击桌面图标启动应用（假设该应用未启动过）为例，从startActivity开启，最终会调用到ActivityService的startProcessLocked()方法。
6，startProcessLocked()方法主要代码：

```java
    private final void startProcessLocked(ProcessRecord app, String hostingType,
            String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {
        
        ...

        try {
            ...

            boolean isActivityProcess = (entryPoint == null);
            //ActivityThread的类路径，使用反射来启用ActivityThread
            if (entryPoint == null) entryPoint = "android.app.ActivityThread";
            ...
            //开始创建进程
            Process.ProcessStartResult startResult = Process.start(entryPoint,
                    app.processName, uid, uid, gids, debugFlags, mountExternal,
                    app.info.targetSdkVersion, app.info.seinfo, requiredAbi, instructionSet,
                    app.info.dataDir, entryPointArgs);
            
        } catch (RuntimeException e) {
            ...
        }
        ...
    }
```
在startProcessLocked中会调用Process.start()方法来开启一个进程，该进程便是Activity运行的进程。

### Process向ServerSocket发送参数信息
/android-6.0.0_r1/frameworks/base/core/java/android/os/Process.java

7，Process.start()方法代码：
```java
    public static final ProcessStartResult start(final String processClass,
                                  final String niceName,
                                  int uid, int gid, int[] gids,
                                  int debugFlags, int mountExternal,
                                  int targetSdkVersion,
                                  String seInfo,
                                  String abi,
                                  String instructionSet,
                                  String appDataDir,
                                  String[] zygoteArgs) {
        try {
            return startViaZygote(processClass, niceName, uid, gid, gids,
                    debugFlags, mountExternal, targetSdkVersion, seInfo,
                    abi, instructionSet, appDataDir, zygoteArgs);
        } catch (ZygoteStartFailedEx ex) {
            ...
        }
    }
```
8，继续往下看，startViaZygote方法代码：
```java
    private static ProcessStartResult startViaZygote(final String processClass,
                                  final String niceName,
                                  final int uid, final int gid,
                                  final int[] gids,
                                  int debugFlags, int mountExternal,
                                  int targetSdkVersion,
                                  String seInfo,
                                  String abi,
                                  String instructionSet,
                                  String appDataDir,
                                  String[] extraArgs)
                                  throws ZygoteStartFailedEx {
        synchronized(Process.class) {
            ArrayList<String> argsForZygote = new ArrayList<String>();

            // --runtime-args, --setuid=, --setgid=,
            // and --setgroups= must go first
            argsForZygote.add("--runtime-args");
            argsForZygote.add("--setuid=" + uid);
            argsForZygote.add("--setgid=" + gid);
            ...//省略了debug相关的参数
            if (mountExternal == Zygote.MOUNT_EXTERNAL_DEFAULT) {
                argsForZygote.add("--mount-external-default");
            } else if (mountExternal == Zygote.MOUNT_EXTERNAL_READ) {
                argsForZygote.add("--mount-external-read");
            } else if (mountExternal == Zygote.MOUNT_EXTERNAL_WRITE) {
                argsForZygote.add("--mount-external-write");
            }
            argsForZygote.add("--target-sdk-version=" + targetSdkVersion);

            // --setgroups is a comma-separated list
            if (gids != null && gids.length > 0) {
                StringBuilder sb = new StringBuilder();
                sb.append("--setgroups=");

                int sz = gids.length;
                for (int i = 0; i < sz; i++) {
                    if (i != 0) {
                        sb.append(',');
                    }
                    sb.append(gids[i]);
                }

                argsForZygote.add(sb.toString());
            }

            if (niceName != null) {
                argsForZygote.add("--nice-name=" + niceName);
            }

            if (seInfo != null) {
                argsForZygote.add("--seinfo=" + seInfo);
            }

            if (instructionSet != null) {
                argsForZygote.add("--instruction-set=" + instructionSet);
            }

            if (appDataDir != null) {
                argsForZygote.add("--app-data-dir=" + appDataDir);
            }

            //将ActivityMangerService设置的参数添加到集合中
            argsForZygote.add(processClass);

            if (extraArgs != null) {
                for (String arg : extraArgs) {
                    argsForZygote.add(arg);
                }
            }

            return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);
        }
    }
```
将传入的参数设置到argsForZygote集合，再调用openZygoteSocketIfNeeded创建了连接Zygote的ServerSocket的Socket连接，然后调用zygoteSendArgsAndGetResult方法。

9，先看看openZygoteSocketIfNeeded：
```java
    private static ZygoteState openZygoteSocketIfNeeded(String abi) throws ZygoteStartFailedEx {
        if (primaryZygoteState == null || primaryZygoteState.isClosed()) {
            try {
                //ZYGOTE_SOCKET为"zygote"
                primaryZygoteState = ZygoteState.connect(ZYGOTE_SOCKET);
            } catch (IOException ioe) {
                throw new ZygoteStartFailedEx("Error connecting to primary zygote", ioe);
            }
        }

        if (primaryZygoteState.matches(abi)) {
            return primaryZygoteState;
        }

        // The primary zygote didn't match. Try the secondary.
        if (secondaryZygoteState == null || secondaryZygoteState.isClosed()) {
            try {
            //SECONDARY_ZYGOTE_SOCKET为"zygote_secondary"
            secondaryZygoteState = ZygoteState.connect(SECONDARY_ZYGOTE_SOCKET);
            } catch (IOException ioe) {
                throw new ZygoteStartFailedEx("Error connecting to secondary zygote", ioe);
            }
        }

        if (secondaryZygoteState.matches(abi)) {
            return secondaryZygoteState;
        }

        throw new ZygoteStartFailedEx("Unsupported zygote ABI: " + abi);
    }
```
方法内先创建了primaryZygoteState连接，如果abi不匹配，则再创建secondaryZygoteState。如果都不匹配，则抛出“Unsupported zygote ABI”异常。这两种连接都是调用ZygoteState.connect()方法获取的。
ZygoteState.connect()内使创建了LocalSocket，

```java
        public static ZygoteState connect(String socketAddress) throws IOException {
            DataInputStream zygoteInputStream = null;
            BufferedWriter zygoteWriter = null;
            final LocalSocket zygoteSocket = new LocalSocket();

            try {
                //连接ServerSocket
                zygoteSocket.connect(new LocalSocketAddress(socketAddress,
                        LocalSocketAddress.Namespace.RESERVED));
                //获取输入流
                zygoteInputStream = new DataInputStream(zygoteSocket.getInputStream());
                //获取输出流
                zygoteWriter = new BufferedWriter(new OutputStreamWriter(
                        zygoteSocket.getOutputStream()), 256);
            } catch (IOException ex) {
                …
            }

            //向ServerSocket获取系统支持的abi列表
            String abiListString = getAbiList(zygoteWriter, zygoteInputStream);
            …
            //封装的ZygoteState，并返回
            return new ZygoteState(zygoteSocket, zygoteInputStream, zygoteWriter,
                    Arrays.asList(abiListString.split(",")));
        }
```
获取abi列表的代码与普通Socket通信一样，先向Server写入请求信息，然后等待并获取Server返回的信息。

10，继续看代码，zygoteSendArgsAndGetResult方法：

```java
    private static ProcessStartResult zygoteSendArgsAndGetResult(
            ZygoteState zygoteState, ArrayList<String> args)
            throws ZygoteStartFailedEx {
        try {
            /**
             * See com.android.internal.os.ZygoteInit.readArgumentList()
             * Presently the wire format to the zygote process is:
             * a) a count of arguments (argc, in essence)
             * b) a number of newline-separated argument strings equal to count
             *
             * After the zygote process reads these it will write the pid of
             * the child or -1 on failure, followed by boolean to
             * indicate whether a wrapper process was used.
             */
            final BufferedWriter writer = zygoteState.writer;
            final DataInputStream inputStream = zygoteState.inputStream;

            //写入参数的数量
            writer.write(Integer.toString(args.size()));
            //写入行分割符号
            writer.newLine();

            //遍历写入参数
            int sz = args.size();
            for (int i = 0; i < sz; i++) {
                String arg = args.get(i);
                if (arg.indexOf('\n') >= 0) {
                    throw new ZygoteStartFailedEx(
                            "embedded newlines not allowed");
                }
                writer.write(arg);
                writer.newLine();
            }

            //刷新缓冲，将数据发送个Server
            writer.flush();

            // Should there be a timeout on this?
            ProcessStartResult result = new ProcessStartResult();
            //获取结果
            result.pid = inputStream.readInt();
            if (result.pid < 0) {
                throw new ZygoteStartFailedEx("fork() failed");
            }
            result.usingWrapper = inputStream.readBoolean();
            return result;
        } catch (IOException ex) {
            zygoteState.close();
            throw new ZygoteStartFailedEx(ex);
        }
    }
```

经过7，8，9，10这几个步骤便执行完ActivityManagerService的startProcessLocked()方法的Process.start这几行代码并获取了执行结果。

Process.start()将参数发送给ServerSocket。ServerSocket收到连接便创建了ZygoteConnection对象。下面开始分析ZygoteConnection。

### ZygoteConnection获取参数和fork进程
/android-6.0.0_r1/frameworks/base/core/java/com/android/internal/os/ZygoteConnection.java

11，ZygoteConnection构造方法：

```java
    ZygoteConnection(LocalSocket socket, String abiList) throws IOException {
        mSocket = socket;
        //ZygoteInit支持的abi列表，即手机支持的abi
        this.abiList = abiList;
        //输出流
        mSocketOutStream
                = new DataOutputStream(socket.getOutputStream());
        //输入流
        mSocketReader = new BufferedReader(
                new InputStreamReader(socket.getInputStream()), 256);
        //设置超时时间
        mSocket.setSoTimeout(CONNECTION_TIMEOUT_MILLIS);

        try {
            peer = mSocket.getPeerCredentials();
        } catch (IOException ex) {
            Log.e(TAG, "Cannot read peer credentials", ex);
            throw ex;
        }
    }
```
主要是获取系统支持的abi，和创建Socket的输入输出流。

12，runOnce()方法

在ZygoteInit的runSelectLoop方法中，有调用到ZygoteConnection的runOnce()方法。
现在分析runOnce()方法的执行流程。

```java
    boolean runOnce() throws ZygoteInit.MethodAndArgsCaller {

        String args[];
        Arguments parsedArgs = null;
        FileDescriptor[] descriptors;

        try {
            //从ServerSocket获取的socket连接输入流读取参数信息
            args = readArgumentList();
            //获取附属的文件描述符（输入输出流）
            descriptors = mSocket.getAncillaryFileDescriptors();
        } catch (IOException ex) {
            Log.w(TAG, "IOException on command socket " + ex.getMessage());
            closeSocket();
            return true;
        }

        //如果无参数则关闭连接，并返回true，
        if (args == null) {
            // EOF reached.
            closeSocket();
            return true;
        }

        /** the stderr of the most recent request, if avail */
        PrintStream newStderr = null;

        if (descriptors != null && descriptors.length >= 3) {
            //设置错误信息输出流
            newStderr = new PrintStream(
                    new FileOutputStream(descriptors[2]));
        }

        int pid = -1;
        FileDescriptor childPipeFd = null;
        FileDescriptor serverPipeFd = null;

        try {
            //解析参数数组args，并赋值给Arguments的属性
            parsedArgs = new Arguments(args);

            //获取abi列表
            if (parsedArgs.abiListQuery) {
                return handleAbiListQuery();
            }

            if (parsedArgs.permittedCapabilities != 0 || parsedArgs.effectiveCapabilities != 0) {
                throw new ZygoteSecurityException("Client may not specify capabilities: " +
                        "permitted=0x" + Long.toHexString(parsedArgs.permittedCapabilities) +
                        ", effective=0x" + Long.toHexString(parsedArgs.effectiveCapabilities));
            }

            ...

            int[][] rlimits = null;

            if (parsedArgs.rlimits != null) {
                rlimits = parsedArgs.rlimits.toArray(intArray2d);
            }

            //此次创建进程，未传入的invokeWith
            if (parsedArgs.invokeWith != null) {
                FileDescriptor[] pipeFds = Os.pipe2(O_CLOEXEC);
                childPipeFd = pipeFds[1];
                serverPipeFd = pipeFds[0];
                Os.fcntlInt(childPipeFd, F_SETFD, 0);
            }

            /**
             * In order to avoid leaking descriptors to the Zygote child,
             * the native code must close the two Zygote socket descriptors
             * in the child process before it switches from Zygote-root to
             * the UID and privileges of the application being launched.
             *
             * In order to avoid "bad file descriptor" errors when the
             * two LocalSocket objects are closed, the Posix file
             * descriptors are released via a dup2() call which closes
             * the socket and substitutes an open descriptor to /dev/null.
             */

            int [] fdsToClose = { -1, -1 };

            FileDescriptor fd = mSocket.getFileDescriptor();

            if (fd != null) {
                fdsToClose[0] = fd.getInt$();
            }

            fd = ZygoteInit.getServerSocketFileDescriptor();

            if (fd != null) {
                fdsToClose[1] = fd.getInt$();
            }

            fd = null;

            //执行fork操作
            pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, parsedArgs.gids,
                    parsedArgs.debugFlags, rlimits, parsedArgs.mountExternal, parsedArgs.seInfo,
                    parsedArgs.niceName, fdsToClose, parsedArgs.instructionSet,
                    parsedArgs.appDataDir);
        } catch (ErrnoException ex) {
            logAndPrintError(newStderr, "Exception creating pipe", ex);
        } catch (IllegalArgumentException ex) {
            logAndPrintError(newStderr, "Invalid zygote arguments", ex);
        } catch (ZygoteSecurityException ex) {
            logAndPrintError(newStderr,
                    "Zygote security policy prevents request: ", ex);
        }

        try {
            //刚fork出来的进程pid为0，如果pid<0则创建进程过程出了错误
            if (pid == 0) {
                // in child
                IoUtils.closeQuietly(serverPipeFd);
                serverPipeFd = null;
                //处理复制的子进程
                handleChildProc(parsedArgs, descriptors, childPipeFd, newStderr);

                // should never get here, the child is expected to either
                // throw ZygoteInit.MethodAndArgsCaller or exec().
                return true;
            } else {
                // in parent...pid of < 0 means failure
                IoUtils.closeQuietly(childPipeFd);
                childPipeFd = null;
                return handleParentProc(pid, descriptors, serverPipeFd, parsedArgs);
            }
        } finally {
            IoUtils.closeQuietly(childPipeFd);
            IoUtils.closeQuietly(serverPipeFd);
        }
    }
```
Zygote.forkAndSpecialize()方法调用的是Zygote类中的native方法nativeForkAndSpecialize()。它来fork Zygote进程来创建虚拟机进程。

Zygote的forkAndSpecialize方法代码注释上对返回值的说明：
> 0 if this is the child, pid of the child if this is the parent, or -1 on error.

返回0为复制Zygote成功，-1为失败。

13，handleChildProc()方法：

```java
    private void handleChildProc(Arguments parsedArgs,
            FileDescriptor[] descriptors, FileDescriptor pipeFd, PrintStream newStderr)
            throws ZygoteInit.MethodAndArgsCaller {
        /**
         * By the time we get here, the native code has closed the two actual Zygote
         * socket connections, and substituted /dev/null in their place.  The LocalSocket
         * objects still need to be closed properly.
         */

        closeSocket();//关闭Socket连接
        ZygoteInit.closeServerSocket();//关闭fork出来的进程的ServerSocket连接

        if (descriptors != null) {
            try {
                Os.dup2(descriptors[0], STDIN_FILENO);//输入
                Os.dup2(descriptors[1], STDOUT_FILENO);//输出
                Os.dup2(descriptors[2], STDERR_FILENO);//错误打印

                for (FileDescriptor fd: descriptors) {
                    IoUtils.closeQuietly(fd);//关闭
                }
                //更改错误输出为System.err
                newStderr = System.err;
            } catch (ErrnoException ex) {
                Log.e(TAG, "Error reopening stdio", ex);
            }
        }

        //设置进程名称
        if (parsedArgs.niceName != null) {
            Process.setArgV0(parsedArgs.niceName);
        }

        ...
        if (parsedArgs.invokeWith != null) {
            WrapperInit.execApplication(parsedArgs.invokeWith,
                    parsedArgs.niceName, parsedArgs.targetSdkVersion,
                    VMRuntime.getCurrentInstructionSet(),
                    pipeFd, parsedArgs.remainingArgs);
        } else {
            RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion,
                    parsedArgs.remainingArgs, null /* classLoader */);
        }
    }
```

进程fork成功后，便关闭了Socket连接，同时也关闭了Zygote的ServerSocket。因为未传入invokeWith参数，所以最后调用RuntimeInit.zygoteInit()方法。

### 应用进程初始化
/android-6.0.0_r1/frameworks/base/core/java/com/android/internal/os/RuntimeInit.java
14，RuntimeInit.zygoteInit()方法代码：

```java
    public static final void zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
        ...
        //重新设置日志流
        redirectLogStreams();
        //对通用的根据进行初始化或重置，如：TimeZone，LogManager，userAgent和NetworkManagementSocketTagger安装等待。
        commonInit();
        //native方法，对创建的进程初始化
        nativeZygoteInit();
        //应用初始化（注：不是创建和初始化Application），对VMRuntime进行设置
        applicationInit(targetSdkVersion, argv, classLoader);
    }
```
15，applicationInit()方法代码：

```java
    private static void applicationInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
        // If the application calls System.exit(), terminate the process
        // immediately without running any shutdown hooks.  It is not possible to
        // shutdown an Android application gracefully.  Among other things, the
        // Android runtime shutdown hooks close the Binder driver, which can cause
        // leftover running threads to crash before the process actually exits.
        nativeSetExitWithoutCleanup(true);

        // We want to be fairly aggressive about heap utilization, to avoid
        // holding on to a lot of memory that isn't needed.
        VMRuntime.getRuntime().setTargetHeapUtilization(0.75f);
        VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);

        final Arguments args;
        try {
            //获取ActivityService的startProcessLocked()方法中设置的参数：
            //"android.app.ActivityThread"
            args = new Arguments(argv);
        } catch (IllegalArgumentException ex) {
            Slog.e(TAG, ex.getMessage());
            // let the process exit
            return;
        }

        // The end of of the RuntimeInit event (see #zygoteInit).
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);

        // Remaining arguments are passed to the start class's static main
        invokeStaticMain(args.startClass, args.startArgs, classLoader);
    }
```
设置了VMRuntime堆内存比例为0.75，设置了SDK版本，对argv进行解析和封装。最后调用invokeStaticMain()方法启动ActvityThread。

15，invokeStaticMain()方法代码：

```java
    private static void invokeStaticMain(String className, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
        Class<?> cl;

        try {
            //通过反射获取ActivityThread的class
            cl = Class.forName(className, true, classLoader);
        } catch (ClassNotFoundException ex) {
            throw new RuntimeException(
                    "Missing class when invoking static main " + className,
                    ex);
        }

        Method m;
        try {
            m = cl.getMethod("main", new Class[] { String[].class });
        } catch (NoSuchMethodException ex) {
            throw new RuntimeException(
                    "Missing static main on " + className, ex);
        } catch (SecurityException ex) {
            throw new RuntimeException(
                    "Problem getting static main on " + className, ex);
        }

        //校验main方法的修饰符是否为static和public
        int modifiers = m.getModifiers();
        if (! (Modifier.isStatic(modifiers) && Modifier.isPublic(modifiers))) {
            throw new RuntimeException(
                    "Main method is not public and static on " + className);
        }

        /*
         * This throw gets caught in ZygoteInit.main(), which responds
         * by invoking the exception's run() method. This arrangement
         * clears up all the stack frames that were required in setting
         * up the process.
         */
        throw new ZygoteInit.MethodAndArgsCaller(m, argv);
    }
```

### 应用进程创建ActivityThread
ZygoteInit.MethodAndArgsCaller类为Zygote的内部类，代码：

```java
    public static class MethodAndArgsCaller extends Exception
            implements Runnable {
        /** method to call */
        private final Method mMethod;

        /** argument array */
        private final String[] mArgs;

        public MethodAndArgsCaller(Method method, String[] args) {
            mMethod = method;
            mArgs = args;
        }

        public void run() {
            try {
                mMethod.invoke(null, new Object[] { mArgs });
            } catch (IllegalAccessException ex) {
                throw new RuntimeException(ex);
            } catch (InvocationTargetException ex) {
                Throwable cause = ex.getCause();
                if (cause instanceof RuntimeException) {
                    throw (RuntimeException) cause;
                } else if (cause instanceof Error) {
                    throw (Error) cause;
                }
                throw new RuntimeException(ex);
            }
        }
    }
```

该异常在ZygoteInit的main方法中被捕获，代码：

```java
    public static void main(String argv[]) {
        try {
            ...
        } catch (MethodAndArgsCaller caller) {
            caller.run();
        } catch (RuntimeException ex) {
           ... 
        }
    }
```
于是执行了ActivityThread的main方法。

/android-6.0.0_r1/frameworks/base/core/java/android/app/ActivityThread.java

16，继续看ActivityThread的main方法：

```java
    public static void main(String[] args) {
        ...

        //创建sMainLooper
        Looper.prepareMainLooper();
        // 创建ActivityThread
        ActivityThread thread = new ActivityThread();
        //执行attach操作
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        // End of event ActivityThreadMain.
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        //无限循环获取消息
        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```
Zygote的main方法中执行了ActivityThread的main方法，而ActivityThread的main方法中`Looper.loop();`会无限循环。如果进程不被杀掉，Zygote和ActivityThread会一直运行。

在看看ActivityThread类的基本代码：

```java
public final class ActivityThread {

    ...
    final ApplicationThread mAppThread = new ApplicationThread();
    final H mH = new H();
    ...
    //内部有对四大组件的操作方法
    ...
    
    private class ApplicationThread extends ApplicationThreadNative {
        ...
        //调用mH发送操作四大组件的消息
    }

    private class H extends Handler {
        ...
        public void handleMessage(Message msg) {
           ...
           //根据消息类别，调用ActivityThread中对应操作方法来处理
        }
    }
}
```

17，再回过头来看看Actvity的attach()方法：

```java
    private void attach(boolean system) {
        sCurrentActivityThread = this;
        mSystemThread = system;
        if (!system) {
            ...
            
            RuntimeInit.setApplicationObject(mAppThread.asBinder());
            //获取ActivityMangerService的IBinder对象
            final IActivityManager mgr = ActivityManagerNative.getDefault();
            try {
                //将应用的ApplicationThread对象设置给ActivityMangerService
                mgr.attachApplication(mAppThread);
            } catch (RemoteException ex) {
                // Ignore
            }
            // Watch for getting close to heap limit.
            BinderInternal.addGcWatcher(new Runnable() {
                @Override public void run() {
                    if (!mSomeActivitiesChanged) {
                        return;
                    }
                    Runtime runtime = Runtime.getRuntime();
                    long dalvikMax = runtime.maxMemory();
                    long dalvikUsed = runtime.totalMemory() - runtime.freeMemory();
                    if (dalvikUsed > ((3*dalvikMax)/4)) {
                        if (DEBUG_MEMORY_TRIM) Slog.d(TAG, "Dalvik max=" + (dalvikMax/1024)
                                + " total=" + (runtime.totalMemory()/1024)
                                + " used=" + (dalvikUsed/1024));
                        mSomeActivitiesChanged = false;
                        try {
                            //VM使用的内存超过分配内存的3/4，则释放一些Activity
                            mgr.releaseSomeActivities(mAppThread);
                        } catch (RemoteException e) {
                        }
                    }
                }
            });
        } else {
            ...// 省略系统设置
        }

        ...
    }
```
ActivityThread对象被创建后，便将ApplicationThread对象（mAppThread）的IBinder传递给ActivityMangerService。ApplicationThread可以向H发送消息，H可以调用ActivityThread的方法来执行相应的操作。虽然ActivityMangerService与ActivityThread无任何直接或间接关联，但是在它获得IBinder之后，便可以通过IBinder来调用ApplicationThread，从而间接地调用ActivityThread的方法。

### 应用进程创建Application

18，`ActivityManagerNative.getDefault().attachApplication(mAppThread);`这行代码调用的其实是ActivityManagerProxy.attachApplication()方法。

/android-6.0.0_r1/frameworks/base/core/java/android/app/ActivityManagerNative.java

```java
    static public IActivityManager asInterface(IBinder obj) {
        if (obj == null) {
            return null;
        }
        IActivityManager in =
            (IActivityManager)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }
        
        //将获取的IBinder赋值给ActivityManagerProxy的mRemote
        return new ActivityManagerProxy(obj);
    }

    private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
            IBinder b = ServiceManager.getService("activity");
            
            IActivityManager am = asInterface(b);
            
            return am;
        }
    };
```
attachApplication()方法

```java
    public void attachApplication(IApplicationThread app) throws RemoteException
    {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        //写入IBinder
        data.writeStrongBinder(app.asBinder());
        mRemote.transact(ATTACH_APPLICATION_TRANSACTION, data, reply, 0);
        reply.readException();
        data.recycle();
        reply.recycle();
    }
```
使用mRemote向ActivityManagerService传递信息。Parcel为存储数据的buffer，data用来写入的数据，reply用来接收ActivityManagerService回应的信息。通过mRemote作为桥梁，ActivityThread便将mAppThread的IBinder传递给ActivityManagerService。ActivityManagerService的onTransact()方法会调用ApplicationThreadNative的asInterface()将mAppThread封装到ApplicationThreadProxy对象，并将该对象作为attachApplication()的参数。因为mAppThread属于创建的应用进程，而ActivityManagerService则在服务进程，所以ActivityManagerService只能通过IBinder来调用mAppThread。

19，继续看ActivityManagerService的attachApplication()方法中的代码

```java
        synchronized (this) {
            //调用Binder的进程id
            int callingPid = Binder.getCallingPid();
            final long origId = Binder.clearCallingIdentity();
            //真正执行赋值的方法
            attachApplicationLocked(thread, callingPid);
            Binder.restoreCallingIdentity(origId);
        }
```

attachApplicationLocked()方法

```java
    private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {

        ...
        try {
            ...
            //创建和绑定应用的Application
            thread.bindApplication(processName, appInfo, providers, app.instrumentationClass,
                    profilerInfo, app.instrumentationArguments, app.instrumentationWatcher,
                    app.instrumentationUiAutomationConnection, testMode, enableOpenGlTrace,
                    isRestrictedBackupMode || !normalMode, app.persistent,
                    new Configuration(mConfiguration), app.compat,
                    getCommonServicesLocked(app.isolated),
                    mCoreSettingsObserver.getCoreSettingsLocked());
            updateLruProcessLocked(app, false, null);
            app.lastRequestedGc = app.lastLowMemory = SystemClock.uptimeMillis();
        } catch (Exception e) {
            …
            return false;
        }

        ...
    }
```
thread.bindApplication()调用的是ApplicationThreadProxy的bindApplication()方法。

ApplicationThreadProxy的bindApplication()方法：

/android-6.0.0_r1/frameworks/base/core/java/android/app/ApplicationThreadNative.java

```java
    public final void bindApplication(String packageName, ApplicationInfo info,
            List<ProviderInfo> providers, ComponentName testName, ProfilerInfo profilerInfo,
            Bundle testArgs, IInstrumentationWatcher testWatcher,
            IUiAutomationConnection uiAutomationConnection, int debugMode,
            boolean openGlTrace, boolean restrictedBackupMode, boolean persistent,
            Configuration config, CompatibilityInfo compatInfo, Map<String, IBinder> services,
            Bundle coreSettings) throws RemoteException {
        Parcel data = Parcel.obtain();
        ...
        mRemote.transact(BIND_APPLICATION_TRANSACTION, data, null,
                IBinder.FLAG_ONEWAY);
        data.recycle();
    }
```

先将设置的参数写入到data，然后mRemote.transact()提交code，data和flags。mRemote.transact()是ApplicationThreadNative.transact()方法。

再看看ApplicationThreadNative.transact()方法相关代码片段：

```java
        case BIND_APPLICATION_TRANSACTION:
        {
            data.enforceInterface(IApplicationThread.descriptor);
            //读取data中的数据
            ...
            bindApplication(packageName, info, providers, testName, profilerInfo, testArgs,
                    testWatcher, uiAutomationConnection, testMode, openGlTrace,
                    restrictedBackupMode, persistent, config, compatInfo, services, coreSettings);
            return true;
        }
```
通过IBinder写入和获取数据，最终会调用到ApplicationThread的bindApplication()。而bindApplication()方法被调用时，该方法又会调用mH发送一个BIND_APPLICATION的消息和传递接收到的数据。

20，ApplicationThread的bindApplication()方法

```java
        public final void bindApplication(String processName, ApplicationInfo appInfo,
                List<ProviderInfo> providers, ComponentName instrumentationName,
                ProfilerInfo profilerInfo, Bundle instrumentationArgs,
                IInstrumentationWatcher instrumentationWatcher,
                IUiAutomationConnection instrumentationUiConnection, int debugMode,
                boolean enableOpenGlTrace, boolean isRestrictedBackupMode, boolean persistent,
                Configuration config, CompatibilityInfo compatInfo, Map<String, IBinder> services,
                Bundle coreSettings) {

            ...

            AppBindData data = new AppBindData();
            ...
            sendMessage(H.BIND_APPLICATION, data);
        }

    private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
        ...
        Message msg = Message.obtain();
        ...
        mH.sendMessage(msg);
    }
```

H的handleMessage()方法会处理该消息的代码片段：

```java
                case BIND_APPLICATION:
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "bindApplication");
                    AppBindData data = (AppBindData)msg.obj;
                    handleBindApplication(data);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    break;
```
mH调用了ActivityThread的handleBindApplication()来处理。

21，ActivityThread的handleBindApplication()方法代码

```java
    private void handleBindApplication(AppBindData data) {
        
        ...

        try {
            // If the app is being launched for full backup or restore, bring it up in
            // a restricted environment with the base application class.
            //创建Application对象
            Application app = data.info.makeApplication(data.restrictedBackupMode, null);
            //赋值到ActivityThread对象的属性
            mInitialApplication = app;

            ...

            // Do this after providers, since instrumentation tests generally start their
            // test thread at this point, and we don't want that racing.
            try {
                mInstrumentation.onCreate(data.instrumentationArgs);
            }
            catch (Exception e) {
                throw new RuntimeException(
                    "Exception thrown in onCreate() of "
                    + data.instrumentationName + ": " + e.toString(), e);
            }

            try {
                //调用Application或子类的onCreate()方法
                mInstrumentation.callApplicationOnCreate(app);
            } catch (Exception e) {
                ...
            }
        } finally {
            ...
        }
    }
```

`data.info`为LoadApk对象，callApplicationOnCreate()方法中调用了Instrumentation的newApplication()方法创建Application对象。

LoadApk的callApplicationOnCreate()方法

/android-6.0.0_r1/frameworks/base/core/java/android/app/LoadedApk.java

```java
   public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
        if (mApplication != null) {
            return mApplication;
        }

        Application app = null;

        String appClass = mApplicationInfo.className;
        //如果为强制默认或者Manifest中未设置，则使用默认
        if (forceDefaultAppClass || (appClass == null)) {
            appClass = "android.app.Application";
        }

        try {
            ...
            ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
            app = mActivityThread.mInstrumentation.newApplication(
                    cl, appClass, appContext);
            appContext.setOuterContext(app);
        } catch (Exception e) {
            
        }
        mActivityThread.mAllApplications.add(app);
        mApplication = app;

        ...

        return app;
    }
```
mInstrumentation为Instrumentation类型，方法中调用了Instrumentation的newApplication()方法通过反射来创建Application或子类对象。

Instrumentation的newApplication()方法

/android-6.0.0_r1/frameworks/base/core/java/android/app/Instrumentation.java


```java
    public Application newApplication(ClassLoader cl, String className, Context context)
            throws InstantiationException, IllegalAccessException, 
            ClassNotFoundException {
        return newApplication(cl.loadClass(className), context);
    }

    static public Application newApplication(Class<?> clazz, Context context)
            throws InstantiationException, IllegalAccessException, 
            ClassNotFoundException {
        Application app = (Application)clazz.newInstance();
        app.attach(context);
        return app;
    }
```
在方法内通过反射创建了对象，并调用attach()给Application或子类对象设置了Context。

19，20，21，22这三个步骤是应用进程创建Application和初始化的过程。



## 分析总结

总结一下流程：

  1，开机，系统会开启Zygote进程，该进程会开启ServerSocket，并无限循环等待和处理连接的Socket。
  
  2，调用startActivity，最终会调用到ActivityMangerService的startProcessLocked()方法来创建进程。该方法调用了Process.start()方法。
  
  3，Process.start()方法会调用创建Socket并连接到ServerSocket，向Server写入startProcessLocked()方法设置的参数，写完便关闭连接。
  
  4，ServerSocket获取到Socket连接并封装到ZygoteConnection对象。ZygoteConnection获取client发送过来的参数，对参数进行解析和封装，调用Zygote.ZygoteConnection()来fork虚拟机进程。进程fork成功后，关闭连接，同时关闭fork出来的进程的ServerSocket。然后进行初始化和通过反射调用ActivityThread。
  
  5，ActivityThread的main方法被调用，创建了ActivityThread对象，运行Looper，并将ApplicationThread对象传递给ActivityMangerService。ActivityMangerService接收到ApplicationThread后，会将其封装到ApplicationThreadProxy对象，至此ActivityMangerService便能通过ApplicationThreadProxy对象来调用ApplicationThread，从而间接地调用ActivityThread的方法来操作四大组件。
  
  6，ActivityMangerService在收到新创建的应用进程传递过来的ApplicationThread的IBinder之后，便会通过其代理对象调用ApplicationThread方法向H发送创建Application消息。H在收到创建消息后便调用ActivityThread的方法来执行Application创建和初始化。执行完这些执行操作，新的App进程也就创建和设置成功了。
  



----------


参考博客：

Android系统进程Zygote启动过程的源代码分析
http://blog.csdn.net/luoshengyang/article/details/6768304

Android应用程序启动过程源代码分析
http://blog.csdn.net/luoshengyang/article/details/6689748


