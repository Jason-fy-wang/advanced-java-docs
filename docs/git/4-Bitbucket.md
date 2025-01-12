---
tags:
  - git
  - stash
  - Bitbucket
---
> 在production 的机器上, 需要download source code from Bitbucket. 但是不可以使用铭文的账户密码, 如何操作?

```shell
# 可以把运行 拉取代码的 账户的ssh key 添加到 bitbucket中, 之后就可以使用 一下命令拉取代码
git clone ssh://git@url

### 注意:
此处的 用户名 `git` 可以换成任意其他的账户, 并且不存在的账户都可以下载成功.
```







