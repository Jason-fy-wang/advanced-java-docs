---
tags:
  - jenkins
  - ant-pattern
  - file-glob
---
一个Jenkinsfile as below, 在使用正则获取文件时, 导致的问题.
```jenkinsfile

def CONFIG="config"

timestamp {

	node('gcp') {
	
		stage('upload file'){
			dir(CONFIG) {
				getFiles()
			}
		}
	}
}

def getFiles() {
	def files = findfile glob: "${WORKSPACE}/${CONFIG}/services/ng*-live*1/*.properties"
	return files
}


```

当Jenkins 运行后报错如下:
```shell
java.io.IOException: Expecting Ant GLOB pattern.  but saw '/opt/jenkins/ws/TT/TEMPLATE-build-job/config/services/ng*-live*1/*.properties'.  See https://ant.apache.org/manual/Types/fileset.html for syntax

```

Reason:

```shell
## 异常原因如下：
在Jenkinsfile中, 通过dir切换到了对应目录,但是在后续的操作中又使用了 ${WORKSPACE}的full path,导致了此问题.

把上面的路径从
"${WORKSPACE}/${CONFIG}/services/ng*-live*1/*.properties"

切换为
"services/ng*-live*1/*.properties"
就可以了.

```


> 注意:
> 在dir切换目录后, 使用相对路径





