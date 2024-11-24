---
tags:
  - Jenkins
  - config-xml
---



```shell
# get original config.xml
curl -v -u "username:token" -X GET --url https://jenkins_url/job/jobname/config.xml > config.xml

## make update in the config.xml. fg. add cron expression 

# update job with the latest config.xml
curl -v -H "Content-type: text/xml" -u "username:token" -X POST --data-binary @config.xml  --url https://jenkins_url/job/jobname/config.xml

```
简单浏览过此config.xml后, 你可以看到所有的 属性以及 配置都在这个文件中.  简单说, job的所有配置, 都可以通过此接口来进行更新.

扩展:
Jenkins job 支持 cron 执行,  可以写一个后端程序, 通过输入  Jenkins job 的url 以及 cron table 表达式, 也实现对任意一个job 的 定时运行.




other userful API:
```shell

### simply: add API after url then you can check the related API

# get job related API
http://jenkins_url/job/jobname/api

# job api json info
http://jenkins_url/job/jobname/api/json


# get specific job info
http://jenkins_url/job/jobname/{build_name}/api/json


```




