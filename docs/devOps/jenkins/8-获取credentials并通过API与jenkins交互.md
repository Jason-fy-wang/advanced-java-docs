---
tags:
  - API
  - jenkins
  - Curl
  - credentials
---
线上的一些任务, 有时需要在凌晨去操作,  虽然已经有了jenkins job, 但仍需要人来 trigger job运行.

那么能否设置cron job, 通过API来出发Jenkins的job 运行呢?
 答案是可以的.

通过API交互, 需要一些 credentials 信息, 有两种方式, 
1. token
2. crumb + username:passwd 

第二种方式有些易错点, 后面会说到.

```shell
## 方式一:  
curl -X POST -u "username:token"  jenkins_url/build/job/jobname/buildwithParameters -F "version=version1.0" -F "tagToRun=deploy" 

```



```shell
## 方式二:
### 获取crumb header
header=(curl -X GET -c cookie.txt -u "username:pwd"  --url "http://jenkins_url/crumbIssuer/api/json?concat(//crumbRequestField, ':', //crumb)" )


curl -X POST -H "${header}" -b cookie.txt  -u "username:password" jenkins_url/build/job/jobname/buildwithParameters -F "version=version1.0" -F "tagToRun=deploy" 

```

> 易错点:
> 1. 在获取 crumb时, 需要使用 -c cookie.txt 来创建并保存cookie
> 2. 在后面访问时, 使用 -b cookie.txt 来使用保存的 cookie.txt




