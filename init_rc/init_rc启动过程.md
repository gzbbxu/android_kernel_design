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

作为android 系统的第一个进程，iniit 将通过解析init.rc 来陆续启动其他关键的系统服务进程，

其中最重要的就是ServiceManager,Zygote 和SystemServer.



ServiceManager 是在Init.rc 里描述并由init 进程启动的。



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











