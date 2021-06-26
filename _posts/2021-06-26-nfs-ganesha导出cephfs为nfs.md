---
layout: post
title: nfs-ganesha导出cephfs为nfs
catalog: true
tag: [Ceph]
---

<!-- TOC -->

- [1. 概述](#1-概述)
- [2. 前提条件](#2-前提条件)
- [3. 版本说明](#3-版本说明)
- [4. 安装](#4-安装)
  - [4.1. 配置yum源](#41-配置yum源)
  - [4.2. 安装软件](#42-安装软件)
- [5. 配置](#5-配置)
- [6. 使用](#6-使用)
- [7. 部署问题](#7-部署问题)
  - [7.1. nfs挂载之后无法创建文件、文件夹](#71-nfs挂载之后无法创建文件文件夹)
- [8. 剩下的问题](#8-剩下的问题)
- [9. 参考文档](#9-参考文档)

<!-- /TOC -->
# 1. 概述

cephfs直接使用不变，需要安装较多的依赖，相对来说nfs更加通用。
**FSAL_CEPH** 调用 **libcephfs2** 将 NFS 转义为 Cephfs 协议再存入到 Ceph 中，通过这种途径来实现cephfs导出为NFS

# 2. 前提条件

- 有一个cephfs集群
- 安装 `nfs-ganesha nfs-ganesha-ceph libcephfs2`
- nfs-ganesha服务需要能连接到ceph的public网络
- 安装nfs必要软件 `rpcbind` `nfs-utils`

# 3. 版本说明

- nfs-ceph 2.8 nautilus

# 4. 安装

## 4.1. 配置yum源

`cat /etc/yum.repos.d/storage.repo`

```ini
[nfsganesha]
name=nfsganesha
baseurl=https://mirrors.cloud.tencent.com/ceph/nfs-ganesha/rpm-V2.8-stable/nautilus/x86_64/
gpgcheck=0
enable=1
```

```bash
yum clean all
yum repolist
```

## 4.2. 安装软件

```bash
yum install nfs-ganesha nfs-ganesha-ceph libcephfs2 -y
```

# 5. 配置

配置文件 `/etc/ganesha/ganesha.conf`

```ini
EXPORT
{
        Export_ID=1;
        # cephfs的目录
        Path = /;
        # nfs-ganesha挂载的目录
        Pseudo = /cephfs;
        Access_Type = RW;
        protocols = 3, 4;
        transports = "UDP", "TCP";
        Squash = no_root_squash;
        FSAL {
                # 访问ceph用户对应的secretkey  
                secret_access_key = "AQDk18FgMo7NABAA4ufuz3O6/0lE4vsVgHs1yQ==";  
                # 访问ceph的用户  
                user_id = "admin";  
                name = "CEPH";  
                # cephfs 的fsname  
                filesystem = "cephfs";  
        }        
}
LOG {                         
        Facility {
                name = FILE;  
                destination = "/var/log/ganesha/ganesha.log";  
                enable = active;  
        }
}
```

启动服务

```bash
systemctl enable nfs-ganesha nfs-utils rpcbind 
systemctl start nfs-ganesha nfs-utils rpcbind
```

# 6. 使用

```bash
mkdir /opt/ganesha
mount -t nfs 172.16.81.237:/cephfs  /opt/ganesha**
```

查看挂载

```bash
df
# output
172.16.81.237:/cephfs 6576680960 96047104 6480633856    2% /opt/ganesha
```

进入挂载点就可以进行文件系统操作了

性能测试结果见， 小集群下性能差距非常小  https://docs.qq.com/doc/DR3RlaGh2ZXpqV1pT

# 7. 部署问题

## 7.1. nfs挂载之后无法创建文件、文件夹

```bash
touch  aax
touch: 无法创建"aax": 权限不够
```

解决方式参照 https://github.com/ceph/ceph-ansible/issues/5300

在 ganesha的配置中添加 `Squash = no_root_squash;` 重启服务即可

> 这个配置项是nfs的配置项，客户端使用 NFS 文件系统的账号若为 root 时，系统该如何判断这个账号的身份？预设的情况下，客户端 root 的身份会由 root_squash 的设定压缩成 nfsnobody， 如此对服务器的系统会较有保障。但如果你想要开放客户端使用 root 身份来操作服务器的文件系统，那么这里就得要开 no_root_squash 才行！

# 8. 剩下的问题

- 如何调用api使得nfs-ganesha生成配置且重新加载
  ceph中的实现是将nfs-ganesha的配置存储为ceph的对象，新建修改删除操作的都是ceph上的对象
  > 需要注意⚠️: 创建nfs-ganesha配置之后还需另外在cephfs上创建对应的目录；删除nfs-ganesha配置之后cephfs对应的目录也不会手动删除，如需清理则需另外操作删除
- nfs-ganesha如何对指定目录做quota
  早期ceph版本在nfs-ganesha配置 `client_quota = true` 即可，后来[社区](https://github.com/ceph/ceph/pull/14978)把这个参数取消了，强制使用cephfs的quota，所以设置好[cephfs的quota](https://docs.ceph.com/en/latest/cephfs/quota/)，在nfs-ganesha不用做什么即可达到目的
- nfs-ganesha如何实现高可用
  nfs-ganesha 本身是个无状态服务，当前有以下几种方式实现高可用
  - pacemaker + corosync
  - ctds + lvs
  - haproxy
  - keepalived
  - k8s service+pod

  既然可以用k8s service来实现，是否也可以通过简单的ha+keepalived来实现(待验证)

# 9. 参考文档

- [鸟哥私房菜nfs](http://cn.linux.vbird.org/linux_server/0330nfs.php)
- [nfs-ganasha](https://github.com/nfs-ganesha/nfs-ganesha/blob/next/src/config_samples/ceph.conf)

