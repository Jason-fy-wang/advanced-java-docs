---
tags:
  - devops
  - jenkins
---
针对`jenkinsfile`中比较复杂的数据, 如: xml, 无规则的字符串 等数据的处理,  可以通过在 jenkinsfile 中动态生成 python 脚本 (或者其他脚本), 之后调用此脚本来进行数据的处理. 


例如：
访问外部接口，其返回值为多个component的version，且格式为xml。

```shell
# result from external API

<tbody>
	<tr><th>component</th><th>version</th></tr>
	<tr><td>NFVO_service</td><td>1.2.50</td></tr>
	<tr><td></td><td></td></tr>
</tbody>
```

那么针对此种类型数据， 可以在Jenkins中动态生成python来进行解析， 并返回component version。


```groovy

node ("builder") {
	stage("version") {

		// call api and get content
		def content = httpRequest .... 

		def version = getComponentVersion(content, "NFVO_service")
	}
}


def getComponentVersion(String content, String component) {
	sh """
		echo '''
import xml.etree.ElementTree as RT

tree = RT.fromstring(\"\"\"${content}\"\"\")
got = ""
for item in treee.iter("td"):
	if got:
		print(f"{item.text}")
		break
	if item.text == "${component}":
		find = item.text
		continue
		''' > parser.py
	"""
	def result = sh returnStdout: true, script: "python3 parser.py"
	return result.trim()
}

```

