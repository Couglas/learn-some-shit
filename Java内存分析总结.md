# 查看java进程的命令

1. ps -ef | grep java
2. jps
3. top
4. 

# 分析工具

1. MAT
2. VisualVM
3. jhat：jhat -port 8000 java_pidxxx.hprof，访问http://localhost:8000访问结果
4. jstat：主要查看full gc的情况

# 查看对象占用

1. jmap -histo pid | head -n 20，-histo代表所有对象，包括已经回收掉的对象
2. jmap -histo:live pid | head -n 20

# 堆转出文件分析

1. 启动时添加：-XX:+HeapDumpOnOutOfMemoryError，自动生成dump文件

2. jmap -dump:format=b,file=filename pid

3. jmap -dump:live, format=b,file=filename pid

4. 使用arthas：heapdump

   

# 线程堆栈文件 thread  dump

可以了解JVM中所有线程的运行情况，比如线程的状态和当前正在坏撇的代码，需要结合占用系统资源的线程id

1. jstack [-l ] <pid> > filename 获取thread dump

# 堆栈文转储文件 heap dump

可以了解堆当时的使用情况，是某个时间点、java进程的内存快照。包含了当时内存中还没有被full GC回收的对象和类信息，比如各个类实例的数量及各个实例所占用的空间大小，内存溢出是可以使用该文件分析和故障定位

# dump文件结构

1. 头部信息：时间、JVM信息

```
2022-01-04 19:26:31
Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.241-b07 mixed mode):
```

2. 线程INFO信息块

```
"http-nio-8081-exec-8" #39 daemon prio=5 os_prio=0 tid=0x0000000020be4000 nid=0xab8 waiting on condition [0x0000000023d7e000]
java.lang.Thread.State: WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
	    - parking to wait for  <0x00000006c32d2198> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
#线程名称：http-nio-8081-exec-8；线程类型：daemon；优先级：5，默认为5；操作系统优先级：0
#JVM线程id：tid=0x0000000020be4000，JVM内部线程的唯一标识（通过java.lang.Thread.getId()获取，通常用自增方式实现）
#对应系统线程id（NativeThread ID）：nid=0xab8，使用十六进制表示的pid
#线程状态：waiting on condition
#起始栈地址：[0x0000000023d7e000]，对象的内存地址，通过JVM内存查看工具，能够看出线程是在哪个对象上等待；
```

3. 线程状态
   1. deadlock：死锁
   2. runnable：正在执行
   3. blocked：阻塞中
   4. waiting on condition：正处于等待资源或等待某个条件发生：等待锁、sleep、网络资源不足等
   5. waiting for monitor entry 或 in Object.wait()：java中synchroinzed借助Monitor实现。有两个队列，一个entry set，一个wait set。当synchronized 升级到重量级锁后，竞争锁失败的线程将记录到**entry set**中，这时线程状态是**waiting for monitor entry**；而竞争到锁的线程若调用锁对象的wait方法时，则记录 到**wait set**中，这时线程状态是 **in Object.wait()**

# 为什么会内存溢出

![image-20241230184006130](/Users/a123456/Library/Application Support/typora-user-images/image-20241230184006130.png)

![image-20241230184024116](/Users/a123456/Library/Application Support/typora-user-images/image-20241230184024116.png)

绝大多数的对象都在young generation被分配，也在young generation被收回，当young generation的空间被填满，GC会进行minor collection(次回收)，速度非常快。其中，young generation中未被回收的对象被转移到tenured generation，当tenured generation被填满时，即触发major collection(FULL GC主回收)，整个应用程序都会停止下来直到回收完成

1. JVM内存过小
2. 程序错误
3. 过多垃圾无法回收
4. 





































































