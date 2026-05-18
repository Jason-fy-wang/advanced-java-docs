---
tags:
  - maven
  - dependency
---
There some  indirect dependency in project, but we can't tell which dependency introduce such jar. 

> How to find out which one introduce this dependency ?


```shell
# display the available command
mvn dependency:help
```

```shell 
# step 1
## check the effect-pom
mvn -B help:effective-pom -Doutput=effective-pom.xml

# step 2
## didn't find the dependency in pom.xml, then try download the dependency one by one
mvn -X -U -e dependency:get -DgroupID:artifactId:version[:package][:classifier]

# step 3
## reolve the plugin one by one
mvn -B -X -U -e dependency:resolve-plugins -DincludeArtifactIds=maven-surefire-plugin

```



> When build package, there some old jar can't download anymore, how to find out which one introduce it? 

```shell
# check if the jar introduced by pugins
### download the specific plugin, check if all good. If not, and the error caused by the specific jar, then means this plugin introduce such jar
mvn -X -U -e dependency:get -DgroupID:artifactId:version[:package][:classifier]


# check test scope jars
mvn -B -U -X -e org.apache.maven.plugins:maven-dependency-plugin:3.9.0:tree -Dverbose -Dscope=test -Dinclude=org.codehaus.plexus:plexus-utils

# also check all plugins dependency, but exclude the release plugin
mvn -B -U -ee dependency:resolve-plugins -DexcludeArtifactIds=maven-release-plugin

```

