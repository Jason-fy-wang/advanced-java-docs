---
tags:
  - JDK17
---

```
--add-exports=java.base/jdk.internal.ref=ALL-UNNAMED  
--add-exports=java.base/sun.nio.ch=ALL-UNNAMED  
--add-exports=jdk.unsupported/sun.misc=ALL-UNNAMED  
--add-exports=jdk.compiler/com.sun.tools.javac.file=ALL-UNNAMED  
--add-opens=jdk.compiler/com.sun.tools.javac=ALL-UNNAMED  
--add-opens=java.base/java.lang=ALL-UNNAMED  
--add-opens=java.base/java.lang.reflect=ALL-UNNAMED  
--add-opens=java.base/java.io=ALL-UNNAMED  
--add-opens=java.base/java.util=ALL-UNNAMED  
--add-opens=java.base/sun.nio.ch=ALL-UNNAMED
```



```
--add-opens java.base/java.lang=ALL-UNNAMED --add-opens java.base/java.io=ALL-UNNAMED --add-opens java.base/java.math=ALL-UNNAMED --add-opens java.base/java.net=ALL-UNNAMED --add-opens java.base/java.nio=ALL-UNNAMED --add-opens java.base/java.security=ALL-UNNAMED --add-opens java.base/java.text=ALL-UNNAMED --add-opens java.base/java.time=ALL-UNNAMED --add-opens java.base/java.util=ALL-UNNAMED --add-opens java.base/jdk.internal.access=ALL-UNNAMED --add-opens java.base/jdk.internal.misc=ALL-UNNAMED
```

add-exports:  把模块导入到 `未命名模块中(ALL-UNNAMED)`, 则可以直接使用.
add-opens:  允许未命名模块通过 `反射访问` Java.base模块中的包.


