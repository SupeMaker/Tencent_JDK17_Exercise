# 1 测试用例

## 1.1 测试代码

```java

import java.util.Random;
import java.util.concurrent.atomic.LongAdder;

public class GCTest {
    private static Random random = new Random();
    public static void main(String[] var0) {
        // 当前毫秒时间戳
        long startMillis = System.currentTimeMillis();
        // 持续运行毫秒数; 可根据需要进行修改，这里选择运行30s
        long timeoutMillis = 30000;
        // 结束时间戳
        long endMillis = startMillis + timeoutMillis;
        LongAdder counter = new LongAdder();
        System.out.println("正在执行...");
        int size=2000;
        Object[] arr = new Object[size];
        while (System.currentTimeMillis() < endMillis) {
            int index = random.nextInt(2*size);
            if(index < size) {
                arr[index] = new byte[100*1024];
            } else {
                byte[] garbage = new byte[100 * 1024];
            }
            counter.increment();
        }
        System.out.println("执行结束!共生成对象次数:" + counter.longValue());
    }
}
/*
-XX:+UseSerialGC -Xlog:gc*=info,gc+heap=debug,gc+ergo*=trace,gc+parse=debug,safepoint:log/gc-oomHeap.log:uptime,level,tags -Xms1024M -Xmx1024M -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=heap/heapdump.hprof
*/
```

# 2 经典垃圾收集器比较

## 摘要：

| 收集器            | 每次GC的用户线程平均暂停时间/ms | 每次GC的用户线程最大暂停时间/ms | 总暂停时间/s | 总并发时间/s | 吞吐量  |
| ----------------- | ------------------------------- | ------------------------------- | ------------ | ------------ | ------- |
| Serial            | 26.7                            | 270                             | 14.304       | 0            | 53.236% |
| Parallel Scavenge | 28.3                            | 480                             | 19.810       | 没有统计     | 35.612% |
| G1                | 14.2                            | 130                             | 17.364       | 2.891        | 42.586% |
| Shenandoah        | 0.291                           | 2.51                            | 0.413        | 6.694        | 98.65%  |
| ZGC               | 0.0101                          | 0.092                           | 0.00685      | 22.462       | 99.977% |



## 2.1 SerialGC

### 2.1.1 Serial收集器运行示意图：

![Serial收集器运行示意图](/img/Serial收集器运行示意图.png)

Serial 收集器是一个单线程的收集器，只是用一个处理器或者收集线程去完成垃圾收集工作。它在新生代采用复制算法，在老年代使用标记-整理算法。在工作的时候，必须暂停其他工作的线程，也就是"Stop The World"。

### 2.1.2 虚拟机参数

```
-XX:+UseSerialGC -Xlog:gc*=info:log/gc-oomHeap.log:uptime,level,tags -Xms1024M -Xmx1024M
```

### 2.1.3 日志分析

完整的log日志将在 ./Serial.log文件给出

#### 2.1.3.1 gc init

接下来这一段在每个GC收集器的日志都差不多，这里统一解释

```
[0.045s][info][gc] Using Serial # 使用的垃圾收集器，这里使用Serial收集器
[0.047s][info][gc,init] Version: 17.0.11-internal+0-adhoc.damowang.TencentKona-17 (release) # JVM版本信息，这里显示的是17.0.11（内部版本），基于腾讯Kona JDK17,自行构建的release版本
[0.048s][info][gc,init] CPUs: 12 total, 12 available # 我的电脑是6核12线程，这里检测到12个CPU核心
[0.048s][info][gc,init] Memory: 3829M # JVM内存总容量，总物理内存大小。
[0.048s][info][gc,init] Large Page Support: Disabled # 未启用大页支持
...
[0.048s][info ][gc,init] Heap Min Capacity: 1G # 最小堆大小1G
[0.048s][info ][gc,init] Heap Initial Capacity: 1G # 初始堆大小1G
[0.048s][info ][gc,init] Heap Max Capacity: 1G # 最大堆大小1G
...
[0.048s][info ][gc,metaspace] Narrow klass base: 0x0000000000000000, Narrow klass shift: 3, Narrow klass range: 0x140000000 # 在这句话之后程序测试代码就正式执行了从这里开始计时

```

#### 2.1.3.2 Young GC和Full GC

一次Young GC和Full GC日志

```
[0.048s][info][gc,metaspace] Narrow klass base: 0x0000000000000000, Narrow klass shift: 3, Narrow klass range: 0x140000000
# 程序开始
...
[1.384s][info][gc,start       ] GC(15) Pause Young (Allocation Failure)
[1.416s][info][gc,heap        ] GC(15) DefNew: 314521K(314560K)->34905K(314560K) Eden: 279616K(279616K)->0K(279616K) From: 34905K(34944K)->34905K(34944K)
[1.416s][info][gc,heap        ] GC(15) Tenured: 598330K(699072K)->680242K(699072K)
[1.416s][info][gc             ] GC(15) Pause Young (Allocation Failure) 891M->698M(989M) 32.364ms
[1.416s][info][gc,cpu         ] GC(15) User=0.01s Sys=0.02s Real=0.03s
...
[1.446s][info][gc,start       ] GC(17) Pause Full (Allocation Failure)
[1.446s][info][gc,phases,start] GC(17) Phase 1: Mark live objects
[1.447s][info][gc,phases      ] GC(17) Phase 1: Mark live objects 1.113ms
[1.447s][info][gc,phases,start] GC(17) Phase 2: Compute new object addresses
[1.449s][info][gc,phases      ] GC(17) Phase 2: Compute new object addresses 1.523ms
[1.449s][info][gc,phases,start] GC(17) Phase 3: Adjust pointers
[1.450s][info][gc,phases      ] GC(17) Phase 3: Adjust pointers 0.893ms
[1.450s][info][gc,phases,start] GC(17) Phase 4: Move objects
[1.490s][info][gc,phases      ] GC(17) Phase 4: Move objects 40.100ms
[1.490s][info][gc,heap        ] GC(17) DefNew: 314521K(314560K)->0K(314560K) Eden: 279616K(279616K)->0K(279616K) From: 34905K(34944K)->0K(34944K)
[1.490s][info][gc,heap        ] GC(17) Tenured: 680242K(699072K)->223171K(699072K)
[1.490s][info][gc             ] GC(17) Pause Full (Allocation Failure) 971M->217M(989M) 43.882ms
[1.490s][info][gc,cpu         ] GC(17) User=0.04s Sys=0.00s Real=0.04s
...
# 程序结束
[1.559s][info][gc,heap,exit   ] Heap
```

1. [1.384s][info][gc,start       ] GC(15) Pause Young (Allocation Failure)

由于分配内存失败，导致针对年轻代的第16次收集。

2. [1.416s][info][gc,heap        ] GC(15) DefNew: 314521K(314560K)->34905K(314560K) Eden: 279616K(279616K)->0K(279616K) From: 34905K(34944K)->34905K(34944K)

DefNew是新生代，经过gc之后，堆内存从314521K变成了34905K，最大堆内存为314560K。

3. [1.416s][info][gc,heap        ] GC(15) Tenured: 598330K(699072K)->680242K(699072K)

可以看到，执行Young GC的时候，新生代的对象并没有全部被回收，有一部分对象进入了老年代，这里老年堆中被使用的内存从598330K变成了680242K。

4. [1.416s][info][gc             ] GC(15) Pause Young (Allocation Failure) 891M->698M(989M) 32.364ms
   [1.416s][info][gc,cpu         ] GC(15) User=0.01s Sys=0.02s Real=0.03s

891M->698M(989M)，本次Young GC，回收了891-698=179M堆内存，可用的堆内存有989M。

Real=0.03s，说明此次GC中暂停用户程序的时间为0.03s，其实这个时间跟 32.364ms是对应的。

5. [1.446s][info][gc,start       ] GC(17) Pause Full (Allocation Failure)

由于Young GC回收的内存无法满足分配的内存，并且老年代的内存也很满的时候，就会进行了一次整堆收集。 

6. [1.446s][info][gc,phases,start] GC(17) Phase 1: Mark live objects
   [1.447s][info][gc,phases      ] GC(17) Phase 1: Mark live objects 1.113ms

开始标记所有存活的对象，花费了1.113ms。

7. [1.447s][info][gc,phases,start] GC(17) Phase 2: Compute new object addresses
   [1.449s][info][gc,phases      ] GC(17) Phase 2: Compute new object addresses 1.523ms。

开始计算存活对象被复制到的新地址，花费了1.523ms。

8. [1.449s][info][gc,phases,start] GC(17) Phase 3: Adjust pointers
   [1.450s][info][gc,phases      ] GC(17) Phase 3: Adjust pointers 0.893ms。

开始并调整所有对存活对象引用的指针，以指向对象的新位置，耗时 0.893ms。

9. [1.450s][info][gc,phases,start] GC(17) Phase 4: Move objects
   [1.490s][info][gc,phases      ] GC(17) Phase 4: Move objects 40.100ms。

移动对象到新地址，花费了 40.100ms。

10. [1.490s][info][gc,heap        ] GC(17) DefNew: 314521K(314560K)->0K(314560K) Eden: 279616K(279616K)->0K(279616K) From: 34905K(34944K)->0K(34944K)
    [1.490s][info][gc,heap        ] GC(17) Tenured: 680242K(699072K)->223171K(699072K)

这次Full GC，新生代的堆内存314521K全部回收；老年代的堆内存从680242K回收到了223171K。

11. [1.490s][info][gc             ] GC(17) Pause Full (Allocation Failure) 971M->217M(989M) 43.882ms
    [1.490s][info][gc,cpu         ] GC(17) User=0.04s Sys=0.00s Real=0.04s

user 部分表示所有 GC 线程消耗的 CPU 时间；sys 部分表示系统调用和系统等待事件消耗的时间。real 则表示应用程序暂停的时间。在Serial收集器中，real与等于user+system。

#### 2.1.3.3 运行30s后的分析

借助 GCeasy 来分析jvm日志。



![Serial1](/img/Serial1.png)

新生代(Young Generation) 内存被分配了307.19MB，老年代(Old Generation) 内存被分配了 682.69MB，整个堆的内存被使用了995.5MB。



![Serial2](/img/Serial2.png)

Throughput(吞吐量): 在测试中，Serial收集器的吞吐量为 53.236%。

Avg Pause GC Time(每次STW的平均时间):26.7ms。

Max Pause GC Time(最大STW时间):270ms。



![image-20240822135556571](/img/Serial3.png)

每次GC所耗费的时间，包括暂停阶段和并发阶段。可以看到，程序运行了5秒之后，每次GC时间趋于平稳，Full GC的平均时间比Young GC的时间要长。



![Serial4](/img/Serial4.png)

根据 ‘real’ time时间进行统计，根据 GC Pause Statistics可以看到，Pause total time(总暂停时间)为14.304秒，程序运行的时间为30.588s，则吞吐量：

吞吐量= （30.588-14.304）/30.588 = 53.236%，跟上面的数据一致。由 GC Average Time(ms)可知，Full GC的平均时间要比 Minor GC的平均时间要长。



## 2.2 Parallel Scavenge收集器

### 2.2.1 Parallel Scavenge收集器运行示意图

![Parallel Scanvenge收集器运行示意图](/img/Parallel Scanvenge收集器运行示意图.png)Parallel收集器对新生代使用“标记-复制”算法，对老年代使用“标记-整理”算法。在GC时间执行期间，多个线程并行地标记和清除垃圾。

### 2.2.2 虚拟机参数

```
-XX:+UseParallelGC -Xlog:gc*=info:log/gc-oomHeap.log:uptime,level,tags -Xms1024M -Xmx1024M -XX:+UseAdaptiveSizePolicy
```

### 2.2.3 日志分析

#### 2.2.3.1 Young GC

没触发Full GC

```
[0.007s][info][gc,init] Parallel Workers: 10 #线程数为10
[0.003s][info][gc,metaspace] Narrow klass base: 0x0000000000000000, Narrow klass shift: 3, Narrow klass range: 0x140000000
#程序开始
...
[1.045s][info][gc,start    ] GC(7) Pause Young (Allocation Failure)
[1.153s][info][gc,heap     ] GC(7) PSYoungGen: 305090K(305152K)->44006K(305152K) Eden: 261120K(261120K)->0K(261120K) From: 43970K(44032K)->44006K(44032K)
[1.153s][info][gc,heap     ] GC(7) ParOldGen: 508175K(699392K)->585703K(699392K)
[1.153s][info][gc          ] GC(7) Pause Young (Allocation Failure) 794M->614M(981M) 107.501ms
[1.153s][info][gc,cpu      ] GC(7) User=0.18s Sys=0.88s Real=0.11s
...
[1.171s][info][gc,heap,exit] Heap
```

1. [1.045s][info][gc,start    ] GC(7) Pause Young (Allocation Failure)

由于分配内存失败，gc开启对年轻代第8次收集。

2. [1.153s][info][gc,heap     ] GC(7) PSYoungGen: 305090K(305152K)->44006K(305152K) Eden: 261120K(261120K)->0K(261120K) From: 43970K(44032K)->44006K(44032K)

PSYoungGen是新生代，经过gc之后，堆内存从305090K变成了44006K，最大堆内存为305152K。

3. [1.153s][info][gc,heap     ] GC(7) ParOldGen: 508175K(699392K)->585703K(699392K)

可以看到，执行Young GC的时候，有部分新生代的对象进入了老年代中。

4. [1.153s][info][gc          ] GC(7) Pause Young (Allocation Failure) 794M->614M(981M) 107.501ms
   [1.153s][info][gc,cpu      ] GC(7) User=0.18s Sys=0.88s Real=0.11s

在Parallel收集器中，real约等于 (user+sys)/线程数，这里的线程数是10，这里说明用户线程暂停了0.11s。

#### 2.2.3.2 Full GC

在前面样例的基础上，增加对象的个数，将size设置为5000，即可触发出Full GC。

```
[0.005s][info][gc,metaspace] Narrow klass base: 0x0000000000000000, Narrow klass shift: 3, Narrow klass range: 0x140000000
...
[1.131s][info][gc,start    ] GC(6) Pause Full (Ergonomics)
[1.131s][info][gc,phases,start] GC(6) Marking Phase
[1.144s][info][gc,phases      ] GC(6) Marking Phase 12.522ms
[1.144s][info][gc,phases,start] GC(6) Summary Phase
[1.144s][info][gc,phases      ] GC(6) Summary Phase 0.035ms
[1.144s][info][gc,phases,start] GC(6) Adjust Roots
[1.144s][info][gc,phases      ] GC(6) Adjust Roots 0.384ms
[1.144s][info][gc,phases,start] GC(6) Compaction Phase
[1.216s][info][gc,phases      ] GC(6) Compaction Phase 72.158ms
[1.216s][info][gc,phases,start] GC(6) Post Compact
[1.220s][info][gc,phases      ] GC(6) Post Compact 3.285ms
[1.220s][info][gc,heap        ] GC(6) PSYoungGen: 43466K(305664K)->0K(305664K) Eden: 0K(262144K)->0K(262144K) From: 43466K(43520K)->0K(43520K)
[1.220s][info][gc,heap        ] GC(6) ParOldGen: 583615K(699392K)->396208K(699392K)
[1.220s][info][gc             ] GC(6) Pause Full (Ergonomics) 612M->386M(981M) 88.606ms
[1.220s][info][gc,cpu         ] GC(6) User=0.77s Sys=0.00s Real=0.09s
...
[1.237s][info][gc,heap,exit   ] Heap
```

1. [1.131s][info][gc,start    ] GC(6) Pause Full (Ergonomics)

Full GC的开始，参数中设置了-XX:+UseAdaptiveSizePolicy，自动设置对空间各分代区域大小。

2. [1.131s][info][gc,phases,start] GC(6) Marking Phase
   [1.144s][info][gc,phases      ] GC(6) Marking Phase 12.522ms

开始标记所有存活的对象，花费了12.522ms。

7. [1.144s][info][gc,phases,start] GC(6) Summary Phase
   [1.144s][info][gc,phases      ] GC(6) Summary Phase 0.035ms

汇总标记阶段的结果，耗时0.035ms。

8. [1.144s][info][gc,phases,start] GC(6) Adjust Roots
   [1.144s][info][gc,phases      ] GC(6) Adjust Roots 0.384ms

调整Roots集，更新引用对象，耗时0.384ms。

9. [1.144s][info][gc,phases,start] GC(6) Compaction Phase
   [1.216s][info][gc,phases      ] GC(6) Compaction Phase 72.158ms

压缩整理堆内存，将存活的对象移动到内存的一侧，耗时72.158ms。

10. [1.216s][info][gc,phases,start] GC(6) Post Compact
    [1.220s][info][gc,phases      ] GC(6) Post Compact 3.285ms

移动完对象之后，清理掉边界外的内存，耗时3.285ms。

10. [1.220s][info][gc,heap        ] GC(6) PSYoungGen: 43466K(305664K)->0K(305664K) Eden: 0K(262144K)->0K(262144K) From: 43466K(43520K)->0K(43520K)
    [1.220s][info][gc,heap        ] GC(6) ParOldGen: 583615K(699392K)->396208K(699392K)

这次Full GC，新生代(PSYoungGen)的堆内存43466K全部回收；老年代(ParOldGen)的堆内存从583615K回收到了396208K。

11. [1.220s][info][gc             ] GC(6) Pause Full (Ergonomics) 612M->386M(981M) 88.606ms
    [1.220s][info][gc,cpu         ] GC(6) User=0.77s Sys=0.00s Real=0.09s

Full GC总耗时为88.606ms。

#### 2.2.3.3 运行30s后的分析

堆内存分配信息：新生代使用了298.5MB；老年代使用了683MB

![Parallel1](/img/Parallel1.png)



吞吐量和线程暂停信息

![image-20240822142807128](/img/Parallel2.png)

![image-20240822142902366](/img/Parallel3.png)

## 2.3 G1收集器

### 2.3.1 G1收集器示意图

![G1收集器运行示意图](/img/G1收集器运行示意图.png)

G1收集器运行过程大致可用分为下面四个步骤：

- 初始标记，需要暂停，但借助MinorGC同步完成，实际上没有额外的停顿。
- 并发标记，从GC Root开始对堆中对象进行可达性分析，与用户程序并发进行。
- 最终标记，处理少量的SATB记录，需要短暂暂停用户线程
- 筛选回收，对回收价值高的Region进行回收，需要移动存活对象，必须暂停用户线程。

### 2.3.2 虚拟机参数

```
-XX:+UseG1GC -Xlog:gc*=debug,gc+heap=debug,gc+ergo*=trace,safepoint:log/gc-oomHeap.log:uptime,level,tags -Xms1024M -Xmx1024M
```

### 2.3.3 日志分析

#### 2.3.3.1 Young GC

```
[0.324s][debug][gc,ergo,ihop   ] Request concurrent cycle initiation (occupancy higher than threshold) occupancy: 483393536B allocation request: 524304B threshold: 483183820B (45.00) source: concurrent humongous allocation
[0.324s][debug][gc,ergo        ] Request concurrent cycle initiation (requested by GC cause). GC cause: G1 Humongous Allocation
...省略多行
[0.324s][debug][gc,ergo        ] GC(8) Initiate concurrent cycle (concurrent cycle initiation requested)
[0.324s][info ][gc,start       ] GC(8) Pause Young (Concurrent Start) (G1 Humongous Allocation)
[0.324s][info ][gc,task        ] GC(8) Using 10 workers of 10 for evacuation
...省略多行
[0.324s][trace][gc,ergo,cset   ] GC(8) Start choosing CSet. Pending cards: 1 target pause time: 200.00ms
[0.324s][trace][gc,ergo,cset   ] GC(8) Added young regions to CSet. Eden: 0 regions, Survivors: 1 regions, predicted eden time: 0.06ms, predicted base time: 9.34ms, target pause time: 200.00ms, remaining time: 190.59ms
...省略多行
[0.333s][info ][gc,phases      ] GC(8)   Pre Evacuate Collection Set: 6.8ms
[0.333s][debug][gc,phases      ] GC(8)     Choose Collection Set: 0.0ms
[0.333s][info ][gc,phases      ] GC(8)   Merge Heap Roots: 0.2ms
[0.333s][debug][gc,phases      ] GC(8)     Remembered Sets (ms):
[0.333s][info ][gc,phases      ] GC(8)   Evacuate Collection Set: 0.7ms
[0.333s][debug][gc,phases      ] GC(8)     Ext Root Scanning (ms):
[0.333s][debug][gc,phases      ] GC(8)     Scan Heap Roots (ms):
[0.333s][debug][gc,phases      ] GC(8)     Code Root Scan (ms):
[0.333s][debug][gc,phases      ] GC(8)     Object Copy (ms):
[0.333s][debug][gc,phases      ] GC(8)     Termination (ms):
[0.333s][debug][gc,phases      ] GC(8)     GC Worker Other (ms):
[0.333s][debug][gc,phases      ] GC(8)     GC Worker Total (ms):
[0.333s][info ][gc,phases      ] GC(8)   Post Evacuate Collection Set: 0.4ms
[0.333s][debug][gc,phases      ] GC(8)     Code Roots Fixup: 0.0ms
...省略多行
[0.333s][debug][gc,phases      ] GC(8)     Post Evacuate Cleanup 1: 0.0ms
[0.333s][debug][gc,phases      ] GC(8)      Clear Logged Cards (ms):
[0.333s][debug][gc,phases      ] GC(8)     Post Evacuate Cleanup 2: 0.1ms
[0.333s][debug][gc,phases      ] GC(8)       Purge Code Roots (ms):
[0.333s][debug][gc,phases      ] GC(8)       Free Collection Set (ms):

[0.333s][info ][gc,phases      ] GC(8)   Other: 0.2ms
[0.333s][info ][gc,heap        ] GC(8) Eden regions: 0->0(99)
[0.333s][info ][gc,heap        ] GC(8) Survivor regions: 1->1(10)
[0.333s][info ][gc,heap        ] GC(8) Old regions: 0->0
[0.333s][info ][gc,heap        ] GC(8) Archive regions: 0->0
[0.333s][info ][gc,heap        ] GC(8) Humongous regions: 461->438

[0.333s][info ][gc             ] GC(8) Pause Young (Concurrent Start) (G1 Humongous Allocation) 461M->438M(1024M) 8.584ms
[0.333s][info ][gc,cpu         ] GC(8) User=0.00s Sys=0.02s Real=0.01s

```

1. [0.324s][debug][gc,ergo,ihop   ] Request concurrent cycle initiation (occupancy higher than threshold) occupancy: 483393536B allocation request: 524304B threshold: 483183820B (45.00) source: concurrent humongous allocation
   [0.324s][debug][gc,ergo        ] Request concurrent cycle initiation (requested by GC cause). GC cause: G1 Humongous Allocation

这次发生 global concurrent marking的原因是：humongous allocation。在大对象分配之前，会检测 old generation 使用占比是否超过了阈值。threshold: 483183820B (45.00)，因为 483393536B + 524304B = 483917840  > 483183820B；

2. [0.324s][info ][gc,start       ] GC(8) Pause Young (Concurrent Start) (G1 Humongous Allocation)
   [0.324s][info ][gc,task        ] GC(8) Using 10 workers of 10 for evacuation

当年轻代空间用满后，应用线程会被暂停，年轻代内存块中的存活对象被拷贝到存活区。如果还没有存活区，则任意选择一部分空闲的内存块作为存活区。这里仅仅清理年轻代空间，并使用10个线程进行清理。

3. [0.324s][trace][gc,ergo,cset   ] GC(8) Start choosing CSet. Pending cards: 1 target pause time: 200.00ms
   [0.324s][trace][gc,ergo,cset   ] GC(8) Added young regions to CSet. Eden: 0 regions, Survivors: 1 regions, predicted eden time: 0.06ms, predicted base time: 9.34ms, target pause time: 200.00ms, remaining time: 190.59ms

选择 Collection Set并将Region添加到 CSet中。predicted eden time是预测需要的时间，target pause time这里是默认的停顿时间， remaining time剩余的时间。

4. [0.333s][debug][gc,phases      ] GC(8)     Choose Collection Set: 0.0ms
   [0.333s][info ][gc,phases      ] GC(8)   Merge Heap Roots: 0.2ms
   [0.333s][debug][gc,phases      ] GC(8)     Remembered Sets (ms):

选择 CSet，合并Roots，整理RS集合。

5. [0.333s][info ][gc,phases      ] GC(8)   Evacuate Collection Set: 0.7ms
   [0.333s][debug][gc,phases      ] GC(8)     Ext Root Scanning (ms): 
   [0.333s][debug][gc,phases      ] GC(8)     Scan Heap Roots (ms):
   [0.333s][debug][gc,phases      ] GC(8)     Code Root Scan (ms):
   [0.333s][debug][gc,phases      ] GC(8)     Object Copy (ms):
   [0.333s][debug][gc,phases      ] GC(8)     Termination (ms):
   [0.333s][debug][gc,phases      ] GC(8)     GC Worker Other (ms):
   [0.333s][debug][gc,phases      ] GC(8)     GC Worker Total (ms):

这里开始执行扫描Roots集合。 Ext Root Scanning扫描堆外内存的 GC Root，如classloaders，JNI引用；Scan Heap Roots，Code Root Scan开始扫描堆中Root，例如线程栈中的局部变量等；Object Copy拷贝存活的对象到新的Region中；Termination：在结束前，它会检查其他线程是否还有未扫描完的引用，如果有，则”偷”过来，完成后再申请结束，这个时间是线程之前互相同步所花费的时间；GC Worker Other：其他的小任务， 因为时间很短，在 GC 日志将他们归结在一起；GC Worker TotalGC 的 worker 线程工作时间总计。

6. [0.333s][info ][gc,phases      ] GC(8)   Post Evacuate Collection Set: 0.4ms
   [0.333s][debug][gc,phases      ] GC(8)     Code Roots Fixup: 0.0ms

释放用于管理并行活动的内部数据，一般都接近于零。这个过程是串行执行的。

7. [0.333s][debug][gc,phases      ] GC(8)     Post Evacuate Cleanup 2: 0.1ms
   [0.333s][debug][gc,phases      ] GC(8)       Purge Code Roots (ms):
   [0.333s][debug][gc,phases      ] GC(8)       Free Collection Set (ms):

Purge Code Roots清理其他部分数据，也是非常快的，如非必要基本上等于零。也是串行执行的过程； Free Collection Set将回收集中被释放的小堆归还所消耗的时间，以便他们能用来分配新的对象。

8. [0.333s][info ][gc             ] GC(8) Pause Young (Concurrent Start) (G1 Humongous Allocation) 461M->438M(1024M) 8.584ms
   [0.333s][info ][gc,cpu         ] GC(8) User=0.00s Sys=0.02s Real=0.01s

gc过后，新生代的内存从461M变成了438M，耗时0.01s

#### 2.3.3.2 Mixed GC

下面是 Mixed GC的一次完整的流程

```
[0.451s][info ][gc                ] GC(25) Concurrent Mark Cycle
[0.452s][info ][gc,marking        ] GC(25) Concurrent Scan Root Regions 0.628ms
[0.452s][info ][gc,marking        ] GC(25) Concurrent Mark
[0.452s][info ][gc,marking        ] GC(25) Concurrent Mark From Roots
[0.452s][info ][gc,marking        ] GC(25) Concurrent Mark From Roots 0.424ms
[0.452s][info ][gc,marking        ] GC(25) Concurrent Preclean
[0.452s][info ][gc,marking        ] GC(25) Concurrent Preclean 0.059ms
[0.453s][info ][gc,start          ] GC(25) Pause Remark
[0.453s][debug][gc,phases,start   ] GC(25) Finalize Marking
[0.453s][debug][gc,phases         ] GC(25) Finalize Marking 0.655ms
[0.454s][info ][gc                ] GC(25) Pause Remark 531M->531M(1024M) 1.584ms
[0.454s][info ][gc,cpu            ] GC(25) User=0.00s Sys=0.00s Real=0.00s
[0.454s][info ][gc,marking        ] GC(25) Concurrent Mark 2.563ms
[0.454s][info ][gc,start          ] GC(25) Pause Cleanup
[0.454s][debug][gc,phases,start   ] GC(25) Finalize Concurrent Mark Cleanup
[0.454s][debug][gc,phases         ] GC(25) Finalize Concurrent Mark Cleanup 0.006ms
[0.454s][info ][gc                ] GC(25) Pause Cleanup 532M->532M(1024M) 0.024ms
[0.454s][info ][gc,cpu            ] GC(25) User=0.00s Sys=0.00s Real=0.00s
[0.456s][info ][gc                ] GC(25) Concurrent Mark Cycle 4.802ms
```

1. 初始标记实际上是借用 Young GC阶段完成的，Mixed GC也是在Young GC之后触发的，由 GC(8) Pause Young (Concurrent Start) (G1 Humongous Allocation) 可知。

2. [0.451s][info ][gc                ] GC(25) Concurrent Mark Cycle

   并发阶段开始

3. [0.452s][info ][gc,marking        ] GC(25) Concurrent Scan Root Regions 0.628ms

并发从GC Root开始对堆中对象进行可达性分析，递归扫描整个堆里面的对象图。

4. [0.453s][info ][gc,start          ] GC(25) Pause Remark
   [0.453s][debug][gc,phases,start   ] GC(25) Finalize Marking
   [0.453s][debug][gc,phases         ] GC(25) Finalize Marking 0.655ms
   [0.454s][info ][gc                ] GC(25) Pause Remark 531M->531M(1024M) 1.584ms

重新标记（也说最终标记），这里需要暂停用户线程，处理并发阶段结束后留下来的少量 SATB 记录。

5. [0.454s][info ][gc,marking        ] GC(25) Concurrent Mark 2.563ms

并发标记阶段结束，之后到达筛选回收阶段

6. [0.454s][info ][gc,start          ] GC(25) Pause Cleanup
   [0.454s][debug][gc,phases,start   ] GC(25) Finalize Concurrent Mark Cleanup
   [0.454s][debug][gc,phases         ] GC(25) Finalize Concurrent Mark Cleanup 0.006ms
   [0.454s][info ][gc                ] GC(25) Pause Cleanup 532M->532M(1024M) 0.024ms
   [0.454s][info ][gc,cpu            ] GC(25) User=0.00s Sys=0.00s Real=0.00s

并发筛选回收，负责对各个Region的回收价值和成本进行排序，将决定回收的那一部分Region的存活对象复制到空的Region中，再清理掉整个旧的Region的全部空间。

7. [0.456s][info ][gc                ] GC(25) Concurrent Mark Cycle 4.802ms

Mixed gc结束，耗时4.802ms。

#### 2.3.3.3 Full GC

G1 是一款自适应的增量垃圾收集器。一般来说，只有在内存严重不足的情况下才会发生 Full GC。比如堆空间不足或者 to-space 空间不足。将每个对象的大小从 100x1024B改成1024x1024B，也就是1M，再进行实验，得到下面的数据。

```
[1.148s][info ][gc,task        ] GC(169) Using 10 workers of 10 for full compaction
[1.148s][info ][gc,start       ] GC(169) Pause Full (G1 Compaction Pause)
[1.148s][info ][gc,phases,start] GC(169) Phase 1: Mark live objects
[1.149s][info ][gc,phases      ] GC(169) Phase 1: Mark live objects 1.013ms
[1.149s][info ][gc,phases,start] GC(169) Phase 2: Prepare for compaction
[1.150s][info ][gc,phases      ] GC(169) Phase 2: Prepare for compaction 0.813ms
[1.150s][info ][gc,phases,start] GC(169) Phase 3: Adjust pointers
[1.150s][info ][gc,phases      ] GC(169) Phase 3: Adjust pointers 0.406ms
[1.150s][info ][gc,phases,start] GC(169) Phase 4: Compact heap
[1.151s][info ][gc,phases      ] GC(169) Phase 4: Compact heap 0.267ms
[1.151s][debug][gc,ergo        ] GC(169) Running G1 Clear Bitmap with 10 workers for 16 work units.
[1.152s][info ][gc,heap        ] GC(169) Eden regions: 0->0(51)
[1.152s][info ][gc,heap        ] GC(169) Survivor regions: 0->0(0)
[1.152s][info ][gc,heap        ] GC(169) Old regions: 1->1
[1.152s][info ][gc,heap        ] GC(169) Archive regions: 0->0
[1.152s][info ][gc,heap        ] GC(169) Humongous regions: 1023->0
[1.152s][info ][gc             ] GC(169) Pause Full (G1 Compaction Pause) 1023M->0M(1024M) 4.149ms
[1.152s][info ][gc,cpu         ] GC(169) User=0.02s Sys=0.00s Real=0.01s
```

#### 2.3.3.4 运行30s后的分析



借助 GCEasy进行分析

新生代和老年代分配内存大小，峰值分配内存大小

![G1_1](/img/G1_1.png)



吞吐量和暂停时间：

![image-20240822114737684](/img/G1_2.png)



Pause GC Duration(用户线程停顿时间)

![image-20240822114925006](/img/G1_3.png)



G1 Collection 各阶段的数据

![image-20240822115233393](/img/G1_4.png)



暂停时间汇总和并行时间汇总

暂停时间包括：初始标记，最终标记，筛选清理

并行时间包括：并发标记

![G1_5](/img/G1_5.png)

## 2.4 Shenandoah 收集器

### 2.4.1 Shenandoah收集器运行示意图

Shenandoah收集器支持并发的整理算法，前面提到的G1的回收阶段是多线程并行的，但却不能与用户线程并发。Shenandoah摒弃了在G1中耗费大量内存和计算资源去维护的记忆集，而是使用“连接矩阵”来记录跨Region的引用关系。

![Shenandoah收集器运行示意图](/img/Shenandoah收集器运行示意图.png)

Shenandoah的工作过程大致可以划分为以下九个阶段：

- 初始标记：与G1一样，首先标记与GC Roots直接关联的对象，这个阶段依然是“Stop The World”的。
- 并发标记：与G1一样，遍历对象图，标记出全部可达的对象，这个阶段是与用户线程一起并发的。
- 最终标记：与G1一样，处理剩下的SATB扫描，并在这个阶段计算出回收价值最高的Region。这个决断也会有一小段短暂的停顿。
- 并发清理：这个阶段用于清理那些整个区域内连一个存活对象都没有找到的Region。
- 并发回收：把会收集里面的存活对象先复制一份到其他未被使用的Region中，这个过程与用户线程是并发执行的，需要使用读屏障和“Brooks Pointers”转发指针来解决旧对象的引用问题。
- 初始引用更新：设立这个阶段只是为了建立一个线程集合点，确保所有并发回收阶段中进行的收集器线程都已完成分配给它们的对象移动任务。这个过程需要简短的停顿。
- 并发引用更新：并发回收阶段复制对象结束后，还需要把堆中所有指向旧对象的引用修正到复制后的新地址，这个操纵叫做引用更新。这个过程与用户线程一起并发的，时间长短取决于内存中涉及的引用数量的多少。
- 最终引用更新：解决了堆中引用更新后，还要修正存在于GC Roots中的引用。这个过程也需要停顿。
- 并发清理：经过并发回收和引用更新之后，整个回收集中所有的Region已再无存活对象，需要再调用一次并发清理过程来回收这些Region的内存空间。

### 2.4.2 虚拟机参数

### 2.4.3 日志分析

#### 2.4.3.1 一个GC过程的分析

```
[30.376s][info][gc,start    ] GC(353) Pause Init Mark (unload classes)
[30.376s][info][gc,task     ] GC(353) Using 6 of 6 workers for init marking
[30.376s][info][gc,ergo     ] GC(353) Pacer for Mark. Expected Live: 196M, Free: 219M, Non-Taxable: 22427K, Alloc Tax Rate: 1.1x
[30.376s][info][gc          ] GC(353) Pause Init Mark (unload classes) 0.209ms
[30.376s][info][gc,start    ] GC(353) Concurrent marking roots
[30.376s][info][gc,task     ] GC(353) Using 3 of 6 workers for concurrent marking roots
[30.376s][info][gc          ] GC(353) Concurrent marking roots 0.337ms
[30.376s][info][gc,start    ] GC(353) Concurrent marking (unload classes)
[30.376s][info][gc,task     ] GC(353) Using 3 of 6 workers for concurrent marking
[30.379s][info][gc          ] GC(353) Concurrent marking (unload classes) 2.859ms
[30.379s][info][gc,start    ] GC(353) Pause Final Mark (unload classes)
[30.380s][info][gc,task     ] GC(353) Using 6 of 6 workers for final marking
[30.380s][info][gc,ergo     ] GC(353) Adaptive CSet Selection. Target Free: 145M, Actual Free: 539M, Max CSet: 43690K, Min Garbage: 0B
[30.380s][info][gc,ergo     ] GC(353) Collectable Garbage: 447M (82%), Immediate: 290M (53%), CSet: 157M (28%)
[30.380s][info][gc,ergo     ] GC(353) Pacer for Evacuation. Used CSet: 199M, Free: 500M, Non-Taxable: 51221K, Alloc Tax Rate: 1.1x
[30.380s][info][gc          ] GC(353) Pause Final Mark (unload classes) 0.722ms
...对线程根、弱引用、弱跟的处理
[30.383s][info][gc,start    ] GC(353) Concurrent evacuation
[30.383s][info][gc,task     ] GC(353) Using 3 of 6 workers for concurrent evacuation
[30.399s][info][gc          ] GC(353) Concurrent evacuation 16.080ms
[30.400s][info][gc,start    ] GC(353) Pause Init Update Refs
[30.400s][info][gc,ergo     ] GC(353) Pacer for Update Refs. Used: 577M, Free: 428M, Non-Taxable: 43845K, Alloc Tax Rate: 1.6x
[30.400s][info][gc          ] GC(353) Pause Init Update Refs 0.057ms
[30.400s][info][gc,start    ] GC(353) Concurrent update references
[30.400s][info][gc,task     ] GC(353) Using 3 of 6 workers for concurrent reference update
[30.404s][info][gc          ] GC(353) Concurrent update references 3.903ms
[30.405s][info][gc,start    ] GC(353) Pause Final Update Refs
[30.405s][info][gc,task     ] GC(353) Using 6 of 6 workers for final reference update
[30.405s][info][gc          ] GC(353) Pause Final Update Refs 0.344ms
[30.405s][info][gc,start    ] GC(353) Concurrent cleanup
[30.405s][info][gc          ] GC(353) Concurrent cleanup 603M->404M(1024M) 0.218ms
[30.405s][info][gc,ergo     ] Free: 567M, Max: 512K regular, 195M humongous, Frag: 66% external, 37% internal; Reserve: 52736K, Max: 512K
[30.405s][info][gc,stats    ] all workers. Dividing the <total> over the root stage time estimates parallelism.
```

1. [30.376s][info][gc,start    ] GC(353) Pause Init Mark (unload classes)
   [30.376s][info][gc,task     ] GC(353) Using 6 of 6 workers for init marking
   [30.376s][info][gc,ergo     ] GC(353) Pacer for Mark. Expected Live: 196M, Free: 219M, Non-Taxable: 22427K, Alloc Tax Rate: 1.1x
   [30.376s][info][gc          ] GC(353) Pause Init Mark (unload classes) 0.209ms

初始标记阶段，这里使用6个线程进行初始标记。预计存活对象大小196MB，空闲内存219MB。

2. [30.376s][info][gc,start    ] GC(353) Concurrent marking roots
   [30.376s][info][gc,task     ] GC(353) Using 3 of 6 workers for concurrent marking roots
   [30.376s][info][gc          ] GC(353) Concurrent marking roots 0.337ms
   [30.376s][info][gc,start    ] GC(353) Concurrent marking (unload classes)
   [30.376s][info][gc,task     ] GC(353) Using 3 of 6 workers for concurrent marking
   [30.379s][info][gc          ] GC(353) Concurrent marking (unload classes) 2.859ms

并发标记阶段，GC线程与用户线程一起运行，遍历对象图。使用3个工作线程进行并发标记根节点，同时卸载类。

3. [30.379s][info][gc,start    ] GC(353) Pause Final Mark (unload classes)
   [30.380s][info][gc,task     ] GC(353) Using 6 of 6 workers for final marking
   [30.380s][info][gc,ergo     ] GC(353) Adaptive CSet Selection. Target Free: 145M, Actual Free: 539M, Max CSet: 43690K, Min Garbage: 0B
   [30.380s][info][gc,ergo     ] GC(353) Collectable Garbage: 447M (82%), Immediate: 290M (53%), CSet: 157M (28%)
   [30.380s][info][gc,ergo     ] GC(353) Pacer for Evacuation. Used CSet: 199M, Free: 500M, Non-Taxable: 51221K, Alloc Tax Rate: 1.1x
   [30.380s][info][gc          ] GC(353) Pause Final Mark (unload classes) 0.722ms

最终标记节点，处理剩余的SATB扫描。

4. [30.383s][info][gc,start    ] GC(353) Concurrent evacuation
   [30.383s][info][gc,task     ] GC(353) Using 3 of 6 workers for concurrent evacuation
   [30.399s][info][gc          ] GC(353) Concurrent evacuation 16.080ms

使用3个线程并发回收，在这里对存活对象进行移动

5. [30.400s][info][gc,start    ] GC(353) Pause Init Update Refs
   [30.400s][info][gc,ergo     ] GC(353) Pacer for Update Refs. Used: 577M, Free: 428M, Non-Taxable: 43845K, Alloc Tax Rate: 1.6x
   [30.400s][info][gc          ] GC(353) Pause Init Update Refs 0.057ms
   [30.400s][info][gc,start    ] GC(353) Concurrent update references
   [30.400s][info][gc,task     ] GC(353) Using 3 of 6 workers for concurrent reference update
   [30.404s][info][gc          ] GC(353) Concurrent update references 3.903ms
   [30.405s][info][gc,start    ] GC(353) Pause Final Update Refs
   [30.405s][info][gc,task     ] GC(353) Using 6 of 6 workers for final reference update
   [30.405s][info][gc          ] GC(353) Pause Final Update Refs 0.344ms

先在初始更新引用阶段等待所有线程集合，然后并发更新引用，最后处理GC Roots中的引用

6. [30.405s][info][gc,start    ] GC(353) Concurrent cleanup
   [30.405s][info][gc          ] GC(353) Concurrent cleanup 603M->404M(1024M) 

并发清理，这里是对会收集的Region进行清理。

#### 2.4.3.2 运行30s后的分析

运行时堆内存分配情况，可以看到，设定的1G的堆内存可以全部被分配。

![Shenandoah1](/img/Shenandoah1.png)



吞吐量达到了98.65%;平均停顿时间为0.291ms，最高停顿时间也不过2.51ms

![Shenandoah2](/img/Shenandoah2.png)



程序运行7秒后，GC触发的事件趋于平稳，每次GC造成的用户线程停顿时间也很短。

![Shenandoah3](/img/Shenandoah3.png)



各阶段的耗时，可以看到并发回收阶段是最耗时的，因为需要将存活的对象移动到新的地址；其次是并发标记节点，需要遍历整个对象图。而其他过程都在2ms内就能完成。

![Shenandoah4](/img/Shenandoah4.png)



用户线程总的停顿时间为413ms，并发时间为6.694s。程序运行时间为30.602s。

![Shenandoah5](/img/Shenandoah5.png)



## 2.5 ZGC收集器

### 2.5.1 ZGG收集器运行示意图

ZGC收集器是一款基于Region内存布局的，不分代，使用了读屏障，染色指针和内存多重隐射等技术来实现可并发的标记-整理算法的、以低延迟为目标的一款垃圾收集器。

ZGC收集器运行的阶段：

- 初始标记：类似于G1，Shenadoah收集器的标记，需要短暂的停顿。
- 并发标记：遍历对象图做可达性分析，ZGC的标记是在指针上而不是在对象上进行的，标记阶段会更新染色质指针中的Marked0、Marked1标记位。
- 再标记（重标记）：处理剩余的SATB扫描。
- 并发转移准备（Concurrent Prepare for Relocate）:将需要清理的Region组成重分配集（Relocation Set）,这个阶段的ZGC会扫描所有的Region，不需要维护记忆集。重分配集RSet只是决定了里面的存活对象会被重新复制到其他的Region中，而原本的Region会被释放。
- 并发重分配（Concurrent Relocate）:把重分配集中存活对象复制到新的Region中，并为重分配集中的每个Region维护一个转发表（Forward Table）,记录从旧对象到新对象的转向关系。
- 并发重映射（Concurrent Remap）：重映射所做的就是修正整个堆中指向重分配集中旧对象的所有引用。可以将这段工作合并到下一次垃圾收集循环中的并发标记阶段里完成，节省一次遍历对象图的开销。

### 2.5.2 测试代码

这里对测试代码进行修改，使代码能够产生小对象，中对象

```java

import java.util.Random;
import java.util.concurrent.atomic.LongAdder;

public class GCTest {
    private static Random random = new Random();
    public static void main(String[] var0) {
        // 当前毫秒时间戳
        long startMillis = System.currentTimeMillis();
        // 持续运行毫秒数; 可根据需要进行修改
        long timeoutMillis = 30000;
        // 结束时间戳
        long endMillis = startMillis + timeoutMillis;
        LongAdder counter = new LongAdder();
        System.out.println("正在执行...");
        int size=2000;
        Object[] arr = new Object[size];
        while (System.currentTimeMillis() < endMillis) {
            int index = random.nextInt(2*size);
            int t = random.nextInt(100) > 50? 256:100;
            byte[] garbage = new byte[t*1024];
            if(index < size) {
                arr[index] = garbage;
            }
            counter.increment();
        }
        System.out.println("执行结束!共生成对象次数:" + counter.longValue());
    }
}
```

### 2.5.3 日志分析

#### 2.5.3.1 阻塞内存分配请求触发

```
[29.555s][info][gc,start    ] GC(221) Garbage Collection (Allocation Stall)
[29.555s][info][gc,phases   ] GC(221) Pause Mark Start 0.007ms
[29.558s][info][gc,phases   ] GC(221) Concurrent Mark 3.255ms
[29.559s][info][gc,phases   ] GC(221) Pause Mark End 0.026ms
[29.559s][info][gc,phases   ] GC(221) Concurrent Mark Free 0.001ms
[29.559s][info][gc,phases   ] GC(221) Concurrent Reset Relocation Set 0.014ms
[29.561s][info][gc,phases   ] GC(221) Concurrent Select Relocation Set 1.697ms
[29.561s][info][gc,phases   ] GC(221) Pause Relocate Start 0.007ms
[29.566s][info][gc          ] Allocation Stall (main) 11.365ms
[29.652s][info][gc,phases   ] GC(221) Concurrent Relocate 90.689ms

[29.652s][info][gc,reloc    ] GC(221) Small Pages: 150 / 300M, Empty: 0M, Relocated: 97M, In-Place: 0
[29.652s][info][gc,reloc    ] GC(221) Medium Pages: 22 / 704M, Empty: 0M, Relocated: 221M, In-Place: 2
[29.652s][info][gc,reloc    ] GC(221) Large Pages: 0 / 0M, Empty: 0M, Relocated: 0M, In-Place: 0
[29.652s][info][gc,reloc    ] GC(221) Forwarding Usage: 0M
[29.652s][info][gc,heap     ] GC(221) Min Capacity: 1024M(100%)
[29.652s][info][gc,heap     ] GC(221) Max Capacity: 1024M(100%)
[29.652s][info][gc,heap     ] GC(221) Soft Max Capacity: 1024M(100%)
[29.652s][info][gc,heap     ] GC(221)                Mark Start          Mark End        Relocate Start      Relocate End           High               Low         
[29.652s][info][gc,heap     ] GC(221)  Capacity:     1024M (100%)       1024M (100%)       1024M (100%)       1024M (100%)       1024M (100%)       1024M (100%)   
[29.652s][info][gc,heap     ] GC(221)      Free:       20M (2%)           20M (2%)           20M (2%)          366M (36%)         386M (38%)          18M (2%)     
[29.652s][info][gc,heap     ] GC(221)      Used:     1004M (98%)        1004M (98%)        1004M (98%)         658M (64%)        1006M (98%)         638M (62%)    
[29.652s][info][gc,heap     ] GC(221)      Live:         -               352M (34%)         352M (34%)         352M (34%)            -                  -          
[29.652s][info][gc,heap     ] GC(221) Allocated:         -                 0M (0%)            0M (0%)          265M (26%)            -                  -          
[29.652s][info][gc,heap     ] GC(221)   Garbage:         -               651M (64%)         651M (64%)          39M (4%)             -                  -          
[29.652s][info][gc,heap     ] GC(221) Reclaimed:         -                  -                 0M (0%)          611M (60%)            -                  -          
[29.652s][info][gc          ] GC(221) Garbage Collection (Allocation Stall) 1004M(98%)->658M(64%)
[29.711s][info][gc,start    ] GC(222) Garbage Collection (Allocation Rate)
```

1. [29.555s][info][gc,start    ] GC(221) Garbage Collection (Allocation Stall)

当垃圾来不及回收，垃圾将堆占满时，会导致部分线程阻塞，触发了GC。

2. [29.555s][info][gc,phases   ] GC(221) Pause Mark Start 0.007ms

初始标记，会STW。

3. [29.558s][info][gc,phases   ] GC(221) Concurrent Mark 3.255ms

并发标记。

4. [29.559s][info][gc,phases   ] GC(221) Pause Mark End 0.026ms

再次标记，会STW。

5. [29.559s][info][gc,phases   ] GC(221) Concurrent Mark Free 0.001ms

并发标记空闲阶段。在这个阶段，垃圾收集器在应用程序的其他线程并发运行的同时，进行标记工作的最后收尾。这个阶段通常包括清理和整理在并发标记阶段收集到的数据，以及准备进行下一阶段的GC工作。

6. [29.559s][info][gc,phases   ] GC(221) Concurrent Reset Relocation Set 0.014ms

并发重置重分配集合阶段。这个阶段是在并发标记阶段之后，用于重置用于跟踪对象移动的集合。

7. [29.561s][info][gc,phases   ] GC(221) Concurrent Select Relocation Set 1.697ms

并发选择重分配集合阶段。这个阶段涉及到选择哪些对象需要在即将到来的重定位阶段被移动。

8. [29.561s][info][gc,phases   ] GC(221) Pause Relocate Start 0.007ms

并发重分配阶段，将存活对象复制到新的Region上。

9. [29.652s][info][gc          ] GC(221) Garbage Collection (Allocation Stall) 1004M(98%)->658M(64%)

gc过后，堆内存从1004M回收到了658M。

10.[29.711s][info][gc,start    ] GC(222) Garbage Collection (Allocation Rate)

另一种触发GC的方式，也是最主要的GC触发方式，其算法原理可简单描述为”ZGC根据近期的对象分配速率以及GC时间，计算出当内存占用达到什么阈值时触发下一次GC”。

#### 2.5.3.2 基于分配速率的自适应算法

处理过程跟2.5.3.1一致。

```
[29.711s][info][gc,task     ] GC(222) Using 3 workers
[29.712s][info][gc,phases   ] GC(222) Pause Mark Start 0.006ms
[29.715s][info][gc,phases   ] GC(222) Concurrent Mark 3.331ms
[29.715s][info][gc,phases   ] GC(222) Pause Mark End 0.006ms
[29.715s][info][gc,phases   ] GC(222) Concurrent Mark Free 0.001ms
[29.716s][info][gc,phases   ] GC(222) Concurrent Process Non-Strong References 0.731ms
[29.716s][info][gc,phases   ] GC(222) Concurrent Reset Relocation Set 0.007ms
[29.717s][info][gc,phases   ] GC(222) Concurrent Select Relocation Set 1.350ms
[29.717s][info][gc,phases   ] GC(222) Pause Relocate Start 0.003ms
[29.721s][info][gc          ] Allocation Stall (main) 9.946ms
[29.808s][info][gc,phases   ] GC(222) Concurrent Relocate 90.929ms
[29.809s][info][gc,heap     ] GC(222)                Mark Start          Mark End        Relocate Start      Relocate End           High               Low         
[29.809s][info][gc,heap     ] GC(222)  Capacity:     1024M (100%)       1024M (100%)       1024M (100%)       1024M (100%)       1024M (100%)       1024M (100%)   
[29.809s][info][gc,heap     ] GC(222)      Free:        0M (0%)            0M (0%)            2M (0%)          400M (39%)         412M (40%)           0M (0%)     
[29.809s][info][gc,heap     ] GC(222)      Used:     1024M (100%)       1024M (100%)       1022M (100%)        624M (61%)        1024M (100%)        612M (60%)    
[29.809s][info][gc,heap     ] GC(222)      Live:         -               350M (34%)         350M (34%)         350M (34%)            -                  -          
[29.809s][info][gc,heap     ] GC(222) Allocated:         -                 0M (0%)            0M (0%)          263M (26%)            -                  -          
[29.809s][info][gc,heap     ] GC(222)   Garbage:         -               673M (66%)         671M (66%)           9M (1%)             -                  -          
[29.809s][info][gc,heap     ] GC(222) Reclaimed:         -                  -                 2M (0%)          663M (65%)            -                  -          
[29.809s][info][gc          ] GC(222) Garbage Collection (Allocation Rate) 1024M(100%)->624M(61%)
```

#### 2.5.3.3 运行30s后的分析

可以看到，堆内存可以分配到1G。

![Z_1](/img/Z_1.png)



吞吐量达到99.977%；平均暂停时间和最高暂停时间都很低。

![Z_2](/img/Z_2.png)

![Z_3](/img/Z_3.png)



各个阶段的数据，可以看到，并发重分配是最耗时间的，因为涉及到对象的移动。

![Z_4](/img/Z_4.png)



总暂停时间为6.85ms，并行运行的时间达到22.462ms。

![Z_5](/img/Z_5.png)
