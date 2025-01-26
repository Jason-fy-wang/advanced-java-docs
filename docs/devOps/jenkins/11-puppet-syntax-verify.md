---
tags:
  - jenkins
  - puppet-verify
---
工作中会用到puppet来对线上的 server 进行一些修改, 那么puppet的包呢也是通过Jenkins来build的.  那么在最近一次通过puppet进行配置修改时, 由于编写的 puppet 文件有 `语法` 问题导致失败了.  
当然了, 在整个准备流程中, 是需要在 vagrant 配置的虚拟机中进行校验的, 但是由于过于自信就省略了此步骤. 
但是为了避免此类问题, 就准备在Jenkins  build packag的过程中整合此 syntax check 的步骤.

添加的内容大体如下:
```shell
stage("syntax check") {
	# download puppet package
	curl http://nexux.host/package.tar.gz -o puppet.tar.gz
	sh """
		# unzip package
		tar -xf puppet.tar.gz
		
		# 把自带的openssl的 so 包放入到 LD_LIBRARY_PATH中
		export LD_LIBRARY_PATH=./puppet/openssl/lib:\$LD_LIBRARY_PATH
		
		# 把ruby的lib 添加到 ruby的search path中, 并进行校验
		puppet/ruby/bin/ruby -I puppet/ruby/lib/ruby/2.5.0  \
		 -I puppet/ruby/lib/ruby/gems/  \
		 -I puppet/ruby/lib/ruby/vendor/2.5.0/  \
		 -I puppet/ruby/lib/ruby/vendor/2.5.0/lib  \
		 -I puppet/ruby/lib/ruby/vendor/2.5.0/lib/x86_64-openssl  \
		 puppet/ruby/bin/puppet apply -debug -confdir=../src/  -modulepath=../src/modules   ../src/root_entry.pp
	"""
}

```

如果不确定要把哪些 folder添加到 ruby的 search path中, 可以找一台正常安装了 ruby的环境,  查看其的search path是什么.
```shell
## 查看 ruby的search path
ruby -e 'puts $:'

```


> 注意点:

1. 设置动态加载库到 `LD_LIBRARY_PATH`
2. 设置ruby 的lib 到search path中
3. 运行puppet时, 设置到对应的 confdir, modulepath 等参数














