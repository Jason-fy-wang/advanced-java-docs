---
tags:
  - jenkins
  - version
  - build-label
---
Jenkins的task 通常会是多个任务的联合, 如:
![](./build-job)

如何获取每一步build好的package 的version 呢?

目前我使用到的方法有两种:
```groovy
## 方式一
#### 在每一个build job的Jenkinsfile中添加如下语句,把version存储到 环境变量中.  此种属于有侵入式的方法
env.version=currentbuild_version

#### 获取版本号
def result = build job: "package-builder", parameters: []
def version = result.getBuildVeriables.get("version")

## 方式二
#### 每一个builder job 把version 按照约定好的格式写到 displayname中, 之后获取版本时, 从diaplayname中获取. 
#### builder job 添加
currentbuild.displayName = "${branch}#${version}"

#### 获取:
display = result.getDiaplayName()
version = diplay.substring(version.indexOf("${branch})+length("${branch}")+1)
									
```



