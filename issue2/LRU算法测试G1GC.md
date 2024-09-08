# LRU算法测试G1GC

## 总结

### 六个问题

[问题](#question1)

[问题2](#question2)

[问题3](#question3)

[问题4](#question4)

[问题5](#question5)

[问题6](#question6)

### 三个猜想并验证

[猜想1](#yanzheng1)

[猜想2](#yanzheng2)

[猜想3](#yanzheng3)

## 1 测试代码

### 1.1 LRU算法的实现

issue要求如下：
使用一些现有的whitebox API（有需要的话可以自己扩展whitebox API）来实现一个典型的LRU cache

本文没有使用whitebox API实现LRU算法，使用了Java的LinkedHashMap实现了LRUCache。

代码如下：

```java
class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;

    public LRUCache(int capacity) {
        super(capacity, 0.75f, true);
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }
}
```

### 1.2 测试代码

测试代码如下，有一些函数的功能会在报告的后面一一展开，这里先关注main函数即可。

main函数中，代码运行10s；在这段时间内，不断生成大小为256 * 1024 k，1024 * 1024k的byte数组和整数对象，并将这些对象放入到LRUCache中。这里设定key值有1/3的概率会重复，如果生成了重复的key值，就会调用cache.get(key);方法，重新激活该对象，让其排在前面，不会被淘汰。

```java
public class TestLRUG1GCTest {

    private static final int CACHE_SIZE = 1000; // LRU能容纳的对象数量
    private static WhiteBox wb = WhiteBox.getWhiteBox();
    private static Random random = new Random();
    private static final int BIG_OBJECT = 1024 * 1024; // 大对象
    private static final int SMALL_OBJECT = 256 * 1024; // 小对象，不超过HeapRegionSize/2

    public static void main(String[] args) throws Exception {
        LRUCache<String, Object> cache = new LRUCache<>(CACHE_SIZE);
        long startMillis = System.currentTimeMillis();
        long timeoutMillis = 10000; // 运行时间为10秒
        long endMillis = startMillis + timeoutMillis;
        while (System.currentTimeMillis() < endMillis) {
            String key = generateStrKeyName();
            Object oldValue = cache.get(key);
            if (null == oldValue) {
                Object value = generateObject();
                cache.put(key, value);
            }
        }
    }
    
    public static String generateStrKeyName() {
        // 如果 cache 填充满了对象，有 1 / 3 的概率这个对象会被重新激活，排在前面
        int index = random.nextInt(3 * CACHE_SIZE);
        return "name_" + index;
    }

    public static Object generateObject() {
        int index = random.nextInt(3); // 返回 0, 1, 2
        switch (index) {
            case 0: {
                // 生成 1M 的对象
                return new byte[BIG_OBJECT];
            }
            case 1: {
                // 生成 256 * 1024 k 的对象
                return new byte[SMALL_OBJECT];
            }
            case 2: {
                // 生成整数
                return random.nextInt(100);
            }
        }
        return null;
    }
}
```



## 2 G1回收器

### 2.1 新生代收集阶段

G1中有两种回收模式：

- 完全新生代GC，只针对新生代的垃圾回收
- 部分新生代回收，也称为混合回收，收集整个新生代以及部分老年代。

完全新生代GC跟其它的收集器差不多，将整个新生代区域加入到回收集合(Collection Set，简称CSet)，新创建的对象分配至Eden区域，然后将标记存活的对象移动至Survivor区，达到晋升年龄的就晋升到老年代区域，然后清空原区域。详细的日志在[Issue1](https://github.com/SupeMaker/Tencent_JDK17_Exercise/blob/main/不同GC收集器的比较.md)中已经给出，这里不做过多的介绍。这里是简短的日志。

```
# 开始新生代收集
[0.331s][info ][gc,start      ] GC(0) Pause Young (Normal) (G1 Evacuation Pause)
[0.333s][info ][gc,task       ] GC(0) Using 10 workers of 10 for evacuation
# 将全部新生代Eden加入到CSet中
[0.334s][trace][gc,ergo,cset  ] GC(0) Start choosing CSet. Pending cards: 308 target pause time: 200.00ms
[0.334s][trace][gc,ergo,cset  ] GC(0) Added young regions to CSet. Eden: 51 regions, Survivors: 0 regions, predicted eden time: 17.75ms, predicted base time: 10.00ms, target pause time: 200.00ms, remaining time: 172.25ms
[0.334s][debug][gc,ergo       ] GC(0) Running G1 Merge Heap Roots using 10 workers for 51 regions
[0.383s][debug][gc,ergo       ] GC(0) Running G1 Rebuild Free List Task using 10 workers for rebuilding free list of regions
# 这里是执行收集的过程
[0.384s][info ][gc,phases     ] GC(0)   Pre Evacuate Collection Set: 0.4ms
[0.384s][info ][gc,phases     ] GC(0)   Merge Heap Roots: 0.2ms
[0.384s][info ][gc,phases     ] GC(0)   Evacuate Collection Set: 40.2ms
[0.384s][info ][gc,phases     ] GC(0)   Post Evacuate Collection Set: 9.3ms
[0.384s][info ][gc,phases     ] GC(0)   Other: 2.8ms
# 收集完成的结果，可以看到51个新生代region全部被收集，有些对象晋升到老年区(Old regions)
[0.384s][info ][gc,heap       ] GC(0) Eden regions: 51->0(44)
[0.384s][info ][gc,heap       ] GC(0) Survivor Survivor: 0->7(7)
[0.384s][info ][gc,heap       ] GC(0) Old regions: 0->43
[0.384s][info ][gc,heap       ] GC(0) Archive regions: 2->2
[0.384s][info ][gc,heap       ] GC(0) Humongous regions: 358->358
[0.384s][info ][gc,metaspace  ] GC(0) Metaspace: 546K(704K)->546K(704K) NonClass: 509K(576K)->509K(576K) 
[0.384s][info ][gc            ] GC(0) Pause Young (Normal) (G1 Evacuation Pause) 409M->408M(1024M) 52.798ms
[0.384s][info ][gc,cpu        ] GC(0) User=0.15s Sys=0.10s Real=0.06s
```

### 2.2 混合GC

混合GC是把一部分老年区的region加到Eden和Survivor后面，合并起来称为Collection Set，在下一次混合GC的时候，将这些老年区一并清理。G1收集器如何决定把哪些老年区Region进行回收呢？这就是并发标记阶段需要做的事情。

#### 2.2.1 并发标记

并发标记的目的是标记每个Region中的存活对象，当堆内存的总体使用比例达到一定的数值，就会触发并发标记，默认为45%，可以通过JVM参数InitiatingHeapOccupancyPercent进行设置。标记存活对象时，是通过可达性分析算法来实现的，至于标记过程中产生的“对象消失”问题，G1使用原始快照(Snapshot At The Beginning, SATB)和写屏障的方法来解决。每个Region中都包含bottm,end,top,prevTAMS和NextTAMS指针。

```c++
# heapRegion.hpp文件
HeapWord* const _bottom;
HeapWord* const _end;
HeapWord* volatile _top;
HeapWord* _prev_top_at_mark_start;
HeapWord* _next_top_at_mark_start;
```

**全局并发标记的过程**

- 初始标记：仅仅是标记一下GC Root能直接关联到的对象，并修改TAMS指针。
- 并发标记：从GC Toot开始对堆中对象进行可达性分析，寻找要回收的对象。
- 最终标记：G1 GC 清空 SATB 缓冲区，跟踪未被访问的存活对象，并执行引用处理。
- 清除垃圾：在这个最后阶段，G1 GC 执行统计和 RSet 净化的 STW 操作。在统计期间，G1 GC 会识别完全空的区域和可供进行混合垃圾回收的区域。清理阶段在将空白区域重置并添加到空闲列表时为部分并发。注意完全空的region不会被加到CSet，都在这个阶段直接回收了。

下面是完整的并发标记日志

```
# 新生代收集
[0.451s][info ][gc,start      ] GC(1) Pause Young (Concurrent Start) (G1 Humongous Allocation)
...
[0.482s][info ][gc            ] GC(1) Pause Young (Concurrent Start) (G1 Humongous Allocation) 536M->536M(1024M) 31.025ms
# 并发标记开始
[0.482s][info ][gc            ] GC(2) Concurrent Mark Cycle
[0.482s][info ][gc,marking    ] GC(2) Concurrent Clear Claimed Marks
[0.482s][info ][gc,marking    ] GC(2) Concurrent Clear Claimed Marks 0.007ms
# 并发扫描Root区
[0.482s][info ][gc,marking    ] GC(2) Concurrent Scan Root Regions
[0.482s][debug][gc,ergo       ] GC(2) Running G1 Root Region Scan using 3 workers for 28 work units.
[0.485s][info ][gc,marking    ] GC(2) Concurrent Scan Root Regions 2.770ms
# 并发标记
[0.485s][info ][gc,marking    ] GC(2) Concurrent Mark
[0.485s][info ][gc,marking    ] GC(2) Concurrent Mark From Roots
[0.485s][info ][gc,task       ] GC(2) Using 3 workers of 3 for marking
[0.488s][info ][gc,marking    ] GC(2) Concurrent Mark From Roots 2.953ms
[0.488s][info ][gc,marking    ] GC(2) Concurrent Preclean
[0.488s][info ][gc,marking    ] GC(2) Concurrent Preclean 0.035ms
# 重标记
[0.492s][info ][gc,start      ] GC(2) Pause Remark
[0.494s][debug][gc,ergo       ] GC(2) Running G1 Update RemSet Tracking Before Rebuild using 3 workers for 1024 regions in heap
[0.495s][info ][gc            ] GC(2) Pause Remark 548M->548M(1024M) 2.433ms
[0.495s][info ][gc,cpu        ] GC(2) User=0.00s Sys=0.00s Real=0.01s
[0.495s][info ][gc,marking    ] GC(2) Concurrent Mark 9.686ms
[0.495s][info ][gc,marking    ] GC(2) Concurrent Rebuild Remembered Sets
[0.496s][info ][gc,marking    ] GC(2) Concurrent Rebuild Remembered Sets 1.135ms
# 清理
[0.497s][info ][gc,start      ] GC(2) Pause Cleanup
[0.497s][debug][gc,ergo,cset  ] GC(2) Pruned 37 regions out of 42, leaving 9696704 bytes waste (allowed 53687091)
[0.497s][info ][gc            ] GC(2) Pause Cleanup 553M->553M(1024M) 0.373ms
```

##### 问题1<a id="question1"></a>

​	是不是并发标记都要跟随在一次新生代的收集后面？如上面的GC(1) Pause Young (Concurrent Start)结束之后才开始并发标记，我发现基本都是这样。在后面会有[验证](#post_yanzheng1)。

#### 2.2.2 混合GC

​	在并发标记阶段后，G1 GC会尝试进行混合GC，但也不是每一次并发标记后都会执行MixedGC。下面的日志显示，在GC(6)中发生了一次新生代GC，这次GC过后就执行了并发标记，之后有5次的新生代收集，依然没有出现MixedGC，而是直接到了Full FC。

​	并发标记阶段有可能会被打断，GC(7)正在进行并发标记阶段，但触发了一个 "G1 Preventive Collection"。在并发标记的过程中，G1 预判当前堆中可用的空闲区域不足，无法满足即将到来的内存分配需求，因此决定提前进行一次 Young GC 来释放空间，避免内存不足的情况发生。

```

[0.916s][info ][gc,start      ] GC(6) Pause Young (Concurrent Start) (G1 Humongous Allocation)
...
[0.919s][info ][gc            ] GC(6) Pause Young (Concurrent Start) (G1 Humongous Allocation) 1011M->1011M(1024M) 3.049ms
[0.919s][info ][gc,cpu        ] GC(6) User=0.01s Sys=0.00s Real=0.00s
[0.919s][info ][gc            ] GC(7) Concurrent Mark Cycle
[0.919s][info ][gc,marking    ] GC(7) Concurrent Clear Claimed Marks
[0.919s][info ][gc,marking    ] GC(7) Concurrent Clear Claimed Marks 0.034ms
[0.919s][info ][gc,marking    ] GC(7) Concurrent Scan Root Regions
[0.919s][debug][gc,ergo       ] GC(7) Running G1 Root Region Scan using 3 workers for 4 work units.
[0.920s][info ][gc,marking    ] GC(7) Concurrent Scan Root Regions 0.334ms
[0.920s][info ][gc,marking    ] GC(7) Concurrent Mark
[0.920s][info ][gc,marking    ] GC(7) Concurrent Mark From Roots
[0.920s][info ][gc,task       ] GC(7) Using 3 workers of 3 for marking
[0.921s][debug][gc,ergo,cset  ] Preventive GC, insufficient free regions. Predicted need 5. Curr Eden 1 (Pred 2). Curr Survivor 4 (Pred 3). Curr Old 140 (Pred 0) Free 5 Alloc 2

[0.922s][info ][gc,start      ] GC(8) Pause Young (Normal) (G1 Preventive Collection)
...
[0.925s][info ][gc            ] GC(8) Pause Young (Normal) (G1 Preventive Collection) 1016M->1015M(1024M) 3.423ms


[0.926s][info ][gc,start      ] GC(9) Pause Young (Normal) (G1 Preventive Collection)
...
[0.929s][info ][gc            ] GC(9) Pause Young (Normal) (G1 Preventive Collection) 1017M->1017M(1024M) 3.042ms


[0.930s][info ][gc,start      ] GC(10) Pause Young (Normal) (G1 Preventive Collection)
...
[0.934s][info ][gc            ] GC(10) Pause Young (Normal) (G1 Preventive Collection) 1018M->1018M(1024M) 3.618ms


[0.934s][info ][gc,start      ] GC(11) Pause Young (Normal) (G1 Preventive Collection)
...
[0.937s][info ][gc            ] GC(11) Pause Young (Normal) (G1 Preventive Collection) 1020M->1021M(1024M) 2.632ms

[0.938s][info ][gc,start      ] GC(12) Pause Young (Normal) (G1 Humongous Allocation)
...
[0.940s][info ][gc            ] GC(12) Pause Young (Normal) (G1 Humongous Allocation) 1022M->1022M(1024M) 2.028ms


[0.961s][info ][gc,start      ] GC(13) Pause Full (G1 Compaction Pause)
...
[1.378s][info ][gc,heap        ] GC(13) Eden regions: 0->0(51)
[1.378s][info ][gc,heap        ] GC(13) Survivor regions: 0->0(0)
[1.378s][info ][gc,heap        ] GC(13) Old regions: 146->115
[1.378s][info ][gc,heap        ] GC(13) Archive regions: 2->2
[1.378s][info ][gc,heap        ] GC(13) Humongous regions: 876->694
...
[1.378s][info ][gc             ] GC(13) Pause Full (G1 Compaction Pause) 1022M->781M(1024M) 416.937ms
```

##### 2.2.2.1 混合GC没有触发的原因

**猜想1<a id="yanzheng1"></a>**： 可能会出现上面提到的并发标记阶段被打断的情况，在本文的测试中，一直在不断地添加对象，G1收集器迫于内存压力，不得不执行一次Young GC 来释放空间。

**猜想2<a id="yanzheng2"></a>**：老年区存活的对象比较多。在GC(13)执行的Full GC 过后，老年区依然存在115个Region，说明老年区的存活对象比较多。执行Mixed GC的收益不高，G1就会延迟或跳出Mixed GC。

##### 2.2.2.2 验证上面的猜测

###### 验证2.2.2.1 中的第一点

```java
  private static final int TOTAL_ELEMENTS = 10000;
  private static WhiteBox wb = WhiteBox.getWhiteBox();
  
  public static void main(String[] args) throws Exception {
        LRUCache<String, Object> cache = new LRUCache<>(CACHE_SIZE);
		
        LongAdder counter = new LongAdder();
        counter.increment();
        while (counter.longValue() < TOTAL_ELEMENTS) {
            String key = generateStrKeyName();
            Object oldValue = cache.get(key);

            if (null == oldValue) {
                Object value = generateObject();
                cache.put(key, value);
                counter.increment();
            }
        }
        wb.g1StartConcMarkCycle();
    }
```

1. 首先查看使用WhiteBox的APi启动并发标记是什么样子的？

可以看到，每次并发标记前，都会执行一次Young GC，整理顺带验证了[问题1](#question1)，不知道对不对？<a id="post_yanzheng1"></a>

可以看到，单纯地调用 wb.g1StartConcMarkCycle();并不会执行后续的Mixed GC的操作。

```
[8.724s][debug][gc,ergo        ] Request concurrent cycle initiation (requested by GC cause). GC cause: WhiteBox Initiated Concurrent Mark
...

[8.724s][info ][gc,start       ] GC(199) Pause Young (Concurrent Start) (WhiteBox Initiated Concurrent Mark)
...
[8.728s][info ][gc             ] GC(199) Pause Young (Concurrent Start) (WhiteBox Initiated Concurrent Mark) 895M->895M(1024M) 4.407ms

[8.728s][info ][gc,cpu         ] GC(199) User=0.02s Sys=0.00s Real=0.00s
# 并发标记开始
[8.728s][info ][gc             ] GC(200) Concurrent Mark Cycle
...省略并发标记的内容
[8.751s][info ][gc             ] GC(200) Concurrent Mark Cycle 22.673ms
#后续没有MixedGC的操作
```

2. 使用WhiteBox的API完整地进行一次Mixed GC的全过程

这里借鉴了 MixedGCProvoker.java 中 provokeMixedGC() 方法的写法。

```java
// test/hotspot/jtreg/gc/testlibrary/g1/MixedGCProvoker.java
public static void provokeMixedGC(List<byte[]> liveOldObjects) {
        Helpers.waitTillCMCFinished(getWhiteBox(), 10);
        getWhiteBox().g1StartConcMarkCycle();
        Helpers.waitTillCMCFinished(getWhiteBox(), 10);
        getWhiteBox().youngGC(); // the "Prepare Mixed" gc
        getWhiteBox().youngGC(); // the "Mixed" gc

        // check that liveOldObjects still alive
        assertTrue(getWhiteBox().isObjectInOldGen(liveOldObjects), "List of the objects is suppose to be in OldGen");
    }
```



这里是测试MixedGC的回收过程，我的测试函数

```java
private static final int TOTAL_ELEMENTS = 10000;
private static WhiteBox wb = WhiteBox.getWhiteBox();

public static void main(String[] args) throws Exception {
        LRUCache<String, Object> cache = new LRUCache<>(CACHE_SIZE);
        LongAdder counter = new LongAdder();
        counter.increment();
        while (counter.longValue() < TOTAL_ELEMENTS) {
            String key = generateStrKeyName();
            Object oldValue = cache.get(key);
            if (null == oldValue) {
                Object value = generateObjectWithInt();
                cache.put(key, value);
                counter.increment();
            }
        }
        startMixedGC();
    }

    public static void startMixedGC() {
        wb.g1StartConcMarkCycle();
        while (wb.g1InConcurrentMark()) {
            try{
                Thread.sleep(1000);
            } catch (Exception e) {
            }
        }
        wb.youngGC(); 
        wb.youngGC(); 
    }
```



完整的Mixed GC日志

```
[10.177s][debug][gc,ergo,ihop   ] Request concurrent cycle initiation (occupancy higher than threshold) occupancy: 844103680B allocation request: 0B threshold: 495570703B (46.15) source: STW humongous allocation
[10.178s][debug][gc,ergo        ] Request concurrent cycle initiation (requested by GC cause). GC cause: WhiteBox Initiated Concurrent Mark
# 并发标记前的Young GC
[10.178s][info ][gc,start       ] GC(299) Pause Young (Concurrent Start) (WhiteBox Initiated Concurrent Mark)
...
[10.180s][info ][gc             ] GC(299) Pause Young (Concurrent Start) (WhiteBox Initiated Concurrent Mark) 774M->774M(1024M) 1.619ms

# 并发标记
[10.180s][info ][gc             ] GC(300) Concurrent Mark Cycle
... 省略并发标记内容
[10.189s][info ][gc             ] GC(300) Concurrent Mark Cycle 8.755ms

# 预先的混合GC
[11.185s][info ][gc,start       ] GC(301) Pause Young (Prepare Mixed) (WhiteBox Initiated Young GC)
...
[11.193s][info ][gc             ] GC(301) Pause Young (Prepare Mixed) (WhiteBox Initiated Young GC) 774M->774M(1024M) 7.987ms

# 混合GC
[11.193s][info ][gc,start       ] GC(302) Pause Young (Mixed) (WhiteBox Initiated Young GC)
...
[11.198s][info ][gc             ] GC(302) Pause Young (Mixed) (WhiteBox Initiated Young GC) 774M->774M(1024M) 4.517ms

```



3. 明白了MixedGC的完整执行流程，现在回到我们需要验证的第一点[验证的第一点](#yanzheng1)

验证思路是：利用WhiteBox的Api启动Mixed GC，但是不等并发标记完成，依然不断地分配对象，看看是否会打断Mixed GC。

测试代码

```java
public static void main(String[] args) throws Exception {
        LRUCache<String, Object> cache = new LRUCache<>(CACHE_SIZE);
        LongAdder counter = new LongAdder();
        counter.increment();
        while (counter.longValue() < TOTAL_ELEMENTS) {
            if (counter.longValue() % 1000 == 0) {
                startMixedGC(); // 开始Mixed GC
            }
            String key = generateStrKeyName();
            Object oldValue = cache.get(key);

            if (null == oldValue) {
                Object value = generateObjectWithInt();
                cache.put(key, value);
                counter.increment();
            }
        }
    }

    public static  void startMixedGC() {
        wb.g1StartConcMarkCycle();
        // 这里注释掉了，没有等待并发标记完成
//        while (wb.g1InConcurrentMark()) {
//            try{
//                Thread.sleep(1000);
//            } catch (Exception e) {
//            }
//        }
        wb.youngGC(); // the "Prepare Mixed" gc
        wb.youngGC(); // the "Mixed" gc
    }
```



可以看到，在测试代码中，打开wb.g1StartConcMarkCycle();之后，并没有等待并发标记完成，在日志中可以看到，并发标记被打断了，后续的Mixed GC也没有执行。[验证1](#yanzheng1)得到验证

```
# 并发标记前的 Young GC
[2.256s][info ][gc,start       ] GC(56) Pause Young (Concurrent Start) (WhiteBox Initiated Concurrent Mark)
...
[2.263s][info ][gc             ] GC(56) Pause Young (Concurrent Start) (WhiteBox Initiated Concurrent Mark) 867M->867M(1024M) 7.588ms

# 并发标记开始
[2.263s][info ][gc             ] GC(57) Concurrent Mark Cycle
[2.263s][info ][gc,marking     ] GC(57) Concurrent Clear Claimed Marks
[2.263s][info ][gc,marking     ] GC(57) Concurrent Clear Claimed Marks 0.005ms
[2.263s][info ][gc,marking     ] GC(57) Concurrent Scan Root Regions
[2.263s][debug][gc,ergo        ] GC(57) Running G1 Root Region Scan using 3 workers for 14 work units.
[2.264s][info ][gc,marking     ] GC(57) Concurrent Scan Root Regions 0.305ms
[2.264s][info ][gc,marking     ] GC(57) Concurrent Mark
[2.264s][info ][gc,marking     ] GC(57) Concurrent Mark From Roots

# 到这里，并发标记被打断了。
[2.264s][debug][gc,heap        ] GC(58) Heap before GC invocations=54 (full 6):
[2.264s][debug][gc,heap        ] GC(58)  garbage-first heap   total 1048576K, used 888673K [0x00000000c0000000, 0x0000000100000000)
[2.264s][debug][gc,heap        ] GC(58)   region size 1024K, 8 young (8192K), 8 survivors (8192K)
[2.264s][debug][gc,heap        ] GC(58)  Metaspace       used 545K, committed 704K, reserved 1114112K
[2.264s][debug][gc,heap        ] GC(58)   class space    used 37K, committed 128K, reserved 1048576K
[2.264s][info ][gc,task        ] GC(57) Using 3 workers of 3 for marking

# 执行第一个wb.youngGC();此时已经不是 Pause Young (Prepare Mixed) (WhiteBox Initiated Young GC)
[2.264s][info ][gc,start       ] GC(58) Pause Young (Normal) (WhiteBox Initiated Young GC)
...
[2.270s][info ][gc             ] GC(58) Pause Young (Normal) (WhiteBox Initiated Young GC) 867M->867M(1024M) 5.630ms

# 执行第二个wb.youngGC();此时已经不是 Pause Young (Mixed) (WhiteBox Initiated Young GC)
[2.270s][info ][gc,start       ] GC(59) Pause Young (Normal) (WhiteBox Initiated Young GC)
...
[2.271s][info ][gc             ] GC(59) Pause Young (Normal) (WhiteBox Initiated Young GC) 867M->867M(1024M) 1.532ms
```



##### 问题2<a id="question2"></a>

​	由完整的Mixed GC产生的问题。为什么完整的MixedGC中，会执行Pause Young (Prepare Mixed)，从具体的日志中可以看到Pause Young (Prepare Mixed)和 Pause Young (Mixed)做的事情一样的?

```
# Pause Young (Prepare Mixed)开始
[11.185s][info ][gc,start       ] GC(301) Pause Young (Prepare Mixed) (WhiteBox Initiated Young GC)
[11.185s][info ][gc,task        ] GC(301) Using 10 workers of 10 for evacuation
[11.185s][trace][gc,ergo,cset   ] GC(301) Start choosing CSet. Pending cards: 34 target pause time: 200.00ms
[11.185s][trace][gc,ergo,cset   ] GC(301) Added young regions to CSet. Eden: 0 regions, Survivors: 1 regions, predicted eden time: 0.09ms, predicted base time: 1.13ms, target pause time: 200.00ms, remaining time: 198.77ms
[11.186s][debug][gc,ergo        ] GC(301) Running G1 Merge Heap Roots using 10 workers for 1 regions
[11.191s][debug][gc,ergo        ] GC(301) Running G1 Rebuild Free List Task using 10 workers for rebuilding free list of regions
[11.192s][debug][gc,ergo,heap   ] GC(301) Heap expansion: short term pause time ratio 21.28% long term pause time ratio 28.78% threshold 7.69% pause time ratio 7.69% fully expanded true resize by 0B
[11.192s][debug][gc,ergo,refine ] GC(301) Concurrent refinement times: Logged Cards Scan time goal: 20.00ms Logged Cards Scan time: 0.07ms HCC time: 0.00ms
[11.192s][trace][gc,ergo,refine ] GC(301) Updating Refinement Zones: logged cards scan time: 0.069ms, processed cards: 25, goal time: 19.999ms
[11.192s][debug][gc,ergo,refine ] GC(301) Updated Refinement Zones: green: 2560, yellow: 7680, red: 12800
[11.192s][info ][gc,phases      ] GC(301)   Pre Evacuate Collection Set: 1.0ms
[11.192s][info ][gc,phases      ] GC(301)   Merge Heap Roots: 1.1ms
[11.192s][info ][gc,phases      ] GC(301)   Evacuate Collection Set: 2.4ms
[11.192s][info ][gc,phases      ] GC(301)   Post Evacuate Collection Set: 2.3ms
[11.192s][info ][gc,phases      ] GC(301)   Other: 1.1ms
[11.192s][info ][gc,heap        ] GC(301) Eden regions: 0->0(50)
[11.192s][info ][gc,heap        ] GC(301) Survivor regions: 1->1(7)
[11.192s][info ][gc,heap        ] GC(301) Old regions: 111->111
[11.192s][info ][gc,heap        ] GC(301) Archive regions: 2->2
[11.192s][info ][gc,heap        ] GC(301) Humongous regions: 692->692
[11.192s][info ][gc,metaspace   ] GC(301) Metaspace: 535K(704K)->535K(704K) NonClass: 500K(576K)->500K(576K) Class: 35K(128K)->35K(128K)
[11.193s][debug][gc,heap        ] GC(301) Heap after GC invocations=279 (full 40):
[11.193s][debug][gc,heap        ] GC(301)  garbage-first heap   total 1048576K, used 792926K [0x00000000c0000000, 0x0000000100000000)
[11.193s][debug][gc,heap        ] GC(301)   region size 1024K, 1 young (1024K), 1 survivors (1024K)
[11.193s][debug][gc,heap        ] GC(301)  Metaspace       used 535K, committed 704K, reserved 1114112K
[11.193s][debug][gc,heap        ] GC(301)   class space    used 35K, committed 128K, reserved 1048576K
[11.193s][info ][gc             ] GC(301) Pause Young (Prepare Mixed) (WhiteBox Initiated Young GC) 774M->774M(1024M) 7.987ms
# Pause Young (Prepare Mixed)结束
 

# Pause Young (Mixed) (WhiteBox Initiated Young GC)开始
[11.193s][info ][gc,cpu         ] GC(301) User=0.01s Sys=0.00s Real=0.01s
[11.193s][debug][gc,heap        ] GC(302) Heap before GC invocations=279 (full 40):
[11.193s][debug][gc,heap        ] GC(302)  garbage-first heap   total 1048576K, used 792926K [0x00000000c0000000, 0x0000000100000000)
[11.193s][debug][gc,heap        ] GC(302)   region size 1024K, 1 young (1024K), 1 survivors (1024K)
[11.193s][debug][gc,heap        ] GC(302)  Metaspace       used 535K, committed 704K, reserved 1114112K
[11.193s][debug][gc,heap        ] GC(302)   class space    used 35K, committed 128K, reserved 1048576K
[11.193s][info ][gc,start       ] GC(302) Pause Young (Mixed) (WhiteBox Initiated Young GC)
[11.194s][info ][gc,task        ] GC(302) Using 10 workers of 10 for evacuation
[11.194s][trace][gc,ergo,cset   ] GC(302) Start choosing CSet. Pending cards: 186 target pause time: 200.00ms
[11.194s][trace][gc,ergo,cset   ] GC(302) Added young regions to CSet. Eden: 0 regions, Survivors: 1 regions, predicted eden time: 0.09ms, predicted base time: 3.08ms, target pause time: 200.00ms, remaining time: 196.83ms
[11.194s][debug][gc,ergo,cset   ] GC(302) Start adding old regions to collection set. Min 2 regions, max 103 regions, time remaining 196.83ms, optional threshold 39.37ms
[11.194s][debug][gc,ergo,cset   ] GC(302) Finish adding old regions to collection set (Predicted time too high).
[11.194s][debug][gc,ergo,cset   ] GC(302) Added 2 initial old regions to collection set although the predicted time was too high.
[11.194s][debug][gc,ergo,cset   ] GC(302) Finish choosing collection set old regions. Initial: 2, optional: 0, predicted old time: 0.00ms, predicted optional time: 0.00ms, time remaining: 0.00
[11.195s][debug][gc,ergo        ] GC(302) Running G1 Merge Heap Roots using 10 workers for 3 regions
[11.197s][debug][gc,ergo        ] GC(302) Running G1 Rebuild Free List Task using 10 workers for rebuilding free list of regions
[11.198s][debug][gc,ergo,heap   ] GC(302) Heap expansion: short term pause time ratio 0.73% long term pause time ratio 8.24% threshold 7.69% pause time ratio 7.69% fully expanded true resize by 0B
[11.198s][debug][gc,ergo,ihop   ] GC(302) Do not request concurrent cycle initiation (still doing mixed collections) occupancy: 843055104B allocation request: 0B threshold: 523723709B (48.78) source: end of GC
[11.198s][debug][gc,ergo,refine ] GC(302) Concurrent refinement times: Logged Cards Scan time goal: 20.00ms Logged Cards Scan time: 0.13ms HCC time: 0.00ms
[11.198s][trace][gc,ergo,refine ] GC(302) Updating Refinement Zones: logged cards scan time: 0.128ms, processed cards: 177, goal time: 19.999ms
[11.198s][debug][gc,ergo,refine ] GC(302) Updated Refinement Zones: green: 2560, yellow: 7680, red: 12800
[11.198s][info ][gc,phases      ] GC(302)   Pre Evacuate Collection Set: 1.0ms
[11.198s][info ][gc,phases      ] GC(302)   Merge Heap Roots: 0.6ms
[11.198s][info ][gc,phases      ] GC(302)   Evacuate Collection Set: 1.1ms
[11.198s][info ][gc,phases      ] GC(302)   Post Evacuate Collection Set: 1.4ms
[11.198s][info ][gc,phases      ] GC(302)   Other: 0.8ms
[11.198s][info ][gc,heap        ] GC(302) Eden regions: 0->0(50)
[11.198s][info ][gc,heap        ] GC(302) Survivor regions: 1->1(7)
[11.198s][info ][gc,heap        ] GC(302) Old regions: 111->110
[11.198s][info ][gc,heap        ] GC(302) Archive regions: 2->2
[11.198s][info ][gc,heap        ] GC(302) Humongous regions: 692->692
[11.198s][info ][gc,metaspace   ] GC(302) Metaspace: 535K(704K)->535K(704K) NonClass: 500K(576K)->500K(576K) Class: 35K(128K)->35K(128K)
[11.198s][debug][gc,heap        ] GC(302) Heap after GC invocations=280 (full 40):
[11.198s][debug][gc,heap        ] GC(302)  garbage-first heap   total 1048576K, used 792926K [0x00000000c0000000, 0x0000000100000000)
[11.198s][debug][gc,heap        ] GC(302)   region size 1024K, 1 young (1024K), 1 survivors (1024K)
[11.198s][debug][gc,heap        ] GC(302)  Metaspace       used 535K, committed 704K, reserved 1114112K
[11.198s][debug][gc,heap        ] GC(302)   class space    used 35K, committed 128K, reserved 1048576K
[11.198s][info ][gc             ] GC(302) Pause Young (Mixed) (WhiteBox Initiated Young GC) 774M->774M(1024M) 4.517ms
[11.198s][info ][gc,cpu         ] GC(302) User=0.01s Sys=0.00s Real=0.00s
```





## 3 获取Region信息

在弄清楚了并发标记和混合GC的过程，下面实现获取Region信息。先通过日志了解Region的打印信息，然后再通过WhiteBox的API获取日志信息。

### 3.1 通过日志获取

在打印日志中，加入虚拟机参数 *gc+liveness=trace* 便可以获取到具体的日志信息，完整的参数列别如下:

```
 -XX:+UnlockDiagnosticVMOptions -XX:+UnlockExperimentalVMOptions -XX:InitiatingHeapOccupancyPercent=50 -XX:G1MixedGCLiveThresholdPercent=85 -XX:+WhiteBoxAPI -XX:G1HeapRegionSize=2m -Xms1024m -Xmx1024m -XX:+UseG1GC -Xlog:gc*=info,gc+heap=debug,gc+ergo*=trace:/home/damowang/Tencent/project/Jdk_issue2/log/G1MixedGC.log:uptime,level,tags

```



获取到的日志形式如下：

```
type                      address-range      used   prev-live  next-live   remset state code-roots
											(bytes)   (bytes)    (bytes)   (bytes)        (bytes)
HUMS 0x00000000fec00000-0x00000000fee00000  2097152    2097152    2097152    4480   CMPLT     16
HUMS 0x00000000fee00000-0x00000000ff000000  2097152    2097152    2097152    4480   CMPLT     16
HUMS 0x00000000ff000000-0x00000000ff200000  2097152    2097152    2097152    4480   CMPLT     16
HUMS 0x00000000ff200000-0x00000000ff400000  2097152    2097152    2097152    4480   CMPLT     16
FREE 0x00000000ff400000-0x00000000ff600000        0          0          0    4480   UNTRA     16
FREE 0x00000000ff600000-0x00000000ff800000        0          0          0    4480   UNTRA     16
SURV 0x00000000ff800000-0x00000000ffa00000   781672     781672     781672    5176   CMPLT     712
EDEN 0x00000000ffa00000-0x00000000ffc00000  1656912    1656912    1656912    4480   CMPLT     16
OARC 0x00000000ffc00000-0x00000000ffe00000  1527808    1527808     357536    5152   UNTRA     688
CARC 0x00000000ffe00000-0x0000000100000000   487424     487424     485816    5248   UNTRA     784
```

type：堆中内存区域的类型。 HUMS（Humongous region），FREE（空闲的Region）,SURV（Survivor region），EDEN（Eden region），OLD（Old region）。

address-range：region的内存地址范围，比如第一个region：0x00000000fec00000-0x00000000fee00000，其中0x00000000fec00000转换为十进制为4273995776，0x00000000fee00000转换为十进制为4276092928，内存大小为4276092928 - 4273995776 = 2097152B，转换为M的话就是 2097152B / 1024 / 1024 = 2M。跟我们设置的-XX:G1HeapRegionSize=2m是一致的。

used：region被使用的内存大小，这里是以字节为单位的。

prev-live和next-live：分别为上一次标记后堆中对象的存活量，当前标记后堆中对象的存活量，也是以字节为单位。



### 3.2 通过扩写WhiteBox的API获取

​	在本章节下面，本文探讨如何通过WhiteBox的API获取内存信息。

​	WhiteBox.java文件中有很多测试方法，也声明了很多本地方法(native method)，用 native 关键字标识方法，但不提供实现。这些native方法会使用c++语言在whitebox.cpp中实现。在 Java 中调用 C++ 代码，通常使用的是 **JNI（Java Native Interface）**，JNI 是 Java 与本地代码之间的桥梁，允许你在 Java 应用程序中调用用 C/C++ 编写的函数，或者从 C/C++ 中调用 Java 方法。它适用于性能优化、与系统硬件的直接交互或是使用现有的 C/C++ 库。

本文实验中想获取 G1 收集器中老区(old region)中的每个region的信息，WhiteBox.java文件中并没有直接提供相应的 API。下面是本文自己实现的API接口的过程。

#### 3.2.1 实现WhiteBox的获取Region信息的API

1. 首先需要在WhiteBox.java中声明本地方法。

```java
// test/lib/jdk/test/whitebox/WhiteBox.java
public native long[][] getG1OldRegionLiveObject(int liveness);
```

2. 在whitebox.cpp中实现这个方法。那这部分该如何写呢？我在WhiteBox.java中找到一个类似的接口g1GetMixedGCInfo(int)，根据接口名称可以猜测该接口可以获取混合GC的信息，然后就在whitebox.cpp文件中搜索这个接口名称。

```java
// test/lib/jdk/test/whitebox/WhiteBox.java
public native long[] g1GetMixedGCInfo(int liveness);
```

最后找到了这个接口的c++实现。

```c++
// src/hotspot/share/prims/whitebox.cpp
WB_ENTRY(jlongArray, WB_G1GetMixedGCInfo(JNIEnv* env, jobject o, jint liveness))
  if (!UseG1GC) {
    THROW_MSG_NULL(vmSymbols::java_lang_UnsupportedOperationException(), "WB_G1GetMixedGCInfo: G1 GC is not enabled");
  }
  if (liveness < 0) {
    THROW_MSG_NULL(vmSymbols::java_lang_IllegalArgumentException(), "liveness value should be non-negative");
  }
	// 获取G1收集器中堆的信息。
  G1CollectedHeap* g1h = G1CollectedHeap::heap();
	// 这里声明一个 OldRegionsLivenessClosure对象，其基类为HeapRegionClosure，它定义了用于遍历和处理 G1 GC 堆中的 HeapRegion 的基本行为
  OldRegionsLivenessClosure rli(liveness);
	// 这里迭代获取每个region信息，我们继续跟踪
  g1h->heap_region_iterate(&rli);

  typeArrayOop result = oopFactory::new_longArray(3, CHECK_NULL);
  result->long_at_put(0, rli.total_count());
  result->long_at_put(1, rli.total_memory());
  result->long_at_put(2, rli.total_memory_to_free());
  return (jlongArray) JNIHandles::make_local(THREAD, result);
WB_END
  
 // src/hotspot/share/gc/g1/g1CollectedHeap.cpp
 void G1CollectedHeap::heap_region_iterate(HeapRegionClosure* cl) const {
  _hrm.iterate(cl); // 由上下文得知 _hrm 是一个HeapRegionManager对象
}

// src/hotspot/share/gc/g1/heapRegionManager.cpp
void HeapRegionManager::iterate(HeapRegionClosure* blk) const {
    // 获取regions的数量
  uint len = reserved_length();

  for (uint i = 0; i < len; i++) {
    if (!is_available(i)) {
      continue;
    }
    guarantee(at(i) != NULL, "Tried to access region %u that has a NULL HeapRegion*", i);
      // 在这里获取每个region的信息，这里调用do_heap_region，又回到了whitebox.cpp中
    bool res = blk->do_heap_region(at(i));
    if (res) {
      blk->set_incomplete();
      return;
    }
  }
}

// src/hotspot/share/prims/whitebox.cpp
class OldRegionsLivenessClosure: public HeapRegionClosure {
 private:
  const int _liveness;
  size_t _total_count;
  size_t _total_memory;
  size_t _total_memory_to_free;
 public:
  OldRegionsLivenessClosure(int liveness) :
    _liveness(liveness),
    _total_count(1),
    _total_memory(2),
    _total_memory_to_free(3) { }

    size_t total_count() { return _total_count; }
    size_t total_memory() { return _total_memory; }
    size_t total_memory_to_free() { return _total_memory_to_free; }

  bool do_heap_region(HeapRegion* r) {
    if (r->is_old()) {
      size_t prev_live = r->marked_bytes(); //前一次标记后存活的对象量
      size_t live = r->live_bytes(); // 本次标记后存活的对象量
      size_t size = r->used(); // HeapRegion中被使用的内存
      size_t reg_size = HeapRegion::GrainBytes; // HeapRegion的内存大小
      if (size > 0 && ((int)(live * 100 / size) < _liveness)) {
        _total_memory += size; // 总的使用量
        ++_total_count; // 标记次数
        if (size == reg_size) {
          _total_memory_to_free += size - prev_live; // 总的释放空间
        }
      }
    }
    return false;
  }
```



至此，整个g1GetMixedGCInfo()接口的实现结束。但是上面的接口不符合本实验的要求，这里是将所有老区的Region信息汇总，而我们需要获取每个Region的信息，所以我们得自己实现WhiteBox的API。

```c++
// src/hotspot/share/prims/whitebox.cpp
WB_ENTRY(jobjectArray, WB_G1OldRegionLiveObject(JNIEnv* env, jobject o, jint liveness)) {
    // 获取G1的堆信息
    G1CollectedHeap* g1h = G1CollectedHeap::heap();
    // 这里获取老区的old_set，这是自己实现的函数，无法获取里面每一个Region，但能获取到老区Region的数量
    HeapRegionSetBase old_set = g1h->get_old_set(); // 获取老区region
    int numRegions = old_set.length(); // 获取老区Region的数量
    // 这里声明一个二维long[][]数组，用来存储每个region的信息
    objArrayOop  result = oopFactory::new_objectArray(numRegions, CHECK_NULL);
    for (int i = 0; i < numRegions; i++) {
        typeArrayOop longArray = oopFactory::new_longArray(4, CHECK_NULL);
        result->obj_at_put(i, longArray);
    }
    // 参照上面的实现
    OldRegionsLivenessClosure rli(liveness);
    // 将每个region的信息加入到result数组中
    g1h->heap_region_iterate_dbarray(numRegions, result, &rli);
    return (jobjectArray) JNIHandles::make_local(THREAD, result);
}
WB_END

// src/hotspot/share/gc/g1/g1CollectedHeap.inline.hpp
inline HeapRegionSet G1CollectedHeap::get_old_set() {
    return _old_set;
}

// src/hotspot/share/gc/g1/g1CollectedHeap.cpp
void G1CollectedHeap::heap_region_iterate_dbarray(int oldSetLenth, objArrayOop arr, HeapRegionClosure* cl) const {
    _hrm.iterate_dbarray(oldSetLenth, arr, cl);
}

// src/hotspot/share/gc/g1/heapRegionManager.cpp
void HeapRegionManager::iterate_dbarray(int oldSetLenth, objArrayOop arr, HeapRegionClosure* blk) const {
  uint len = reserved_length(); // 这里获取的是所有的region，包括新生代，老年代，大对象区
  int index = 0;
  for (uint i = 0; i < len; i++) {
	...
    bool res = blk->do_heap_region_dbarray(index, arr, at(i));
    if (res) {
      index++;
      if (index==oldSetLenth) return;
    }
  }
}


// src/hotspot/share/gc/g1/heapRegion.cpp
 bool do_heap_region_dbarray(int index, objArrayOop objArr, HeapRegion* r){
  if (r->is_old()) {
        typeArrayOop arr = objArr->obj_at(index);
        size_t prev_live = r->marked_bytes(); // 先前标记对象量
        size_t live = r->live_bytes(); // 现在标记对象量
        size_t size = r->used(); // region中已使用的内存
        size_t reg_size = HeapRegion::GrainBytes; //region大小
      	// 下面依次将region信息放入 objArrayOop数组中
        arr->long_at_put(0, prev_live);
        arr->long_at_put(1, live);
        arr->long_at_put(2, size);
        arr->long_at_put(3, reg_size);
       return true; // 如果是老区就返回 true
    }
    return false; // 不是老区返回false
}
```

##### 问题3<a id="question3"></a>

如何保证 get_old_set()接口能返回所有老区的信息，并且能获取老区的region数量？（后来知道）其实也可以直接通过 G1ColletedHeap的 uint old_regions_count() const { return _old_set.length(); } 方法获取。

1. 首先找到 _old_set 的声明

```c++
// src/hotspot/share/gc/g1/g1CollectedHeap.hpp
class G1CollectedHeap : public CollectedHeap {
 ...
     HeapRegionSet _old_set;
 ...
}
// 往 _old_set增加元素时
inline void G1CollectedHeap::old_set_add(HeapRegion* hr) {
  _old_set.add(hr);
}

// HeapRegionSet在src/hotspot/share/gc/g1/heapRegionSet.hpp中实现，它继承了HeapRegionSetBase，而它继承了HeapRegionSetBase里面有add方法的实现。
// src/hotspot/share/gc/g1/heapRegionSet.inline.hpp
inline void HeapRegionSetBase::add(HeapRegion* hr) {
  ...
  _length++; // 可以看到，每增加一个Region，这个_length属性就增加一
  ...
}
```

2. 通过实验证明

实验代码：

```java
  public static void main(String[] args) throws Exception {
        LRUCache<String, Object> cache = new LRUCache<>(CACHE_SIZE);
        long startMillis = System.currentTimeMillis();
        long timeoutMillis = 10000;
        long endMillis = startMillis + timeoutMillis;
        System.out.println("正在执行...");
        LongAdder counter = new LongAdder();
        while (System.currentTimeMillis() < endMillis) {
            String key = generateStrKeyName();
            Object oldValue = cache.get(key);
            if (null == oldValue) {
                Object value = generateObjectWithInt();
                cache.put(key, value);
                counter.increment();
            }
        }
        System.out.println("每个Region的结果:");
      // 在这里调用getG1OldRegionLiveObject，获取每个region 的信息
        long[][] regionLen = wb.getG1OldRegionLiveObject(50);
        for(int i = 0; i < regionLen.length; i++) {
            System.out.println("第" + i + "个：" + regionLen[i][0] + ' ' + regionLen[i][1] + ' ' + regionLen[i][2] + ' ' + regionLen[i][3]);
        }
    }
```

输出的结果

```
每个Region的结果:
第0个：961176 961176 961176 1048576
第1个：875984 875984 875984 1048576
...
第116个：525112 525112 525112 1048576
第117个：262632 262632 262632 1048576

```



再查看日志证明：这是最后一次垃圾收集打印出来的信息堆栈，可以看到老年区有118个region，跟上面输出的结果是一样的。

```
[10.199s][info ][gc,heap        ] GC(292) Eden regions: 0->0(51)
[10.199s][info ][gc,heap        ] GC(292) Survivor regions: 2->0(7)
[10.199s][info ][gc,heap        ] GC(292) Old regions: 158->118
[10.199s][info ][gc,heap        ] GC(292) Archive regions: 2->2
[10.199s][info ][gc,heap        ] GC(292) Humongous regions: 856->652
```



在这里也发现一个有趣的现象，在周志明的《深入理解Java虚拟机——JVM高级特性与最佳实践》一书中，这样子介绍Humongous区域。

```
G1认为只要大小超过了一个Region容量一半的对象即可判定为大对象。...。G1的大多数行为都把Humongous Region作为老年代的一部分来看待。
```

在我们实际得到的结果中，老年代的 Old regions和大对象的Humongous regions是分开的，我们使用HeapRegion的方法is_old()来判断Region的分代时，也是不将Humongous regions当做老年代来看待的。

#### 3.2.2 测试获得Region信息

##### 3.2.2.1 测试代码1

在分配完所有对象之后，查看Region信息

```java
public class TestLRUG1GCTest {

    private static final int CACHE_SIZE = 1000;
    private static final int TOTAL_ELEMENTS = 10000;
    private static WhiteBox wb = WhiteBox.getWhiteBox();
    private static Random random = new Random();
    private static final int BIG_OBJECT = 1024 * 1024;
    private static final int SMALL_OBJECT = 256 * 1024;
    private static String poolName = "G1 Old Gen";

    public static void main(String[] args) throws Exception {
        LRUCache<String, Object> cache = new LRUCache<>(CACHE_SIZE);
        long startMillis = System.currentTimeMillis();
        long timeoutMillis = 10000;
        long endMillis = startMillis + timeoutMillis;
        System.out.println("正在执行...");
        LongAdder counter = new LongAdder();
        while (System.currentTimeMillis() < endMillis) {
            String key = generateStrKeyName();
            Object oldValue = cache.get(key);
            if (null == oldValue) {
                Object value = generateObject();
                cache.put(key, value);
                counter.increment();
            }
        }
        System.out.println("每个Region的结果:");
        long[][] regionLen = wb.getG1OldRegionLiveObject(50); // 这里获取Regions信息
        long total_usage = 0;
        for(int i = 0; i < regionLen.length; i++) {
            total_usage += regionLen[i][2];
            System.out.println("第" + i + "个：" + regionLen[i][0] + ' ' + regionLen[i][1] + ' ' + regionLen[i][2] + ' ' + regionLen[i][3]);
        }
        System.out.println("总使用量: " + total_usage);
    }

    public static Object generateObject() {
        int index = random.nextInt(3);
        switch (index) {
            case 0: {
                return new byte[BIG_OBJECT];
            }
            case 1: {
                return new byte[SMALL_OBJECT];
            }
            case 2: {
                return random.nextInt(100);
            }
        }
        return null;
    }
    
    public static String generateStrKeyName() {
        // 如果 cache 填充满了对象，有 1 / 3 的概率这个对象会被重新激活，排在前面
        int index = random.nextInt(3 * CACHE_SIZE);
        return "name_" + index;
    }
}

```

虚拟机参数

```
 -XX:+UnlockDiagnosticVMOptions -XX:+UnlockExperimentalVMOptions -XX:InitiatingHeapOccupancyPercent=50 -XX:G1MixedGCLiveThresholdPercent=85 -XX:+WhiteBoxAPI -XX:G1HeapRegionSize=1m -Xms1024m -Xmx1024m -XX:+UseG1GC
```



输出的日志

```
每个Region的结果:
	 pre  next   used   region_size
第0个：0  822576  822576  1048576
第1个：0  854376  854376  1048576
第2个：0  1022168 1022168 1048576
第3个：0  787304  787304  1048576
...
第106个：0 786480 786480  1048576
第107个：0 786480 786480  1048576
第108个：0 262160 262160  1048576
总使用量: 83796256
```

###### 问题4<a id="question4"></a>

输出的内容如上面：可以观察到很有趣的现象，这样子测试的话，上一次标记后对象的存活量（Region）为0（这里不明白什么原因，望老师指教）。



##### 3.2.2.2 测试代码2

如果在测试代码中加上下面的代码，getOldGenInfo()函数用来获取老区的信息，这个方法是借鉴了test/hotspot/jtreg/gc/g1/mixedgc/TestOldGenCollectionUsage.java下的run()方法的实现。

```java
public static void main(String[] args){
    	...
    	getUsageInfo();
}

 public static void getUsageInfo() {
        System.out.println("每个Region的结果:");
        long[][] regionLen = wb.getG1OldRegionLiveObject2(50);
        long total_usage = 0;
        long total_live = 0;
        for(int i = 0; i < regionLen.length; i++) {
            total_usage += regionLen[i][2];
            total_live += regionLen[i][1];
            System.out.println("第" + i + "个：" + regionLen[i][0] + ' ' + regionLen[i][1] + ' ' + regionLen[i][2] + ' ' + regionLen[i][3]);
        }
        System.out.println("总使用量: " + total_usage);
        System.out.println("总存活量: " + total_live);
        System.out.println("输出老年代的结果: ");
        getOldGenInfo();
    }

public static void getOldGenInfo() {
    List<MemoryPoolMXBean> pools = ManagementFactory.getMemoryPoolMXBeans();
    MemoryPoolMXBean pool = null;
    boolean foundPool = false;
    for (int i = 0; i < pools.size(); i++) {
        pool = pools.get(i);
        String name = pool.getName();
        // 这里查找名称包含 "G1 Old Gen" 的内存池
        if (name.contains(poolName)) {
            System.out.println("Found pool: " + name);
            foundPool = true;
            break;
        }
    }
    if (!foundPool) {
        throw new RuntimeException(poolName + " not found, test with -XX:+UseG1GC");
    }
    long baseUsage = pool.getCollectionUsage().getUsed(); // 获取老区的使用量
    System.out.println(poolName + ": usage after GC = " + baseUsage);
}
```

###### 问题5<a id="question5"></a>

输出的结果如下：可以看到自己写的代码中统计出来的老区的region的使用量跟使用MemoryPoolMXBean统计出来的堆使用量出入很大？问题出在哪里呢?

```
// 这里数字跟上面的一样，我是跑一次实验，分开来叙述，这里的单位都是字节
总使用量: 83796256 (79.9M)
输出老年代的结果:
Found pool: G1 Old Gen
G1 Old Gen: usage after GC = 835543328(796.8M)
```

我的猜测是MemoryPoolMXBean统计的是全部的老区(old regions + humongous regions )区域的使用量，我统计的仅仅是 old regions的使用量。

**验证问题5**

我在WSL中编译的JDK源码，并在本地的IDEA进行开发（这可能是很笨的方法），没有办法从代码层面进行debug验证，下面通过测试进行验证。在原本的测试代码中，生成的大对象是 1M 的，刚好等于HeapRegion的大小，导致很多对象直接被分配到Humongous Region里面了，在我的测试代码中获取不到这部分区域。所以下面修改生成对象的方法，不允许生成大对象，只生成小对象。

验证思路：只生成小对象，这样子 Humongous Region中就没有对象了，老区的对象只存储在 Old Region中，计算 Old Region中已使用的内存，跟MemoryPoolMXBean统计的数据相比较就可以。

```java
 public static Object generateObject() {
    int index = random.nextInt(100, 500);
    return new byte[index * 1024];
}
```



输出结果如下，在我的每次实验中，总使用量都会比通过MemoryPoolMXBean获取到的使用量要高。为什么会这样呢？

**猜想3**<a id="yanzheng3"></a>： 具体原因可能是在最后一次GC之后，程序还没有停止，仍然在分配对象，只是没有再触发GC。

在有些数据中，当前存活率，也就是next这一栏，跟pre或者used是一样的，说明当前标记后，该Region中没有需要删除的对象。

```
正在执行...
每个Region的结果:
	  pre     next   used   region_size 
第0个：372752 372752 1048576 1048576
第1个：933936 933936 1048576 1048576
第2个：0 1048576 1048576 1048576
...
第150个：423952 423952 1048576 1048576
...
第300个：0 1048576 1048576 1048576
...
第500个：882200 882200 1048576 1048576
...
第513个：696352 696352 1048576 1048576
第514个：979680 979680 1048576 1048576
...
第680个：0 1048576 1048576 1048576
总使用量: 713738752（680.7M，这里跟old Region的个数高度一致，old中的每个region都塞满了）
总存活量: 521138920（497M）
输出老年代的结果:
Found pool: G1 Old Gen
G1 Old Gen: usage after GC = 668835328（637.85M）
```

通过下面的日志可以看到，Humongous region是没有对象的，对象都在old regions老区中

```
[10.164s][info ][gc,heap       ] GC(207) Eden regions: 110->0(36)
[10.164s][info ][gc,heap       ] GC(207) Survivor regions: 7->15(15)
[10.164s][info ][gc,heap       ] GC(207) Old regions: 570->681
[10.164s][info ][gc,heap       ] GC(207) Archive regions: 2->2
[10.164s][info ][gc,heap       ] GC(207) Humongous regions: 0->0
```



继续改变测试方法来[验证猜想3](#yanzheng3)，这次使用WhiteBox启动依次混合GC，看看GC后的效果

```java
public static void main(String[] args){
    	...
    	getUsageInfo(); // GC前
        startMixedGC();
        getUsageInfo(); // GC后
}
public static void startMixedGC() {
    wb.g1StartConcMarkCycle();
    while (wb.g1InConcurrentMark()) {
        try{
            Thread.sleep(1000);
        } catch (Exception e) {

        }
    }
    wb.youngGC(); // the "Prepare Mixed" gc
    wb.youngGC(); // the "Mixed" gc
}
```

```
# MixedG 前的效果：
第656个：0 1048576 1048576 1048576
第657个：0 1048576 1048576 1048576
总使用量: 689481216
总存活量: 627239136
输出老年代的结果:
Found pool: G1 Old Gen
G1 Old Gen: usage after GC = 557734400


# MixedG 后的输出
第374个：469008 469008 1048576 1048576
第375个：814112 814112 1048576 1048576
总使用量: 393831936
总存活量: 311838856
输出老年代的结果:
Found pool: G1 Old Gen
G1 Old Gen: usage after GC = 394798592

可以看到：Mixed GC后，总使用量跟MemoryPoolMXBean统计出来的堆使用量已经基本一致。
```

```
# MixedG前的Region情况
[10.151s][info ][gc,heap       ] GC(201) Eden regions: 126->0(79)
[10.151s][info ][gc,heap       ] GC(201) Survivor regions: 7->17(17)
[10.151s][info ][gc,heap       ] GC(201) Old regions: 531->658
[10.151s][info ][gc,heap       ] GC(201) Archive regions: 2->2
[10.151s][info ][gc,heap       ] GC(201) Humongous regions: 0->0

# MixedGC后的Region情况
[11.413s][info ][gc,heap       ] GC(205) Eden regions: 0->0(51)
[11.413s][info ][gc,heap       ] GC(205) Survivor regions: 0->0(7)
[11.413s][info ][gc,heap       ] GC(205) Old regions: 460->376 （一个完整的MixedGC会伴随很多Young GC, Region一点点变少）
[11.413s][info ][gc,heap       ] GC(205) Archive regions: 2->2
[11.413s][info ][gc,heap       ] GC(205) Humongous regions: 0->0

可以看到：Region从658变成了376个
```

至此，[猜想3](#yanzheng3)得到验证，[问题5](#question5)也得到解决，不知道这样对不对，期望老师指导。



###### 问题6<a id="question6"></a>

这里还有一个疑问，为什么每个Region的used大小都为 1048576(1M)，代码中分配的随机对象大小为 (100x1024 B 到 500x1024 B)，是怎么做到每个Region都塞满的呢？

在源码中，used的计算是通过两个指针的内存位置进行减法，得到的数值就是被使用的内存。在分配内存时，\_bottom和\_top是如何移动的，就需要进一步探索，麻烦老师指条明路。

```c++
size_t used() const { return byte_size(bottom(), top()); }
HeapWord* bottom() const         { return _bottom; }
HeapWord* top() const { return _top; }
```

