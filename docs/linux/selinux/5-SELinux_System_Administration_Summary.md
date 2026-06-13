---
tags:
  - SELinux
---
# SELinux System Administration — 深度技术实战笔记

> **书名**: SELinux System Administration  
> **作者**: Sven Vermeulen  
> **出版社**: Packt Publishing (2013)  
> **ISBN**: 9781783283187  
> **阅读时长**: 6小时24分 | **笔记**: 97条  
> **本文基于原书内容，更新至 RHEL 9 / Fedora 40+ 的现代命令和实践**

---

## ⚠️ 原书 vs 现代版本的重要变化

原书出版于 2013 年，部分命令在新版本中已变化。以下是关键差异：

| 原书命令 | 现代替代 | 变化说明 |
|---------|---------|---------|
| `chcon` | `semanage fcontext` + `restorecon` | chcon 仅临时修改，重启/relable 后丢失；**生产环境应使用 semanage 永久规则** |
| `chcat` | `semanage login -r` / `semanage user` | chcat 在 RHEL 9 中已弃用，用 semanage 替代 |
| `fixfiles` | `restorecon -R` / `semanage fcontext` | fixfiles 仍可用但功能被 restorecon 覆盖 |
| `rlpkg` | `restorecon -R` | rlpkg 在新版本中已移除 |
| `/etc/selinux/config` 中 `SELINUX=disabled` | 内核参数 `selinux=0` | RHEL 9 起不再支持配置文件中设 disabled |
| `semodule --build` | `semodule -B` | 简写等效 |
| `audit2allow -R` | `audit2allow -R` | 仍然支持，但优先尝试标签和布尔值 |
| 原生策略语言 `.te` | CIL (Common Intermediate Language) | RHEL 9 默认使用 CIL 格式，但 `.te` 仍支持 |

---

## 全书结构

| 章 | 标题 | 核心主题 |
|----|------|----------|
| 1 | Fundamental SELinux Concepts | 安全模型、标签、策略类型 |
| 2 | Understanding SELinux Decisions and Logging | 日志审计与排错 |
| 3 | Managing User Logins | 用户、角色、上下文管理 |
| 4 | Process Domains and File-level Access Controls | 进程域与文件访问控制 |
| 5 | Controlling Network Communications | 网络通信控制 |
| 6 | Working with SELinux Policies | 策略定制与模块开发 |

---

# 第1章：SELinux 基础概念

## 1.1 为什么需要 SELinux — DAC 的问题

### DAC (Discretionary Access Control) — 自主访问控制

传统 Linux 权限模型：文件所有者自行决定谁可以访问（rwx + owner/group/other）。

**问题**：
- root 用户不受任何限制——一旦提权成功，系统完全失控
- 用户可以 chmod 777，主动降低安全性
- SUID 程序（如 `passwd`）以 root 身份运行，是经典攻击面
- 进程继承用户权限，恶意软件可以利用用户身份

```bash
# DAC 下的典型问题：一个被入侵的 httpd 进程
# 如果 httpd 以 root 运行（或被提权），它可以读 /etc/shadow
cat /etc/shadow   # 在 DAC 下，root 可以读任何文件
```

### MAC (Mandatory Access Control) — 强制访问控制

SELinux 实现的模型：**系统策略强制执行，用户和进程都无法覆盖**。

- 策略由管理员定义，进程无法自行修改
- 即使是 root，也受 SELinux 策略约束
- 基于 **LSM (Linux Security Modules)** 框架，在内核层实现
- 源自美国国防部 **TCSEC (Trusted Computer System Evaluation Criteria)** 标准

**SELinux 不是替换 DAC，而是在 DAC 之上增加一层**：

```
访问请求 → DAC 检查 → SELinux 检查 → 允许/拒绝
           ↓              ↓
        传统权限       强制策略
        (先检查)       (后检查，更严格)
```

> **关键点**：即使 DAC 允许（文件权限 777），SELinux 仍然可以拒绝。两层都必须通过。

## 1.2 一切皆有标签 — SELinux 上下文

SELinux 的核心思想：**系统中的每个进程、文件、端口、设备都有一个安全标签（上下文）**。策略基于标签之间的规则来允许或拒绝访问。

### 上下文格式

```
user:role:type:sensitivity[:category]
```

| 字段 | 含义 | 示例 |
|------|------|------|
| `user` | SELinux 用户身份 | `system_u`, `unconfined_u` |
| `role` | SELinux 角色 | `system_r`, `object_r` |
| `type` | 类型/域（**最关键字段**） | `httpd_t`, `httpd_sys_content_t` |
| `sensitivity` | 敏感度级别（MLS） | `s0` |
| `category` | 类别（MCS） | `c0.c1023` |

```bash
# 查看文件的安全上下文
ls -Z /var/www/html/index.html
# system_u:object_r:httpd_sys_content_t:s0

# 查看进程的安全上下文
ps -eZ | grep httpd
# system_u:system_r:httpd_t:s0    1234 ?  00:00:01 httpd

# 查看端口的安全上下文
sudo semanage port -l | grep http
# http_port_t     tcp   80, 81, 443, 488, 8008, 8009, 8443, 9000
```

### Type Enforcement (TE) — 类型强制

RHEL/CentOS 的 targeted 策略主要基于 **type** 字段：

- 进程的 type 称为 **域 (domain)**：如 `httpd_t`
- 资源的 type 称为 **类型 (type)**：如 `httpd_sys_content_t`
- 策略规则定义：**哪个域可以访问哪个类型，以及什么权限**

```
httpd_t 进程 → 允许读 → httpd_sys_content_t 文件
httpd_t 进程 → 拒绝写 → shadow_t 文件
```

### MLS (Multi-Level Security) — 多级安全

用于需要数据分级的场景（军事、情报、金融）：

- 敏感度从 s0（最低）到 sN（最高）
- **No Read Up**：不能读比自己级别高的数据
- **No Write Down**：不能往比自己级别低的写数据
- 防止高密级信息泄露到低密级

```
进程 s2 → 可读 s0, s1, s2 → 可写 s2, s3, ... → 不可读 s3+
```

### MCS (Multi-Category Security) — 多类别安全

MLS 的简化版，用于**容器隔离**等场景：

- 使用类别（c0, c1, c2, ...）标记不同组的数据
- 不同 MCS 类别之间互相隔离
- Podman/Docker 使用 MCS 实现容器间文件隔离

## 1.3 策略类型

| 策略 | 说明 | 适用场景 |
|------|------|---------|
| **Targeted** | 仅限制特定网络服务，其他进程 unconfined | RHEL/CentOS 默认，生产推荐 |
| **MLS** | 多级安全策略 | 军事/政府/高密级环境 |
| **Strict** | 所有进程都受限 | 极端安全环境（很少用） |

```bash
# 查看当前加载的策略
sestatus | grep "Loaded policy"
# Loaded policy name:             targeted

# 查看策略的 deny_unknown 状态
sestatus | grep "deny_unknown"
# Policy deny_unknown status:     allowed
```

### unconfined 域

在 targeted 策略下，用户进程通常运行在 `unconfined_t` 域——几乎不受 SELinux 限制。**只有网络服务进程被限制**（如 httpd_t, named_t, sshd_t）。

```bash
# 查看当前用户进程的域
ps -eZ | grep $$ | head -1
# unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023

# 查看受限的服务域
seinfo -t | grep _t | wc -l
# 约 5000+ 种类型
```

---

# 第2章：SELinux 决策与日志 — 排错全流程

这是**运维最常用的章节**。当 SELinux 拒绝了合法操作时，你需要一套系统的排错方法。

## 2.1 SELinux 运行模式

```bash
# 查看当前模式
getenforce
# Enforcing / Permissive / Disabled

# 详细状态
sestatus
# SELinux status:                 enabled
# SELinuxfs mount:                /sys/fs/selinux
# SELinux root directory:         /etc/selinux
# Loaded policy name:             targeted
# Current mode:                   enforcing
# Mode from config file:          enforcing
# Policy MLS status:              enabled
# Policy deny_unknown status:     allowed
# Max kernel policy version:      33
```

### 模式切换

```bash
# 临时切换（重启后恢复）
setenforce 0    # → Permissive（仅记录不拒绝，调试用）
setenforce 1    # → Enforcing（强制执行）

# 永久修改
sudo vi /etc/selinux/config
# SELINUX=enforcing    # 或 permissive
# ⚠️ RHEL 9 不再支持 SELINUX=disabled，需用内核参数 selinux=0
```

> **💡 调试技巧**：遇到疑似 SELinux 问题时，先 `setenforce 0` 切到 Permissive。如果问题消失，确认是 SELinux 拒绝，然后按下面的排错流程处理，最后切回 Enforcing。

## 2.2 AVC (Access Vector Cache) — SELinux 的决策缓存

SELinux 每次访问检查都查 AVC 缓存。缓存未命中时查策略，结果写入 AVC。被拒绝的访问会被记录到审计日志。

**AVC 拒绝消息格式**：

```
type=AVC msg=audit(1686123456.789:1234): avc:  denied  { read } for  pid=1234
  comm="httpd" name="index.html" dev="sda1" ino=56789
  scontext=system_u:system_r:httpd_t:s0
  tcontext=unconfined_u:object_r:default_t:s0
  tclass=file permissive=0
```

| 字段 | 含义 |
|------|------|
| `denied { read }` | 被拒绝的权限 |
| `scontext` | 源上下文（发起访问的进程） |
| `tcontext` | 目标上下文（被访问的资源） |
| `tclass` | 目标类别（file/dir/socket 等） |
| `permissive=0` | Enforcing 模式（1=Permissive，仅记录） |

## 2.3 🔥 SELinux 拒绝排错全流程

### Step 1：确认是否是 SELinux 问题

```bash
# 方法1：临时切到 Permissive
setenforce 0
# 重现操作 → 如果成功，确认是 SELinux 拒绝
setenforce 1    # 排错完成后切回

# 方法2：查看审计日志中是否有 AVC 拒绝
sudo ausearch -m AVC,USER_AVC,SELINUX_ERR,USER_SELINUX_ERR -ts recent
```

### Step 2：收集拒绝日志

```bash
# 查看最近的 AVC 拒绝
sudo ausearch -m AVC -ts recent

# 查看特定时间段的拒绝
sudo ausearch -m AVC -ts today
sudo ausearch -m AVC -ts "06/13/2026 14:00" -te "06/13/2026 15:00"

# 通过 dmesg 查看（审计守护进程未运行时）
dmesg | grep -i avc | tail -20

# 通过 journalctl 查看
journalctl -t audit -g avc | tail -20

# 查看特定进程的拒绝（如 httpd）
sudo ausearch -m AVC -c httpd
```

### Step 3：解读拒绝消息

```bash
# 典型拒绝消息示例
type=AVC msg=audit(1686123456.789:1234): avc:  denied  { write } for
  pid=5678 comm="httpd" name="uploads" dev="sda1" ino=98765
  scontext=system_u:system_r:httpd_t:s0
  tcontext=system_u:object_r:var_t:s0
  tclass=dir permissive=0
```

**解读**：
- **谁** (`scontext`): httpd 进程（httpd_t 域）
- **想做什么** (`denied`): 写入
- **到哪里** (`tcontext`): var_t 类型的目录
- **目标类型** (`tclass`): 目录
- **根因**：httpd_t 域不允许写入 var_t 类型的目录。应该是 httpd_sys_content_t 或 httpd_sys_rw_content_t。

### Step 4：使用 setroubleshoot 自动诊断

```bash
# 安装（如果未安装）
sudo dnf install setroubleshoot-server setroubleshoot-plugins

# setroubleshoot 会自动分析 AVC 拒绝并提供建议
# 查看诊断结果
sealert -a /var/log/audit/audit.log

# 查看最近的诊断
journalctl -t setroubleshoot | tail -20

# 更详细的分析
sealert -l <alert-uuid>
```

**sealert 输出示例**：

```
SELinux is preventing httpd from write access on the directory /var/www/uploads.

*****  Plugin catchall_boolean (89.3 confidence) suggests  *******************

If you want to allow httpd to modify public files
Then you must tell SELinux about this by enabling the 'httpd_unified' boolean.

Do
setsebool -P httpd_unified on

*****  Plugin catchall_labels (10.0 confidence) suggests  *******************

If you want to allow httpd to write to the uploads directory
Then the directory must be labeled with a type that httpd is allowed to write to.

Do
semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/uploads(/.*)?"
restorecon -Rv /var/www/uploads
```

> **💡 sealert 的置信度评分 (confidence score)**：表示建议的可靠性。89.3% 的布尔值方案 vs 10% 的标签方案——优先尝试高置信度建议。

### Step 5：按照排错决策树处理 ⭐⭐⭐

**排错决策树（优先级从高到低）**：

```
                    SELinux 拒绝
                         │
            ┌────────────┴────────────┐
            │ 1. 标签是否正确？        │
            │   (最常见原因)           │
            └────────────┬────────────┘
                    正确 ↓  不正确 → 修复标签
            ┌────────────┴────────────┐
            │ 2. 布尔值能否解决？      │
            │   (策略预设的开关)       │
            └────────────┬────────────┘
                    能 ↓  不能 → 继续
            ┌────────────┴────────────┐
            │ 3. 源进程标签是否正确？   │
            └────────────┬────────────┘
                    正确 ↓  不正确 → 修复
            ┌────────────┴────────────┐
            │ 4. 是否按预期方式使用？   │
            │   (而非通过非标准方式)    │
            └────────────┬────────────┘
                    是 ↓  否 → 改用标准方式
            ┌────────────┴────────────┐
            │ 5. audit2allow 生成规则  │
            │   (⚠️ 最后手段)          │
            └─────────────────────────┘
```

**详细解释每一层**：

#### 第1层：修复标签 ✅（最常见，占 ~70% 的情况）

非默认路径/端口的资源标签不正确：

```bash
# 示例：Web 内容放在 /srv/www 而非默认的 /var/www
ls -Zd /srv/www
# unconfined_u:object_r:var_t:s0    ← 错误！应该是 httpd_sys_content_t

# 修复：添加永久标签规则 + 应用
sudo semanage fcontext -a -t httpd_sys_content_t "/srv/www(/.*)?"
sudo restorecon -Rv /srv/www
# Relabeled /srv/www from unconfined_u:object_r:var_t:s0 to system_u:object_r:httpd_sys_content_t:s0

# 非默认端口
sudo semanage port -a -t http_port_t -p tcp 84
```

#### 第2层：切换布尔值 ✅（占 ~20% 的情况）

策略预设了可切换的开关：

```bash
# 列出所有布尔值
getsebool -a

# 查看与 httpd 相关的布尔值
getsebool -a | grep httpd
# httpd_can_network_connect --> off
# httpd_unified --> off
# httpd_enable_homedirs --> off

# 开启布尔值（-P 表示永久生效）
sudo setsebool -P httpd_can_network_connect on

# 查看布尔值说明
semanage boolean -l | grep httpd_can_network
# httpd_can_network_connect  (off , off)  Allow httpd to can network connect
```

> **布尔值 vs 标签**：布尔值是策略预设的开关，适用于常见场景（如"允许 httpd 连接网络"）。标签是更精确的方案。优先尝试标签，标签无法解决时再考虑布尔值。

#### 第3层：修复源进程标签 ✅

进程本身的标签不对（通常因为非标准安装方式）：

```bash
# 查看进程标签
ps -eZ | grep myapp
# unconfined_u:unconfined_r:unconfined_t:s0  ← 不对，应该被限制

# 修复：确保可执行文件的标签正确
ls -Z /usr/local/bin/myapp
sudo restorecon -v /usr/local/bin/myapp
```

#### 第4层：按预期方式使用 ✅

有时候"问题"其实是操作方式不对：

- 不应该通过 `ssh` 直接在服务器上运行服务，而应该通过 systemd
- 不应该用 `cp` 拷贝文件（保留源标签），而应该用 `install` 或 `mv`

```bash
# cp 保留源标签（可能导致标签不正确）
cp /tmp/myfile /var/www/html/
ls -Z /var/www/html/myfile
# unconfined_u:object_r:user_tmp_t:s0  ← 错误！

# 修复
restorecon -v /var/www/html/myfile

# 正确做法：使用 install（会创建新标签）或 mv 后 restorecon
install -m 644 /tmp/myfile /var/www/html/
```

#### 第5层：audit2allow ⚠️（最后手段）

当以上方法都无法解决时，使用 audit2allow 从 AVC 拒绝日志自动生成策略规则：

```bash
# 方法1：分析所有 AVC 拒绝
sudo ausearch -m AVC -ts recent | audit2allow -M myapp_custom
# 生成两个文件：myapp_custom.te（源码）和 myapp_custom.pp（编译后的模块）

# 查看生成的规则
cat myapp_custom.te

# 加载模块
sudo semodule -i myapp_custom.pp

# 方法2：生成参考策略宏版本（更易维护）
sudo ausearch -m AVC -ts recent | audit2allow -R -M myapp_custom

# 方法3：仅查看建议规则（不生成模块）
sudo ausearch -m AVC -c myapp | audit2allow
```

> **⚠️ audit2allow 的风险**：
> - 它只是把拒绝变成了允许，没有考虑安全影响
> - 可能过度授权（allow 了不需要的权限）
> - 每次使用都应该审查生成的规则
> - **永远是最后手段**——先尝试标签和布尔值

### 完整排错实战示例

**场景**：自定义 Web 应用部署在 /opt/webapp，httpd 无法读取文件。

```bash
# Step 1: 确认是 SELinux 问题
sudo setenforce 0
# → 问题消失，确认是 SELinux

# Step 2: 收集拒绝日志
sudo ausearch -m AVC -c httpd -ts recent
# type=AVC msg=audit(...): avc:  denied  { read } for pid=1234 comm="httpd"
#   name="index.html" scontext=system_u:system_r:httpd_t:s0
#   tcontext=unconfined_u:object_r:usr_t:s0  tclass=file

# Step 3: 解读 — httpd_t 不能读 usr_t 文件

# Step 4: sealert 自动诊断
sudo sealert -a /var/log/audit/audit.log
# 建议：将 /opt/webapp 标记为 httpd_sys_content_t

# Step 5: 按决策树处理 → 第1层（标签问题）
sudo semanage fcontext -a -t httpd_sys_content_t "/opt/webapp(/.*)?"
sudo restorecon -Rv /opt/webapp

# Step 6: 切回 Enforcing 并验证
sudo setenforce 1
curl http://localhost/  # → 成功
```

## 2.4 日志工具详解

```bash
# ausearch — 搜索审计日志
sudo ausearch -m AVC                    # 所有 AVC 拒绝
sudo ausearch -m AVC -c httpd           # httpd 的拒绝
sudo ausearch -m AVC -ts today          # 今天的拒绝
sudo ausearch -m AVC -ts "06/13/2026"   # 指定日期

# ausearch — 按上下文搜索
sudo ausearch -m AVC -s httpd_t         # 源域为 httpd_t
sudo ausearch -m AVC -e default_t       # 目标类型含 default_t

# aureport — 审计报告
sudo aureport -a                        # 所有审计事件摘要
sudo aureport -a -i                     # 带解释的摘要
sudo aureport --avc                     # AVC 拒绝报告

# seinfo — 查询策略信息
seinfo -t                               # 列出所有类型
seinfo -t httpd                         # 搜索含 httpd 的类型
seinfo -r                               # 列出所有角色
seinfo -u                               # 列出所有 SELinux 用户
seinfo --stats                          # 策略统计

# sesearch — 搜索策略规则
sesearch -A -s httpd_t                  # httpd_t 域的所有 allow 规则
sesearch -A -s httpd_t -t httpd_sys_content_t  # httpd_t 对 httpd_sys_content_t 的规则
sesearch --dontaudit -s httpd_t         # dontaudit 规则

# semodule — 管理策略模块
semodule -l                             # 列出已安装模块
semodule -l | grep myapp                # 搜索模块
semodule -i myapp.pp                    # 安装模块
semodule -r myapp                       # 删除模块
semodule -B                             # 重建策略（等同于 --build）
semodule --disable_dontaudit            # 临时禁用 dontaudit 规则（调试用）
# 恢复：semodule -B

# setroubleshoot
sudo sealert -a /var/log/audit/audit.log   # 分析所有拒绝
sudo sealert -l <uuid>                      # 查看特定告警详情
```

---

# 第3章：用户与上下文管理

## 3.1 Linux 用户 vs SELinux 用户

**它们是不同的东西**：

```
Linux 用户 (dan) ←→ SELinux 用户 (staff_u) ←→ SELinux 角色 (staff_r) ←→ SELinux 域 (staff_t)
```

| 维度 | Linux 用户 | SELinux 用户 |
|------|-----------|-------------|
| 数量 | 可能有数百个 | 通常只有几个（staff_u, user_u, sysadm_u, system_u, unconfined_u） |
| 用途 | 系统登录身份 | 定义可扮演的角色和敏感度范围 |
| 映射 | 一对多：多个 Linux 用户可映射到同一个 SELinux 用户 | |

```bash
# 查看当前用户映射
id -Z
# unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023

# 查看所有 SELinux 用户
sudo semanage user -l
#              Labeling   MLS/       MLS/                          
# SELinux User Prefix     MCS Level  MCS Range                      SELinux Roles
# staff_u     staff       s0         s0-s0:c0.c1023                 staff_r sysadm_r
# sysadm_u    sysadm      s0         s0-s0:c0.c1023                 sysadm_r
# system_u    system      s0         s0-s0:c0.c1023                 system_r
# unconfined_u unconfined  s0         s0-s0:c0.c1023                 unconfined_r
# user_u      user        s0         s0                             user_r

# 查看登录映射
sudo semanage login -l
# Login Name    SELinux User    MLS/MCS Range   Service
# __default__   unconfined_u    s0-s0:c0.c1023  *
# root          unconfined_u    s0-s0:c0.c1023  *
# dan           staff_u         s0-s0:c0.c1023  *
```

## 3.2 创建和管理 SELinux 用户

```bash
# 创建新的 SELinux 用户
sudo semanage user -a -R staff_r -R sysadm_r -L s0 -r s0-s0:c0.c1023 myadmin_u
# -R: 允许的角色（可指定多个）
# -L: 默认 MLS 级别
# -r: 允许的 MLS/MCS 范围

# 修改 SELinux 用户
sudo semanage user -m -R staff_r myadmin_u    # 移除 sysadm_r 角色

# 删除 SELinux 用户
sudo semanage user -d myadmin_u

# 将 Linux 用户映射到 SELinux 用户
sudo semanage login -a -s staff_u dan
# dan → staff_u

# 将用户组映射到 SELinux 用户
sudo semanage login -a -s user_u "%developers"
# developers 组 → user_u

# 修改映射
sudo semanage login -m -s sysadm_u dan
# dan → sysadm_u

# 删除映射
sudo semanage login -d dan

# 映射后恢复上下文
sudo restorecon -Rv /home/dan
```

## 3.3 角色切换

### newrole — 切换角色和敏感度

```bash
# 切换角色
newrole -r sysadm_r
# 需要输入密码

# 切换到特定敏感度
newrole -l s0:c0.c100
# 需要 MCS 范围在用户允许范围内

# 退出新角色
exit    # 返回原来的角色
```

### sudo 集成

```bash
# 通过 sudo 切换角色和类型
sudo -r dbadm_r -t dbadm_t command

# 在 sudoers 中配置
# /etc/sudoers.d/dan
dan ALL=(ALL) TYPE=dbadm_t ROLE=dbadm_r ALL
```

### runcon — 在指定上下文中运行命令

```bash
# 在特定上下文中运行命令
runcon system_u:system_r:httpd_t:s0 /usr/sbin/httpd

# 仅指定角色
runcon -r object_r -t httpd_sys_content_t ls -Z /var/www

# ⚠️ runcon 的使用场景有限，主要用于测试和调试
```

## 3.4 PAM 登录自动分配

用户登录时，PAM 模块自动根据 `semanage login` 的映射分配 SELinux 上下文：

```
用户登录 → PAM → 查询 semanage login 映射 → 分配 SELinux 用户/角色/域
```

```bash
# getseuser — 查看 Linux 用户到 SELinux 用户的映射
getseuser dan
# dan: staff_u

# 查看用户的完整上下文
id -Z
```

---

# 第4章：进程域与文件访问控制

## 4.1 🔥 文件上下文管理 — 最核心的实操内容

### chcon vs semanage fcontext — 临时 vs 永久

| 维度 | `chcon` | `semanage fcontext` + `restorecon` |
|------|---------|--------------------------------------|
| 持久性 | ❌ 临时，重启/relable 后丢失 | ✅ 永久，写入策略规则 |
| 适用场景 | 快速测试 | 生产环境 |
| 匹配能力 | 精确路径 | 正则表达式匹配 |
| 推荐度 | ⚠️ 仅测试用 | ✅ 生产推荐 |

> **现代实践**：永远不要在生产环境使用 `chcon`。使用 `semanage fcontext` 添加规则，然后 `restorecon` 应用。`chcon` 的修改会在 `restorecon` 或系统重新标记时被覆盖。

### semanage fcontext — 永久文件上下文管理

```bash
# 列出所有文件上下文规则
sudo semanage fcontext -l

# 搜索特定路径的规则
sudo semanage fcontext -l | grep /var/www

# 添加新规则
sudo semanage fcontext -a -t httpd_sys_content_t "/srv/www(/.*)?"
# -a: 添加
# -t: 指定类型
# "/srv/www(/.*)?": 正则匹配 /srv/www 及其所有子内容

# 添加规则 — 指定文件类型
sudo semanage fcontext -a -t httpd_sys_rw_content_t -f d "/srv/www/uploads(/.*)?"
# -f d: 仅匹配目录（-f --: 普通文件, -f l: 符号链接, -f c: 字符设备）

# 修改现有规则
sudo semanage fcontext -m -t httpd_sys_rw_content_t "/srv/www/uploads(/.*)?"

# 删除规则
sudo semanage fcontext -d "/srv/www/uploads(/.*)?"

# 删除所有本地自定义规则
sudo semanage fcontext -D

# 查看仅本地自定义规则
sudo semanage fcontext -l -C

# 应用规则（关键！添加规则后必须执行）
sudo restorecon -Rv /srv/www
# -R: 递归
# -v: 显示变更
```

### restorecon — 恢复默认上下文

```bash
# 恢复单个文件
sudo restorecon -v /var/www/html/index.html

# 递归恢复目录
sudo restorecon -Rv /var/www/html/

# 强制恢复（即使已有正确标签也重新设置）
sudo restorecon -Fv /var/www/html/

# 排除某些路径
sudo restorecon -Rv -e /var/www/html/uploads /var/www/html/

# 使用匹配的父目录上下文
sudo restorecon -v /var/www/html/newfile
```

### 全量重新标记

当系统标签大面积混乱时：

```bash
# 方法1：创建 .autorelabel 文件，下次重启时自动全量重新标记
sudo touch /.autorelabel
sudo reboot
# ⚠️ 大文件系统可能需要很长时间

# 方法2：使用 fixfiles（仍可用但推荐 restorecon）
sudo fixfiles relabel

# 方法3：使用 restorecon 全量恢复
sudo restorecon -Rv /
```

### 匹配优先级规则

当多条 `semanage fcontext` 规则匹配同一文件时：

1. **无正则表达式的行**优先于有正则的行
2. **更具体的路径**优先（正则前字符数少的更具体 → 原书规则，新版本中更复杂的优先级算法已改进）
3. **后添加的本地规则**优先于系统默认规则

```bash
# 查看匹配某个路径的所有规则
sudo semanage fcontext -l | grep "/opt/web"
```

### 卷标签的特殊处理

在容器环境中，绑定挂载的卷需要 SELinux 标签处理：

```bash
# :z — 共享标签（多容器共享读写）
podman run -v /host/data:/data:z myapp

# :Z — 私有标签（仅当前容器可读写）
podman run -v /host/data:/data:Z myapp

# ⚠️ 不要混用 :z 和 :Z — 同一路径不能同时标记为共享和私有
```

## 4.2 进程上下文管理

```bash
# 查看进程的安全上下文
ps -eZ
ps -eZ | grep httpd

# 查看进程的完整上下文
cat /proc/<pid>/attr/current

# setexeccon — 设置下次 exec 的上下文
# 在脚本中使用：
setexeccon system_u:system_r:httpd_t:s0
/usr/sbin/httpd

# 查看进程域的允许操作
sesearch -A -s httpd_t
```

### 进程域转换

当进程 exec 一个新程序时，SELinux 会自动进行域转换：

```
init_t → 执行 /usr/sbin/httpd → httpd_t
unconfined_t → 执行 /usr/sbin/sshd → sshd_t
```

```bash
# 查看域转换规则
sesearch -T -s init_t | grep httpd
# type_transition init_t httpd_exec_t:process httpd_t
```

---

# 第5章：网络通信控制

## 5.1 TCP/UDP 端口标签

```bash
# 列出所有端口标签
sudo semanage port -l

# 查看特定类型的端口
sudo semanage port -l | grep http_port_t
# http_port_t  tcp  80, 81, 443, 488, 8008, 8009, 8443, 9000

# 添加新端口标签
sudo semanage port -a -t http_port_t -p tcp 84
# -a: 添加
# -t: 类型
# -p: 协议（tcp/udp/udp_dccp/sctp）

# 修改端口标签
sudo semanage port -m -t http_port_t -p tcp 84

# 删除端口标签
sudo semanage port -d -t http_port_t -p tcp 84

# 添加端口范围
sudo semanage port -a -t http_port_t -p tcp 8080-8090
```

**常见场景**：Nginx 监听非标准端口 8443

```bash
# 1. 查看当前 http_port_t 允许的端口
sudo semanage port -l | grep http_port_t

# 2. 如果 8443 不在列表中，添加
sudo semanage port -a -t http_port_t -p tcp 8443

# 3. 验证
sudo semanage port -l | grep http_port_t
```

## 5.2 SELinux 与 Netfilter (iptables/nftables) 集成

### SECMARK — 为网络包打 SELinux 标签

```bash
# 使用 nftables 为 HTTP 包打标签（现代方式）
nft add rule ip filter INPUT tcp dport 80 meta secmark set "http_server_packet_t"

# 使用 iptables 的 SECMARK（旧方式，仍兼容）
iptables -t security -A SEL_M_HTTP -j SECMARK --selctx system_u:object_r:http_server_packet_t:s0
```

## 5.3 标记网络 (Labeled Networking)

- **接口检查 (interface checking)**：基于网络接口的安全标签
- **节点检查 (node checking)**：基于主机 IP 的安全标签
- **对等检查 (peer checking)**：基于对端的安全标签（Netlabel/CIPSO）
- 适用于跨主机的强制访问控制（多机 MLS 环境）

---

# 第6章：策略定制与模块开发

## 6.1 布尔值管理

```bash
# 列出所有布尔值
getsebool -a

# 带说明的布尔值列表
semanage boolean -l

# 查看布尔值状态
getsebool httpd_can_network_connect
# httpd_can_network_connect --> off

# 设置布尔值（-P 永久生效）
sudo setsebool -P httpd_can_network_connect on

# 同时设置多个
sudo setsebool -P httpd_can_network_connect on httpd_enable_homedirs on

# 布尔值字段说明
semanage boolean -l | grep virt_use_nfs
# virt_use_nfs  (off , off)  Allow virt to use nfs
#               ^^^^^   ^^^^
#               当前值   默认值
# off=D(disabled) / on=E(enabled)
# 默认值来自 T(true)/F(false)
```

## 6.2 🔥 创建自定义 SELinux 策略模块

### 方法1：使用 audit2allow 快速生成

```bash
# Step 1: 重现问题，收集 AVC 拒绝
sudo setenforce 0    # 切到 Permissive 以收集足够日志
# 执行被拒绝的操作...
sudo ausearch -m AVC -ts recent > /tmp/avc.log

# Step 2: 生成策略模块
cat /tmp/avc.log | audit2allow -M myapp_policy
# 生成 myapp_policy.te 和 myapp_policy.pp

# Step 3: 审查生成的规则（重要！）
cat myapp_policy.te
# module myapp_policy 1.0;
# require {
#     type httpd_t;
#     type var_t;
#     class dir write;
# }
# allow httpd_t var_t:dir write;
# ⚠️ 确认这些权限是必要的！

# Step 4: 加载模块
sudo semodule -i myapp_policy.pp

# Step 5: 切回 Enforcing 并验证
sudo setenforce 1
```

### 方法2：手动编写 CIL 策略模块（RHEL 9 推荐格式）

CIL (Common Intermediate Language) 是现代 SELinux 的原生策略语言：

```bash
# 创建 CIL 模块文件
cat > myapp.cil << 'EOF'
; 自定义应用策略模块
(block myapp
  (type process)
  (type executable)
  (type config_file)
  (type data_file)
  
  ; 进程域
  (roletype system_r process)
  
  ; 进程可执行文件入口
  (typeattributeset domain (process))
  (typeattributeset entry_type (executable))
  
  ; 自动域转换：当 init_t 执行 myapp_exec_t 时 → 转到 myapp_t
  (typetransition init_t executable process)
  
  ; 允许进程读取配置文件
  (allow process config_file (file (read open getattr)))
  
  ; 允许进程读写数据文件
  (allow process data_file (file (read write open getattr create unlink)))
  
  ; 允许进程写日志
  (allow process var_log_t (file (append open)))
  
  ; 允许进程使用网络
  (allow process self (tcp_socket (connect create read write)))
  (allow process self (udp_socket (connect create read write)))
)
EOF

# 安装 CIL 模块
sudo semodule -i myapp.cil
```

### 方法3：手动编写传统 .te 策略模块

```bash
# 创建类型强制文件
cat > myapp.te << 'EOF'
policy_module(myapp, 1.0)

# 需要的类型声明
require {
    type init_t;
    type var_log_t;
    class file { read write open append create unlink getattr };
}

# 声明新类型
type myapp_t;
type myapp_exec_t;
type myapp_config_t;
type myapp_data_t;

# 域属性
domain_type(myapp_t)
domain_entry_file(myapp_t, myapp_exec_t)

# 域转换
domain_auto_transition_pattern(init_t, myapp_exec_t, myapp_t)

# 允许规则
allow myapp_t myapp_config_t:file { read open getattr };
allow myapp_t myapp_data_t:file { read write open create unlink getattr };
allow myapp_t var_log_t:file { append open };

# 网络权限
corenet_tcp_connect_generic_port(myapp_t)
EOF

# 创建文件上下文文件
cat > myapp.fc << 'EOF'
/usr/local/bin/myapp    --  gen_context(system_u:object_r:myapp_exec_t,s0)
/etc/myapp(/.*)?        --  gen_context(system_u:object_r:myapp_config_t,s0)
/var/lib/myapp(/.*)?    --  gen_context(system_u:object_r:myapp_data_t,s0)
EOF

# 编译和安装
make -f /usr/share/selinux/devel/Makefile myapp.pp
sudo semodule -i myapp.pp
sudo restorecon -Rv /usr/local/bin/myapp /etc/myapp /var/lib/myapp
```

## 6.3 管理策略模块

```bash
# 列出所有已安装模块
semodule -l

# 列出模块（带版本和启用状态）
semodule -lfull

# 安装模块
sudo semodule -i myapp.pp

# 安装 CIL 模块
sudo semodule -i myapp.cil

# 删除模块
sudo semodule -r myapp

# 启用/禁用模块
sudo semanage module --enable myapp
sudo semanage module --disable myapp

# 重建策略（修改模块后）
sudo semodule -B

# 导出本地自定义
sudo semanage export > local_customizations.txt

# 导入自定义（迁移到新系统时）
sudo semanage import < local_customizations.txt
```

## 6.4 创建自定义角色和用户域

```bash
# 创建新角色
sudo semanage user -a -R myapp_r -L s0 -r s0-s0:c0.c1023 myapp_u

# 创建 permissive 域（调试用，允许所有但记录）
sudo semanage permissive -a myapp_t

# 列出 permissive 域
sudo semanage permissive -l

# 删除 permissive 域
sudo semanage permissive -d myapp_t
```

---

# 附录：命令速查表

## 状态与模式

| 命令 | 用途 | 示例 |
|------|------|------|
| `sestatus` | 查看详细状态 | `sestatus` |
| `getenforce` | 查看当前模式 | `getenforce` |
| `setenforce` | 临时切换模式 | `setenforce 0` |
| `id -Z` | 查看当前用户上下文 | `id -Z` |
| `ps -eZ` | 查看进程上下文 | `ps -eZ \| grep httpd` |
| `ls -Z` | 查看文件上下文 | `ls -Z /var/www/html/` |

## 上下文管理

| 命令 | 用途 | 示例 |
|------|------|------|
| `semanage fcontext -a` | 添加永久文件上下文 | `semanage fcontext -a -t httpd_sys_content_t "/srv/www(/.*)?"` |
| `semanage fcontext -m` | 修改文件上下文 | `semanage fcontext -m -t httpd_sys_rw_content_t "/srv/www/uploads(/.*)?"` |
| `semanage fcontext -d` | 删除文件上下文 | `semanage fcontext -d "/srv/www/uploads(/.*)?"` |
| `semanage fcontext -l` | 列出所有文件上下文 | `semanage fcontext -l \| grep myapp` |
| `semanage fcontext -l -C` | 仅列出本地自定义 | `semanage fcontext -l -C` |
| `restorecon` | 恢复默认上下文 | `restorecon -Rv /srv/www` |
| `restorecon -F` | 强制恢复上下文 | `restorecon -Fv /srv/www` |

## 用户与角色

| 命令 | 用途 | 示例 |
|------|------|------|
| `semanage login -a` | 添加登录映射 | `semanage login -a -s staff_u dan` |
| `semanage login -l` | 列出登录映射 | `semanage login -l` |
| `semanage user -a` | 添加 SELinux 用户 | `semanage user -a -R staff_r -L s0 myadmin_u` |
| `semanage user -l` | 列出 SELinux 用户 | `semanage user -l` |
| `newrole` | 切换角色 | `newrole -r sysadm_r` |
| `runcon` | 指定上下文运行 | `runcon -t httpd_t /usr/sbin/httpd` |

## 端口与网络

| 命令 | 用途 | 示例 |
|------|------|------|
| `semanage port -a` | 添加端口标签 | `semanage port -a -t http_port_t -p tcp 84` |
| `semanage port -l` | 列出端口标签 | `semanage port -l \| grep http` |
| `semanage port -d` | 删除端口标签 | `semanage port -d -t http_port_t -p tcp 84` |

## 布尔值

| 命令 | 用途 | 示例 |
|------|------|------|
| `getsebool -a` | 列出所有布尔值 | `getsebool -a` |
| `setsebool -P` | 永久设置布尔值 | `setsebool -P httpd_can_network_connect on` |
| `semanage boolean -l` | 带说明的布尔值列表 | `semanage boolean -l \| grep httpd` |

## 日志与排错

| 命令 | 用途 | 示例 |
|------|------|------|
| `ausearch -m AVC` | 搜索 AVC 拒绝 | `ausearch -m AVC -ts recent` |
| `aureport --avc` | AVC 拒绝报告 | `aureport --avc` |
| `audit2allow` | 生成策略规则 | `ausearch -m AVC \| audit2allow -M mypolicy` |
| `sealert` | 自动诊断 | `sealert -a /var/log/audit/audit.log` |
| `seinfo` | 查询策略信息 | `seinfo -t \| grep httpd` |
| `sesearch` | 搜索策略规则 | `sesearch -A -s httpd_t` |
| `semodule` | 管理策略模块 | `semodule -i myapp.pp` |

## 策略模块

| 命令 | 用途 | 示例 |
|------|------|------|
| `semodule -i` | 安装模块 | `semodule -i myapp.pp` |
| `semodule -r` | 删除模块 | `semodule -r myapp` |
| `semodule -l` | 列出模块 | `semodule -l` |
| `semodule -B` | 重建策略 | `semodule -B` |
| `semanage module -l` | 列出模块（含状态） | `semanage module -l` |
| `semanage export` | 导出本地自定义 | `semanage export > backup.txt` |
| `semanage import` | 导入自定义 | `semanage import < backup.txt` |

---

*基于 SELinux System Administration (Sven Vermeulen, 2013) 读书笔记  
更新至 RHEL 9 / Fedora 40+ 的现代命令和实践  
2026-06-13*
