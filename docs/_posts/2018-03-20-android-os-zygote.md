---
layout: post
title: "zygote分析"
---
zygote，作为系统启动后打开java世界的第一个进程，正如它的名字一样，java层其他的进程都是由它分裂出来。

android系统大致可以划分为四层，最底层的是linux内核，主要是跟硬件打交道的驱动;往上是native层，主要是各种库和bin文件，
包括虚拟机、surface等;再往上是framework层，包括SystemServer、ActivityManagerServer等，构成java世界的运行机制;
最顶层就是application层，包括各种apk了，拨打电话的dialer、home界面的launcher等等。

linux内核和native是c/c++实现的，framework和applicant则基本上都是java实现。

手机通电后加载引导程序，然后拉起linux内核，内核启动的第一个用户进程是init进程，然后fork出子进程zygote，
此后zygote分裂出了首个子进程system_server，系统中所有的service都是在进程system_server中被加载起来，
包括常见的ActivityManagerService、WindowManagerService等，整个过程可以清晰的分为三个部分，
用[下图]({{ site.url }}/android/images/android-os-zygote/zygote.png){:target="_blank"}表示：
![]({{ site.url }}/android/images/android-os-zygote/zygote.png)


系统源码版本是Android 4.4.4_r1 - KitKat.

## [](#header-2) &diams; linux内核
init进程起来后会加载配置文件init.rc，以便启动zygote进程，zygote进程的配置参数如下：
```
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
    class main
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart media
    onrestart restart netd
```
主要参数的意义是zygote进程是通过执行/system/bin/目录下面的可执行文件app_process来启动，
参数有两个`--zygote --start-system-server`。另外还打开一个名为zygote的socket，
当zygote进程异常退出时，向socket写数据，init进程收到数据则知道要重启zygote了。


## [](#header-2) &diams; zygote进程
init进程fork出子进程后，首先调用main函数，在这里把进程名改为zygote，传入的参数即是init.rc中配置的参数:
* `argv[0] = zygote`
* `argv[1] = start-system-server`

相关的实现代码和注释如下，android系统的源码数量巨大，每次分析必须抓住重点，只跟踪相关的代码，
其他的省略，否则会陷入无底深渊，迷失在茫茫的代码中。

```c++
// /frameworks/base/cmds/app_process/app_main.cpp
int main(int argc, char* const argv[])
{
 .....//省略其他代码

    while (i < argc) {
        const char* arg = argv[i++];
        
 .....//省略其他代码

	 else if (strcmp(arg, "--zygote") == 0) {//两个参数中的第一个
            zygote = true;
            niceName = "zygote";
        } else if (strcmp(arg, "--start-system-server") == 0) {//两个参数中的第二个
            startSystemServer = true;
        } 

 .....//省略其他代码
    }

 .....//省略其他代码

    if (niceName && *niceName) {
        setArgv0(argv0, niceName);
        set_process_name(niceName);//修改进程名为zygote
    }

 .....//省略其他代码

    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit",
                startSystemServer ? "start-system-server" : "");//startSystemServer为true，
								//传入字符串"start-system-server"
    }

 .....//省略其他代码
}
 
```


接着进入start函数，传入的参数为:
* `className = com.android.internal.os.ZygoteInit`
* `options = start-system-server`

```c++
// /frameworks/base/core/jni/AndroidRuntime.cpp
void AndroidRuntime::start(const char* className, const char* options)
{
 .....//省略其他代码

    /* start the virtual machine */
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;
    if (startVm(&mJavaVM, &env) != 0) {//启动虚拟机
        return;
    }
    onVmCreated(env);//空函数

    /*
     * Register android functions.
     */
    if (startReg(env) < 0) {//注册android的jni函数
        ALOGE("Unable to register all android natives\n");
        return;
    }

 .....//省略其他代码


    char* slashClassName = toSlashClassName(className);
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
            env->CallStaticVoidMethod(startClass, startMeth, strArray);//调用ZygoteInit.java的main函数

 .....//省略其他代码
        }
    }

 .....//省略其他代码

}

```

通过jni进入java层，调用ZygoteInit的main函数，传入的参数为:
* `argv[0] = com.android.internal.os.ZygoteInit`
* `argv[1] = start-system-server`

```java
// /frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
    public static void main(String argv[]) {

        try {
 .....//省略其他代码

            registerZygoteSocket();//①注册socket

 .....//省略其他代码

            if (argv[1].equals("start-system-server")) {
                startSystemServer();//②启动system server
            } 

 .....//省略其他代码

            runSelectLoop();//上面注册的socket进入loop

 .....//省略其他代码

        } catch (MethodAndArgsCaller caller) {//③捕获异常
            caller.run();
        } 

 .....//省略其他代码

    }

```

①注册socket
`ANDROID_SOCKET_ENV = "ANDROID_SOCKET_zygote"`就是指init.rc中定义的socket，在init中把这个socket通过函数`public_socket`
向系统发布，此后通过ANDROID_SOCKET_ENV来连接名为zygote的socket  
这里创建了ServerSocket用于接收fork进程的请求，在framework和applicant层，只有通过这个socket向zygote发请求才能fork出子进程，
zygote的名字就在这里体现出来了。

```java
// /frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
    private static void registerZygoteSocket() {
        if (sServerSocket == null) {
            int fileDesc;
            try {
                String env = System.getenv(ANDROID_SOCKET_ENV);
                fileDesc = Integer.parseInt(env);
            } 

 .....//省略其他代码

            try {
                sServerSocket = new LocalServerSocket(
                        createFileDescriptor(fileDesc));
            } 

 .....//省略其他代码

        }
    }
```

②启动system server

```java
    private static boolean startSystemServer()
            throws MethodAndArgsCaller, RuntimeException {

 .....//省略其他代码

        /* Hardcoded command line to start the system server */
        String args[] = {
            "--setuid=1000",
            "--setgid=1000",
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1032,3001,3002,3003,3006,3007",
            "--capabilities=" + capabilities + "," + capabilities,
            "--runtime-init",
            "--nice-name=system_server",//fork出新进程后修改进程名为nice-name的值
            "com.android.server.SystemServer",//新进程调用的java代码文件
        };
        ZygoteConnection.Arguments parsedArgs = null;

        int pid;

        try {
            parsedArgs = new ZygoteConnection.Arguments(args);//把字符串转换成Arguments类型
            ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
            ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);

            /* Request to fork the system server process */
            pid = Zygote.forkSystemServer(//fork进程
                    parsedArgs.uid, parsedArgs.gid,
                    parsedArgs.gids,
                    parsedArgs.debugFlags,
                    null,
                    parsedArgs.permittedCapabilities,
                    parsedArgs.effectiveCapabilities);
        } catch (IllegalArgumentException ex) {
            throw new RuntimeException(ex);
        }

        /* For child process */
        if (pid == 0) {
            handleSystemServerProcess(parsedArgs);//在system_server进程运行了
        }

        return true;
    }
```

fork出system server进程
```java
// /libcore/dalvik/src/main/java/dalvik/system/Zygote.java
    public static int forkSystemServer(int uid, int gid, int[] gids, int debugFlags,
            int[][] rlimits, long permittedCapabilities, long effectiveCapabilities) {
        preFork();
        int pid = nativeForkSystemServer(//调用native层方法fork进程
                uid, gid, gids, debugFlags, rlimits, permittedCapabilities, effectiveCapabilities);
        postFork();
        return pid;
    }
```

nativeForkSystemServer定义和实现
```c++
// /dalvik/vm/native/dalvik_system_Zygote.cpp
const DalvikNativeMethod dvm_dalvik_system_Zygote[] = {

 .....//省略其他代码

    { "nativeForkSystemServer", "(II[II[[IJJ)I",
      Dalvik_dalvik_system_Zygote_forkSystemServer },
    { NULL, NULL, NULL },
};

static void Dalvik_dalvik_system_Zygote_forkSystemServer(
        const u4* args, JValue* pResult)
{
    pid_t pid;
    pid = forkAndSpecializeCommon(args, true);

 .....//省略其他代码
}

static pid_t forkAndSpecializeCommon(const u4* args, bool isSystemServer)
{
 .....//省略其他代码

    pid = fork();

    if (pid == 0) {//设置进程的各种参数等

 .....//省略其他代码
    }

 .....//省略其他代码
}
```

## [](#header-2) &diams; system_server进程
这里是进入到前面fork出的子进程。
```java

    private static void handleSystemServerProcess(
            ZygoteConnection.Arguments parsedArgs)
            throws ZygoteInit.MethodAndArgsCaller {

        closeServerSocket();//关闭ANDROID_SOCKET_zygote，这样只有zygote进程才能fork出子进程

        // set umask to 0077 so new files and directories will default to owner-only permissions.
        Libcore.os.umask(S_IRWXG | S_IRWXO);//这里限制了进程的权限，不然就是zygote的权限那就是root了

        if (parsedArgs.niceName != null) {
            Process.setArgV0(parsedArgs.niceName);//设置进程名为前面定义的"--nice-name=system_server"
        }

        if (parsedArgs.invokeWith != null) {
            WrapperInit.execApplication(parsedArgs.invokeWith,
                    parsedArgs.niceName, parsedArgs.targetSdkVersion,
                    null, parsedArgs.remainingArgs);
        } else {
            /*
             * Pass the remaining arguments to SystemServer.
             */

            //打个log看一下传入zygoteInit的参数
            Log.d(TAG, "+++++ parsedArgs.targetSdkVersion:" + parsedArgs.targetSdkVersion);
            for (int i = 0; i < parsedArgs.remainingArgs.length; ++i) {
                Log.d(TAG, "+++++ parsedArgs.remainingArgs[" + i + "]:" + parsedArgs.remainingArgs[i]);
            }

            RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs);
        }

        /* should never reach here */
    }
```
![]({{ site.url }}/android/images/android-os-zygote/zygoteInit_params.png)
传入的参数如[上图]({{ site.url }}/android/images/android-os-zygote/zygoteInit_params.png){:target="_blank"}所示，
只有最初定义的"com.android.server.SystemServer"
这里把进程名设为system_server，但在ActivityManagerServer创建的时候，通过下面这句函数把system_server改成"system_process",
在调试的时候看到的system_process进程即是system_server，只是被换个名字而已。
```java
            android.ddm.DdmHandleAppName.setAppName("system_process",
                                                    UserHandle.myUserId());
```

```java

    public static final void zygoteInit(int targetSdkVersion, String[] argv)
            throws ZygoteInit.MethodAndArgsCaller {

 .....//省略其他代码

        nativeZygoteInit();//在这里创建了binder的native层

        applicationInit(targetSdkVersion, argv);
    }

    private static void applicationInit(int targetSdkVersion, String[] argv)
            throws ZygoteInit.MethodAndArgsCaller {

 .....//省略其他代码

	//args.startClass = "com.android.server.SystemServer"
	//args.startArgs = ""

        // Remaining arguments are passed to the start class's static main
        invokeStaticMain(args.startClass, args.startArgs);
    }

    private static void invokeStaticMain(String className, String[] argv)
            throws ZygoteInit.MethodAndArgsCaller {
 
 .....//省略其他代码

        try {
            cl = Class.forName(className);
        }  

.....//省略其他代码

        Method m;
        try {
            m = cl.getMethod("main", new Class[] { String[].class });
        } 

 .....//省略其他代码

        /*
         * This throw gets caught in ZygoteInit.main(), which responds
         * by invoking the exception's run() method. This arrangement
         * clears up all the stack frames that were required in setting
         * up the process.
         */
        throw new ZygoteInit.MethodAndArgsCaller(m, argv);//抛出异常了
    }
```

③捕获异常
```java

    public static void main(String argv[]) {

 .....//省略其他代码

         catch (MethodAndArgsCaller caller) {
            caller.run();

        } 
 .....//省略其他代码

    }

    public static class MethodAndArgsCaller extends Exception
            implements Runnable {

 .....//省略其他代码

        public MethodAndArgsCaller(Method method, String[] args) {
            mMethod = method;
            mArgs = args;
        }

        public void run() {
            try {
                mMethod.invoke(null, new Object[] { mArgs });
            }  

 .....//省略其他代码

        }
    }

```
最终用反射的方法调用了com.android.server.SystemServer中的main函数，
在新进程里启动了SystemServer

```java

    public static void main(String[] args) {

 .....//省略其他代码

        // This used to be its own separate thread, but now it is
        // just the loop we run on the main thread.
        ServerThread thr = new ServerThread();
        thr.initAndLoop();
    }

    public void initAndLoop() {

 .....//省略其他代码

	//都是类似的形式注册各种service

        try {
            Slog.i(TAG, "Display Manager");
            display = new DisplayManagerService(context, wmHandler);
            ServiceManager.addService(Context.DISPLAY_SERVICE, display, true);

 .....//省略其他代码
	}

 .....//省略其他代码

   }
```

至此加载完各种service就把android系统的框架搭起来了，然后就是安装一系列的apk，
包括顶部状态栏是systemui，home界面是launcher，系统设置是setting等等。


android源码代码量巨大，阅读过程中的笔记里，时序图是必不可少的，否则跳转完一圈最后不记得前面是在哪里了，
涉及到类的数量多的话,各种继承和关联的关系也错综复杂，所以类图也是不可少的。

前面分析过程的时序图如[下图]({{ site.url }}/android/images/android-os-zygote/SystemService.png){:target="_blank"}
![]({{ site.url }}/android/images/android-os-zygote/SystemService.png)


