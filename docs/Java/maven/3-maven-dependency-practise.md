---
tags:
  - maven
  - best-practise
  - dependency
---

### 1.  使用dependencyManagement
在maven项目中, 尤其是针对多模块项目,  优先使用 `dependencyManagement` 来锁定版本号.  并且模块对应的`scope`也可以定义好, 这样在子模块使用时, 单单导入 grouoId 和 artificatId 就可.
[[2-dependency-best-practise]]

### 2. 使用bom 文件
像 netty 和 grpc 的功能, 其依赖有很多个, 但是在项目并不是使用所有的模块. 那么这对此种情况, 可以在  dependencyManagement 导入其对应的`bom`文件, `scope`为 `import`, 那么就可以锁定其所有模块的版本.


### 3. 把子模块通用要导入的模块提取到 parent中
把通用模块提取到 parent中, 如: junit, mockito 测试模块.  这样子模块就不需要再进行导入. dont repeat.

### 4. plugin 也同样可以通过dependencyManagement来管理







