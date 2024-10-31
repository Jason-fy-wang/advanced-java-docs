---
tags:
  - linux
  - java
  - work-dir
---

在java的配置文件中, 经常会使用一些相对目录, 此相对目录是相对于当前工作目录来计算的.  那么在linux中如何获取当前运行的java的程序的工作目录呢?

方式有三种:
```shell
## 1. pwdx
pwdx -p pid


## 2. proc 文件系统
readlink -f /proc/pid/cwd

## 3. lsof
lsof -p pod | grep cwd

```

