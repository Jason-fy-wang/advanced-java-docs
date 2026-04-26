---
tags:
  - maven
  - maven_cmd
---
Basic mvn command usage.

#### 1. help
```shell
# mvn help
mvn -h

# help plugin
mvn help:help

### more detail output 
mvn help:help -Ddetail

# get current effect pom
mvn help:effective-pom

# get current maven effective seeting
mvn help:effective-settings

# get profiles
mvn help:all-profiles

# get active profile
mvn help:active-profiles

# specific cmd
mvn compiler:help
mvn install:help
mvn clean:help


```



#### 2. check specific command usage

```shell
# check cmd 'help:describe' help
mvn help:describe -Dcmd=help:describe

mvn help:describe -Dcmd=install

# package
mvn help:describe -Dcmd=package

#jar
mvn jar:help

```


#### 3.  check help for plugin
```shell
mvn help:describe -Dplugin=org.apache.maven.plugins:maven-help-plugin

mvn help:describe -DgroupId=org.apache.maven.plugins -DartifactId=maven-help-plugin

mvn help:describe -Dplugin=org.apache.maven.plugins:maven-jar-plugin



```




