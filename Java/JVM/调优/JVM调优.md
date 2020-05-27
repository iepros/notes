# JVM内存模型及垃圾收集算法

## 根据java虚拟机规范，JVM将内存划分为：

- New（年轻代）

  用来存放JVM刚分配的Java对象

- Tenured（老年代）

  年轻代中经过垃圾回收没有回收掉的对象呗copy到老年代

- Perm（永久代）

  存放Class、Method元信息，其大小跟项目的规模、类方法的量有关，一般设置为128M就足够，设置原则是预留30%的空间

其中**New**和**Tenured**属于**堆内存**，堆内存会从JVM启动参数（**-Xmx:3G**）指定的内存中分配，**Perm不属于堆内存，有虚拟机直接分配**。单可以通过**-XX:PermSize -XX:MaxPermSize**等参数调整其大小。

New又分为几个部分：

- Eden：用来存放JVM刚分配的对象
- Survivor1
- Survivor2：这两个**Survivor空间一样大**，当Eden中的对象经过垃圾回收没有被回收掉时，会在两个Survivor之间来回copy，当满足某个条件，比如copy次数，就会被copy到Tenured。显然，**Survivor只是增加了对象再年轻代中的逗留时间**，增加了**被垃圾回收的可能性**。

## 垃圾回收算法



垃圾回收算法分为三类，都基于**标记-清除（复制）算法**

- Serial算法（单线程）
- 并行算法
- 并发算法

JVM会根据机器的硬件配置对每个内存代选择适合的回收算法，比如，如果机器多于1个核，会对年轻代选择并行算法，关于选择细节请参考JVM调优文档。

 稍微解释下的是，**并行算法是用多线程进行垃圾回收，回收期间会暂停程序的执行，而并发算法，也是多线程回收，但期间不停止应用执行。**所以，并发算法适用于交互性高的一些程序。经过观察，并发算法会减少年轻代的大小，其实就是使用了一个大的年老代，这反过来跟并行算法相比吞吐量相对较低。

还有一个问题是，垃圾回收动作何时执行？

- 当年轻代存满时，会引发一次普通GC，该GC进回收年轻代。需要强调的是：**年轻代满是指Eden代满，Survivor满不会引发GC**
- 当老年代满时，会引发Full GC，Full GC将会同时回收年轻代、老年代
- 当永久代满时，也会引发Full GC，导致Class、Method元信息的卸载

另一个问题是，合适会抛出OutOfMemoryException，并不是内存被好空的时候才抛出

- JVM98%的时间都花费在内存回收
- 每次回收的内存小于2%

满足这两个条件将触发OutOfMemoryException，这将会留给系统一个微小的间隙以做一些Down之前的操作，比如手动打印Heap Dump

# 内存泄漏及解决方法

## 系统崩溃前的一些现象

- 每次垃圾回收的时间越来越长,由之前的10ms延长到50ms左右，Full GC的时间也有之前的0.5s延长到4，5s
- Full GC的次数越来越多，最频繁时隔不到一分钟就进行一次Full GC
- 老年代的内存越来越大并且每次Full GC后老年代没有内存释放

之后系统会无法响应新的请求，主键到达OutOfMemoryError的临界值

### 生成堆的dump文件

通过JMX的MBean生成当前的Heap信息，大小为一个3G（整个堆的大小）的hprof文件，如果没有启动JMX，可以通过Java的jmap命令来生成该文件

#### 分析dump文件

如何打开这个3G的堆信息文件：

- Visual VM
- IBM HeapAnalyzer
- JDK 自带的Hprof工具
- eclipse静态内存分析工具：Mat

#### 分析内存泄漏

通过Mat我们能清楚地看到，哪些对象被怀疑为内存泄漏，哪些对象占的空间最大及对象的调用关系。针对本案，在ThreadLocal中有很多的JbpmContext实例，经过调查是JBPM的Context没有关闭所致。

 另，通过Mat或JMX我们还可以分析线程状态，可以观察到线程被阻塞在哪个对象上，从而判断系统的瓶颈

#### 回归问题

**Q**：为什么崩溃前垃圾回收的时间越来越长？

  **A**:根据内存模型和垃圾回收算法，垃圾回收分两部分：内存标记、清除（复制），标记部分只要内存大小固定时间是不变的，变的是复制部分，因为每次垃圾回收都有一些回收不掉的内存，所以增加了复制量，导致时间延长。所以，垃圾回收的时间也可以作为判断内存泄漏的依据

  **Q**：为什么Full GC的次数越来越多？

  **A**：因此内存的积累，逐渐耗尽了年老代的内存，导致新对象分配没有更多的空间，从而导致频繁的垃圾回收

  **Q**:为什么年老代占用的内存越来越大？

  **A**:因为年轻代的内存无法被回收，越来越多地被Copy到年老代

# 性能调优

## 线程池

大多数JVM6上的应用采用的线程池都是JDK自带的线程池，之所以把成熟的Java线程池进行罗嗦说明，是因为该线程池的行为与我们想象的有点出入。Java线程池有几个重要的配置参数：

- corePoolSize：核心线程数（最新线程数）
- maximumPoolSize：最大线程数，超过这个数量的任务会被拒绝，用户可以通过RejectedExecutionHandler接口自定义处理方式
- keepAliveTime：线程保持活动的时间
- workQueue：工作队列，存放执行的任务

Java线程池需要传入一个Queue参数（workQueue）用来存放执行的任务，而对Queue的不同选择，线程池有完全不同的行为：

- SynchronousQueue：一个无容量的等待队列，一个线程的insert操作必须等待另一线程的
- LinkedBlockingQueue：无界队列，采用该Queue，线程池将忽略` maximumPoolSize参数，仅用corePoolSize的线程处理所有的任务，未处理的任务便在`LinkedBlockingQueue中排队
- ArrayBlockingQueue：`有界队列，在有界队列和` maximumPoolSize的作用下，程序将很难被调优：更大的Queue和小的maximumPoolSize将导致CPU的低负载；小的Queue和大的池，Queue就没起动应有的作用

其实我们的要求很简单，希望线程池能跟连接池一样，能设置最小线程数、最大线程数，当最小数<任务<最大数时，应该分配新的线程处理；当任务>最大数时，应该等待有空闲线程再处理该任务

但线程池的设计思路是，任务应该放到Queue中，当Queue放不下时再考虑用新线程处理，如果Queue满且无法派生新线程，就拒绝该任务。设计导致“先放等执行”、“放不下再执行”、“拒绝不等待”。所以，根据不同的Queue参数，要提高吞吐量不能一味地增大maximumPoolSize

当然，要达到我们的目标，必须对线程池进行一定的封装，幸运的是ThreadPoolExecutor中留了足够的自定义接口以帮助我们达到目标。我们封装的方式是：

- 以SynchronousQueue作为参数，使maximumPoolSize发挥作用，以防止线程被无限制的分配，同时可以通过提高maximumPoolSize来提高系统吞吐量
- 自定义一个RejectedExecutionHandler，当线程数超过maximumPoolSize时进行处理，处理方式为隔一段时间检查线程池是否可以执行新Task，如果可以把拒绝的Task重新放入到线程池，检查的时间依赖keepAliveTime的大小

## 连接池

在使用org.apache.commons.dbcp.BasicDataSource的时候，因为之前采用了默认配置，所以当访问量大时，通过JMX观察到很多Tomcat线程都阻塞在BasicDataSource使用的Apache ObjectPool的锁上，直接原因当时是因为BasicDataSource连接池的最大连接数设置的太小，默认的BasicDataSource配置，仅使用8个最大连接

我还观察到一个问题，当较长的时间不访问系统，比如2天，DB上的Mysql会断掉所以的连接，导致连接池中缓存的连接不能用。为了解决这些问题，我们充分研究了BasicDataSource，发现了一些优化的点：

- Mysql默认支持100个链接，所以每个连接池的配置要根据集群中的机器数进行，如有2台服务器，可每个设置为60
- initialSize：参数是一直打开的连接数
- minEvictableIdleTimeMillis：该参数设置每个连接的空闲时间，超过这个时间连接将被关闭
- timeBetweenEvictionRunsMillis：后台线程的运行周期，用来检测过期连接
- maxActive：最大能分配的连接数
- maxIdle：最大空闲数，当连接使用完毕后发现连接数大于maxIdle，连接将被直接关闭。只有initialSize < x < maxIdle的连接将被定期检测是否超期。这个参数主要用来在峰值访问时提高吞吐量
- initialSize是如何保持的？经过研究代码发现，BasicDataSource会关闭所有超期的连接，然后再打开initialSize数量的连接，这个特性与minEvictableIdleTimeMillis、timeBetweenEvictionRunsMillis一起保证了所有超期的initialSize连接都会被重新连接，从而避免了Mysql长时间无动作会断掉连接的问题

## JVM参数

在JVM启动参数中，可以设置跟内存、垃圾回收相关的一些参数设置，默认情况不做任何设置JVM会工作的很好，但对一些配置很好的Server和具体的应用必须仔细调优才能获得最佳性能。通过设置我们希望达到一些目标：

- **GC的时间足够的小**
- **GC的次数足够的少**
- **发生Full GC的周期足够的长**

前两个目前是相悖的，要想GC时间小必须要一个更小的堆，要保证GC次数足够少，必须保证一个更大的堆，我们只能取其平衡

- 针对JVM堆的设置，一般可以通过-Xms -Xmx限定其最小、最大值，**为了防止垃圾收集器在最小、最大之间收缩堆而产生额外的时间，我们通常把最大、最小设置为相同的值**

- **年轻代和年老代将根据默认的比例（1：2）分配堆内存**，可以通过调整二者之间的比率NewRadio来调整二者之间的大小，也可以针对回收代，比如年轻代，通过 -XX:newSize -XX:MaxNewSize来设置其绝对大小。同样，为了防止年轻代的堆收缩，我们通常会把-XX:newSize -XX:MaxNewSize设置为同样大小

- 年轻代和年老代设置多大才算合理？这个我问题毫无疑问是没有答案的，否则也就不会有调优。我们观察一下二者大小变化有哪些影响

  1. **更大的年轻代必然导致更小的年老代，大的年轻代会延长普通GC的周期，但会增加每次GC的时间；小的年老代会导致更频繁的Full GC**
  2. **更小的年轻代必然导致更大年老代，小的年轻代会导致普通GC很频繁，但每次的GC时间会更短；大的年老代会减少Full GC的频率**
  3. 如何选择应该依赖应用程序**对象生命周期的分布情况**：如果应用存在大量的临时对象，应该选择更大的年轻代；如果存在相对较多的持久对象，年老代应该适当增大。但很多应用都没有这样明显的特性，在抉择时应该根据以下两点：
     1. 本着Full GC尽量少的原则，让年老代尽量缓存常用对象，JVM的默认比例1：2也是这个道理
     2. 通过观察应用一段时间，看其他在峰值时年老代会占多少内存，在不影响Full GC的前提下，根据实际情况加大年轻代，比如可以把比例控制在1：1。但应该给年老代至少预留1/3的增长空间

- **在配置较好的机器上（比如多核、大内存），可以为年老代选择并行收集算法**： **-XX:+UseParallelOldGC** ，默认为Serial收集

- 线程堆栈的设置：每个线程默认会开启1M的堆栈，用于存放栈帧、调用参数、局部变量等，对大多数应用而言这个默认值太了，一般256K就足用。理论上，在内存不变的情况下，减少每个线程的堆栈，可以产生更多的线程，但这实际上还受限于操作系统

- 可以通过下面的参数打Heap Dump信息

  1. -XX:HeapDumpPath
  2. -XX:+PrintGCDetails
  3. -XX:+PrintGCTimeStamps
  4. -Xloggc:/usr/aaa/dump/heap_trace.txt

  通过下面参数可以控制OutOfMemoryError时打印堆的信息

  1. -XX:+HeapDumpOnOutOfMemoryError

  请看一下一个时间的Java参数配置：（服务器：Linux 64Bit，8Core×16G）：

  **AVA_OPTS="$JAVA_OPTS -server -Xms3G -Xmx3G -Xss256k -XX:PermSize=128m -XX:MaxPermSize=128m -XX:+UseParallelOldGC -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/usr/aaa/dump -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -Xloggc:/usr/aaa/dump/heap_trace.txt -XX:NewSize=1G -XX:MaxNewSize=1G"**

  经过观察该配置非常稳定，每次普通GC的时间在10ms左右，Full GC基本不发生，或隔很长很长的时间才发生一次

  通过分析dump文件可以发现，每个1小时都会发生一次Full GC，经过多方求证，只要在JVM中开启了JMX服务，JMX将会1小时执行一次Full GC以清除引用，关于这点请参考附件文档

  

# 调优方法

## 调优原则

1. 多数的Java应用不需要在服务器上进行GC优化；
2. 多数导致GC问题的Java应用，都不是因为我们参数设置错误，而是代码问题；
3. 在应用上线之前，先考虑将机器的JVM参数设置到最优（最适合）；
4. 减少创建对象的数量；
5. 减少使用全局变量和大对象；
6. GC优化是到最后不得已才采用的手段；
7. 在实际使用中，分析GC情况优化代码比优化GC参数要多得多；



## **GC优化的目的有两个**

（http://www.360doc.com/content/13/0305/10/15643_269388816.shtml）

1. **将转移到老年代的对象数量降低到最小；**
2. **减少full GC的执行时间；**

为了达到上面的目的，一般地，你需要做的事情有：

1. 减少使用全局变量和大对象；
2. 调整新生代的大小到最合适；
3. 设置老年代的大小为最合适；
4. 选择合适的GC收集器；

真正熟练的使用GC调优，是建立在多次进行GC监控和调优的实战经验上的，进行监控和调优的一般步骤为：

1. **监控GC的状态**

2. **分析结果，判断是否需要优化**

3. **调整GC类型和内存分配**

4. **不断的分析和调整**

5. **全面应用参数**

   

# 调优实战

## 实例一

笔者昨日发现部分开发测试机器出现异常：java.lang.OutOfMemoryError: GC overhead limit exceeded，这个异常代表：

GC为了释放很小的空间却耗费了太多的时间，其原因一般有两个：

- 堆太小
- 有死循环或大对象

首先排除了第2个原因，因为这个应用同时是在线上运行的，如果有问题，早就挂了。所以怀疑是这台机器中堆设置太小

使用ps -ef |grep "java"查看，发现：

`-Xms768m -Xmx768m -XX:NewSize=320m -XX:MaxNewSize=320m`

该应用的堆区设置只有768m，而机器内存有2g，机器上只跑这一个java应用，没有其他需要占用内存的地方。另外，这个应用比较大，需要占用的内存也比较多

笔者通过上面的情况判断，只需要改变堆中各区域的大小设置即可，于是改成下面的情况：

`-Xms1280m -Xmx1280m -XX:NewSize=500m -XX:MaxNewSize=500m`

跟踪运行情况发现，相关异常没有再出现



## 实例二

（http://www.360doc.com/content/13/0305/10/15643_269388816.shtml）

一个服务系统，**经常出现卡顿，分析原因，发现Full GC时间太长**：

jstat -gcutil:

S0   S1   E   O    P     YGC YGCT FGC FGCT  GCT

12.16 0.00 5.18 63.78 20.32  54  2.047 5   6.946  8.993 

分析上面的数据，发现Young GC执行了54次，耗时2.047秒，每次Young GC耗时37ms，在正常范围，**而Full GC执行了5次，耗时6.946秒，每次平均1.389s，数据显示出来的问题是：Full GC耗时较长**，分析该系统的是指发现，NewRatio=9，也就是说，新生代和老生代大小之比为1:9，这就是问题的原因：

1，新生代太小，导致对象提前进入老年代，触发老年代发生Full GC；

2，老年代较大，进行Full GC时耗时较大；

优化的方法是调整NewRatio的值，调整到4，发现Full GC没有再发生，只有Young GC在执行。这就是把对象控制在新生代就清理掉，没有进入老年代（这种做法对一些应用是很有用的，但并不是对所有应用都要这么做）



## 实例三

一应用在性能测试过程中，发现内存占用率很高，Full GC频繁，使用sudo -u admin -H  jmap -dump:format=b,file=文件名.hprof pid 来dump内存，生成dump文件，并使用Eclipse下的mat差距进行分析，发现：

线程存在问题，队列LinkedBlockingQueue所引用的大量对象并未释放，导致整个线程占用内存高达378m，此时通知开发人员进行代码优化，将相关对象释放掉即可





# 监控GC工具

## JPS

jps命令用于查询正在运行的JVM进程，常用的参数为：

- -q:只输出LVMID，省略主类的名称
-  -m:输出虚拟机进程启动时传给主类main()函数的参数
- -l:输出主类的全类名，如果进程执行的是Jar包，输出Jar路径
-  -v:输出虚拟机进程启动时JVM参数

**命令格式:jps [option] [hostid]**



`mxf@MaXiangFeng ~ % jps -l`

`57200 org.jetbrains.jps.cmdline.Launcher`

`53121 org.jetbrains.jps.cmdline.Launcher`

`57778 sun.tools.jps.Jps`

`42195` 

`436` 



## JSTAT

jstat可以实时显示本地或远程JVM进程中类装载、内存、垃圾收集、JIT编译等数据（如果要显示远程JVM信息，需要远程主机开启RMI支持）。如果在服务启动时没有指定启动参数-verbose:gc，则可以用jstat实时查看gc情况。

jstat有如下选项：

- -class:监视类装载、卸载数量、总空间及类装载所耗费的时间
-   -gc:监听Java堆状况，包括Eden区、两个Survivor区、老年代、永久代等的容量，以用空间、GC时间合计等信息
-  -gccapacity:监视内容与-gc基本相同，但输出主要关注java堆各个区域使用到的最大和最小空间
-   -gcutil:监视内容与-gc基本相同，但输出主要关注已使用空间占总空间的百分比
-   -gccause:与-gcutil功能一样，但是会额外输出导致上一次GC产生的原因
-   -gcnew:监视新生代GC状况
-   -gcnewcapacity:监视内同与-gcnew基本相同，输出主要关注使用到的最大和最小空间
-   -gcold:监视老年代GC情况
-   -gcoldcapacity:监视内同与-gcold基本相同，输出主要关注使用到的最大和最小空间
-   -gcpermcapacity:输出永久代使用到最大和最小空间
-   -compiler:输出JIT编译器编译过的方法、耗时等信息
-   -printcompilation:输出已经被JIT编译的方法

**命令格式:jstat [option vmid [interval[s|ms] [count]]]**

jstat可以监控远程机器，命令格式中VMID和LVMID特别说明：如果是本地虚拟机进程，VMID和LVMID是一致的，如果是远程虚拟机进程，那么VMID格式是: [protocol:][//]lvmid[@hostname[:port]/servername]，如果省略interval和count，则只查询一次

mxf@MaXiangFeng ~ % jstat -gc 57954 1000 5
 S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT   
38400.0 47104.0  0.0    0.0   556544.0 41742.7   276992.0   128150.1  164224.0 156125.9 20096.0 18910.1     34    0.742   5      1.045    1.787
38400.0 47104.0  0.0    0.0   556544.0 42186.9   276992.0   128150.1  164224.0 156125.9 20096.0 18910.1     34    0.742   5      1.045    1.787
38400.0 47104.0  0.0    0.0   556544.0 42219.2   276992.0   128150.1  164224.0 156125.9 20096.0 18910.1     34    0.742   5      1.045    1.787
38400.0 47104.0  0.0    0.0   556544.0 42221.2   276992.0   128150.1  164224.0 156125.9 20096.0 18910.1     34    0.742   5      1.045    1.787
38400.0 47104.0  0.0    0.0   556544.0 42221.2   276992.0   128150.1  164224.0 156125.9 20096.0 18910.1     34    0.742   5      1.045    1.787

**jstat -gc 57954 1000 5代表：搜集vid为57954的java进程的整体gc状态，每1000ms收集一次，共收集5次；**

S0C：S0区容量（S1区相同，略）

S0U：S0区已使用

EC：E区容量

EU：E区已使用

OC：老年代容量

OU：老年代已使用

PC：Perm容量

PU：Perm区已使用

YGC：Young GC（Minor GC）次数

YGCT：Young GC总耗时

FGC：Full GC次数

FGCT：Full GC总耗时

GCT：GC总耗时

## JINFO

用于查询当前运行这的JVM属性和参数的值。

jinfo可以使用如下选项：

- -flag:显示未被显示指定的参数的系统默认值
- -flag [+|-]name或-flag name=value: 修改部分参数
-  -sysprops:打印虚拟机进程的System.getProperties()

**命令格式:jinfo [option] pid**

## JMAP

用于显示当前Java堆和永久代的详细信息（如当前使用的收集器，当前的空间使用率等）

- **-dump:生成java堆转储快照**
- **-heap:显示java堆详细信息(只在Linux/Solaris下有效)**
- **-F:当虚拟机进程对-dump选项没有响应时，可使用这个选项强制生成dump快照(只在Linux/Solaris下有效)**
- -finalizerinfo:显示在F-Queue中等待Finalizer线程执行finalize方法的对象(只在Linux/Solaris下有效)
-  -histo:显示堆中对象统计信息
- -permstat:以ClassLoader为统计口径显示永久代内存状态(只在Linux/Solaris下有效)

**命令格式:jmap [option] vmid**

其中前面3个参数最重要，如：

1. 查看对详细信息：sudo jmap -heap 309
2. 生成dump文件： sudo jmap -dump:file=./test.prof 309
3. 部分用户没有权限时，采用admin用户：sudo -u admin -H jmap -dump:format=b,file=文件名.hprof pid
4. 查看当前堆中对象统计信息：sudo jmap -histo 309：该命令显示3列，分别为对象数量，对象大小，对象名称，通过该命令可以查看是否内存中有大对象；
5. 有的用户可能没有jmap权限：sudo -u admin -H jmap -histo 309 | less

## JHAT

用于分析使用jmap生成的dump文件，是JDK自带的工具，**使用方法为： jhat -J -Xmx512m [file]**

不过jhat没有mat好用，推荐使用mat（Eclipse插件： http://www.eclipse.org/mat ），mat速度更快，而且是图形界面。

## JSTACK

用于生成当前JVM的所有线程快照，线程快照是虚拟机每一条线程正在执行的方法,目的是定位线程出现长时间停顿的原因。

- -F:当正常输出的请求不被响应时，强制输出线程堆栈
- -l:除堆栈外，显示关于锁的附加信息
- -m:如果调用到本地方法的话，可以显示C/C++的堆栈

**命令格式:jstack [option] vmid**



















 





