---
tags:
  - mount
  - bind
---
mount 一般挂在设备文件 到目录.

> 那么如果我只是想挂载一个存在的目录/文件到 dest directory, 如何办 ?

```shell
## 此种情况可以使用mount  bind
mount --bind src  dest
```




