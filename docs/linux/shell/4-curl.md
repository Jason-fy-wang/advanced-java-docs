---
tags:
  - shell
  - Curl
---
> curl访问时, 映射主机名到其他IP

```shell
# 把www.example.com映射到 百度IP 并进行访问
curl -X GET -k --url https://www.example.com  --resolve www.example.com:110.242.68.66

```

![](./images/curl/resolve1.png)




