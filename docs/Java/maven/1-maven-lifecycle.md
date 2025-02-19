---
tags:
  - maven
  - lifecycle
---
maven 现在默认有三种`LifeCycle`,  分别是 `default, clean, site`.  三个lifecycle之间是独立的, 可以并行执行.
> `lifeCycle` 由 `Phase`组成
> `Phase` 由多种 `Goal`组成. Goal是其中具体执行任务的 Step.

![](./images/Lifecycle)

一个Goal可以绑定到多个 Phase上面, 那么此Goal在每次 phase执行时, 都会被触发.






> reference

[maven liefcycle](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html#Lifecycle_Reference)
[config maven plugin](https://maven.apache.org/guides/mini/guide-configuring-plugins.html)