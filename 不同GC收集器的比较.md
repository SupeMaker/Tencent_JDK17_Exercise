# 1 测试用例

## 1.1 测试代码

```java
import java.util.ArrayList;
import java.util.List;

public class GCTest {
    public static void main(String[] args) {
        List<byte[]> list = new ArrayList<>();
        while(true) {
            // 分配1MB的数组
            list.add(new byte[1024 * 1024]);
        }
    }
}
```

# 2 经典垃圾收集器比较

## 2.1 SerialGC

Serial收集器运行示意图：

![深入理解Java虚拟机](img\Serial收集器运行示意图.png)

虚拟机参数

```
-XX:+UseSerialGC -Xlog:gc*=info,gc+heap=debug,safepoint:log/gc-oomHeap.log:uptime,level,tags -Xms20M -Xmx20M -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=heap/heapdump.hprof
```

完整的log日志将在 ./Serial.log文件给出

接下来这一段在每个GC收集器的日志都差不多，这里统一解释

```
[0.042s][info][gc] Using Serial # 使用的垃圾收集器，这里使用Serial收集器
[0.043s][info][gc,init] Version: 17.0.11-internal+0-adhoc.damowang.TencentKona-17 (release) # JVM版本信息，这里显示的是17.0.11（内部版本），基于腾讯Kona JDK17,自行构建的release版本
[0.044s][info][gc,init] CPUs: 12 total, 12 available # 我的电脑是6核12线程，这里检测到12个CPU核心
[0.044s][info][gc,init] Memory: 3829M # JVM内存总容量，总物理内存大小。
[0.044s][info][gc,init] Large Page Support: Disabled # 未启用大页支持
...
[0.044s][info][gc,init] Heap Min Capacity: 20M # 最小堆大小20MB
[0.044s][info][gc,init] Heap Initial Capacity: 20M # 初始堆大小20MB
[0.044s][info][gc,init] Heap Max Capacity: 20M # 最大堆大小20MB
...
```

一次Young GC和Full GC日志

```
[0.156s][info ][safepoint   ] Safepoint "GenCollectForAllocation", Time since last: 344800 ns, Reaching safepoint: 2400 ns, Cleanup: 2900 ns, At safepoint: 2289900 ns, Total: 2295200 ns
[0.157s][info ][gc,start    ] GC(3) Pause Young (Allocation Failure)
[0.157s][debug][gc,heap     ] GC(3) Heap before GC invocations=3 (full 0):
[0.157s][debug][gc,heap     ] GC(3)  def new generation   total 6144K, used 5222K [0x00000000fec00000, 0x00000000ff2a0000, 0x00000000ff2a0000)
[0.157s][debug][gc,heap     ] GC(3)   eden space 5504K,  94% used [0x00000000fec00000, 0x00000000ff119a38, 0x00000000ff160000)
[0.157s][debug][gc,heap     ] GC(3)   from space 640K,   0% used [0x00000000ff200000, 0x00000000ff200050, 0x00000000ff2a0000)
[0.157s][debug][gc,heap     ] GC(3)   to   space 640K,   0% used [0x00000000ff160000, 0x00000000ff160000, 0x00000000ff200000)
[0.157s][debug][gc,heap     ] GC(3)  tenured generation   total 13696K, used 12028K [0x00000000ff2a0000, 0x0000000100000000, 0x0000000100000000)
[0.157s][debug][gc,heap     ] GC(3)    the space 13696K,  87% used [0x00000000ff2a0000, 0x00000000ffe5f3b8, 0x00000000ffe5f400, 0x0000000100000000)
[0.157s][debug][gc,heap     ] GC(3)  Metaspace       used 6101K, committed 6272K, reserved 1114112K
[0.157s][debug][gc,heap     ] GC(3)   class space    used 529K, committed 640K, reserved 1048576K
[0.157s][debug][gc,heap     ] GC(3) Heap after GC invocations=4 (full 0):
[0.157s][debug][gc,heap     ] GC(3)  def new generation   total 6144K, used 5222K [0x00000000fec00000, 0x00000000ff2a0000, 0x00000000ff2a0000)
[0.157s][debug][gc,heap     ] GC(3)   eden space 5504K,  94% used [0x00000000fec00000, 0x00000000ff119a38, 0x00000000ff160000)
[0.157s][debug][gc,heap     ] GC(3)   from space 640K,   0% used [0x00000000ff200000, 0x00000000ff200050, 0x00000000ff2a0000)
[0.157s][debug][gc,heap     ] GC(3)   to   space 640K,   0% used [0x00000000ff160000, 0x00000000ff160000, 0x00000000ff200000)
[0.157s][debug][gc,heap     ] GC(3)  tenured generation   total 13696K, used 12028K [0x00000000ff2a0000, 0x0000000100000000, 0x0000000100000000)
[0.157s][debug][gc,heap     ] GC(3)    the space 13696K,  87% used [0x00000000ff2a0000, 0x00000000ffe5f3b8, 0x00000000ffe5f400, 0x0000000100000000)
[0.157s][debug][gc,heap     ] GC(3)  Metaspace       used 6101K, committed 6272K, reserved 1114112K
[0.157s][debug][gc,heap     ] GC(3)   class space    used 529K, committed 640K, reserved 1048576K
[0.157s][info ][gc          ] GC(3) Pause Young (Allocation Failure) 16M->16M(19M) 0.088ms
[0.157s][info ][gc,cpu      ] GC(3) User=0.00s Sys=0.00s Real=0.00s
[0.157s][info ][gc,start    ] GC(4) Pause Full (Allocation Failure)
[0.157s][debug][gc,heap     ] GC(4) Heap before GC invocations=4 (full 0):
[0.157s][debug][gc,heap     ] GC(4)  def new generation   total 6144K, used 5222K [0x00000000fec00000, 0x00000000ff2a0000, 0x00000000ff2a0000)
[0.157s][debug][gc,heap     ] GC(4)   eden space 5504K,  94% used [0x00000000fec00000, 0x00000000ff119a38, 0x00000000ff160000)
[0.157s][debug][gc,heap     ] GC(4)   from space 640K,   0% used [0x00000000ff200000, 0x00000000ff200050, 0x00000000ff2a0000)
[0.157s][debug][gc,heap     ] GC(4)   to   space 640K,   0% used [0x00000000ff160000, 0x00000000ff160000, 0x00000000ff200000)
[0.157s][debug][gc,heap     ] GC(4)  tenured generation   total 13696K, used 12028K [0x00000000ff2a0000, 0x0000000100000000, 0x0000000100000000)
[0.157s][debug][gc,heap     ] GC(4)    the space 13696K,  87% used [0x00000000ff2a0000, 0x00000000ffe5f3b8, 0x00000000ffe5f400, 0x0000000100000000)
[0.157s][debug][gc,heap     ] GC(4)  Metaspace       used 6101K, committed 6272K, reserved 1114112K
[0.157s][debug][gc,heap     ] GC(4)   class space    used 529K, committed 640K, reserved 1048576K
[0.157s][info ][gc,phases,start] GC(4) Phase 1: Mark live objects
[0.158s][info ][gc,phases      ] GC(4) Phase 1: Mark live objects 0.936ms
[0.158s][info ][gc,phases,start] GC(4) Phase 2: Compute new object addresses
[0.158s][info ][gc,phases      ] GC(4) Phase 2: Compute new object addresses 0.163ms
[0.158s][info ][gc,phases,start] GC(4) Phase 3: Adjust pointers
[0.159s][info ][gc,phases      ] GC(4) Phase 3: Adjust pointers 0.536ms
[0.159s][info ][gc,phases,start] GC(4) Phase 4: Move objects
[0.160s][info ][gc,phases      ] GC(4) Phase 4: Move objects 0.932ms
[0.160s][info ][gc,heap        ] GC(4) DefNew: 5222K(6144K)->4096K(6144K) Eden: 5222K(5504K)->4096K(5504K) From: 0K(640K)->0K(640K)
[0.160s][info ][gc,heap        ] GC(4) Tenured: 12028K(13696K)->13052K(13696K)
[0.160s][info ][gc,metaspace   ] GC(4) Metaspace: 6101K(6272K)->6101K(6272K) NonClass: 5572K(5632K)->5572K(5632K) Class: 529K(640K)->529K(640K)
[0.160s][debug][gc,heap        ] GC(4) Heap after GC invocations=4 (full 1):
[0.160s][debug][gc,heap        ] GC(4)  def new generation   total 6144K, used 4096K [0x00000000fec00000, 0x00000000ff2a0000, 0x00000000ff2a0000)
[0.160s][debug][gc,heap        ] GC(4)   eden space 5504K,  74% used [0x00000000fec00000, 0x00000000ff0000a8, 0x00000000ff160000)
[0.160s][debug][gc,heap        ] GC(4)   from space 640K,   0% used [0x00000000ff200000, 0x00000000ff200000, 0x00000000ff2a0000)
[0.160s][debug][gc,heap        ] GC(4)   to   space 640K,   0% used [0x00000000ff160000, 0x00000000ff160000, 0x00000000ff200000)
[0.160s][debug][gc,heap        ] GC(4)  tenured generation   total 13696K, used 13052K [0x00000000ff2a0000, 0x0000000100000000, 0x0000000100000000)
[0.160s][debug][gc,heap        ] GC(4)    the space 13696K,  95% used [0x00000000ff2a0000, 0x00000000fff5f3c8, 0x00000000fff5f400, 0x0000000100000000)
[0.160s][debug][gc,heap        ] GC(4)  Metaspace       used 6101K, committed 6272K, reserved 1114112K
[0.160s][debug][gc,heap        ] GC(4)   class space    used 529K, committed 640K, reserved 1048576K
[0.160s][info ][gc             ] GC(4) Pause Full (Allocation Failure) 16M->16M(19M) 2.738ms
[0.160s][info ][gc,cpu         ] GC(4) User=0.00s Sys=0.00s Real=0.01s
[0.160s][info ][safepoint      ] Safepoint "GenCollectForAllocation", Time since last: 387500 ns, Reaching safepoint: 3200 ns, Cleanup: 2800 ns, At safepoint: 2853200 ns, Total: 2859200 ns

```

