初阶任务：
写一个测试用例，通过不同的GC参数（Serial GC，Parallel Scavenge，G1GC，ZGC，Shenandoah GC），通过打印GC日志完整的展示GC的各个阶段
由于很多JDK发行版（包括KonaJDK）不带Shenandoah GC发布，同学们需要自己从代码到构建一个包含Shenandoah GC的JDK版本（统一基于Tag：TencentKona-17.0.11）
目标：通过该任务让同学们熟悉JDK的编译和构建方法，包括如何打开和关闭某个GC的方法。以及通过一个测试用例，比较各个GC的特点。通过GC日志看到各个GC的不同阶段和GC暂停时间的差异

中阶任务：
专注于G1GC算法，写一个JDK的jtreg测试用例，使用一些现有的whitebox API（有需要的话可以自己扩展whitebox API）来实现一个典型的LRU cache，随机的增加LRU cache内容。运行一段时间之后统计old region的对象存活情况。

目标：通过该任务让同学们熟悉JDK测试特别是GC测试的书写和运行方式，并且通过该测试了解G1GC的运行原理和各个阶段的含义。通过其中老区的存活来了解G1GC中concurrent Mark和mixed GC的运行机理。

高阶任务：
利用任务二中的一些数据，分析目前情况下G1GC Adaptive IHOP的原理以及对于mixed GC的影响。根据相关数据，尝试优化一些mixed GC的参数的数值，使其能够根据Java进程的运行状态同样的进行动态调整。
背景：
目前G1GC触发老区的回收的阈值默认情况下是动态调整的，而老区的回收策略（包括old region存活对象阈值，老区数量的选择，以及mixed GC次数的选择）都是固定的静态值。
目标：
通过该任务让同学们实际并且进行G1GC的深度优化。解决一些社区目前的难题。
