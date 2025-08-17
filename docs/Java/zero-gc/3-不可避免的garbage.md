---
tags:
  - java
  - zero-gc
  - unavoid
  - unaviod-garbage
---
在web项目中(microservice or not),  API 层面有一些不可避免的garbage.

1, backend gateways --> UI components
2, backend service  --> external data service

![](./images/API.excalidraw)


在此种情况, 我们知道API会产生Garbage, 但是不允许影响其他的服务.  `所以可以把GateWay 和 其他服务分离开来.`

![](./images/services.excalidraw)

通过这种把message 发送到 messageBus, 需要的component直接监听此bus,(无需连接, 无需交互, 无需关心版本号). 所有监听的component拿到消息后缓存本地, 加快查找速度.


