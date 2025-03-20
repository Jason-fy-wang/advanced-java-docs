---
tags:
  - sdk
  - linux
  - develop-env
---
Usually, we will prepare a develop environment with lots of SDK installed. And to install these SDK will cost a lot of time. 

There is a tool `SDK` can help install the SDK that we used in daily develop.

SDK install:
```shell
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"

```


```shell
# list all available sdk
sdk list

# list all available groovy
sdk list groovy
sdk list java

# install groovy
sdk install groovy 4.0.26

# install java
sdk install java 17.0.14-zulu

```



install nodejs
[[1-node多环境管理]]




