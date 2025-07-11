---
tags:
  - awscli
  - install
  - cloud
---
aws cli 在ubuntu上的安装：

```shell
# download
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"  -vv

# unzip
unzip awscliv2.zip

# install
sudo ./install

# upgrade
sudo ./install --bin-dir /usr/local/bin --install-dir /usr/local/aws-cli --update
```



