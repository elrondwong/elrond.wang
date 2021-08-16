---
layout: post
title: CephFS subvolume
catalog: true
tag: [Ceph]
---


<!-- TOC -->

- [1. 卷基本操作](#1-卷基本操作)
- [2. 卷信息](#2-卷信息)
- [3. 卷权限设置](#3-卷权限设置)
	- [3.1. cephx的mds cap](#31-cephx的mds-cap)
	- [3.2. 创建用户并应用于指定路径](#32-创建用户并应用于指定路径)
	- [3.3. 挂载测试](#33-挂载测试)
		- [3.3.1. 正向测试](#331-正向测试)
		- [3.3.2. 反向测试](#332-反向测试)
			- [3.3.2.1. 挂载一个非rw授权的目录，例如 `/volumes/group1/volume1/`](#3321-挂载一个非rw授权的目录例如-volumesgroup1volume1)
			- [3.3.2.2. 只对指定目录授权](#3322-只对指定目录授权)
- [4. 总结](#4-总结)
	- [4.1. 好用的点](#41-好用的点)
- [5. 参考文档](#5-参考文档)

<!-- /TOC -->
# 1. 卷基本操作

- 卷创建
每个cephfs创建之后会自动创建一个文件系统同名卷,一个文件系统就是一个卷，N版多文件系统功能还在试验性阶段，暂时不考虑使用
> 这个集群为cephfs

- 卷组创建

> 创建卷组可以指定后端的data_pool,和用户uid gid mode等
> 指定data_pool生产环境使用不太好，考虑隔离性可以再新建个集群；考虑多租户，如果池太多导致pg个数过多，mon处理巨量pg压力过大，不利于集群维护。

```bash
ceph fs subvolumegroup create cephfs group1
```

- 子卷创建

> 可以指定后端的data_pool，用户uid gid mod 、rados namespace等等

```bash
# 大小为100G
ceph fs subvolume create cephfs volume1  100000000000 --group-name group1
```

# 2. 卷信息

- 卷

```bash
ceph fs volume ls
# output
[
    {
        "name": "cephfs"
    }
]
```

- 卷组

```bash
ceph fs subvolumegroup ls cephfs
# output
[
    {
        "name": "csi"
    }
]
```

- 子卷

```bash
ceph fs subvolume ls cephfs csi
[
    {
        "name": "csi-vol-0af68b6a-cfff-11eb-b335-fad30d8a412c"
    }
]
```

- 子卷详情

```bash
ceph fs subvolume info cephfs volume1 group1
# output
{
    "atime": "2021-07-02 11:06:49",
    "bytes_pcent": "0.00",
    "bytes_quota": 100000000000,
    "bytes_used": 0,
    "created_at": "2021-07-02 11:06:49",
    "ctime": "2021-07-02 11:06:49",
    "data_pool": "cephfs_data",
    "features": [
        "snapshot-clone",
        "snapshot-autoprotect",
        "snapshot-retention"
    ],
    "gid": 0,
    "mode": 16877,
    "mon_addrs": [
        "10.10.10.237:6789",
        "10.10.10.238:6789",
        "10.10.10.239:6789"
    ],
    "mtime": "2021-07-02 11:06:49",
    "path": "/volumes/group1/volume1/c78865f8-6196-45de-bdb7-e61c1fca0963",
    "pool_namespace": "",
    "state": "complete",
    "type": "subvolume",
    "uid": 0
}
```

- 卷组挂载路径

```bash
ceph fs subvolumegroup getpath cephfs csi
# output
/volumes/csi
```

- 子卷挂载路径

```bash
ceph fs subvolume getpath cephfs csi-vol-0af68b6a-cfff-11eb-b335-fad30d8a412c csi
# output
/volumes/csi/csi-vol-0af68b6a-cfff-11eb-b335-fad30d8a412c/c721c454-23d6-4d73-8a4f-6d051e322487
```

# 3. 卷权限设置

> 卷权限设置使用cephx主要针对mds cephfs路径进行限制，可以指定用户对指定用户拥有读写权限

## 3.1. cephx的mds cap

[CEPHFS CLIENT CAPABILITIES](https://docs.ceph.com/en/latest/cephfs/client-auth/)

通过cephx的mds cap可以设置

- 文件夹访问控制

rw标识

```bash
client.foo
  key: *key*
  caps: [mds] allow r, allow rw path=/bar
  caps: [mon] allow r
  caps: [osd] allow rw tag cephfs data=cephfs_a
```

- 文件布局和quota功能限制

(文件夹布局)[https://docs.ceph.com/en/latest/cephfs/file-layouts/]

p标识

```bash
client.0
    key: AQAz7EVWygILFRAAdIcuJ12opU/JKyfFmxhuaw==
    caps: [mds] allow rwp
    caps: [mon] allow r
    caps: [osd] allow rw tag cephfs data=cephfs_a

```

- 网络白名单限制

```bash
client.foo
  key: *key*
  caps: [mds] allow r network 10.0.0.0/8, allow rw path=/bar network 10.0.0.0/8
  caps: [mon] allow r network 10.0.0.0/8
  caps: [osd] allow rw tag cephfs data=cephfs_a network 10.0.0.0/8
```

- 文件系统限制

多个文件共存时只允许看到指定的文件系统
mds fsname
mon fsname
osd data

```bash
[client.someuser]
    key = AQAmthpf89M+JhAAiHDYQkMiCq3x+J0n9e8REQ==
    caps mds = "allow rw fsname=cephfs"
    caps mon = "allow r fsname=cephfs"
    caps osd = "allow rw tag cephfs data=cephfs"
```

- mds连接限制

多个文件系统共存的时候，只能访问用户权限绑定的文件系统
mds fsname
osd data

```bash
[client.someuser]
    key = AQBPSARfg8hCJRAAEegIxjlm7VkHuiuntm6wsA==
    caps mds = "allow rw fsname=cephfs"
    caps mon = "allow r"
    caps osd = "allow rw tag cephfs data=cephfs"
```

- root squash

> no_root_squash：登入 NFS 主机使用分享目录的使用者，如果是 root 的话，那么对于这个分享的目录来说，他就具有 root 的权限
> root_squash：在登入 NFS 主机使用分享之目录的使用者如果是 root 时，那么这个使用者的权限将被压缩成为匿名使用者，通常他的 UID 与 GID 都会变成 nobody 的身份

root_squash标识

```bash
[client.test_a]
    key = AQBZcDpfEbEUKxAADk14VflBXt71rL9D966mYA==
    caps mds = "allow rw fsname=a root_squash, allow rw fsname=a path=/volumes"
    caps mon = "allow r fsname=a"
    caps osd = "allow rw tag cephfs data=a"
```

## 3.2. 创建用户并应用于指定路径


```bash
# ceph fs authorize <filesystem> <user entity> <poth> <mode> <path> <mode> ...
# 设置根目录为只读，/volumes/group1/volume1/c78865f8-6196-45de-bdb7-e61c1fca0963目录读写
# /volumes/group1/volume1/c78865f8-6196-45de-bdb7-e61c1fca0963 为我们上面创建的子卷
ceph fs authorize  cephfs client.user1 / r /volumes/group1/volume1/c78865f8-6196-45de-bdb7-e61c1fca0963 rw
[client.user1]
    key = AQCrhd5g9Tu/GRAAJ5+wMjiwcnD7AQRlxfJdPA==
# 查看用户详情
ceph auth get client.user1
exported keyring for client.user1
[client.user1]
    key = AQCrhd5g9Tu/GRAAJ5+wMjiwcnD7AQRlxfJdPA==
    caps mds = "allow r, allow rw path=/volumes/group1/volume1/c78865f8-6196-45de-bdb7-e61c1fca0963"
    caps mon = "allow r"
    caps osd = "allow rw tag cephfs data=cephfs"
```

## 3.3. 挂载测试

### 3.3.1. 正向测试

- 获取keyring到本地

*ceph.conf 也需要获取到/etc/ceph/目录*

```bash
ceph auth get client.user1-o /etc/ceph/ceph.client.user1.keyring
```

- 挂载

```bash
ceph-fuse -n client.user1 -r /volumes/group1/volume1/c78865f8-6196-45de-bdb7-e61c1fca0963 /opt/data1
```

- 查看挂载

```bash
df -hT
# output
# 可以看到目录大小也是创建时设置的100G左右
ceph-fuse                                                  fuse.ceph-fuse   94G     0   94G    0% /opt/data1
```

- 获取配额

```bash
getfattr -n ceph.quota.max_bytes /opt/data1
# 实际上上面的容量也是通过quota来实现的
getfattr: Removing leading '/' from absolute path names
# file: opt/data1
ceph.quota.max_bytes="100000000000"
```

### 3.3.2. 反向测试

#### 3.3.2.1. 挂载一个非rw授权的目录，例如 `/volumes/group1/volume1/`

```bash
ceph-fuse -n client.user1 -r /volumes/group1/volume1 /opt/data2
```

挂载成功，但不能写数据,但可以读取

```bash
touch xxx
touch: 无法创建"xxx": 权限不够
```
#### 3.3.2.2. 只对指定目录授权

- 创建用户绑定子卷

```bash
# 相比上文授权，去除对根目录只读授权
ceph fs authorize  cephfs client.user2 /volumes/group1/volume1/c78865f8-6196-45de-bdb7-e61c1fca0963 rw
```

- 挂载子卷

```bash
ceph-fuse -n client.user1 -r /volumes/group1/volume1/c78865f8-6196-45de-bdb7-e61c1fca0963 /opt/data2
touch /opt/data2/w_op_test
# 写数据成功
```

- 挂载其他文件夹

```bash
ceph-fuse -n client.user2 -r /volumes/group1/volume1/ /opt/data2
ceph-fuse[1895408]: starting ceph client
2021-07-02 11:46:04.867 7f6e657bbf80 -1 init, newargv = 0x55c51e0db140 newargc=9
2021-07-02 11:46:04.873 7f6e3f7fe700 -1 client.418593 mds.0 rejected us (non-allowable root '/volumes/group1/volume1/')
ceph-fuse[1895408]: ceph mount failed with (1) Operation not permitted
```

其他文件夹不可挂载

# 4. 总结

子卷的实现是对文件夹的抽象，直观体现在以下几方面
- 文件夹配额使用
- mds path权限限制
- 目录层级自动管理

## 4.1. 好用的点

- 多租户情况下不用考虑租户用户文件夹对应关系
  - 租户 -- subvolumegroup
  - 用户 -- subvolume

总体来看比较鸡肋

# 5. 参考文档

- [FS VOLUMES AND SUBVOLUMES](https://docs.ceph.com/en/latest/cephfs/fs-volumes/)
- [CEPHFS CLIENT CAPABILITIES](https://docs.ceph.com/en/latest/cephfs/client-auth/)
- [FILE LAYOUTS](https://docs.ceph.com/en/latest/cephfs/file-layouts/)