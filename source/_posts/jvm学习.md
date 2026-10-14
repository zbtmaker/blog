# JVM 学习
之前学习JVM都是停留在理论层面，每一次学习完成之后我都不知道怎么样定位问题，每次出问题以后都有些懵逼。这一次趁着假期我来好好学一下JVM的问题排查和调优。

## 程序死循环
通过程序死循环来看CPU使用率飙升的场景。在一个正常的to C 应用，平常流量的CPU使用率会在一个很小范围的波动，我们这里模拟程序出现死循环，此时CPU会不断的调度当前线程（这里只要当前线程的状态是Runnable，那么当前线程就会被调起，然后由CPU运行指令）。我们看一下代码
```java
    @GetMapping("/cpu")
    public void cpu() {
        // 打印当前进程的 PID，方便后续排查
        long pid = ProcessHandle.current().pid();
        System.out.println("程序已启动，当前进程 PID = " + pid);

        // 创建一个专门用于模拟高 CPU 消耗的后台线程
        Thread highCpuThread = new Thread(() -> {
            long counter = 0;
            while (true) {
                // 模拟疯狂的死循环计算，导致 CPU 飙升
                counter++;
                if (counter == Long.MAX_VALUE) {
                    counter = 0;
                }
            }
        }, "High-CPU-Simulation-Thread"); // 给线程起一个容易辨认的名字

        // 启动线程
        highCpuThread.start();

        // 主线程进入休眠，保持程序运行
        try {
            Thread.sleep(Long.MAX_VALUE);
        } catch (InterruptedException e) {
        }
    }
```
将程序打包
```bash
mvn clean package
```
将打包后的程序运行
```bash
java \
  -Xms256m \
  -Xmx256m \
  -XX:+UseG1GC \
  '-Xlog:gc*,safepoint:file=logs/gc.log:time,uptime,level,tags:filecount=3,filesize=20M' \
  -jar target/jvm-lab.war
```
执行curl命令请求接口
```bash
curl "http://localhost:8080/jvm/cpu"
```
运行top命令查看占用CPU最高的进程是哪一个
```bash
zoubaitao@zoubaitaodeMac-mini jvm-lab % top

Processes: 867 total, 3 running, 864 sleeping, 4312 threads                                                                    16:53:12
Load Avg: 2.33, 2.06, 1.81  CPU usage: 11.17% user, 1.14% sys, 87.67% idle  SharedLibs: 1127M resident, 193M data, 148M linkedit.
MemRegions: 0 total, 0B resident, 0B private, 3008M shared. PhysMem: 23G used (1843M wired, 2133M compressor), 653M unused.
VM: 433T vsize, 57G framework vsize, 0(0) swapins, 0(0) swapouts. Networks: packets: 24372533/6918M in, 23246764/5009M out.
Disks: 3745141/84G read, 2339046/39G written.

PID   COMMAND      %CPU      TIME     #TH    #WQ   #PORT MEM    PURG   CMPRS  PGRP PPID STATE    BOOSTS            %CPU_ME %CPU_OTHRS
8859  java         100.6     00:24.04 47/1   1     181   217M   0B     0B     8859 906  running  *0[1]             0.00000 0.00000
8520  top          3.9       04:36.97 1/1    0     41    10M    0B     3120K  8520 921  running  *0[1]             0.00000 0.00000
431   WindowServer 3.2       28:58.08 24     7     6343  683M   22M    53M    431  1    sleeping *0[1]             0.33338 0.15412
0     kernel_task  2.9       24:51.95 663/10 0     0     512K   0B     0B     0    0    running   0[0]             0.00000 0.00000
3743  idea         1.8       22:46.68 105    3     593   2070M  75M    155M   678  1    sleeping  0[4052]          0.00000 0.00000
7761  Citrix Viewe 1.2       01:20.13 17     4     522   70M    2288K  24M    7761 1    sleeping *0[1160]          0.02968 0.00000
1082  logioptionsp 1.1       04:21.32 143    2     1087  89M    0B     27M    1082 1    sleeping *0[1]             0.49397 0.05168
6495  Google Chrom 1.0       02:28.30 28     1     252   327M   208K   25M    679  679  sleeping *0[37041]         0.00000 0.00000
```
这里我们可以看到是java进程占据了一整个CPU的运行。但是这里有个问题，如果我的程序里里面有很多线程在运行，因为只有Runnable状态的线程才能被CPU调度。所以我可以采用下面的方式来查看运行的线程，然后一个线程一个线程来排查。
```bash
zoubaitao@192 ~ % jstack 8859 | grep -A 100 "RUNNABLE"
   java.lang.Thread.State: RUNNABLE
	at java.lang.ref.Reference.waitForReferencePendingList(java.base@17.0.20/Native Method)
	at java.lang.ref.Reference.processPendingReferences(java.base@17.0.20/Reference.java:253)
	at java.lang.ref.Reference$ReferenceHandler.run(java.base@17.0.20/Reference.java:215)

"Finalizer" #3 daemon prio=8 os_prio=31 cpu=0.04ms elapsed=592.60s tid=0x0000007ab1548000 nid=0x7b03 in Object.wait()  [0x000000016e816000]
   java.lang.Thread.State: WAITING (on object monitor)
	at java.lang.Object.wait(java.base@17.0.20/Native Method)
	- waiting on <0x000000013c1400b8> (a java.lang.ref.ReferenceQueue$Lock)
	at java.lang.ref.ReferenceQueue.remove(java.base@17.0.20/ReferenceQueue.java:155)
	- locked <0x000000013c1400b8> (a java.lang.ref.ReferenceQueue$Lock)
	at java.lang.ref.ReferenceQueue.remove(java.base@17.0.20/ReferenceQueue.java:176)
	at java.lang.ref.Finalizer$FinalizerThread.run(java.base@17.0.20/Finalizer.java:172)

"Signal Dispatcher" #4 daemon prio=9 os_prio=31 cpu=0.08ms elapsed=592.59s tid=0x0000007ab1548c00 nid=0x7503 waiting on condition  [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"Service Thread" #5 daemon prio=9 os_prio=31 cpu=0.11ms elapsed=592.59s tid=0x0000007ab1549200 nid=0x6703 runnable  [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"Monitor Deflation Thread" #6 daemon prio=9 os_prio=31 cpu=13.61ms elapsed=592.59s tid=0x0000007ab1549800 nid=0x6903 runnable  [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"C2 CompilerThread0" #7 daemon prio=9 os_prio=31 cpu=766.70ms elapsed=592.59s tid=0x0000007ab1549e00 nid=0x6b03 waiting on condition  [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE
   No compile task

"C1 CompilerThread0" #10 daemon prio=9 os_prio=31 cpu=477.28ms elapsed=592.59s tid=0x0000007ab154a400 nid=0x7303 waiting on condition  [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE
   No compile task

"Sweeper thread" #11 daemon prio=9 os_prio=31 cpu=1.72ms elapsed=592.59s tid=0x0000007ab154aa00 nid=0x6e03 runnable  [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"Notification Thread" #12 daemon prio=9 os_prio=31 cpu=0.01ms elapsed=592.59s tid=0x0000007ab154b000 nid=0x7203 runnable  [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"Common-Cleaner" #13 daemon prio=8 os_prio=31 cpu=0.75ms elapsed=592.59s tid=0x0000007ab154b600 nid=0xa903 in Object.wait()  [0x000000016fb9a000]
   java.lang.Thread.State: TIMED_WAITING (on object monitor)
	at java.lang.Object.wait(java.base@17.0.20/Native Method)
	- waiting on <0x000000013c19ff98> (a java.lang.ref.ReferenceQueue$Lock)
	at java.lang.ref.ReferenceQueue.remove(java.base@17.0.20/ReferenceQueue.java:155)
	- locked <0x000000013c19ff98> (a java.lang.ref.ReferenceQueue$Lock)
	at jdk.internal.ref.CleanerImpl.run(java.base@17.0.20/CleanerImpl.java:140)
	at java.lang.Thread.run(java.base@17.0.20/Thread.java:840)
	at jdk.internal.misc.InnocuousThread.run(java.base@17.0.20/InnocuousThread.java:162)

"Cleaner-0" #14 daemon prio=8 os_prio=31 cpu=2.34ms elapsed=592.57s tid=0x0000007ab1610600 nid=0xa703 in Object.wait()  [0x000000016fda6000]
   java.lang.Thread.State: TIMED_WAITING (on object monitor)
	at java.lang.Object.wait(java.base@17.0.20/Native Method)
	- waiting on <0x000000013c180020> (a java.lang.ref.ReferenceQueue$Lock)
	at java.lang.ref.ReferenceQueue.remove(java.base@17.0.20/ReferenceQueue.java:155)
	- locked <0x000000013c180020> (a java.lang.ref.ReferenceQueue$Lock)
	at jdk.internal.ref.CleanerImpl.run(java.base@17.0.20/CleanerImpl.java:140)
	at java.lang.Thread.run(java.base@17.0.20/Thread.java:840)
	at jdk.internal.misc.InnocuousThread.run(java.base@17.0.20/InnocuousThread.java:162)

"Catalina-utility-1" #19 prio=1 os_prio=31 cpu=27.79ms elapsed=591.88s tid=0x0000007ab3b40000 nid=0x8803 waiting on condition  [0x0000000171636000]
   java.lang.Thread.State: TIMED_WAITING (parking)
	at jdk.internal.misc.Unsafe.park(java.base@17.0.20/Native Method)
	- parking to wait for  <0x000000013eeecb88> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
	at java.util.concurrent.locks.LockSupport.parkNanos(java.base@17.0.20/LockSupport.java:252)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.awaitNanos(java.base@17.0.20/AbstractQueuedSynchronizer.java:1679)
	at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(java.base@17.0.20/ScheduledThreadPoolExecutor.java:1182)
	at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(java.base@17.0.20/ScheduledThreadPoolExecutor.java:899)
	at java.util.concurrent.ThreadPoolExecutor.getTask(java.base@17.0.20/ThreadPoolExecutor.java:1062)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(java.base@17.0.20/ThreadPoolExecutor.java:1122)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(java.base@17.0.20/ThreadPoolExecutor.java:635)
	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:63)
	at java.lang.Thread.run(java.base@17.0.20/Thread.java:840)

"Catalina-utility-2" #20 prio=1 os_prio=31 cpu=28.50ms elapsed=591.88s tid=0x0000007ab3b40600 nid=0x8903 waiting on condition  [0x0000000171842000]
   java.lang.Thread.State: WAITING (parking)
	at jdk.internal.misc.Unsafe.park(java.base@17.0.20/Native Method)
	- parking to wait for  <0x000000013eeecb88> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
	at java.util.concurrent.locks.LockSupport.park(java.base@17.0.20/LockSupport.java:341)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionNode.block(java.base@17.0.20/AbstractQueuedSynchronizer.java:506)
	at java.util.concurrent.ForkJoinPool.unmanagedBlock(java.base@17.0.20/ForkJoinPool.java:3465)
	at java.util.concurrent.ForkJoinPool.managedBlock(java.base@17.0.20/ForkJoinPool.java:3436)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(java.base@17.0.20/AbstractQueuedSynchronizer.java:1630)
	at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(java.base@17.0.20/ScheduledThreadPoolExecutor.java:1177)
	at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(java.base@17.0.20/ScheduledThreadPoolExecutor.java:899)
	at java.util.concurrent.ThreadPoolExecutor.getTask(java.base@17.0.20/ThreadPoolExecutor.java:1062)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(java.base@17.0.20/ThreadPoolExecutor.java:1122)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(java.base@17.0.20/ThreadPoolExecutor.java:635)
	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:63)
	at java.lang.Thread.run(java.base@17.0.20/Thread.java:840)

"container-0" #21 prio=5 os_prio=31 cpu=0.67ms elapsed=591.88s tid=0x0000007ab3b40c00 nid=0x9503 waiting on condition  [0x0000000171a4e000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
	at java.lang.Thread.sleep(java.base@17.0.20/Native Method)
	at org.apache.catalina.core.StandardServer.await(StandardServer.java:520)
	at org.springframework.boot.web.embedded.tomcat.TomcatWebServer$1.run(TomcatWebServer.java:219)

"http-nio-8080-exec-1" #22 daemon prio=5 os_prio=31 cpu=25.19ms elapsed=591.78s tid=0x0000007ab3b43000 nid=0x8c03 waiting on condition  [0x0000000171c58000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
	at java.lang.Thread.sleep(java.base@17.0.20/Native Method)
	at org.example.jvmlab.controller.JvmController.cpu(JvmController.java:152)
	at jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(java.base@17.0.20/Native Method)
	at jdk.internal.reflect.NativeMethodAccessorImpl.invoke(java.base@17.0.20/NativeMethodAccessorImpl.java:77)
	at jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(java.base@17.0.20/DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(java.base@17.0.20/Method.java:569)
	at org.springframework.web.method.support.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:255)
	at org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:188)
	at org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:118)
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandlerMethod(RequestMappingHandlerAdapter.java:926)
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:831)
	at org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:87)
	at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:1089)
	at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:979)
	at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:1014)
	at org.springframework.web.servlet.FrameworkServlet.doGet(FrameworkServlet.java:903)
	at jakarta.servlet.http.HttpServlet.service(HttpServlet.java:564)
	at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:885)
	at jakarta.servlet.http.HttpServlet.service(HttpServlet.java:658)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:195)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:140)
	at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:51)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:164)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:140)
	at org.springframework.web.filter.RequestContextFilter.doFilterInternal(RequestContextFilter.java:100)
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:116)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:164)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:140)
	at org.springframework.web.filter.FormContentFilter.doFilterInternal(FormContentFilter.java:93)
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:116)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:164)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:140)
	at org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:201)
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:116)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:164)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:140)
	at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:167)
	at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:90)
	at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:483)
	at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:115)
--
   java.lang.Thread.State: RUNNABLE
	at sun.nio.ch.KQueue.poll(java.base@17.0.20/Native Method)
	at sun.nio.ch.KQueueSelectorImpl.doSelect(java.base@17.0.20/KQueueSelectorImpl.java:122)
	at sun.nio.ch.SelectorImpl.lockAndDoSelect(java.base@17.0.20/SelectorImpl.java:129)
	- locked <0x000000013cb72658> (a sun.nio.ch.Util$2)
	- locked <0x000000013cb72468> (a sun.nio.ch.KQueueSelectorImpl)
	at sun.nio.ch.SelectorImpl.select(java.base@17.0.20/SelectorImpl.java:141)
	at org.apache.tomcat.util.net.NioEndpoint$Poller.run(NioEndpoint.java:755)
	at java.lang.Thread.run(java.base@17.0.20/Thread.java:840)

"http-nio-8080-Acceptor" #33 daemon prio=5 os_prio=31 cpu=3.52ms elapsed=591.78s tid=0x0000007ab3bb7600 nid=0xf903 runnable  [0x00000001732de000]
   java.lang.Thread.State: RUNNABLE
	at sun.nio.ch.Net.accept(java.base@17.0.20/Native Method)
	at sun.nio.ch.ServerSocketChannelImpl.implAccept(java.base@17.0.20/ServerSocketChannelImpl.java:425)
	at sun.nio.ch.ServerSocketChannelImpl.accept(java.base@17.0.20/ServerSocketChannelImpl.java:391)
	at org.apache.tomcat.util.net.NioEndpoint.serverSocketAccept(NioEndpoint.java:518)
	at org.apache.tomcat.util.net.NioEndpoint.serverSocketAccept(NioEndpoint.java:71)
	at org.apache.tomcat.util.net.Acceptor.run(Acceptor.java:128)
	at java.lang.Thread.run(java.base@17.0.20/Thread.java:840)

"DestroyJavaVM" #34 prio=5 os_prio=31 cpu=768.75ms elapsed=591.77s tid=0x0000007ab3bc0600 nid=0x2603 waiting on condition  [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"High-CPU-Simulation-Thread" #35 daemon prio=5 os_prio=31 cpu=588610.55ms elapsed=589.25s tid=0x0000007ab3bc0c00 nid=0xb103 runnable  [0x00000001734ea000]
   java.lang.Thread.State: RUNNABLE
	at org.example.jvmlab.controller.JvmController.lambda$cpu$0(JvmController.java:141)
	at org.example.jvmlab.controller.JvmController$$Lambda$787/0x000000c801408440.run(Unknown Source)
	at java.lang.Thread.run(java.base@17.0.20/Thread.java:840)

"Attach Listener" #36 daemon prio=9 os_prio=31 cpu=0.72ms elapsed=488.52s tid=0x0000007ab3bc1200 nid=0x820b waiting on condition  [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"G1 Conc#1" os_prio=31 cpu=2.55ms elapsed=592.01s tid=0x0000007ab0c61500 nid=0x8603 runnable  

"GC Thread#8" os_prio=31 cpu=0.56ms elapsed=592.01s tid=0x0000007ab0c61e00 nid=0x9803 runnable  

"GC Thread#7" os_prio=31 cpu=0.59ms elapsed=592.01s tid=0x0000007ab0c61b00 nid=0x9a07 runnable  

"GC Thread#6" os_prio=31 cpu=0.61ms elapsed=592.01s tid=0x0000007ab0c61800 nid=0x9c0f runnable  

"GC Thread#5" os_prio=31 cpu=5.52ms elapsed=592.43s tid=0x0000007ab097e700 nid=0x9e03 runnable  

"GC Thread#4" os_prio=31 cpu=5.53ms elapsed=592.43s tid=0x0000007ab097e400 nid=0x9f03 runnable  

"GC Thread#3" os_prio=31 cpu=5.08ms elapsed=592.43s tid=0x0000007ab097e100 nid=0xa103 runnable  

"GC Thread#2" os_prio=31 cpu=5.01ms elapsed=592.43s tid=0x0000007ab097de00 nid=0xa303 runnable  

"GC Thread#1" os_prio=31 cpu=5.52ms elapsed=592.43s tid=0x0000007ab097db00 nid=0xa403 runnable  

"VM Periodic Task Thread" os_prio=31 cpu=64.28ms elapsed=592.59s tid=0x0000007ab097c000 nid=0x7003 waiting on condition  

"VM Thread" os_prio=31 cpu=10.62ms elapsed=592.60s tid=0x000000010316c490 nid=0x4703 runnable  

"G1 Service" os_prio=31 cpu=32.07ms elapsed=592.62s tid=0x0000000103169f20 nid=0x5303 runnable  

"G1 Refine#0" os_prio=31 cpu=0.02ms elapsed=592.62s tid=0x00000001031695b0 nid=0x4203 runnable  

"G1 Conc#0" os_prio=31 cpu=2.68ms elapsed=592.62s tid=0x00000001031673e0 nid=0x4003 runnable  

"G1 Main Marker" os_prio=31 cpu=0.31ms elapsed=592.62s tid=0x0000000103166930 nid=0x3703 runnable  

"GC Thread#0" os_prio=31 cpu=5.02ms elapsed=592.62s tid=0x0000000103166140 nid=0x3303 runnable  

JNI global refs: 15, weak refs: 0

```