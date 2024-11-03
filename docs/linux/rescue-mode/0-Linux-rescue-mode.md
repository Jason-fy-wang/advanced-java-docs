---
tags:
  - linux
  - index
  - rescue
---

```shell

## 追加下面内容到 启动参数后, 之后重启进入 rescue mode
rescue
1
systemd.unit=rescue.target
```

![](./images/1-rescue.png)

![](./images/2-rescue.png)



