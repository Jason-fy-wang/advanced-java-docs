---
tags:
  - jenkins
  - repository
  - stash
---
在平时的 devOps开发中, 一些自动化的工作, 开端都是从 git repository中获取到code 或 config. 之后进行其他操作.  针对此一般的 clone操作,  使用 ssh key来进行 clone 就可以.

如果通过Jenkins来创建修改 repository中的文件, 步骤如下:
1. 在Jenkins中创建此task需要的 唯一性 branch
2. 在Jenkins中 修改文件, 并把文件提交到 上面创建的branch
3. 使用此 新建的branch 去build package.

那么前两步需要和 remote repository进行交互, 此时再使用 ssh key就不行了.
可以通过使用一个提前创建好的账号, 并且增加对 repository的读写权限. 那么就可以通过此 账号来进行git 操作.
```groovy
withCredentials([gitUsernamePassword](credentialsId: "credentialId", gitToolName: "default")) {
	sh """
		git config user.name "此credential 对应的username"
		git config user.email "email"
		git checkout -b newbranch 

		git add filename
		git commit -m'commit message'
		git push origin newbranch
	
	"""

}

```




