# 1 测试用例

## 1.1 测试代码

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.atomic.LongAdder;

public class GCTest {
    public static void main(String[] args) {
        // 当前毫秒时间戳
        long startMillis = System.currentTimeMillis();
        // 持续运行毫秒数; 可根据需要进行修改
        long timeoutMillis = 1000;
        // 结束时间戳
        long endMillis = startMillis + timeoutMillis;
        System.out.println("startMillis:  " + startMillis);
        System.out.println("endMillis: " + endMillis);
        LongAdder counter = new LongAdder();
        System.out.println("开始执行....");
        List<byte[]> list = new ArrayList<>();
        while(System.currentTimeMillis() < endMillis) {
            // 分配1MB的数组
            list.add(new byte[1024 * 1024]);
            counter.increment();
        }
        System.out.println("执行结束！共生成对象次数:" + counter.longValue());
    }
}

```

# 2 经典垃圾收集器比较

## 2.1 SerialGC

### 2.1.1 Serial收集器运行示意图：

![Serial收集器运行示意图](/img/Serial收集器运行示意图.png)

### 2.1.2 虚拟机参数

```
-XX:+UseSerialGC -Xlog:gc*=info:log/gc-oomHeap.log:uptime,level,tags -Xms1024M -Xmx1024M
```

### 2.1.3 日志分析

完整的log日志将在 ./Serial.log文件给出

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

整堆回收，暂停了用户程序0.04秒。

### 2.1.4 暂停时间

Serial收集器是一个单线程工作的收集器，采取复制算法，在GC过程中会暂停所有用户线程。

由于测试用例无法保证每个程序运行的时间是一样的，这里计算总的暂停时间和吞吐量。具体看Serial.log，将每个暂停时间相加。由log文件可以知道，一共进行了19轮gc，得到的暂停总时间是

总暂停时间：Sum(暂停)=554.98ms

程序开始时间：0.048s

程序结束时间：1.559s

程序运行总时间：Sum(程序)=1.559-0.048=1.511

吞吐量=运行用户代码时间/(运行总时间)

这里的运行总时间=运行用户程序时间+运行垃圾收集时间，其实就是上面的Sum(程序)

所以吞吐量 = (1.511 - 0.555)/1.511=0.633

## 2.2 Parallel Scavenge收集器

### 2.2.1 Parallel Scavenge收集器运行示意图

![Parallel Scanvenge收集器运行示意图](/img/Parallel Scanvenge收集器运行示意图.png)

Parallel收集器对新生代使用“标记-复制”算法，对老年代使用“标记-整理”算法。在GC时间执行期间，多个线程并行地标记和清除垃圾。

### 2.2.2 虚拟机参数

```
-XX:+UseParallelGC -Xlog:gc*=info:log/gc-oomHeap.log:uptime,level,tags -Xms1024M -Xmx1024M -XX:+UseAdaptiveSizePolicy
```

### 2.2.3 日志分析

没触发Full GC

```
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

本次Young GC，花费了0.11s。

在这个样例中，并没能触发Full GC。

### 2.2.4 暂停时间

根据 Parallel Scavenge.log文件

总暂停时间：Sum(暂停)=827.171ms

程序开始时间：0.003s

程序结束时间：1.171s

程序运行总时间：Sum(程序)=1.171s-0.003s=1.168s

吞吐量=(1.168-0.827)/1.168=0.292

### 2.2.5 Full GC

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

### 2.2.6 暂停时间

根据 Parallel Scavenge.log文件

总暂停时间：Sum(暂停)=922.312ms

程序开始时间：0.005s

程序结束时间：1.237s

程序运行总时间：Sum(程序)=1.237s-0.005s=1.232s

吞吐量=(1.232-0.922)/1.232=0.252

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
-XX:+UseG1GC -Xlog:gc*=info:log/gc-oomHeap.log:uptime,level,tags -Xms1024M -Xmx1024M
```

### 2.3.3 日志分析

将程序的运行时间修改为30s，数组的大小设置为5000，其他地方不变，运行30s

```java
// 当前毫秒时间戳
long startMillis = System.currentTimeMillis();
// 持续运行毫秒数; 可根据需要进行修改
long timeoutMillis = 30000;
// 结束时间戳
long endMillis = startMillis + timeoutMillis;
...
int size=5000;
Object[] arr = new Object[size];
...
```

借助 GCEasy进行分析

![image-20240822112910892](/img/G1_1.png)
