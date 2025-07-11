---
tags:
  - docker
  - volume
  - migrate
  - volume-migrate
---
volume 的迁移核心是`完整转移卷中的数据`到目的环境. (同一主机的不同卷, 不同主机, 设置不同容器引擎如docker podman之间.). 迁移的关键是避免直接操作卷的底层存储目录(docker/podman存储路径不透明), 而是通过 `备份 -- > 传输 --> 恢复` 的流程实现.  确保数据一致性和兼容性.  

### 同一主机内的卷迁移 (重命名/复制到新卷)
同一主机内, 如将old-volume 数据迁移到 new-volume.
1,  创建目的卷(new volume)

```shell
docker volume create new-volume

# with label
docker volume create --label "purpose=migrate-data" new-volume
```

2, 用临时容器复制数据
通过使用临时容器同时挂载源卷和目标卷, 将数据从源拷贝到目标卷.
```shell
docker run --rm -v old-volume:/source -v new-colume:/target alpine sh -c "cp -ra /source/.  /target && chown -R $(stat -c '%u:%g /source)  /target"

#cp -a:  保留文件的权限， 所有制，时间戳等metadata
# chown -R $(stat -c '%u:%g /source)  /target: 相同的UID/GID
```

3, 验证迁移结果
检查目标卷数据是否完整
```shell
# check new volume
docker run --rm -v new-volume:/target alpine ls -la /target/

# compare with old
docker run --rm -v old-volume:/source alpine sh -c "du -sh /source && find /source | wc -l"

docker run --rm -v new-volume:/target alpine sh -c "du -sh /target && find /target | wc -l"

```

4, 切换容器使用新卷
确认数据一致后, 修改容器启动命令, 将容器挂载卷从 old-volume 切换到  new-volume
```shell
docker stop container
docker rm container

docker run -d -v new-volume:/data --name container my-image:latest
```


5, 清理卷
```shell
docker volume rm old-volume
```

### 跨主机迁移
当需要从主机A 迁移到主机B（目标主机)，  核心是通过`备份文件`传输数据.
1,  在源主机A 备份数据
将source-volume数据打包, 并保存.
```shell
docker run --rm -v source-volume:/source -v $(pwd):/backup alpine tar -zcf /backup/volume-backup.tar.gz -C /source .
```
生成的volume-backup.tar.gz 包含源卷的所有数据及权限信息.

2,  传输备份文件到目标主机 B.
scp ./volume-backup.tar.gz  user@hostB:/tmp

3, 在目标主机 B 创建新卷并恢复数据

```shell
# create new volume
docker volume create target-colume

docker run --rm -v target-colume:/target -v /tmp:/backup alpine sh -c "tar -xvf /backup/volume-backup.tar.gz -C /target && chown -R 1000:1000 /target"

```
4, 验证目标卷数据

```shell
docker run --rm -v target-volume:/target alpine ls -la /target

docker run --rm -v target-volume:/target alpine sh -c "du -sh /target && find /target | wc -l"
```


5， 在目标主机部署容器
确认数据无误后, 就可以在目标主机启动容器.
```shell
docker run -d --name container -v target-volume:/data my-app:latest

```


### 跨容器引擎迁移(从 docker 到 podman)

docker 和podman 卷机制兼容, 命令几乎一致. 迁移方法和 `跨主机迁移` 完全相同. 核心是 数据本身迁移, 与引擎无关.


### 网络卷迁移
如果卷是基于网络存储（NFS,  Gluster FS)的外部卷,  迁移相对简单:
1, 源卷
```shell
docker volume create --driver local --opt type=nfs --opt o=addr=源NFS服务器,rw --opt device=:/path/to/nfs/share

## parameter
addr: ip address of the nfs server
device: path to the NFS export on the server 
o:  Mount options (rw=read-write, ro=read-only)

# directly mount in docker run
docker run -d --mount type=volume,source=my-nfs-vol,target=/app/data,volume-drive=local,volume-drive=local,volume-opt=type=nfs,volume-opt=device=:/path/to/nfs/share,volume-opt=o=addr=nfs_server_IP  nginx

```
2, 迁移时, 只需要在目标环境创建同名卷,  指向相同的网络存储路径.

迁移注意事项：
* 数据一致性:  迁移前需停止使用源卷的容器, 避免迁移过程中数据写入导致不一致. 
* 权限匹配: 目标卷的权限必须与容器内运行用户UID/GID 匹配, 否则可能出现 'permission denied'
* 避免直接复制存储目录: docker默认存储在/var/lib/docker/volumes/ , podman root模式下: /var/lib/containers/storage/volumes/,  路径可能因为版本/配置变化 , 直接复制可能导致元数据丢失或不兼容
* 敏感数据处理:  如卷包含密码等敏感数据,  传输时需加密.