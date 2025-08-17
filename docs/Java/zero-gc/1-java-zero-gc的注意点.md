---
tags:
  - java
  - zero-gc
---
#### java zero-gc代码的常用思路
1, 优先使用 primitive type
2, 避免 autoBox
3, 使用 memory-pool
4, 使用 off-heap memory
5, 避免使用 temporay objects, re-use object
6, 避免使用 String, 尽量使用 stringinterner
7, 使用 Array 代替 ArrayList  LinkedList
8, 避免使用 iterator,  Stream, lambda.  使用 c style foreach
9, 使用 flyweight 模式使用 内存共享
10, 逃逸分析, 临时对象尽量分配到栈上
11, 日志打印方式







#### 高效思路
> 1. 序列化(serial), 不使用自带的序列化方式,  可以使用特定化的序列化, 参考(chronicle 实现) 把对象使用二进制写入到 对堆外内存,  这样数据比较紧凑, 而且序列化, 反序列更高效
> 2. Auo generate code. (generate code from FIX special)











