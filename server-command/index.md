# Server Command


# 0 背景
命令行是程序员诊断问题和分析问题的基础。
# 1 业务日志分析
如果应用系统出现异常，一般都会在业务日志中体现
统计当天业务日志中ERROR出现数量：egrep ERROR --color logname | wc -l  ，如果错误数量过大，一般都是有问题的
查看日志中ERROR后10行具体报错：egrep -A 10 ERROR logname | less ，或 -C 10 查看ERROR前后10行日志

按照时间段过滤日志
```
sed -n '/起始时间/,/结束时间/p' 日志文件
sed -n '/2018-12-06 00:00:00/,/2018-12-06 00:03:00/p' logname  # 查询三分钟内的日志，后再跟grep 过滤相应关键字
sed -n '/2018-12-06 08:38:00/,$p' logname  |  less # 查询指定时间到当前日志
ps：禁止使用vim直接打开日志文件
```

# 2 JVM相关

名称	主要作用
jps	JVM Process Status Tool,用来查看基于HotSpot的JVM里面中，所有具有访问权限的Java进程的具体状态, 包括进程ID，进程启动的路径及启动参数等等，与unix上的ps类似，只不过jps是用来显示java进程，可以把jps理解为ps的一个子集。
jstat	JVM Statistics Monitoring Tool,jstat是用于监视虚拟各种运行状态信息的命令行工具，它可以显示本地或者远程虚拟机进程中的类装载、内存、垃圾收集、JIT编译等运行数据。
jinfo	Configuration info for java，命令的作用是实时的查看和调整虚拟机的参数。
jmap	Memory Map for java，生成虚拟机的内存转储快照(heapdump)
jhat	JVM Heap Dump Browser，用于分析heapdump文件，它会建立一个Http/HTML服务器，让用户可以在浏览器上查看分析结果
jstack	Stack Trace for java，显示虚拟机的线程快照。

使用--help，查看命令具体使用

常用：
```
jps -v
jstat -gc 118694 500 5 
jmap -dump:live,format=b,file=dump.hprof 29170
jmap -heap 29170
jmap -histo:live 29170 | more
jmap -permstat 29170
jstack -l 29170 |more
```

```
jstack -l pid |wc -l
jstack -l pid |grep "BLOCKED"|wc -l
jstack -l pid |grep "Waiting on condition"|wc -l

线程block问题通常是等待io、等待网络、等待监视器锁等造成，可能会导致请求超时、造成造成线程数暴涨导致系统502等。
假设出现这样的问题，主要是关注jstack 出来的BLOCKED、Waiting on condition、Waiting on monitor entry等状态信息。
假设大量线程在“waiting for monitor entry”：可能是一个全局锁堵塞住了大量线程。
假设短时间内打印的 thread dump 文件反映。随着时间流逝。waiting for monitor entry 的线程越来越多，没有降低的趋势，可能意味着某些线程在临界区里呆的时间太长了，以至于越来越多新线程迟迟无法进入临界区。
假设大量线程在“waiting on condition”：可能是它们又跑去获取第三方资源，迟迟获取不到Response，导致大量线程进入等待状态。
假设发现有大量的线程都处在 Wait on condition，从线程堆栈看，正等待网络读写，这可能是一个网络瓶颈的征兆，由于网络堵塞导致线程无法运行。
```

# 3 日志相关
 ![alt 内存分配机制](https://perday30kilo.github.io/command.png)
  ![alt 内存分配机制](https://perday30kilo.github.io/command2.png)
