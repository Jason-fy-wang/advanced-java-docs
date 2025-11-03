---
tags:
  - docker
  - credentials
---
当在制作一个镜像时, 通常会把一些带 credentials的配置文件放入到镜像中,并通过这些配置文件那安装一些依赖, 如: python的pip.conf 和 nexus的账户密码 等.  那么这其实就是把这些信息暴漏出去了, 任何人都可以运行容器并得到其中的信息.

那么针对此种情况, 最好就是把这些配置文件挂载到镜像中, 然后镜像依赖的配置, 最后配置完成后卸载对应的文件.  这样就不会存在敏感信息的泄露.
如:

```shell
bind:  Bind mount context directories (read-only)
cache: Mount a temporary directory to cache directories for compilers and package managers. 
tmpfs: Mount a tmpfs in the build container
secret: Allow the build container to access secure files such as private keys without baking them into the image.
ssh: Allow the build container to access SSH keys via SSH agents, with support for passphrases.
```

```dockerfile
RUN --mount=type=bind:
options:
target,dst,destination:  Mount path
source: Source path in the from, Defaults to the root of the from
from: Build stage, context, or image name for the root of the source. Defaults to the build context.
rw,readwrite: Allow writes on the mount. Written data will be discarded.


# mount local file into image
RUN --mount=type=bind,source=pip.conf,target=/etc/pip.conf \
	pip install request

```


```text
RUN --mount=type=cache
options:
id: Optional ID to identify separate/different cache. defaults to value of target.
target,dst,destination: Mount path
ro,readonly: readonly if set. 
sharing: One of `shared, private, locked`.Defaults is shared. A `shared` cache mount can be used concurrently by multiple writers.`private` creates a new mount if there are multiples writes. `locked` pauses the second writer until the first one release the mount. 
from: Build stage, context, or image name to use as a base of the cache mount. Default to empty directory.
source: Subpath in the from to mount. 
mode: File mode for new cache directory in octal. defaults 0755.
uid: User ID for new cache directory. default 0.
gid: Group ID for new cache directory. default 0.


# example
RUN --mount=type=cache,target=/root/.cache/go-build \
	go build ...


RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
	--mount=type=cache,target=/var/lib/apt,sharing=locked \
	apt update && apt get --no-install-recommends install -y gcc

```


```shell
RUN --mount=type=tmpfs
options:
target,dst,destination:  Mount path
size: specify an upper limit on the size of the filesystem


```


```shell
RUN --mount=type=secret
options:
id:  ID of the secret. Defaults to basename of the target path.
target,dst,destination: Mount the secret to the specified path. default to /run/secrets/+id if unset and if env is also unset.
env: Mount the secret to an environment variable instead of a file or both.
required: if set to true, the instruction errors out when secret is unavailable. default false.
mode: File mode for secret file in octal. default 0400.
uid: User ID for secret file. default 0.
gid: Group ID for secret file. default 0.


RUN --mount=type=secret,id=aws,target=/root/.aws/credentials \
	aws s3 cp s3://... ...


docker buildx build --secret id=aws,sec=$HOME/.aws/credentials .

RUN --mount=type=secret,id=API_KEY,env=API_KEY  \
	command --token-from-env $API_KEY
	
set API_KEY in build:
docker buildx build --secret id=API_KEY
```

```shell
# this mount type allows the build container to access SSH keys via SSH agents. with support for passphrases.
RUN --mount=type=ssh

options:
id:
target,dst,destination:
required:
mode:
uid:
gid:


RUN --mount=type=ssh \
	ssh -q -T git@gitlab.com 2>&1 | tee /hello
# "Welcome to GitLab, @GITLAB_USERNAME_ASSOCIATED_WITH_SSHKEY" should be printed here
# with the type of build progress is defined as `plain`.


```


[docker run command](https://docs.docker.com/reference/dockerfile/#run---mount)