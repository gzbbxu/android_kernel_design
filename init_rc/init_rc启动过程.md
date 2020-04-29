### 第一个系统进程

作为android 第一个被启动的进程，init 的pid 值为0；通过解析init.rc 脚本来构建系统的初始化运行状态。

纯文本编写，可读性高，开发商控制android 系统启动的一大利器。



#### init.rc 语法

与init.rc 解析相关的代码大部分集中在Init_parser.c 中

一个完整的init.rc脚本由4种类型的声明组成：

- Action(动作)
- Commands (命令)
- Service是（服务）
- Options (选项)





**Action**

动作的一般格式如下

on  <trigger> ##触发条件

​     <command1> ##执行命令

​    <command2> ## 可以执行多条命令



aciton 其实是一个响应过程。即当<trigger>所描述的触发实践产生时，会依次执行各种command

**Commands(命令)**

命令将在所属事件发生时被一个个地执行



**Service 服务**

Service 其实是可以执行程序，他们在特定的选项的约束下会被init 程序运行或者重启。

Service 可以在配置中指定是否需要退出时重启，这样当service 出现异常Crash 时就可以有机会复原。

一般格式：

​	services <name><pathname> [<argument>] *

​        <option>

​		<option>

**name**表示service 名称

**pahtname** 表示所在路径，是可执行文件

**argument** ,启动所需要的参数
**option**  service 约束。





### 关键服务启动解析

作为android 系统的第一个进程**，iniit 将通过解析init.rc 来陆续启动其他关键的系统服务进程**，

其中最重要的就是ServiceManager,Zygote 和SystemServer.



**ServiceManager** 是在Init.rc 里描述并由init 进程启动的。



```
/*system/core/rootdir/Init.rc*/

service servicemanager /system/bin/servicemanager

		class core

		user system

        group system
        
        critical
        
        onrestart restart healthd
        onrestart restart zygote
        onrestart restart media
        onrestart restart surfaceflinger
        onrestart restart drm

```



servicemanager 是一个linux程序。它在设备中存储的路径/system/bin/service-manager ,

源码路径则/frameworks/native/cmds/servicemanager.

servicemanager 所属class 是core,同类进程包含，ueventd,console,sdbd等。

根据core 特性。**这些进程会同时启动或者停止。另外critical 也说明它是系统关键进程**。

意味着如果进程不幸4分钟异常退出超过4次，则设备将重启进入还原模式。

当seriviceManager 重启时候，其他关键进程Zygote,edia,surfaceflinger **也会restart.**



### Zygote 

android 中大多数应用进程和系统进程都是通过Zygote 来生成的

和serviceManager 类似，Zygote 也是由init 解析rc脚本时启动的。Zygote 的启动需要根据不同的情况分别对待。（32位和64位）



根据系统属性**ro.zygote** 的具体值，加载不同描述的Zyote 的rc 脚本。

```java

import /init.environ.rc
import /system/etc/init/hw/init.usb.rc
import /init.${ro.hardware}.rc
import /vendor/etc/init/hw/init.${ro.hardware}.rc
import /system/etc/init/hw/init.usb.configfs.rc
import /system/etc/init/hw/init.${ro.zygote}.rc
```

init.zygote64.rc 为例

```java
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server
    class main
    priority -20
    user root
    group root readproc reserved_disk
    socket zygote stream 660 root system
    socket usap_pool_primary stream 660 root system
    onrestart write /sys/power/state on
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart netd
    onrestart restart wificond
    writepid /dev/cpuset/foreground/tasks


```

从上面的脚本可以看出

serviceName:zygote

paht:/system/bin/app_process

Argumnts: -Xzygote /system/bin --zygote -start-system-server



它所在的程序名叫“app_process64”,而不像ServiceManager 一样在一个独立的程序中。通过制定--zygote 参数，

app_process 可以识别出用户是否需要启动zygote. 那么，app_process 又是什么？

源码位于 /frameworks/base/cmds/app_process 。



#### app_process 

 /frameworks/base/cmds/app_process/App_main.cpp

```c++
int main(int argc, char* const argv[])
{
    if (!LOG_NDEBUG) {
      String8 argv_String;
      for (int i = 0; i < argc; ++i) {
        argv_String.append("\"");
        argv_String.append(argv[i]);
        argv_String.append("\" ");
      }
      ALOGV("app_process main with argv: %s", argv_String.string());
    }

    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
    // Process command line arguments
    // ignore argv[0]
    argc--;
    argv++;

    const char* spaced_commands[] = { "-cp", "-classpath" };
    // Allow "spaced commands" to be succeeded by exactly 1 argument (regardless of -s).
    bool known_command = false;

    int i;
    for (i = 0; i < argc; i++) {
        if (known_command == true) {
          runtime.addOption(strdup(argv[i]));
          // The static analyzer gets upset that we don't ever free the above
          // string. Since the allocation is from main, leaking it doesn't seem
          // problematic. NOLINTNEXTLINE
          ALOGV("app_process main add known option '%s'", argv[i]);
          known_command = false;
          continue;
        }

        for (int j = 0;
             j < static_cast<int>(sizeof(spaced_commands) / sizeof(spaced_commands[0]));
             ++j) {
          if (strcmp(argv[i], spaced_commands[j]) == 0) {
            known_command = true;
            ALOGV("app_process main found known command '%s'", argv[i]);
          }
        }

        if (argv[i][0] != '-') {
            break;
        }
        if (argv[i][1] == '-' && argv[i][2] == 0) {
            ++i; // Skip --.
            break;
        }

        runtime.addOption(strdup(argv[i]));
        // The static analyzer gets upset that we don't ever free the above
        // string. Since the allocation is from main, leaking it doesn't seem
        // problematic. NOLINTNEXTLINE
        ALOGV("app_process main add option '%s'", argv[i]);
    }

    // Parse runtime arguments.  Stop at first unrecognized option.
    bool zygote = false;
    bool startSystemServer = false;
    bool application = false;
    String8 niceName;
    String8 className;

    ++i;  // Skip unused "parent dir" argument.
    while (i < argc) {
        const char* arg = argv[i++];
        if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            niceName = ZYGOTE_NICE_NAME;
        } else if (strcmp(arg, "--start-system-server") == 0) {
            startSystemServer = true;
        } else if (strcmp(arg, "--application") == 0) {
            application = true;
        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
            niceName.setTo(arg + 12);
        } else if (strncmp(arg, "--", 2) != 0) {
            className.setTo(arg);
            break;
        } else {
            --i;
            break;
        }
    }

    Vector<String8> args;
    if (!className.isEmpty()) {
        // We're not in zygote mode, the only argument we need to pass
        // to RuntimeInit is the application argument.
        //
        // The Remainder of args get passed to startup class main(). Make
        // copies of them before we overwrite them with the process name.
        args.add(application ? String8("application") : String8("tool"));
        runtime.setClassNameAndArgs(className, argc - i, argv + i);

        if (!LOG_NDEBUG) {
          String8 restOfArgs;
          char* const* argv_new = argv + i;
          int argc_new = argc - i;
          for (int k = 0; k < argc_new; ++k) {
            restOfArgs.append("\"");
            restOfArgs.append(argv_new[k]);
            restOfArgs.append("\" ");
          }
          ALOGV("Class name = %s, args = %s", className.string(), restOfArgs.string());
        }
    } else {
        // We're in zygote mode.
        maybeCreateDalvikCache();

        if (startSystemServer) {
            args.add(String8("start-system-server"));
        }

        char prop[PROP_VALUE_MAX];
        if (property_get(ABI_LIST_PROPERTY, prop, NULL) == 0) {
            LOG_ALWAYS_FATAL("app_process: Unable to determine ABI list from property %s.",
                ABI_LIST_PROPERTY);
            return 11;
        }

        String8 abiFlag("--abi-list=");
        abiFlag.append(prop);
        args.add(abiFlag);

        // In zygote mode, pass all remaining arguments to the zygote
        // main() method.
        for (; i < argc; ++i) {
            args.add(String8(argv[i]));
        }
    }

    if (!niceName.isEmpty()) {
        runtime.setArgv0(niceName.string(), true /* setProcName */);
    }

    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
    }
}

```





--zyote : 表示当前进程用于承载zygote

--start-system-server : 是否需要启动system server 

--application:启动进入独立的程序模式

--nice-name 此进程的“别名”



对于非 zygote 的情况下，在上述参数的末尾会跟上main class 的名称。然后其他参数则属于这个class 的主函数入参数。

对于zygote 的情况，所有参数则会作为它的主函数入参使用。



init.rc 指定了 --zygote 选项，因而app_process 接下来启动ZygoteInit ,并传入start-system-server ，之后，ZygoteInit 会运行于java 虚拟机上，为什么？



原因是runtime 这个变量，它实际上是一个AndroidRuntime 对象，其start 函数源码如下：

```c
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
	JNIEnv* env;
    if (startVm(&mJavaVM, &env, zygote, primary_zygote) != 0) { //启动虚拟机
        return;
    }
    onVmCreated(env);
}
```



假设VM可以创建成功启动，并进入ZygoteInit 执行中：

fameworks/base/core/java/com/android/internal/os/ZygoteInit.java



```java
  public static void main(String argv[]) {
        ZygoteServer zygoteServer = null;

        // Mark zygote start. This ensures that thread creation will throw
        // an error.
        ZygoteHooks.startZygoteNoThreadCreation();

        // Zygote goes into its own process group.
        try {
            Os.setpgid(0, 0);
        } catch (ErrnoException ex) {
            throw new RuntimeException("Failed to setpgid(0,0)", ex);
        }

        Runnable caller;
        try {
            // Report Zygote start time to tron unless it is a runtime restart
            if (!"1".equals(SystemProperties.get("sys.boot_completed"))) {
                MetricsLogger.histogram(null, "boot_zygote_init",
                        (int) SystemClock.elapsedRealtime());
            }

            String bootTimeTag = Process.is64Bit() ? "Zygote64Timing" : "Zygote32Timing";
            TimingsTraceLog bootTimingsTraceLog = new TimingsTraceLog(bootTimeTag,
                    Trace.TRACE_TAG_DALVIK);
            bootTimingsTraceLog.traceBegin("ZygoteInit");
            RuntimeInit.preForkInit();

            boolean startSystemServer = false;
            String zygoteSocketName = "zygote";
            String abiList = null;
            boolean enableLazyPreload = false;
            for (int i = 1; i < argv.length; i++) {
                if ("start-system-server".equals(argv[i])) {
                    startSystemServer = true;
                } else if ("--enable-lazy-preload".equals(argv[i])) {
                    enableLazyPreload = true;
                } else if (argv[i].startsWith(ABI_LIST_ARG)) {
                    abiList = argv[i].substring(ABI_LIST_ARG.length());
                } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
                    zygoteSocketName = argv[i].substring(SOCKET_NAME_ARG.length());
                } else {
                    throw new RuntimeException("Unknown command line argument: " + argv[i]);
                }
            }

            final boolean isPrimaryZygote = zygoteSocketName.equals(Zygote.PRIMARY_SOCKET_NAME);

            if (abiList == null) {
                throw new RuntimeException("No ABI list supplied.");
            }

            // In some configurations, we avoid preloading resources and classes eagerly.
            // In such cases, we will preload things prior to our first fork.
            if (!enableLazyPreload) {
                bootTimingsTraceLog.traceBegin("ZygotePreload");
                EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_START,
                        SystemClock.uptimeMillis());
                preload(bootTimingsTraceLog);
                EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_END,
                        SystemClock.uptimeMillis());
                bootTimingsTraceLog.traceEnd(); // ZygotePreload
            }

            // Do an initial gc to clean up after startup
            bootTimingsTraceLog.traceBegin("PostZygoteInitGC");
            gcAndFinalize();
            bootTimingsTraceLog.traceEnd(); // PostZygoteInitGC

            bootTimingsTraceLog.traceEnd(); // ZygoteInit

            Zygote.initNativeState(isPrimaryZygote);

            ZygoteHooks.stopZygoteNoThreadCreation();

            zygoteServer = new ZygoteServer(isPrimaryZygote);

            if (startSystemServer) {
                Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer);

                // {@code r == null} in the parent (zygote) process, and {@code r != null} in the
                // child (system_server) process.
                if (r != null) {
                    r.run();
                    return;
                }
            }

            Log.i(TAG, "Accepting command socket connections");

            // The select loop returns early in the child process after a fork and
            // loops forever in the zygote.
            caller = zygoteServer.runSelectLoop(abiList);
        } catch (Throwable ex) {
            Log.e(TAG, "System zygote died with exception", ex);
            throw ex;
        } finally {
            if (zygoteServer != null) {
                zygoteServer.closeServerSocket();
            }
        }

        // We're in the child process and have exited the select loop. Proceed to execute the
        // command.
        if (caller != null) {
            caller.run();
        }
    }
```



ZygoteInit 的主函数并不复杂，它主要完成两项工作。

1,注册一个Socket

​    一旦有新程序运行时，系统会通过这个socket 在第一时间通知“总管家”，并由它负责实际的进程孵化过程。

2,预加载各类资源





#### 启动System Server 





#### zygote 的启动

