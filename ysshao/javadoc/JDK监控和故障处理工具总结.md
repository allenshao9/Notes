# JDK监控和故障处理工具

为了便于我们分析JVM虚拟机性能与诊断故障，java自带了命令行工具；

## JDK 命令行工具

| **名称** | **功能**描述                          |
| -------- | ------------------------------------- |
| jcmd     | 查看Java进程的类、线程、JVM属性等信息 |
| jps      | 显示当前所有java进程pid的命令         |
| jinfo    | 查看虚拟机的配置信息                  |
| jstat    | 监视虚拟机各种运行状态信息            |
| jmap     | 生成堆转储快照、查看JVM信息           |
| jstack   | 生成虚拟机当前时刻的线程快照          |

### jcmd

常用操作：

1. 列出当前所有运行的 java 进程：`jcmd -l`

2. 查看当前运行的 java 进程的信息：`jcmd PID XXX`

   如查看JVM 的启动时长`jcmd PID VM.uptime` 

| 命令                   | 描述                                                         |
| ---------------------- | ------------------------------------------------------------ |
| VM.uptime              | 查看 JVM 的启动时长                                          |
| GC.class_histogram     | 查看 JVM 的类信息，这个可以查看每个类的实例数量和占用空间大小。 |
| VM.flags               | 查看jvm使用了哪些虚拟机参数                                  |
| VM.system_properties   | 查看jvm使用了哪些系统属性                                    |
| VM.command_line        | 查看 JVM 的启动命令行                                        |
| Thread.print           | 查看 JVM 的Thread Dump (javacore)                            |
| GC.heap_dump FILE_NAME | 查看 JVM 的Heap Dump,注意，如果只指定文件名，默认会生成在启动 JVM 的目录里。 |
| GC.run                 | 对 JVM 执行 java.lang.System.gc()，告诉垃圾收集器打算进行垃圾收集，而垃圾收集器进不进行收集是不确定的 |
| PerfCounter.print      | 查看指定进程的性能统计信息                                   |

jcmd拥有jmap的大部分功能，并且Oracle官方也建议使用jcmd代替jmap。

### jps

常用操作：

1. 输出完全的包名，应用主类名，jar的完全路径名 :`jsp -l`

2. 输出传递给JVM的参数 :`jsp -v`
3. 输出传递给 Java 进程 main() 函数的参数 :`jps -m`

### jinfo

常用操作：

1. 输出当前 jvm 进程的全部参数和系统属性 (第一部分是系统的属性，第二部分是 JVM 的参数)：`jinfo vmid` 
2. 查看jvm的参数`jinfo -flags pid`
3. 查看java系统参数`jinfo -sysprops pid`

### jstat

​	常用操作:

​    查看JVM信息：`jstat -XXX PID `

​	固定次数频率查看信息`	jstat -gc pid [间隔时间/毫秒] [查询次数]`

​	详细查看堆内各个部分的使用量，以及加载类的数量。

| 命令           | 描述                                                     |
| -------------- | -------------------------------------------------------- |
| class          | 显示加载class的数量，及所占空间等信息。                  |
| compiler       | 显示VM实时编译的数量等信息。                             |
| gc             | 显示gc的信息，查看gc的次数，及时间                       |
| gccapacity     | 显示，VM内存中三代（young,old,perm）对象的使用和占用大小 |
| gcnew          | 显示新生代信息                                           |
| gcnewcapcacity | 显示新生代大小与使用情况                                 |
| gcold          | 显示老年代和永久代的信息                                 |
| gcutil         | 显示垃圾收集信息                                         |

> jstat -gc pid: 可以显示**gc的信息**，查看gc的次数，及时间。
>
> jstat -gc  pid 3000 3：每隔3秒打印一次，共计3次。
>
> | 显示列名 | 具体描述                                       |
> | -------- | ---------------------------------------------- |
> | S0C      | 年轻代中第一个survivor（幸存区）的容量         |
> | S1C      | 年轻代中第二个survivor（幸存区）的容量         |
> | S0U      | 年轻代中第一个survivor（幸存区）目前已使用空间 |
> | S1U      | 年轻代中第二个survivor（幸存区）目前已使用空间 |
> | EC       | 年轻代中Eden（伊甸园）的容量                   |
> | EU       | 年轻代中Eden（伊甸园）目前已使用空间           |
> | OC       | Old代的容量                                    |
> | OU       | Old代目前已使用空间                            |
> | PC/MC    | 永久代/元空间的容量                            |
> | PU/MU    | 永久代/元空间目前已使用空间                    |
> | YGC      | 从应用程序启动到采样时年轻代中gc次数           |
> | YGCT     | 从应用程序启动到采样时年轻代中gc所用时间(s)    |
> | FGC      | 从应用程序启动到采样时old代(全gc)gc次数        |
> | FGCT     | 从应用程序启动到采样时old代(全gc)gc所用时间(s) |
> | GCT      | 从应用程序启动到采样时gc用的总时间(s)          |
>
> jstat -class pid: 可以显示**类加载统计**。
>
> | 显示列名 | 具体描述        |
> | -------- | --------------- |
> | Loaded   | 加载class的数量 |
> | Bytes    | 所占用空间大小  |
> | Unloaded | 未加载数量      |
> | Time     | 时间            |

### jmap

`jmap`（Memory Map for Java）命令用于生成堆转储快照。 如果不使用 `jmap` 命令，要想获取 Java 堆转储，可以使用 `“-XX:+HeapDumpOnOutOfMemoryError”` 参数，可以让虚拟机在 OOM 异常出现之后自动生成 dump 文件，Linux 命令下可以通过 `kill -3` 发送进程退出信号也能拿到 dump 文件。

`jmap` 的作用并不仅仅是为了获取 dump 文件，它还可以查询 finalizer 执行队列、Java 堆和永久代的详细信息，如空间使用率、当前使用的是哪种收集器等。和`jinfo`一样，`jmap`有不少功能在 Windows 平台下也是受限制的。

常用操作：

`jmap -heap pid` 查看当前堆的大小分配情况。

`jmap -dump:format=b,file=/home/weblogic/temp/heap.hprof pid` 生产堆转储文件。

### jstack

`jstack`（Stack Trace for Java）命令用于生成虚拟机当前时刻的线程快照。线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈的集合.

生成线程快照的目的主要是定位线程长时间出现停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间等待等都是导致线程长时间停顿的原因。线程出现停顿的时候通过`jstack`来查看各个线程的调用堆栈，就可以知道没有响应的线程到底在后台做些什么事情，或者在等待些什么资源。

常用操作：

`jstack -l pid`查看线程堆栈信息

`jstack -l pid > javacore_pid.txt` 输出线程堆栈信息到文件。

## JDK 可视化分析工具

### JConsole

Jconsole是JDK自带的监控工具，在JDK/bin目录下可以找到（jconsole.exe）。它用于连接正在运行的本地或者远程的JVM，对运行在java应用程序的资源消耗和性能进行监控，并画出大量的图表，提供强大的可视化界面。而且本身占用的服务器内存很小，甚至可以说几乎不消耗。

![image-20211124102216135](images\image-20211124102216135.png)

> 如果需要使用 JConsole 连接远程进程，可以在远程 Java 程序启动时加上下面这些参数:
>
> ```shell
> -Djava.rmi.server.hostname=ip
> -Dcom.sun.management.jmxremote.port=60001   //监控的端口号
> -Dcom.sun.management.jmxremote.authenticate=false   //关闭认证
> -Dcom.sun.management.jmxremote.ssl=false
> ```

![image-20211124105947939](images\image-20211124105947939.png)

Jconsole能捕获到以下信息：

1. 概述 － JVM概述和一些监控变量的信息
2. 内存 － 内存的使用信息
3. 线程 － 线程的使用信息
4. 类 － 加载java类的信息
5. VM － JVM摘要
6. MBeans － 所有MBeans的信息

### Visual VM

VisualVM 是一款免费的，集成了多个 JDK 命令行工具的可视化工具，JDK/bin目录下可以找到（jvisualvm.exe）它能为您提供强大的分析能力，对 Java 应用程序做性能分析和调优。这些功能包括生成和分析海量数据、跟踪内存泄漏、监控垃圾回收器、执行内存和 CPU 分析，同时它还支持在 MBeans 上进行浏览和操作。本文主要介绍如何使用 VisualVM 进行性能分析及调优。

远程连接方式同JConsole。需要监控的进程添加jvm参数。

![image-20211124110021595](images\image-20211124110021595.png)

VisualVM 可提供

**监视**：监视是一种用来查看应用程序运行时行为的一般方法。通常会有多个视图（View）分别实时地显示 CPU 使用情况、内存使用情况、线程状态以及其他一些有用的信息，以便用户能很快地发现问题的关键所在。

转储：性能分析工具从内存中获得当前状态数据并存储到文件用于静态的性能分析。Java 程序是通过在启动 Java 程序时添加适当的条件参数来触发转储操作的。它包括以下三种：

- 系统转储：JVM 生成的本地系统的转储，又称作核心转储。一般的，系统转储数据量大，需要平台相关的工具去分析，如 Windows 上的 windbg 和 Linux 上的 gdb。
- Java 转储：JVM 内部生成的格式化后的数据，包括线程信息，类的加载信息以及堆的统计数据。通常也用于检测死锁。
- 堆转储：JVM 将所有对象的堆内容存储到文件。

**快照**：应用程序启动后，性能分析工具开始收集各种运行时数据，其中一些数据直接显示在监视视图中，而另外大部分数据被保存在内部，直到用户要求获取快照，基于这些保存的数据的统计信息才被显示出来。快照包含了应用程序在一段时间内的执行信息，通常有 CPU 快照和内存快照两种类型。

- CPU 快照：主要包含了应用程序中函数的调用关系及运行时间，这些信息通常可以在 CPU 快照视图中进行查看。
- 内存快照：主要包含了内存的分配和使用情况、载入的所有类、存在的对象信息及对象间的引用关系等。这些信息通常可以在内存快照视图中进行查看。

**性能分析**：性能分析是通过收集程序运行时的执行数据来帮助开发人员定位程序需要被优化的部分，从而提高程序的运行速度或是内存使用效率，主要有以下三个方面：

- CPU 性能分析：CPU 性能分析的主要目的是统计函数的调用情况及执行时间，或者更简单的情况就是统计应用程序的 CPU 使用情况。通常有 CPU 监视和 CPU 快照两种方式来显示 CPU 性能分析结果。
- 内存性能分析：内存性能分析的主要目的是通过统计内存使用情况检测可能存在的内存泄露问题及确定优化内存使用的方向。通常有内存监视和内存快照两种方式来显示内存性能分析结果。
- 线程性能分析：线程性能分析主要用于在多线程应用程序中确定内存的问题所在。一般包括线程的状态变化情况，死锁情况和某个线程在线程生命期内状态的分布情况等。

### Arthas jvm

**阿里巴巴开源的性能分析神器Arthas**

介绍了几种常见的jvm方面调优的场景，用的都是jdk自带的小工具，比如jps、jmap、jstack等。用这些自带的工具排查问题时最大的痛点就是过程比较麻烦，就好比如排查cpu占用率过高的问题，就要top->jps->printf->jstack等一系列的操作。

上面两款工具他们的**优点**是可以图形界面上看到各维度的性能数据，使用者根据这些数据进行综合分析，然后判断哪里出现了性能问题。

但是这两款工具也有个**缺点**，都必须在服务端项目进程中配置相关的监控参数。然后工具通过远程连接到项目进程，获取相关的数据。这样就会带来一些不便，比如线上环境的网络是隔离的，本地的监控工具根本连不上线上环境。那么Arthas就是一款工具不需要远程连接，也不需要配置监控参数的监控工具。

它能带给你解决以下几个问题：

> 这个类从哪个 jar 包加载的？为什么会报各种类相关的 Exception？
>
> 我改的代码为什么没有执行到？难道是我没 commit？分支搞错了？
>
> 遇到问题无法在线上 debug，难道只能通过加日志再重新发布吗？
>
> 线上遇到某个用户的数据处理有问题，但线上同样无法 debug，线下无法重现！
>
> 是否有一个全局视角来查看系统的运行状况？
>
> 有什么办法可以监控到JVM的实时运行状态？

- **安装和使用**

linux下安装：`wget https://alibaba.github.io/arthas/arthas-boot.jar`

运行`java -jar arthas-boot.jar`

![image-20211124111644645](images\image-20211124111644645.png)

- **整体监控数据**

在arthas的命令行界面，输入`dashboard`，会实时展示当前进程的的多线程状态、Jvm各区域、GC情况等信息

![image-20211124112025332](images\image-20211124112025332.png)

- **查看线程监控数据**

  常用参数

  输入thread会显示所有线程的状态信息

  输入thread -n 3会显示当前最忙的3个线程，可以用来排查线程CPU消耗

  输入thread -b 会显示当前处于BLOCKED状态的线程，可以排查线程锁的问题

  ![image-20211124112312937](images\image-20211124112312937.png)

- **查看JVM数据**

  输入jvm，查看jvm详细的性能数据

- 其他命令

  arthas还提供了很多用于监控的命令，比如监控某个方法的执行时间，反编译线上的class文件，甚至在不重启java应用的情况下直接替换某个类。官方的使用文档已经写得太详细了，这里就不再一一介绍了，大家可以自己尝试。[Arthas 用户文档 — Arthas 3.5.4 文档 (aliyun.com)](https://arthas.aliyun.com/doc/)

  > - [dashboard](https://arthas.aliyun.com/doc/dashboard.html)——当前系统的实时数据面板
  > - [thread](https://arthas.aliyun.com/doc/thread.html)——查看当前 JVM 的线程堆栈信息
  > - [jvm](https://arthas.aliyun.com/doc/jvm.html)——查看当前 JVM 的信息
  > - [sysprop](https://arthas.aliyun.com/doc/sysprop.html)——查看和修改JVM的系统属性
  > - [sysenv](https://arthas.aliyun.com/doc/sysenv.html)——查看JVM的环境变量
  > - [vmoption](https://arthas.aliyun.com/doc/vmoption.html)——查看和修改JVM里诊断相关的option
  > - [perfcounter](https://arthas.aliyun.com/doc/perfcounter.html)——查看当前 JVM 的Perf Counter信息
  > - [logger](https://arthas.aliyun.com/doc/logger.html)——查看和修改logger
  > - [getstatic](https://arthas.aliyun.com/doc/getstatic.html)——查看类的静态属性
  > - [ognl](https://arthas.aliyun.com/doc/ognl.html)——执行ognl表达式
  > - [mbean](https://arthas.aliyun.com/doc/mbean.html)——查看 Mbean 的信息
  > - [heapdump](https://arthas.aliyun.com/doc/heapdump.html)——dump java heap, 类似jmap命令的heap dump功能
  > - [vmtool](https://arthas.aliyun.com/doc/vmtool.html)——从jvm里查询对象，执行forceGc

### 补充

​    上面为具体的监控工具，针对**Java Core和Heap Dump**还需要专业工具进行特定分析。

1. MemoryAnalyzer工具堆转储文件，用于分析内存溢出或泄露情况。
2. IBM的相关jar包，jca465.jar分析javacore文件；ha457.jar分析堆dmp文件。
