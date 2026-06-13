---
tags:
  - podman
  - in-action
---
# Podman 实战 — 深度技术读书笔记

> **书名**: Podman 实战 (Podman in Action)  

---

## 全书架构

全书分 **4 个部分**，共 **11 章 + 6 个附录**：

| 部分 | 章节 | 主题 |
|------|------|------|
| 第1部分：基础 | 1-4 | 容器概念、命令行、卷、Pod |
| 第2部分：设计 | 5-6 | 配置文件、非特权容器 |
| 第3部分：高级主题 | 7-9 | systemd集成、K8s协同、Podman服务 |
| 第4部分：容器安全 🔒 | 10-11 | 安全隔离、纵深防御 |

---

## 第1章：Podman — 下一代容器引擎

### 1.1 术语说明

| 术语 | 含义 |
|------|------|
| **OCI** (Open Container Initiative) | 容器标准化组织，定义镜像格式和运行时规范 |
| **容器镜像** | 包含应用程序及其依赖的只读文件系统 |
| **容器** | 镜像的运行实例，拥有隔离的进程空间 |
| **Pod** | 一组共享网络/存储命名空间的容器 |

### 1.2 容器简介

容器技术的核心价值：**隔离 + 可移植性**。容器不是虚拟机——它共享宿主机内核，通过 Linux 内核特性（命名空间、cgroup、SELinux、seccomp）实现隔离。

### 1.3 有了 Docker 为什么还要 Podman？

关键差异：

| 维度 | Docker | Podman |
|------|--------|--------|
| 架构 | 守护进程 (daemon) | 无守护进程 (daemonless) |
| 权限 | 默认 root 运行 daemon | 支持非特权 (rootless) |
| 安全模型 | daemon 拥有所有容器 | fork/exec 模型，每个容器独立 |
| Pod 支持 | 无 | 原生支持 |
| systemd 集成 | 有限 | 深度集成 |
| Kubernetes YAML | 不支持生成 | `podman generate kube` |

**核心论点**：Docker 的守护进程模型是安全风险的集中点——一旦 daemon 被攻破，所有容器都暴露。Podman 的 fork/exec 模型让每个容器进程独立，没有单点攻击面。

### 1.4 什么时候不使用 Podman

- 需要 Docker Swarm 的场景
- 依赖 Docker 特有 API 的工具链
- Windows 原生容器（Podman 通过 WSL2 运行 Linux 容器）

---

## 第2章：Podman 命令行

这是全书最长的一章（36000字），覆盖日常使用的所有核心命令。

### 2.1 容器生命周期管理

```bash
# 运行容器
podman run -d --name myapp -p 8080:80 nginx:latest

# 端口映射说明：-p HOST_PORT:CONTAINER_PORT
podman run -p 8080:80 nginx          # 主机8080→容器80
podman run --publish-all ...          # 自动映射所有暴露端口
podman port myapp                     # 查看容器端口映射

# 查看容器日志
podman logs myapp

# 在运行中的容器内执行命令
podman exec -it myapp /bin/bash

# 停止/启动/重启
podman stop myapp
podman start myapp
podman restart myapp

# 删除容器
podman rm myapp
podman rm --ignore myapp              # 容器不存在时不报错
```

### 2.2 镜像管理

```bash
# 拉取镜像
podman pull docker.io/library/nginx:latest

# 列出镜像
podman images

# 检查镜像详细信息
podman image inspect myimage
podman image inspect --format '{{ .Config.Cmd }}' myimage  # 查看默认命令

# 查看镜像变更
podman image diff myimage              # 查看镜像层变更

# 挂载镜像（直接查看文件系统）
mnt=$(podman image mount quay.io/rhatdan/myimage)
ls $mnt
podman image unmount quay.io/rhatdan/myimage

# 推送镜像
podman push myimage quay.io/rhatdan/myimage

# 清理空悬镜像
podman image prune

# 认证文件
podman login --auth-file ~/myauth.json quay.io
```

> **💡 空悬镜像 (dangling image)**：没有标签的中间镜像层，通常由构建过程产生，`podman image prune` 可清理。

### 2.3 构建镜像

Podman 支持两种构建方式：

```bash
# 使用 Containerfile (等同于 Dockerfile)
podman build -t myapp:v1 .

# 使用 Buildah 原生命令（更灵活）
buildah bud -t myapp:v1 .
```

---

## 第3章：卷

容器存储是 ephemeral 的——容器删除后数据丢失。卷提供持久化存储。

### 3.1 绑定挂载 vs 命名卷

```bash
# 绑定挂载：HOST-DIR:CONTAINER-DIR
podman run -v /host/data:/container/data myapp

# 命名卷
podman volume create mydata
podman run -v mydata:/container/data myapp

# 卷的导出/导入（跨机器迁移）
podman volume export mydata > mydata.tar
podman volume import mydata < mydata.tar
```

### 3.2 SELinux 卷标签

在 SELinux 系统上，卷挂载需要处理标签问题：

```bash
# :z — 重新标记为共享标签（多个容器可读写）
podman run -v /host/data:/data:z myapp

# :Z — 重新标记为私有标签（仅当前容器可读写）
podman run -v /host/data:/data:Z myapp

# :U — 将源目录所有权改为容器主进程的 UID
# 适用于非特权容器中数据库等需要特定文件所有权的场景
podman run -v ./mariadb:/var/lib/mariadb:U mariadb

# 禁用 SELinux 标签（不推荐，仅调试用）
podman run --security-opt label=disable myapp
```

> **⚠️ 生产环境忠告**：永远不要使用 `--security-opt label=disable`，这等于关闭了容器内 SELinux 保护。应该使用 `:z` 或 `:Z` 选项正确处理标签。

### 3.3 idmap 卷挂载

```bash
# idmap 选项：自动映射 UID/GID，使容器内看到的文件所有权正确
podman run -v /host/data:/data:idmap myapp
```

`idmap` 让卷中的文件在容器内显示为由用户命名空间的 root 拥有，基于标准权限可读写。这对于非特权容器中需要访问宿主机文件的场景非常关键。

---

## 第4章：Pod

Pod 是 Podman 的标志性功能——一组共享网络和存储命名空间的容器，类似 Kubernetes Pod。

```bash
# 创建 Pod
podman pod create --name mypod -p 8080:80

# 向 Pod 添加容器
podman run --pod mypod --name web nginx
podman run --pod mypod --name app myapp

# 启动/停止 Pod（自动管理所有容器）
podman pod start mypod
podman pod stop mypod

# 列出 Pod
podman pod list

# 删除 Pod（会删除其中所有容器）
podman pod rm mypod
```

**Pod 内容器共享**：
- 🌐 网络命名空间（共享 localhost、端口空间）
- 📦 存储命名空间（共享卷）
- 🔗 可以通过 localhost 互相通信

---

## 第5章：自定义和配置文件

### 5.1 存储配置 (`storage.conf`)

```bash
# 系统级配置
/usr/share/containers/storage.conf

# 用户级配置
$HOME/.config/containers/storage.conf
```

关键配置项：

| 配置项 | 说明 |
|--------|------|
| `graphRoot` | 镜像/容器的存储根路径 |
| `rootless_storage_path` | 非特权模式的存储路径 |
| `mount_program` | FUSE 挂载程序（如 fuse-overlayfs） |
| `ignore_chown_errors` | 忽略 chown 错误（非特权模式有用） |

查看完整配置手册：`containers-storage.conf(5)`

### 5.2 注册服务器配置 (`registries.conf`)

控制 Podman 从哪些注册服务器拉取/推送镜像：

```toml
# /etc/containers/registries.conf
[registries.search]
registries = ['docker.io', 'quay.io']

[registries.insecure]
registries = ['localhost:5000']

[registries.block]
registries = ['untrusted.io']
```

### 5.3 策略配置 (`policy.json`)

定义镜像拉取的安全策略——哪些注册服务器的镜像可信：

```bash
# /etc/containers/policy.json
```

### 5.4 引擎配置 (`containers.conf`)

```bash
# 通过 CONTAINERS_CONF 环境变量覆盖配置路径
CONTAINERS_CONF=/path/to/custom.conf podman run ...
```

---

## 第6章：非特权容器 🔒

这是 Podman 最重要的设计决策——**无需 root 权限即可运行容器**。

### 6.1 非特权 Podman 的工作原理

传统 Docker 需要 root 权限运行守护进程，这带来了巨大的安全风险。Podman 通过 **用户命名空间 (user namespace)** 实现 rootless 模式：

```
宿主机 UID 1000 ←→ 容器内 UID 0 (root)
宿主机 UID 100000-165535 ←→ 容器内 UID 1-65535
```

**关键机制**：

1. **newuidmap / newgidmap**：Podman 启动 `/usr/bin/newuidmap` 和 `/usr/bin/newgidmap` 来配置 UID/GID 映射
2. **从属用户/组**：需要配置 `/etc/subuid` 和 `/etc/subgid`

```bash
# /etc/subuid 格式
username:起始UID:数量
# 例如：
dan:100000:65536

# /etc/subgid 格式
username:起始GID:数量
# 例如：
dan:100000:65536
```

三个字段：**登录名或UID** / **数字从属用户ID或组ID** / **数字从属用户ID或组ID计数**

### 6.2 非特权容器的技术内幕

非特权容器涉及的能力（capabilities）：

| 能力 | 用途 |
|------|------|
| `CAP_SETUID` | 修改进程 UID |
| `CAP_NET_ADMIN` | 网络管理（非特权模式受限） |
| `CAP_SYS_ADMIN` | 系统管理（非特权模式受限最多） |

**网络限制**：非特权容器默认使用 **slirp4netns** 模拟网络栈，性能不如特权模式但安全。新方案 **pasta** 性能更好。

```bash
# 迁移存储格式（升级 Podman 后需要）
podman system migrate
```

---

## 第7章：与 systemd 集成

Podman 与 systemd 的深度集成是其生产环境部署的核心优势。

### 7.1 在容器中运行 systemd

Podman 支持在容器内运行 systemd 作为 PID 1，这意味着容器可以像完整 Linux 系统一样管理多个服务。

### 7.2 journald 日志

```toml
# containers.conf
[containers]
log_driver = "journald"
```

`events_logger` 配置控制事件日志的输出方式。

### 7.3 系统启动时自动启动容器

```bash
# 生成 systemd 单元文件（推荐 --new 模式）
podman generate systemd --new --name myapp > /etc/systemd/system/myapp.service

# 启用并启动
systemctl enable --now myapp.service
```

`--new` 模式意味着每次启动服务时创建新容器，而非复用旧容器——更干净、更可靠。

### 7.4 notify 单元文件

```bash
# Type=notify 模式：容器内应用通过 NOTIFY_SOCKET 通知 systemd 就绪
podman run -d --name myapp \
  -e NOTIFY_SOCKET=/run/notify.sock \
  myapp
```

### 7.5 自动更新与回滚

```bash
# 标记容器为自动更新
podman run -d \
  --label "io.containers.autoupdate=registry" \
  myapp

# systemd 单元 + autoupdate label = 自动拉取新镜像并重启
# 如果新版本失败，自动回滚到上一版本
```

### 7.6 套接字激活

Podman 支持 systemd 套接字激活——只在有连接时才启动容器，节省资源。

---

## 第8章：与 Kubernetes 协同工作

### 8.1 生成 Kubernetes YAML

```bash
# 从运行的 Pod 生成 K8s YAML
podman generate kube myapp > myapp.yaml
```

### 8.2 从 K8s YAML 创建 Pod

```bash
# 从 YAML 创建 Pod 和容器
podman play kube myapp.yaml

# 指定容器名称
podman play kube myapp.yaml --ctr-names myapp-web,myapp-db

# 拆除 K8s YAML 创建的资源
podman play kube myapp.yaml --down
```

### 8.3 在容器内运行 Podman

DooD (Docker outside of Docker) 的 Podman 等价——在容器内运行 Podman 命令。

---

## 第9章：Podman 服务

### 9.1 Podman 服务模式

```bash
# 启动 Podman REST API 服务
systemctl enable --now podman.socket

# 或手动启动
podman system service --time=0 unix:///run/podman/podman.sock
```

### 9.2 兼容 Docker API

Podman 服务兼容 Docker API，大多数 Docker 客户端工具可以直接使用。

### 9.3 Python 库

```python
from podman import PodmanClient

with PodmanClient() as client:
    container = client.containers.run("nginx", detach=True)
```

### 9.4 docker-compose 兼容

Podman 服务可以替代 Docker 后端，让 docker-compose 直接工作。

### 9.5 远程管理

```bash
# 管理 SSH 连接
podman system connection add remote-server ssh://user@host:/run/podman/podman.sock

# 远程执行 Podman 命令
podman --remote ps
```

---

# 🔒 第4部分：容器安全 — 详细解读

> 这是全书的精华部分，作者 Daniel Walsh 本人就是 Red Hat SELinux 和容器安全的核心开发者。这两章从 Linux 内核层面系统性地解释了容器安全的每一个隔离层。

---

## 第10章：安全容器隔离

容器安全的本质是 **Linux 内核提供的多层隔离机制**。容器并不是一种新的虚拟化技术，而是对已有内核特性的组合运用。

### 10.1 只读的 Linux 内核伪文件系统

Linux 内核通过伪文件系统向用户空间暴露信息，容器需要限制对这些文件系统的访问：

| 文件系统 | 内容 | 容器中的处理 |
|----------|------|-------------|
| `/proc` | 进程和内核信息 | 部分遮蔽 (mask) |
| `/sys` | 设备和驱动信息 | 严格限制 |
| `/dev` | 设备节点 | 最小化设备集 |

### 10.2 Linux 能力 (Capabilities) ⭐

传统 Linux 只有 root 和非 root 两种权限。Linux Capabilities 将 root 权限细分为几十种独立能力，容器只需其中少量即可运行。

**默认能力列表**（Podman 运行容器时默认授予）：

```
CAP_AUDIT_WRITE    — 写审计日志
CAP_CHOWN          — 修改文件所有者
CAP_DAC_OVERRIDE   — 绕过文件权限检查
CAP_FOWNER         — 绕过文件所有者检查
CAP_FSETID         — 设置 setuid/setgid 位
CAP_KILL           — 向任意进程发信号
CAP_MKNOD          — 创建设备节点
CAP_NET_BIND_SERVICE — 绑定 <1024 端口
CAP_NET_RAW        — 使用原始套接字（ping等）
CAP_SETFCAP        — 设置文件能力
CAP_SETPCAP        — 修改进程能力
CAP_SETUID         — 修改进程 UID
CAP_SETGID         — 修改进程 GID
CAP_SYS_CHROOT     — chroot
CAP_SYS_PTRACE     — 追踪进程
```

**最小化能力原则**：

```bash
# 丢弃所有能力，只添加必要的
podman run --cap-drop=all --cap-add CAP_SETUID,CAP_SETGID myapp

# 丢弃特定能力
podman run --cap-drop=CAP_NET_BIND_SERVICE myapp

# 添加特定能力
podman run --cap-add CAP_NET_RAW myapp   # 如需 ping
```

> **💡 实战建议**：生产环境应该 `--cap-drop=all` 然后只添加必需的能力。这是最小权限原则的直接应用。

**危险能力**：

- `CAP_SYS_ADMIN` — 几乎等于 root，**永远不要给容器**。许多容器逃逸漏洞都依赖此能力。
- `CAP_NET_ADMIN` — 可修改网络配置，非特权容器默认不具备
- `CAP_SYS_PTRACE` — 可追踪和注入其他进程，潜在逃逸风险

**no-new-privileges**：

```bash
# 阻止容器内进程通过 setuid/setgid 提权
podman run --security-opt no-new-privileges myapp
```

这个选项确保容器内的子进程无法获得比父进程更多的权限。非常适合运行不可信代码的场景。

### 10.3 UID 隔离：用户命名空间 ⭐⭐

用户命名空间是容器安全最重要的隔离机制——它将容器内的 root 映射到宿主机的普通用户。

```
容器内 UID    宿主机 UID
─────────    ──────────
0 (root)  ←→ 1000 (普通用户)
1         ←→ 100001
2         ←→ 100002
...           ...
65534     ←→ 165534
```

**关键特性**：
- 容器内进程以为自己是 root，但在宿主机上只是普通用户
- 即使容器被攻破，攻击者在宿主机上没有 root 权限
- 这就是 Podman rootless 模式的核心

```bash
# 自动分配用户命名空间
podman run --userns=auto:size=500 myapp

# 自动模式从 /etc/subuid 和 /etc/subgid 分配映射范围
# size=500 表示分配 500 个 UID 映射
```

**卷的 UID 映射问题**：

当非特权容器挂载宿主机目录时，文件所有权是核心痛点：

```bash
# 方案1：使用 :U 选项自动修改所有权
podman run -v ./data:/data:U myapp
# U选项将源目录的所有权改为容器主进程的UID

# 方案2：使用 :idmap 选项自动映射
podman run -v ./data:/data:idmap myapp
# 文件在容器内显示为由root拥有，基于标准权限可读写
```

### 10.4 进程隔离：PID 命名空间

PID 命名空间让容器拥有独立的进程 ID 空间：

- 容器内 PID 1 是容器的主进程
- 容器内看不到宿主机或其他容器的进程
- 防止容器内进程向宿主机进程发信号

```bash
# 共享宿主机 PID 命名空间（危险！调试用）
podman run --pid=host myapp
```

> **⚠️ `--pid=host` 的风险**：容器可以看到并可能影响宿主机所有进程。仅在调试时使用，生产环境禁止。

### 10.5 网络隔离：网络命名空间

网络命名空间给容器独立的网络栈：

- 独立的网络接口、IP地址、路由表、iptables
- 容器间默认网络隔离
- 通过端口映射 (-p) 或 Pod 共享网络

非特权容器的网络选项：

| 方案 | 性能 | 安全性 | 说明 |
|------|------|--------|------|
| slirp4netns | 慢 | 高 | 用户空间网络栈 |
| pasta | 中 | 高 | 改进版 slirp |
| host 网络模式 | 快 | 低 | `--net=host` 共享宿主机网络 |

### 10.6 IPC 隔离：IPC 命名空间

IPC 命名空间隔离 System V IPC 和 POSIX 消息队列：

```bash
# 共享宿主机 IPC（某些应用需要，如共享内存）
podman run --ipc=host myapp

# 两个容器共享 IPC 命名空间
podman run --ipc=container:another_container myapp
```

### 10.7 文件系统隔离：挂载命名空间

挂载命名空间让容器拥有独立的文件系统视图：

- 容器只能看到自己的文件系统
- `mount` 操作在容器内不可见（默认 propagation=slave）
- 内核通过 `pivot_root` 切换根文件系统

### 10.8 文件系统隔离：SELinux ⭐⭐⭐

SELinux 是容器文件系统隔离的最后一道防线——即使进程逃出了命名空间，SELinux 仍然阻止未授权的文件访问。

**容器的 SELinux 工作方式**：

1. 每个容器进程获得一个唯一的 SELinux 类型（如 `container_t`）
2. 容器内的文件获得容器文件类型（如 `container_file_t`）
3. SELinux 策略只允许 `container_t` 进程读写 `container_file_t` 文件
4. 不同容器之间的 `container_file_t` 文件互相隔离（通过 MCS 类别）

**MCS (Multi-Category Security) 隔离**：

```bash
# 自定义 SELinux 标签
podman run --security-opt label=level:s0:c454,c510 myapp
# c454,c510 是 MCS 类别，确保此容器只能访问同类别的文件
```

**与 SELinux 的互动**：

```bash
# Docker 那种危险操作——不要做！
docker run -ti --name hack -v /:/host --privileged \
  registry.access.redhat.com/ubi8-micro chroot /host
# 这会将整个宿主机文件系统暴露给容器

# Podman 非特权模式天然阻止这种攻击
# 即使加了 --privileged，rootless 模式也没有真正的 root 权限
```

**/proc 和 /sys 的遮蔽 (mask)**：

```bash
# 默认遮蔽的 /proc 路径（安全保护）
--security-opt mask=/proc/sys/dev

# 取消遮蔽（需要访问特定内核信息时）
--security-opt unmask=/proc/scsi
```

### 10.9 系统调用隔离：seccomp ⭐⭐

seccomp (secure computing mode) 限制容器可以使用的系统调用——即使进程有足够权限，也不允许调用被禁止的 syscall。

**默认 seccomp 配置**：

Podman 使用 `/usr/share/containers/seccomp.json` 作为默认配置，阻止了约 40+ 个危险系统调用。

```bash
# 使用自定义 seccomp 配置
podman run --security-opt seccomp=/tmp/seccomp.json myapp

# 禁用 seccomp（仅调试用！）
podman run --security-opt seccomp=unconfined myapp
```

> **⚠️ 生产环境永远不要使用 `seccomp=unconfined`**。如果遇到 seccomp 阻止，应该将特定 syscall 添加到白名单，而不是完全禁用。

**自定义 seccomp 配置示例**：

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": ["open", "read", "write", "close", "execve"],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

### 10.10 虚拟机隔离

Podman 支持 **GVisor (runsc)** 和 **Kata Containers** 作为 OCI 运行时，提供虚拟机级别的隔离：

```bash
# containers.conf 中配置运行时
[engine]
runtime = "runc"      # 默认，基于命名空间
# runtime = "runsc"   # Google GVisor，用户空间内核
# runtime = "kata"    # Kata Containers，轻量级 VM

# 命令行指定运行时
podman run --runtime runsc myapp
```

```bash
# 查看 OCI 运行时
podman info --format '{{ .OCIRuntime }}'
runc
```

| 运行时 | 隔离级别 | 性能开销 | 适用场景 |
|--------|---------|---------|---------|
| runc | 命名空间 | 最低 | 通用 |
| runsc (GVisor) | 系统调用过滤 | 中 | 不受信任的工作负载 |
| kata (VM) | 硬件虚拟化 | 最高 | 多租户、合规要求 |

---

## 第11章：其他安全注意事项

### 11.1 守护进程 vs fork/exec 模型 ⭐

这是 Docker 和 Podman 最根本的架构差异：

**Docker (守护进程模型)**：

```
用户 → Docker CLI → Docker Daemon (root) → 容器进程
                    ↑
                    单点攻击面！
```

- Docker Daemon 以 root 运行，控制所有容器
- 如果 Daemon 被攻破，所有容器都暴露
- Daemon 崩溃可能导致所有容器丢失
- 所有操作都通过 Daemon 的 API，存在提权风险

**Podman (fork/exec 模型)**：

```
用户 → Podman → fork/exec → 容器进程（直接子进程）
```

- Podman 进程直接 fork 出容器进程
- 没有 Daemon，没有单点攻击面
- 每个用户的 Podman 进程独立
- 容器进程是 Podman 的直接子进程，继承用户权限
- 天然支持非特权运行

> **💡 为什么 fork/exec 更安全？** 在 Unix 安全模型中，进程只能操作其子进程。fork/exec 模型下，容器进程就是用户进程的子进程，受用户权限约束。守护进程模型下，所有容器都是 Daemon 的子进程，用户通过 API 与 Daemon 交互，存在权限代理的风险。

### 11.2 Podman 机密处理 ⭐

容器需要敏感数据（密码、API密钥、证书），但不应该写在镜像或环境变量中。

```bash
# 创建机密
echo "my_password" | podman secret create my_secret -

# 在容器中使用机密（作为环境变量）
podman run --secret my_secret,type=env myapp

# 在容器中使用机密（作为文件挂载）
podman run --secret my_secret,type=mount,target=/run/secrets/my_secret myapp

# 列出机密
podman secret ls

# 删除机密
podman secret rm my_secret
```

**机密 vs 环境变量的优势**：
- 🔒 机密不会出现在 `podman inspect` 输出中
- 🔒 机密不会写入镜像层
- 🔒 机密有独立的权限控制
- 🔒 机密可以轮换而不需要重建容器

### 11.3 Podman 镜像信任 ⭐

控制允许拉取和推送的镜像源，防止供应链攻击：

```bash
# 查看当前信任策略
podman image trust list

# 拒绝来自 docker.io 的所有镜像
sudo podman image trust set -t reject docker.io

# 只信任 docker.io/library（官方镜像）
sudo podman image trust set -t accept docker.io/library

# 设置默认拒绝策略
sudo podman image trust set --type=reject default
```

**信任策略层级**：

```
1. 默认策略（兜底）
2. 注册服务器级策略
3. 仓库级策略（最具体，优先级最高）
```

**签名存储**：

```
sigstore: https://sigstore.example.com
sigstore-staging: file:///var/lib/containers/sigstore
```

### 11.4 Podman 镜像扫描

```bash
# 扫描镜像中的已知漏洞
podman image scan myimage
```

镜像扫描检查镜像层中的软件包是否有已知 CVE 漏洞。

### 11.5 纵深安全 (Defense in Depth) ⭐⭐⭐

这是全书安全哲学的总结——**没有任何单一安全机制是足够的，必须多层防御**。

Podman 的纵深安全层：

```
┌──────────────────────────────────────┐
│  第1层：用户命名空间 (User NS)        │  ← UID 隔离，容器 root ≠ 宿主 root
├──────────────────────────────────────┤
│  第2层：PID 命名空间                  │  ← 进程隔离
├──────────────────────────────────────┤
│  第3层：网络命名空间                   │  ← 网络隔离
├──────────────────────────────────────┤
│  第4层：IPC 命名空间                  │  ← IPC 隔离
├──────────────────────────────────────┤
│  第5层：挂载命名空间                   │  ← 文件系统隔离
├──────────────────────────────────────┤
│  第6层：Linux Capabilities            │  ← 最小权限，只保留必要能力
├──────────────────────────────────────┤
│  第7层：SELinux / MCS                │  ← 强制访问控制，文件级隔离
├──────────────────────────────────────┤
│  第8层：seccomp                      │  ← 系统调用过滤
├──────────────────────────────────────┤
│  第9层：只读文件系统                   │  ← 防止运行时修改
├──────────────────────────────────────┤
│  第10层：镜像信任与扫描                │  ← 供应链安全
├──────────────────────────────────────┤
│  第11层：机密管理                     │  ← 敏感数据保护
├──────────────────────────────────────┤
│  第12层：fork/exec 模型               │  ← 无守护进程，无单点攻击面
├──────────────────────────────────────┤
│  第13层：GVisor / Kata (可选)         │  ← 虚拟机级隔离
└──────────────────────────────────────┘
```

**只读容器**：

```bash
# 容器根文件系统只读
podman run --read-only myapp

# 但 /tmp 和 /var/tmp 也需要可写时
podman run --read-only --read-only-tmpfs=false myapp
# --read-only-tmpfs: 控制 tmpfs 是否只读
# --read-only-tmpfs=false 允许 tmpfs 可写
```

> **💡 纵深安全思维**：每一层都可能被绕过，但组合在一起时，攻击者需要同时突破所有层。例如，即使攻击者通过内核漏洞逃出了命名空间，仍然受到 SELinux 和 seccomp 的约束。

---

## 附录：容器工具生态

### Buildah

```bash
# 在用户命名空间中运行的 shell（用于调试/构建）
buildah unshare
# buildah unshare 命令会创建在用户命名空间中运行的 shell
```

### Skopeo

```bash
# 检查远程镜像信息（不拉取）
skopeo inspect docker://quay.io/rhatdan/myimage
```

### OCI 运行时

```bash
# 查看配置的运行时
cat /usr/share/containers/containers.conf | grep runtime
# [engine]
# runtime = "runc"

# 指定自定义运行时
podman run --runtime=/usr/bin/runc myapp

# 查看运行时版本
runc --version
```

---

## 🔑 安全命令速查表

| 安全特性 | 命令 | 说明 |
|---------|------|------|
| **能力** | `--cap-drop=all` | 丢弃所有能力 |
| | `--cap-add CAP_NET_RAW` | 添加特定能力 |
| | `--security-opt no-new-privileges` | 禁止提权 |
| **用户命名空间** | `--userns=auto:size=500` | 自动分配用户NS |
| **PID 隔离** | `--pid=host` | 共享宿主机PID（危险） |
| **IPC 隔离** | `--ipc=host` | 共享宿主机IPC |
| | `--ipc=container:NAME` | 共享另一容器IPC |
| **SELinux** | `--security-opt label=level:s0:c454,c510` | 自定义MCS标签 |
| | `--security-opt label=disable` | 禁用SELinux（危险） |
| | `--security-opt mask=/proc/sys/dev` | 遮蔽路径 |
| | `--security-opt unmask=/proc/scsi` | 取消遮蔽 |
| | `-v /data:/data:z` | 共享SELinux标签 |
| | `-v /data:/data:Z` | 私有SELinux标签 |
| | `-v /data:/data:U` | 自动修改所有权 |
| | `-v /data:/data:idmap` | UID映射 |
| **seccomp** | `--security-opt seccomp=/path/seccomp.json` | 自定义seccomp |
| | `--security-opt seccomp=unconfined` | 禁用seccomp（危险） |
| **只读** | `--read-only` | 只读根文件系统 |
| | `--read-only-tmpfs=false` | 允许tmpfs可写 |
| **机密** | `podman secret create` | 创建机密 |
| | `--secret my_secret,type=env` | 环境变量方式使用 |
| **镜像信任** | `podman image trust set -t reject` | 拒绝镜像源 |
| | `podman image trust set -t accept` | 信任镜像源 |
| **日志** | `log_driver="journald"` | journald日志驱动 |
| **自动更新** | `--label "io.containers.autoupdate=registry"` | 自动更新标签 |

---

## 🧠 读书笔记关键词分析

从 164 条笔记中提取的核心关注点：

| 类别 | 数量 | 说明 |
|------|------|------|
| 🔒 安全命令 | ~60 | `--cap-drop`, `--security-opt`, seccomp, SELinux, 信任策略 |
| 📦 存储与卷 | ~20 | `storage.conf`, 卷挂载选项 (`:z/:Z/:U/:idmap`) |
| 🏗️ 构建与镜像 | ~15 | `podman image mount/inspect`, Buildah, Skopeo |
| 🌐 网络与K8s | ~10 | `podman generate kube`, 端口映射, slirp4netns |
| ⚙️ systemd | ~10 | `podman generate systemd`, NOTIFY_SOCKET, autoupdate |
| 📖 概念理解 | ~10 | OCI, manifest, capabilities 理论 |

**你的阅读重点明显在第4部分（容器安全）**，安全相关笔记占了近 40%。这说明你是有意识地在补充容器安全知识，特别是从 SELinux System Administration 过渡到 Podman 安全，形成了完整的 Linux 安全知识链：

```
SELinux (理论基础) → Podman容器隔离 (实践应用) → 纵深防御 (体系构建)
```

---

*基于微信读书笔记数据生成 | 2026-06-13*  
*阅读版本: v1761861006 | 全书 30 章*
