---
title: "深入理解Java虚拟机笔记4——虚拟机监控处理工具" 
date: 2021-06-26T12:03:39+08:00
draft: false
tags: ["JVM","笔记"]
categories:
  - "JVM"
  - "笔记"
---



给一个系统定位问题的时候，知识、经验是关键基础，数据是依据，工具是运用知识处理数据的手段。这里说的数据包括但不限于异常堆栈、虚拟机运行日志、垃圾收集器日志、线程快照（`threaddump/javacore`文件）、堆转储快照（`heapdump/hprof`文件）等。



# 虚拟机监控处理工具



## 基础故障处理工具

`JDK`的bin目录中除了常见的`java.exe`、`javac.exe`这两个命令行工具，还有很多可用于打包、部署、签名、调试、监控、运维等各种场景都可能用到的小工具。



### jps:虚拟机进程状况工具

`JDK`的很多小工具的名字都参考了UNIX命令的命名方式，`jps`（`JVM Process Status Tool`）是其中的典型。除了名字像UNIX的`ps`命令之外，它的功能也和`ps`命令类似：可以列出正在运行的虚拟机进程，并显示虚拟机执行主类（Main Class，main()函数所在的类）名称以及这些进程的本地虚拟机唯一ID（`LVMID`，`Local Virtual Machine Identifier`）。虽然功能比较单一，但它绝对是**使用频率最高的`JDK`命令行工具，因为其他的`JDK`工具大多需要输入它查询到的`LVMID`来确定要监控的是哪一个虚拟机进程**。**对于本地虚拟机进程来说，`LVMID`与操作系统的进程ID（`PID`，Process Identifier）是一致的**，使用Windows的任务管理器或者UNIX的`ps`命令也可以查询到虚拟机进程的`LVMID`，但如果同时启动了多个虚拟机进程，无法根据进程名称定位时，那就必须依赖`jps`命令显示主类的功能才能区分了。



`jps`命令格式：

```shell
jps [ options ] [ hostid ]
```



`jps`执行样例：

```shell
jps -l
2388 D:\Develop\glassfish\bin\..\modules\admin-cli.jar
2764 com.sun.enterprise.glassfish.bootstrap.ASMain
3788 sun.tools.jps.Jps
```



`jps`还可以通过`RMI协议`查询开启了`RMI服务`的远程虚拟机进程状态，参数`hostid`为`RMI注册表`中注册的主机名。



`jps`工具主要选项：

| 选项 | 作用                                                 |
| ---- | ---------------------------------------------------- |
| -q   | 只输出`LVMID`，省略主类的名称                        |
| -m   | 输出虚拟机进程启动时传递给主类main()函数的名称       |
| -l   | 输出主类的全名，如果进程执行的是JAR包，则输出JAR路径 |
| -v   | 输出虚拟机进程启动时的`JVM`参数                      |





### jstat：虚拟机统计信息监视工具

`jstat`（`JVM Statistics Monitoring Tool`）是用于**监视虚拟机各种运行状态信息的命令行工具**。它**可以显示本地或者远程虚拟机进程中的类加载、内存、垃圾收集、即时编译等运行时数据**，在没有GUI图形界面、只提供了纯文本控制台环境的服务器上，它将是运行期定位虚拟机性能问题的常用工具。



`jstat`命令格式为：

```shell
jstat [ option vmid [interval[s|ms] [count]] ]
```

对于命令格式中的`VMID`与`LVMID`需要特别说明一下：如果是本地虚拟机进程，`VMID`与`LVMID`是一致的；如果是远程虚拟机进程，那`VMID`的格式应当是：

```shell
[protocol:][//]lvmid[@hostname[:port]/servername]
```



参数interval和count代表查询间隔和次数，如果省略这2个参数，说明只查询一次。假设需要每250毫秒查询一次进程2764垃圾收集状况，一共查询20次，那命令应当是：

```shell
jstat -gc 2764 250 20
```



选项option代表用户希望查询的虚拟机信息，主要分为三类：类加载、垃圾收集、运行期编译状况：

| 选项              | 作用                                                         |
| ----------------- | ------------------------------------------------------------ |
| -class            | 监视类加载、卸载数量、总空间以及类装载所耗费的时间           |
| -gc               | 监视java堆状况，包括Eden区、2个Survivor区、老年代、永久代等的容量，已用空间，垃圾收集信息合计等信息 |
| -gccapacity       | 监视内容与-gc基本相同，但输出主要关注Java堆各个区域使用到的最大、最小空间 |
| -gcutil           | 监视内容与-gc基本相同，但输出主要关注已使用空间占总空间的百分比 |
| -gccause          | 与-gcutil功能一样，但是会额外输出导致上一次垃圾收集产生的原因 |
| -gcnew            | 监视新生代垃圾收集状况                                       |
| -gcnewcapacity    | 监视内容与-gcnew基本相同，输出主要关注使用到的最大、最小空间 |
| -gcold            | 监视老年代垃圾收集状况                                       |
| -gcoldcapacity    | 监视内容与-gcold基本相同，输出主要关注使用到的最大、最小空间 |
| -gcpermcapacity   | 输出永久代使用到的最大、最小空间                             |
| -compiler         | 输出即时编译器编译过的方法、耗时等信息                       |
| -printcompilation | 输出已经被即时编译的方法                                     |



使用`jstat`工具在纯文本状态下监视虚拟机状态的变化，在用户体验上也许不如`JMC`、`VisualVM`等可视化的监视工具直接以图表展现那样直观，但在实际生产环境中不一定可以使用图形界面，而且多数服务器管理员也都已经习惯了在文本控制台工作，直接在控制台中使用`jstat`命令依然是一种常用的监控方式。



### jinfo：Java配置信息工具

`jinfo`（Configuration Info for Java）的作用是**实时查看和调整虚拟机各项参数**。使用`jps`命令的`-v参数`可以**查看虚拟机启动时显式指定的参数列表**，但**如果想知道未被显式指定的参数的系统默认值，除了去找资料外，就只能使用`jinfo`的`-flag`选项进行查询**了（如果只限于`JDK 6`或以上版本的话，使用`java-XX：+PrintFlagsFinal`查看参数默认值也是一个很好的选择）。`jinfo`还可以使用`-sysprops`选项把虚拟机进程的`System.getProperties()`的内容打印出来。这个命令在`JDK 5`时期已经随着Linux版的`JDK`发布，当时只提供了信息查询的功能，`JDK 6`之后，`jinfo`在Windows和Linux平台都有提供，并且加入了在运行期修改部分参数值的能力（可以使用-flag[+|-]name或者-flag name=value在运行期修改一部分运行期可写的虚拟机参数值）。在`JDK 6`中，`jinfo`对于Windows平台功能仍然有较大限制，只提供了最基本的`-flag`选项。



`jinfo`命令格式：

```shell
jinfo [ option ] pid
```



执行样例：查询`CMSInitiatingOccupancyFraction`参数值:

```shell
jinfo -flag CMSInitiatingOccupancyFraction 1444
-XX:CMSInitiatingOccupancyFraction=85
```



### jmap：Java内存映像工具

`jmap`（Memory Map for Java）命令用于生成堆转储快照（一般称为`heapdump`或`dump`文件）。如果不使用`jmap`命令，要想获取Java堆转储快照也还有一些比较“暴力”的手段：譬如在第2章中用过的`-XX：+HeapDumpOnOutOfMemoryError`参数，可以让虚拟机在内存溢出异常出现之后自动生成堆转储快照文件，通过`-XX：+HeapDumpOnCtrlBreak`参数则可以使用`[Ctrl]+[Break]`键让虚拟机生成堆转储快照文件，又或者在Linux系统下通过Kill-3命令发送进程退出信号“恐吓”一下虚拟机，也能顺利拿到堆转储快照。



`jmap`的作用并**不仅仅是为了获取堆转储快照，它还可以查询finalize执行队列、Java堆和方法区的详细信息，如空间使用率、当前用的是哪种收集器等**。



和`jinfo`命令一样，`jmap`有部分功能在Windows平台下是受限的，除了生成堆转储快照的-dump选项和用于查看每个类的实例、空间占用统计的`-histo`选项在所有操作系统中都可以使用之外，其余选项都只能在`Linux/Solaris`中使用：

`jmap`命令格式：

```shell
jmap [ option ] vmid
```



`jmap`工具主要选项：

| 选项           | 作用                                                         |
| -------------- | ------------------------------------------------------------ |
| -dump          | 生成Java堆转储快照。格式为-dump:[live,]format=b,file=<filename>，其中live子参数说明是否只dump出存活的对象 |
| -finalizerinfo | 显示在F-Queue中等待Finalizer线程执行finalize方法的对象。只在Linux平台下有效 |
| -heap          | 显式Java堆详细信息，如使用哪种回收期、参数配置、分代状况等。只在Linux平台下有效 |
| -histo         | 显式堆中对象统计信息，包括类、实例数量、合计容量             |
| -permstat      | 以ClassLoader为统计口径显式永久代内存状态。只在Linux平台下有效 |
| -F             | 当虚拟机进程对-dump选项没有响应时，可以使用这个选项强制生成dump快照，只在Linux平台下有效 |



使用`jmap`生成dump文件：

```shell
jmap -dump:format=b,file=eclipse.bin 3500
Dumping heap to C:\Users\IcyFenix\eclipse.bin ...
Heap dump file created
```



3500是通过`jps`命令查询到的`LVMID`。



### jhat：虚拟机堆转储快照分析工具

`JDK`提供`jhat`（`JVM Heap Analysis Tool`）命令与`jmap`搭配使用，来分析`jmap`生成的堆转储快照。`jhat`内置了一个微型的`HTTP/Web`服务器，生成堆转储快照的分析结果后，可以在浏览器中查看。不过实事求是地说，**在实际工作中，除非手上真的没有别的工具可用，否则多数人是不会直接使用`jhat`命令来分析堆转储快照文件的**，主要原因有两个方面。一是一般不会在部署应用程序的服务器上直接分析堆转储快照，即使可以这样做，也会尽量将堆转储快照文件复制到其他机器上进行分析，因为分析工作是一个耗时而且极为耗费硬件资源的过程，既然都要在其他机器上进行，就没有必要再受命令行工具的限制了。另外一个原因是`jhat`的分析功能相对来说比较简陋，后文将会介绍到的`VisualVM`，以及专业用于分析堆转储快照文件的`Eclipse Memory Analyzer`、`IBM HeapAnalyzer`等工具，都能实现比`jhat`更强大专业的分析功能。



使用`jhat`分析dump文件：

```shell
jhat eclipse.bin
Reading from eclipse.bin...
Dump file created Fri Nov 19 22:07:21 CST 2010
Snapshot read, resolving...
Resolving 1225951 objects...
Chasing references, expect 245 dots....
Eliminating duplicate references...
Snapshot resolved.
Started HTTP server on port 7000
Server is ready.
```



屏幕显示“Server is ready.”的提示后，用户在浏览器中输入http://localhost:7000/可以看到分析结果。



分析结果默认以包为单位进行分组显示，分析内存泄漏问题主要会使用到其中的“`Heap Histogram`”（与`jmap-histo`功能一样）与`OQL`页签的功能，前者可以找到内存中总容量最大的对象，后者是标准的对象查询语言，使用类似`SQL`的语法对内存中的对象进行查询统计。





### jstack：Java堆栈跟踪工具

`jstack`（`Stack Trace for Java`）命令用于**生成虚拟机当前时刻的线程快照（一般称为`threaddump`或者`javacore`文件）**。**线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈的集合，生成线程快照的目的通常是定位线程出现长时间停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间挂起等，都是导致线程长时间停顿的常见原因**。**线程出现停顿时通过`jstack`来查看各个线程的调用堆栈，就可以获知没有响应的线程到底在后台做些什么事情，或者等待着什么资源**。



`jstack`命令格式：

```shell
jstack [ option ] vmid
```



option选项：

| 选项 | 作用                                         |
| ---- | -------------------------------------------- |
| -F   | 当正常输出的请求不被响应时，强制输出线程堆栈 |
| -l   | 除堆栈外，显示关于锁的附加信息               |
| -m   | 如果调用到本地方法的话，可以显示C/C++的堆栈  |



使用`jstack`查看线程堆栈：

```shell
jstack -l 3500
2010-11-19 23:11:26
Full thread dump Java HotSpot(TM) 64-Bit Server VM (17.1-b03 mixed mode):
"[ThreadPool Manager] - Idle Thread" daemon prio=6 tid=0x0000000039dd4000 nid= 0xf50 in Object.wait() [0x000000003c96f000]
    java.lang.Thread.State: WAITING (on object monitor)
        at java.lang.Object.wait(Native Method)
        - waiting on <0x0000000016bdcc60> (a org.eclipse.equinox.internal.util.impl.tpt.threadpool.Executor)
        at java.lang.Object.wait(Object.java:485)
        at org.eclipse.equinox.internal.util.impl.tpt.threadpool.Executor.run (Executor. java:106)
        - locked <0x0000000016bdcc60> (a org.eclipse.equinox.internal.util.impl.tpt.threadpool.Executor)
    Locked ownable synchronizers:
    	- None
```



从`JDK 5`起，`java.lang.Thread`类新增了一个`getAllStackTraces()`方法用于获取虚拟机中所有线程的`StackTraceElement`对象。使用这个方法可以通过简单的几行代码完成`jstack`的大部分功能，在实际项目中不妨调用这个方法做个管理员页面，可以随时使用浏览器来查看线程堆栈。

```html
<%@ page import="java.util.Map"%>
<html>
<head>
<title>服务器线程信息</title>
</head>
<body>
<pre>
<%
    for (Map.Entry<Thread, StackTraceElement[]> stackTrace : Thread.getAllStack-Traces().entrySet()) {
        Thread thread = (Thread) stackTrace.getKey();
        StackTraceElement[] stack = (StackTraceElement[]) stackTrace.getValue();
        if (thread.equals(Thread.currentThread())) {
        	continue;
        }
        out.print("\n线程：" + thread.getName() + "\n");
        for (StackTraceElement element : stack) {
        	out.print("\t"+element+"\n");
        }
    }
%>
</pre>
</body>
</html>
```



## 可视化故障处理工具

`JDK`中除了附带大量的命令行工具外，还提供了几个功能集成度更高的可视化工具，用户可以使用这些可视化工具以更加便捷的方式进行进程故障诊断和调试工作。这类工具主要包括`JConsole`、`JHSDB`、`VisualVM`和`JMC`四个。



### JConsole：Java监视与管理控制台

`JConsole`（Java Monitoring and Management Console）是一款基于`JMX`（`Java Manage-ment Extensions`）的可视化监视、管理工具。它的**主要功能是通过`JMX`的`MBean`（Managed Bean）对系统进行信息收集和参数动态调整**。`JMX`是一种开放性的技术，不仅可以用在虚拟机本身的管理上，还可以运行于虚拟机之上的软件中，典型的如中间件大多也基于`JMX`来实现管理与监控。虚拟机对`JMXMBean`的访问也是完全开放的，可以使用代码调用`API`、支持`JMX`协议的管理控制台，或者其他符合`JMX`规范的软件进行访问。



启动JConsole：

通过`JDK/bin`目录下的`jconsole.exe`启动`JCon-sole`后，会自动搜索出本机运行的所有虚拟机进程，而不需要用户自己使用`jps`来查询，双击选择其中一个进程便可进入主界面开始监控。`JMX`支持跨服务器的管理，也可以使用下面的“远程进程”功能来连接远程服务器，对远程虚拟机进行监控。



主界面里共包括“概述”“内存”“线程”“类”“`VM`摘要”“`MBean`”六个页签。“概述”页签里显示的是整个虚拟机主要运行数据的概览信息，包括“堆内存使用情况”“线
程”“类”“CPU使用情况”四项信息的曲线图，这些曲线图是后面“内存”“线程”“类”页签的信息汇总。



内存监控：

“内存”页签的作用**相当于可视化的`jstat`命令**，用于监视被收集器管理的虚拟机内存（被收集器直接管理的Java堆和被间接管理的方法区）的变化趋势。



线程监控：

如果说`JConsole`的“内存”页签相当于可视化的`jstat`命令的话，**那“线程”页签的功能就相当于可视化的`jstack`命令**了，遇到**线程停顿的时候可以使用这个页签的功能进行分析**。线程长时间停顿的主要原因有等待外部资源（数据库连接、网络资源、设备资源等）、死循环、锁等待等。



### VisualVM：多合-故障处理工具

`VisualVM`（`All-in-One Java Troubleshooting Tool`）是功能最强大的运行监视和故障处理程序之一，曾经在很长一段时间内是Oracle官方主力发展的虚拟机故障处理工具。Oracle曾在`VisualVM`的软件说明中写上了“`All-in-One`”的字样，预示着它除了常规的运行监视、故障处理外，还将提供其他方面的能力，譬如性能分析（Profiling）。`VisualVM`的性能分析功能比起`JProfiler`、`YourKit`等专业且收费的`Profiling`工具都不遑多让。而且相比这些第三方工具，`VisualVM`还有一个很大的优点：不需要被监视的程序基于特殊Agent去运行，因此它的通用性很强，对应用程序实际性能的影响也较小，使得它可以直接应用在生产环境中。这个优点是`JProfiler`、`YourKit`等工具无法与之媲美的。







