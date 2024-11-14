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




