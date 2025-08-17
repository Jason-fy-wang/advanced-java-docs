---
tags:
  - java
  - zero-gc
  - log-style
---
总是把日志的打印放入到一个 if 判断中, 这样就避免了创建log buffer, 而且其有可能因为 log level而不会使用到.
```java
if (logger.isDebugEnabled()){
logger.debug("message {}", msg)
}
```

