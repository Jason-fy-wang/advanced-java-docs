---
tags:
  - java
  - zero-gc
  - collection
---

### 1. library 
一些low garbage 的collection  library:
[Trove](Trovehttps://trove4j.sourceforge.net/html/overview.html)
[FastUtil](https://github.com/vigna/fastutil)
[Agrona](https://aeron.io/docs/agrona/data-structures/)
[Agrona-code](https://github.com/aeron-io/agrona)

以上是一些高效的库, 提供了以 Primitive type 的collection. 占用内存更小, 效率更高. 同时也避免了 AutoBox
> Long2LongHashMap   InthashSet ..etc



### 2. iterator
避免使用 iterator是因为collection中的iterator是一个临时对象, 每次遍历collection时, 都会创建一个临时 iterator.



### 3 why use array instead of ArrayList

> within certain bounds, CPU can take the hitnt that data we're iterating over happens to all be at a preditable 'stride'  in memory.  and can make our lives a lot faster by 'pre-fetching'  the next elements of data before we even read them. 
if  we have an array of objects in Java, there is no guarantee that theses objects actually live next to each other in memory.  The JVM may allocate in disparate places in memor,  removing any possiability for the CPU and memory subsystem to help us by figuring out the 'stride' and pre-fetching.










