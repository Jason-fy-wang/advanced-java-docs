---
tags:
  - git
  - resolve-conflict
  - conflict
---
In devops script, we need operate some repository automatic, but sometimes the operation will encounter merge `conflict`, how to resolve the conflict in script automatic ?

In this page, we will accept the remote change to resolve the conflict and get the latest repository.  After that, we can make our change .

```shell
## sync from remote and accept remote to resolve the conflict if any.

git pull -X theirs origin master

if [[ $? -ne 0 ]]; then
	conflict_fiiles=$(git diff --name-only --diff-filter=U)
	# revert conflict as remote
	git diff --name-only --diff-filter=U | xargs -I{} git checkout --theirs {}
	git add .
	git commit -m"resolve conflict"
fi
echo "repository sync from remote"
```





