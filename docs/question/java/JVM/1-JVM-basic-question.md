---
tags:
  - JVM
  - interview
---
### 1. 解释一下minor/major/full GC
#### 1) minor GC
* 只回收 年轻代 (Young Generation), 即Eden 和 Survivor 区.
* 触发条件: ` Eden 空间不足`
* 速度快, 停顿时间短

#### 2) major GC
* 通常只是回收老年代(Old Generation) 的GC
* 不一定会触及整个堆. (比如不回收 Young Generation)
* 在HotSpot中, Major GC 常与 Old Generation CMS 收集器的'Concurrent Mark-Sweep'  回收对应;  或 G1 中的 'Old  GC' 对应


#### 3) full GC
* 回收整个堆:  Young +Old + (PermGen/MetaSpace)
* 无论哪种收集器, 只要它对整个堆做了 STW收回,  都称作 Full  GC
* 停顿时间最长,  对性能影响最大.


### 2. GC 的触发点是什么?
#### 1) minor GC
* `Eden 空间不足`
	当分配新对象时, 如果Eden区没有足够连续空间,  就出发一次Minor GC, 尝试回收 Young Generation
* 大对象分配
	如果像 Eden 直接分配大于 `PretenureSizeThreshold`的对象, 也可能直接触发Minor GC

* 动态年龄阈值
	Minor GC中, 对象在Survivor区不断复制, 年龄增长;  当达到 `MaxTenuringThreshold`, 对象晋升(Promotion)到Old Gen,  此时也可能触发一次 Minor GC 来腾出 Survivor

#### 2) Major GC/Old Gen GC
* Old Gen 占用超过阈值
	当晋升到Old Gen 的对象过多, Old Gen 使用率达到一定比例 (如CMS中 `-XX:CMSInitiatingOccupancyFraction`), 就会启动 Major GC.
* Survivor 区晋升失败
   Minor GC过程中, 如果Survivor 没有足够空间容纳幸存对象,  晋升操作可能溢出到 Old Gen,  导致对 Old Gen的一次即时回收 (Major GC)

* Promotion Failure(晋升失败)
	对象晋升时 Old Gen 空间不足, 就直接触发一次Old Gen GC.

> Major GC 仅针对老年代, 停顿比Minor GC 长, 但通常不回收 Young
>  GEN
#### 3) full  GC
* 手动调用 System.gc()
	默认映射到 Full GC (通过 -XX:+DisableExplicitGC 禁用)
* CMS 回收失败降级
  CMS在并发回收过程中, 如果无法及时完成 (空间回收不足或并发模型失败), 会降级为一次STW的Full GC
 * Metaspace/PermGen 溢出
	 元数据区(Metaspace/PermGen) 没有足够空间分配新类, 常量等, 也会触发Full GC 先回收在尝试扩展.
* 大对象/Humongous Allocation(G1)
	在G1模式下, 分配大对象 (> half region)时, 会触发一次 Full GC 或者 Humongous region 回收.

* Promotion Failure
	Old Gen 回收不够, 晋升对象失败,  会触发一次 Full GC.


| Collector | Young GC 触发点          | Old GC 触发点                               | Full GC 触发点                                          |
| --------- | --------------------- | ---------------------------------------- | ---------------------------------------------------- |
| Parallel  | Eden 空间不足             | Old Gen 占用率高, Survivor 溢出                | 手动System.GC(),  晋升失败                                 |
| CMS       | 同上                    | 达到CMSInitiatingOccupancyFraction且无足够空闲时间 | CMS Concurrent 模式失败降级, 元数据溢出, system.gc()            |
| G1        | Eden (Young Region)用满 | Old Region 空间阈值, Humongous 分配            | Humongous Allocation, Promotion Failure, Explicit GC |
|           |                       |                                          |                                                      |
|           |                       |                                          |                                                      |








