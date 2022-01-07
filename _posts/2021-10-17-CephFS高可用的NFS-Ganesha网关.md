---
layout: post
title: CephFS高可用的NFS-Ganesha网关
catalog: true
tag: [Ceph, NFS, Linux]
---

<!-- TOC -->

- [1. 概述](#1-概述)
- [2. 术语](#2-术语)
- [3. nfs-ganesha](#3-nfs-ganesha)
  - [3.1. 介绍](#31-介绍)
  - [3.2. 架构](#32-架构)
    - [3.2.1. 总体架构图](#321-总体架构图)
    - [3.2.2. 架构说明](#322-架构说明)
    - [3.2.3. genesha-rados-cluster设计](#323-genesha-rados-cluster设计)
      - [3.2.3.1. 客户端恢复(单体情况)](#3231-客户端恢复单体情况)
      - [3.2.3.2. 宽限期(单体情况)](#3232-宽限期单体情况)
      - [3.2.3.3. Reboot Epochs](#3233-reboot-epochs)
      - [3.2.3.4. gracedb](#3234-gracedb)
      - [3.2.3.5. 集群](#3235-集群)
  - [3.3. 高可用集群实现](#33-高可用集群实现)
- [4. 部署](#4-部署)
  - [4.1. 环境说明](#41-环境说明)
  - [4.2. 安装软件](#42-安装软件)
    - [4.2.1. 配置yum源](#421-配置yum源)
    - [4.2.2. 安装](#422-安装)
  - [4.3. ganesha配置](#43-ganesha配置)
    - [4.3.1. /etc/ganesha/ganesha.conf](#431-etcganeshaganeshaconf)
    - [4.3.2. 创建export文件并上传到ceph](#432-创建export文件并上传到ceph)
    - [4.3.3. 创建第一个export目录](#433-创建第一个export目录)
    - [4.3.4. 将节点加入gracedb集群](#434-将节点加入gracedb集群)
  - [4.4. haproxy+keepalived](#44-haproxykeepalived)
    - [4.4.1. haproxy](#441-haproxy)
    - [4.4.2. keepalived](#442-keepalived)
      - [4.4.2.1. 系统配置](#4421-系统配置)
      - [4.4.2.2. 启动服务](#4422-启动服务)
- [5. 验证](#5-验证)
  - [5.1. 客户端没有io写入时服务端断网](#51-客户端没有io写入时服务端断网)
  - [5.2. 客户端高io时服务端断网超过五分钟](#52-客户端高io时服务端断网超过五分钟)
    - [5.2.1. 测试概况](#521-测试概况)
    - [5.2.2. 具体测试现象与排查](#522-具体测试现象与排查)
      - [5.2.2.1. 现象](#5221-现象)
        - [5.2.2.1.1. 客户端](#52211-客户端)
          - [5.2.2.1.1.1. 服务端恢复之后](#522111-服务端恢复之后)
          - [5.2.2.1.1.2. 挂载一个ceph-fuse客户端](#522112-挂载一个ceph-fuse客户端)
        - [5.2.2.1.2. 服务端](#52212-服务端)
          - [5.2.2.1.2.1. 系统日志](#522121-系统日志)
- [6. 附录](#6-附录)
  - [6.1. 附录1: gracedb数据结构](#61-附录1-gracedb数据结构)
- [7. 参考](#7-参考)

<!-- /TOC -->

> 涉及系统较多，概念介绍内容稍多，使用的话可以直接看部署实践，通过实践来理解各种系统和概念

# 1. 概述

NFS是linux操作系统共享文件系统使用广泛的协议，也是各种共享文件产品的标准功能，客户端安装简单、使用简单。CephFS相对来说要安装CephFS客户端，添加客户端配置等，对于用户来说NFS更方便使用。

本文旨在说明如何为CEPHFS部署一个高可用的NFS网关集群

- 为什么要给NFS做集群
  - nfs是个有状态服务，有状态服务高可用必须要保持状态的一致性，状态为
    - Open files
    - File locks
    - ...
- 当前有什么比较成熟的技术方案
  - rook是当前唯一可以找到的资料比较齐全的CephFS导出NFS的实现
    - 通过K8S的 ing svc deployment来保证服务的高可用
    - 通过ceph集群来存储文件系统的状态
- 已有的技术方案有什么技术优缺点
  - 优点: 社区支持
  - 缺点: 只能通过容器化方式ROOK来部署，生产Ceph上容器是一件极具挑战的事。不过nfs集群使用容器，ceph集群使用进程也是一种方式，但是这样增加了一些交付难度。

# 2. 术语

- nfs[^1]: 是一个有状态服务，客户端服务端之间维护如下信息
  - Open files
  - File locks

- nfs服务属性[^3]:
  - **Minimum supported version（支持的最低版本）**
  - **Grace period（宽限期)** 定义从计划外中断重新引导设备（从 15 秒到 600 秒）后所有客户机必须在多少秒内恢复锁定状态。该属性只影响 NFSv4.0 和 NFSv4.1 客户机（NFSv3 是无状态协议，因此没有要恢复的状态）。在此期间内，NFS 服务只处理旧锁定状态的回收。在宽限期结束之前，不会处理对服务的其他请求。默认宽限期为 90 秒。减小宽限期将使得 NFS 客户机在服务器重新引导之后能够更快地恢复操作，但也会增加客户机无法恢复其所有锁定状态的可能性。
  - **Enable NFSv4 delegation（启用 NFSv4 委托）** 选择此属性将允许客户机在本地缓存文件且无需联系服务器即可进行修改。此选项默认情况下启用且通常可提高性能，但在极少的情况下可能会引起问题。如果要禁用此设置，只能在对特定工作负荷进行仔细的性能测量并验证了这样设置具有相当的性能优势后进行。此选项只影响 NFSv4.0 和 NFSv4.1 挂载。
  - **Mount visibility（挂载可见性** 此属性允许您限制有关 NFS 客户机共享资源访问列表和远程挂载等信息的可用性。完全允许完全访问。受限的限制访问，比如客户机仅能查看允许其访问的共享资源。客户机看不到在服务器上定义的共享资源或服务器中其他客户机完成的远程挂载的访问列表。默认情况下，此属性设置为 "Full"（完全）。
  - **Maximum supported version（支持的最高版本）**
  - **Maximum # of server threads（最大服务器线程数）** 定义并发 NFS 请求的最大数目（从 20 到 1000）。这至少应当涵盖您预期的并发 NFS 客户机的数目。默认值是 500。

# 3. nfs-ganesha

## 3.1. 介绍

nfs-ganesha[^2]: 是一个nfs服务器，支持将不同的文件系统导通过 `FSAL(File System Abstraction Layer)` 导出为nfs , 支持以下文件系统导出为NFS。

- cephfs
- Gluster
- GPFS
- VFS
- XFS LUSTER
- RadosGW

## 3.2. 架构

### 3.2.1. 总体架构图

- 社区的架构图
  ![nfs-arch](https://raw.githubusercontent.com/wiki/nfs-ganesha/nfs-ganesha/images/nfs-arch.png)
- 更加详细的IBM的架构图[^4]

### 3.2.2. 架构说明

要使用nfs-ganesha重点关注以下几点

nfs-ganesha作为各种分布式文件系统的客户端，客户端使用文件系统时需要将目录结构(Dentry)和本地文件系统跟后端存储介质/远程文件系统的映射(inode)做缓存[^5]

- 需要导出为nfs的各种分布式文件系统
- MDCACHE: 后端文件系统的元数据缓存(Dentry Innode)
- FSAL: 需要到处的文件系统抽象层，统一了后端不同文件系统的api
- NFS: 用户使用各种分布式存储导出的nfs服务
- LOG: nfs-ganesha服务日志
  - 默认日志路径: `/var/log/ganesha/ganesha.log`
  - 其他[log配置](https://github.com/nfs-ganesha/nfs-ganesha/blob/next/src/doc/man/ganesha-log-config.rst)

### 3.2.3. genesha-rados-cluster设计

> 这里是Ceph的专场，设计文档源自官方文档的翻译[^6]

#### 3.2.3.1. 客户端恢复(单体情况)

NFSv4是基于租期的协议, 客户端与服务端建立连接之后，必须定期更新足月以维持文件系统状态(open files, locks, delegations or layouts)
当一个nfs服务重启后所有的状态丢失。当服务重新上线，客户端会检测到服务已启动，从服务端获取到上一次的连接状态恢复连接。

#### 3.2.3.2. 宽限期(单体情况)

grace period, 简言之，客户端和服务端在连接中断后恢复上一次的连接状态时，如果超时了则恢复超时(宽限期)则重新建立连接，在恢复过程中，客户端被禁获取新的状态，只允许回收状态。

#### 3.2.3.3. Reboot Epochs

服务版本

- **R** Recovery
- **N** Normal

版本状态

- **C** 当前版本，体现在grace db dump输出上为 cur
- **R** 非0值表示宽限期的有效时间，0表示结束，体现在grace db dump输出上为 rec

#### 3.2.3.4. gracedb

有状态的数据记录在数据库，rados-cluster的recovery信息存储在rados omap，通过 ganesha-rados-grace[^7]命令行可以操作数据，例如将节点加入集群，将节点踢出集群，加入集群之后节点有两个状态

- **N(NEED)** 该节点是否有需要recovery的的客户端
- **E(ENFORCING)** 强制执行宽限期

#### 3.2.3.5. 集群

上面的客户端恢复、宽限期、gracedb说明了单体服务的恢复原理和状态存储，对于集群只是将单体服务部署多个，状态保持一致

ganesha集群场景: 部署多个ganesha服务，服务端通过VIP或者DNS等方式访问其中一个，如果正在访问的ganesha服务挂了，客户端换到另一个ganesha服务进行访问(需要注意的是上次访问的状态数据也同步到了集群的其他节点的数据库中)，新的访问连接恢复了原来的状态，如果恢复不了则数据库删除原来的状态进行重建连接。

## 3.3. 高可用集群实现

- 无状态部分: 部署多个ganesha服务，通过keepalive+haproxy进行高可用负载
- 有状态部分: 状态数据通过gracedb管理，存储在ceph rados对象的omap中

以上，架构确定了，开始部署

# 4. 部署

## 4.1. 环境说明

|组件|版本|备注|
|:-:|:-:|:-:|
|操作系统|CentOS Linux release 7.8.2003 (Core)||
|操作系统内核|3.10.0-1127.el7.x86_64||
|nfs-Ganesha|2.8.1||
|ceph|ceph version 14.2.22 (ca74598065096e6fcbd8433c8779a2be0c889351) nautilus (stable)||
|haproxy|1.5.18-9||
|keepalived|1.3.5-19||
|节点数|3||

## 4.2. 安装软件

### 4.2.1. 配置yum源

```conf
cat /etc/yum.repos.d/nfs-ganasha.repo
[nfsganesha]
name=nfsganesha
baseurl=https://mirrors.cloud.tencent.com/ceph/nfs-ganesha/rpm-V2.8-stable/nautilus/x86_64/
gpgcheck=0
enable=1
```

### 4.2.2. 安装

```bash
yum install -y nfs-ganesha nfs-ganesha-ceph \
  nfs-ganesha-rados-grace nfs-ganesha-rgw \
  haproxy keepalived
```

## 4.3. ganesha配置

### 4.3.1. /etc/ganesha/ganesha.conf

> 三个节点都配置,注意修改配置文件中节点对应的配置项

```ini
NFS_CORE_PARAM {
	Enable_NLM = false;
	Enable_RQUOTA = false;
	Protocols = 4;
}

MDCACHE {
	Dir_Chunk = 0;
}

EXPORT_DEFAULTS {
	Attr_Expiration_Time = 0;
}

NFSv4 {
	Delegations = false;
	RecoveryBackend = 'rados_cluster';
	Minor_Versions = 1, 2;
}

RADOS_KV {
    # Ceph 配置文件
	ceph_conf = '/etc/ceph/ceph.conf';
    # ganesha访问ceph的用户
	userid = admin;
    # ganesha节点名称
    nodeid = "ceph01";
    # ganesha 状态存储池
	pool = "cephfs_data";
    # ganesha 状态文件存储命名空间
	namespace = "nfs-ns";
}

RADOS_URLS {
	ceph_conf = '/etc/ceph/ceph.conf';
	userid = admin;
	watch_url = 'rados://cephfs_data/nfs-ns/conf-ceph01';
}

# 通过watch ceph rados上面的ganesha配置来实现配置在线设置功能
%url	rados://cephfs_data/nfs-ns/conf-ceph01
```

### 4.3.2. 创建export文件并上传到ceph

> 集群操作 在其中一台上操作即可

```bash
# 创建文件
cat conf-ceph
%url "rados://cephfs_data/nfs-ns/export-1"
# 上传 各个节点都需要
rados put -p cephfs_data -N nfs-ns conf-ceph01 conf-ceph
rados put -p cephfs_data -N nfs-ns conf-ceph02 conf-ceph
rados put -p cephfs_data -N nfs-ns conf-ceph03 conf-ceph
```

### 4.3.3. 创建第一个export目录

cat export

```ini
EXPORT {
    FSAL {
        # ceph用户，和下面的访问权限挂钩
        user_id = "admin";
        # 上面用户对应的secret
        secret_access_key = "AQC5Z1Rh6Nu3BRAAc98ORpMCLu9kXuBh/k3oHA==";
        name = "CEPH";
        # 文件系统名称
        filesystem = "cephfs";
    }
    # 客户通过nfs访问时带的路径
    pseudo = "/test";
    # 权限
    squash = "no_root_squash";
    # 访问nfs目录的权限
    access_type = "RW";
    # cephfs路径
    path = "/test001";
    # 一般为数字
    export_id = 1;
    transports = "UDP", "TCP";
    protocols = 3, 4;
}
```

和上面配置一样需要上传到集群

```bash
rados put -p cephfs_data -N nfs-ns export-1 export
```

### 4.3.4. 将节点加入gracedb集群

有状态部分处理，将三个节点加入gracedb集群
**注意⚠️** 节点名、池名、命名空间名称注意和配置文件中的对应

```bash
ganesha-rados-grace -p cephfs_data --ns nfs-ns add ceph01 ceph02 ceph03
```

- 刚执行完命令之后状态如下

```bash
ganesha-rados-grace -p cephfs_data --ns nfs-ns
cur=5 rec=4
======================================================
ceph01	NE
ceph02	NE
ceph03	NE
```

- 启动三个ganesha服务

```bash
systemctl start nfs-ganesha
```

- 等服务启动一会儿之后再看gracedb里面的节点状态，恢复如下状态即正常

```bash
ganesha-rados-grace -p cephfs_data --ns nfs-ns
cur=5 rec=0
======================================================
ceph01
ceph02
ceph03
```

> 关于输出中的cur res 和 N E 参见nfs-ganesha章节的介绍

## 4.4. haproxy+keepalived

### 4.4.1. haproxy

```ini
global
    log         127.0.0.1 local2

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     8000
    user        haproxy
    group       haproxy
    daemon
    stats socket /var/lib/haproxy/stats
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 8000

listen stats
   bind 172.16.80.86:9000
   mode http
   stats enable
   stats uri /
   stats refresh 15s
   stats realm Haproxy\ Stats
   stats auth admin:admin

frontend nfs-in
    bind 172.16.80.244:2049
    mode tcp
    option tcplog
    default_backend             nfs-back

backend nfs-back
    balance     source
    mode        tcp
    log         /dev/log local0 debug
    server      ceph01   172.16.80.86:2049 check
    server      ceph02   172.16.80.136:2049 check
    server      ceph03   172.16.80.10:2049 check
```

### 4.4.2. keepalived

```ini
global_defs {
   router_id CEPH_NFS
}

vrrp_script check_haproxy {
    script "killall -0 haproxy"
    weight -20
    interval 2
    rise 2
    fall 2
}

vrrp_instance VI_0 {
    state BACKUP
    priority 100
    interface eth0
    virtual_router_id 51
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1234
    }
    virtual_ipaddress {
        172.16.80.244/24 dev eth0
    }
    track_script {
        check_haproxy
    }
}
```

> 注意：三个keepalived，priority(权重) 不能一样 virtual_router_id 一定一样

#### 4.4.2.1. 系统配置

> 如果不做如下操作，haproxy无法正常启动

```bash
echo "net.ipv4.ip_nonlocal_bind = 1" >> /etc/sysctl.conf
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p
```

#### 4.4.2.2. 启动服务

```bash
systemctl restart haproxy keepalived
```

> 至此， 我们搭建了一个高可用的cephfs的nfs网关，至于rgw，同理

# 5. 验证

```log
Oct  8 20:52:03 wanggangfeng-dev kernel: nfs: server 172.16.80.244 not responding, still trying
Oct  8 20:53:07 wanggangfeng-dev kernel: nfs: server 172.16.80.244 OK
```

```log
# 重新切回
cat aaa
cat: aaa: 远程 I/O 错误
# df 挂载消失 需要重新挂载
```

## 5.1. 客户端没有io写入时服务端断网

客户端会卡一段时间(约1分钟) 然后恢复

## 5.2. 客户端高io时服务端断网超过五分钟

> 后面排查过程比较长，先上结论

### 5.2.1. 测试概况

开始测试断网五分钟之后，nfs-ganesha客户端会被cephfs超时驱逐，并加入和名单，之后网络恢复，客户端一直卡着，最终通过调整参数解决该问题

在nfs-ganesha服务器上 `/etc/ceph/ceph.conf` 增加了如下配置

```ini
# mds session超时不加入黑名单
mds_session_blacklist_on_timeout = false
# mds 驱逐之后不加黑名单
mds_session_blacklist_on_evict = false
# 客户端状态如果是stale则重新建立连接
client_reconnect_stale = true
```

### 5.2.2. 具体测试现象与排查

#### 5.2.2.1. 现象

##### 5.2.2.1.1. 客户端

频繁读写时内核crash(io超时情况下的正常现象) -- Ceph集群有慢请求: 这个是因为ceph跑在虚拟机性能太差导致的，实际上切换成功在vip的服务器上使用 `netstat -anolp|grep 2049` 就能看到连接

```log
Oct  8 21:47:16 wanggangfeng-dev kernel: INFO: task dd:2053 blocked for more than 120 seconds.
Oct  8 21:47:16 wanggangfeng-dev kernel: "echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
Oct  8 21:47:16 wanggangfeng-dev kernel: dd              D ffff986436d9acc0     0  2053   2023 0x00000080
Oct  8 21:47:16 wanggangfeng-dev kernel: Call Trace:
Oct  8 21:47:16 wanggangfeng-dev kernel: [<ffffffffaab83ed0>] ? bit_wait+0x50/0x50
Oct  8 21:47:16 wanggangfeng-dev kernel: [<ffffffffaab85d89>] schedule+0x29/0x70
Oct  8 21:47:16 wanggangfeng-dev kernel: [<ffffffffaab83891>] schedule_timeout+0x221/0x2d0
Oct  8 21:47:16 wanggangfeng-dev kernel: [<ffffffffaa4e69a1>] ? put_prev_entity+0x31/0x400
Oct  8 21:47:16 wanggangfeng-dev kernel: [<ffffffffaa46d39e>] ? kvm_clock_get_cycles+0x1e/0x20
Oct  8 21:47:16 wanggangfeng-dev kernel: [<ffffffffaab83ed0>] ? bit_wait+0x50/0x50
Oct  8 21:47:16 wanggangfeng-dev kernel: [<ffffffffaab8547d>] io_schedule_timeout+0xad/0x130
Oct  8 21:47:16 wanggangfeng-dev kernel: [<ffffffffaab85518>] io_schedule+0x18/0x20
Oct  8 21:47:16 wanggangfeng-dev kernel: [<ffffffffaab83ee1>] bit_wait_io+0x11/0x50
Oct  8 21:47:16 wanggangfeng-dev kernel: [<ffffffffaab83a07>] __wait_on_bit+0x67/0x90
Oct  8 21:47:16 wanggangfeng-dev kernel: [<ffffffffaab83ed0>] ? bit_wait+0x50/0x50
Oct  8 21:47:16 wanggangfeng-dev kernel: [<ffffffffaab83b71>] out_of_line_wait_on_bit+0x81/0xb0
Oct  8 21:47:16 wanggangfeng-dev kernel: [<ffffffffaa4c7840>] ? wake_bit_function+0x40/0x40
Oct  8 21:47:16 wanggangfeng-dev kernel: [<ffffffffc0669193>] nfs_wait_on_request+0x33/0x40 [nfs]
Oct  8 21:47:16 wanggangfeng-dev kernel: [<ffffffffc066e483>] nfs_updatepage+0x153/0x8e0 [nfs]
Oct  8 21:47:16 wanggangfeng-dev kernel: [<ffffffffc065d5c1>] nfs_write_end+0x171/0x3c0 [nfs]
Oct  8 21:47:16 wanggangfeng-dev kernel: [<ffffffffaa5be044>] generic_file_buffered_write+0x164/0x270
Oct  8 21:47:16 wanggangfeng-dev kernel: [<ffffffffc06a4e90>] ? nfs4_xattr_set_nfs4_label+0x50/0x50 [nfsv4]
Oct  8 21:47:16 wanggangfeng-dev kernel: [<ffffffffc06a4e90>] ? nfs4_xattr_set_nfs4_label+0x50/0x50 [nfsv4]
Oct  8 21:47:16 wanggangfeng-dev kernel: [<ffffffffaa5c0872>] __generic_file_aio_write+0x1e2/0x400
Oct  8 21:47:16 wanggangfeng-dev kernel: [<ffffffffaa5c0ae9>] generic_file_aio_write+0x59/0xa0
Oct  8 21:47:16 wanggangfeng-dev kernel: [<ffffffffc065ca2b>] nfs_file_write+0xbb/0x1e0 [nfs]
Oct  8 21:47:16 wanggangfeng-dev kernel: [<ffffffffaa64c663>] do_sync_write+0x93/0xe0
Oct  8 21:47:16 wanggangfeng-dev kernel: [<ffffffffaa64d150>] vfs_write+0xc0/0x1f0
Oct  8 21:47:16 wanggangfeng-dev kernel: [<ffffffffaa64df1f>] SyS_write+0x7f/0xf0
Oct  8 21:47:16 wanggangfeng-dev kernel: [<ffffffffaab92ed2>] system_call_fastpath+0x25/0x2a
```

dd 进程不会恢复状态，需要手动将其终结，客户端一会儿之后可以恢复状态

客户端 

```log
Oct  8 21:54:22 wanggangfeng-dev kernel: NFS: nfs4_reclaim_open_state: unhandled error -121
```

###### 5.2.2.1.1.1. 服务端恢复之后

```log
# 重新切回
cat aaa
cat: aaa: 远程 I/O 错误
# df 挂载消失 需要重新挂载
```

ganesha日志报错 `/var/log/ganesha/ganesha.log`

```log
09/10/2021 11:13:02 : epoch 616104eb : ceph01 : ganesha.nfsd-78386[svc_8] posix2fsal_error :FSAL :CRIT :Mapping 108(default) to ERR_FSAL_SERVERFAULT
```

mds日志

```log
2021-10-09 11:05:59.220 7fc5b6e8c700  0 log_channel(cluster) log [WRN] : evicting unresponsive client ceph01 (44152), after 302.173 seconds
2021-10-09 11:05:59.220 7fc5b6e8c700  1 mds.0.12 Evicting (and blacklisting) client session 44152 (v1:172.16.80.86:0/3860306704)
2021-10-09 11:05:59.220 7fc5b6e8c700  0 log_channel(cluster) log [INF] : Evicting (and blacklisting) client session 44152 (v1:172.16.80.86:0/3860306704)
2021-10-09 11:06:04.220 7fc5b6e8c700  0 log_channel(cluster) log [WRN] : evicting unresponsive client ceph01 (49033), after 302.374 seconds
2021-10-09 11:06:04.220 7fc5b6e8c700  1 mds.0.12 Evicting (and blacklisting) client session 49033 (172.16.80.86:0/2196066904)
2021-10-09 11:06:04.220 7fc5b6e8c700  0 log_channel(cluster) log [INF] : Evicting (and blacklisting) client session 49033 (172.16.80.86:0/2196066904)
2021-10-09 11:09:47.444 7fc5bae94700  0 --2- [v2:172.16.80.10:6818/3513818921,v1:172.16.80.10:6819/3513818921] >> 172.16.80.86:0/2196066904 conn(0x5637be582800 0x5637bcbcf800 crc :-1 s=SESSION_ACCEPTING pgs=465 cs=0 l=0 rev1=1 rx=0 tx=0).handle_reconnect no existing connection exists, reseting client
2021-10-09 11:09:53.035 7fc5bae94700  0 --1- [v2:172.16.80.10:6818/3513818921,v1:172.16.80.10:6819/3513818921] >> v1:172.16.80.86:0/3860306704 conn(0x5637be529c00 0x5637bcc17000 :6819 s=ACCEPTING_WAIT_CONNECT_MSG_AUTH pgs=0 cs=0 l=0).handle_connect_message_2 accept we reset (peer sent cseq 1), sending RESETSESSION
2021-10-09 11:09:53.047 7fc5b8e90700  0 mds.0.server  ignoring msg from not-open sessionclient_reconnect(1 caps 1 realms ) v3
2021-10-09 11:09:53.047 7fc5bae94700  0 --1- [v2:172.16.80.10:6818/3513818921,v1:172.16.80.10:6819/3513818921] >> v1:172.16.80.86:0/3860306704 conn(0x5637be50dc00 0x5637bcc14800 :6819 s=OPENED pgs=26 cs=1 l=0).fault server, going to standby
```

加入黑名单

```bash
ceph osd blacklist ls
172.16.80.86:0/4234145215 2021-10-09 11:23:34.169972
```

修改黑名单配置

```ini
mds_session_blacklist_on_timeout = false
mds_session_blacklist_on_evict = false
```

配置具体说明参照 [CEPH FILE SYSTEM CLIENT EVICTION](https://docs.ceph.com/en/latest/cephfs/eviction/)

ganesha 日志

```log
15/10/2021 10:37:42 : epoch 6168e43f : ceph01 : ganesha.nfsd-359850[svc_102] rpc :TIRPC :EVENT :svc_vc_wait: 0x7f9a88000c80 fd 67 recv errno 104 (will set dead)
15/10/2021 10:37:44 : epoch 6168e43f : ceph01 : ganesha.nfsd-359850[dbus_heartbeat] nfs_health :DBUS :WARN :Health status is unhealthy. enq new: 6289, old: 6288; deq new: 6287, old: 6287
```

###### 5.2.2.1.1.2. 挂载一个ceph-fuse客户端

客户端断网等他超时

```log
x7f28740060c0 unknown :-1 s=START_CONNECT pgs=0 cs=0 l=1 rev1=0 rx=0 tx=0).stop
2021-10-15 14:21:11.445 7f28ad46b700  1 -- 172.16.80.86:0/1706554648 --> [v2:172.16.80.136:3300/0,v1:172.16.80.136:6789/0] -- mon_subscribe({config=0+,mdsmap=246+,monmap=2+}) v3 -- 0x7f28740080d0 con 0x7f2874008600
2021-10-15 14:21:11.445 7f28ad46b700  1 --2- 172.16.80.86:0/1706554648 >> [v2:172.16.80.136:3300/0,v1:172.16.80.136:6789/0] conn(0x7f2874008600 0x7f2874009770 secure :-1 s=READY pgs=149220 cs=0 l=1 rev1=1 rx=0x7f289c0080c0 tx=0x7f289c03dd20).ready entity=mon.2 client_cookie=0 server_cookie=0 in_seq=0 out_seq=0
2021-10-15 14:21:11.445 7f28a57fa700 10 client.1058179.objecter ms_handle_connect 0x7f2874008600
2021-10-15 14:21:11.445 7f28a57fa700 10 client.1058179.objecter resend_mon_ops
2021-10-15 14:21:11.446 7f28a57fa700  1 -- 172.16.80.86:0/1706554648 <== mon.2 v2:172.16.80.136:3300/0 1 ==== mon_map magic: 0 v1 ==== 422+0+0 (secure 0 0 0) 0x7f289c040310 con 0x7f2874008600
2021-10-15 14:21:11.446 7f28a57fa700  1 -- 172.16.80.86:0/1706554648 <== mon.2 v2:172.16.80.136:3300/0 2 ==== config(0 keys) v1 ==== 4+0+0 (secure 0 0 0) 0x7f289c040510 con 0x7f2874008600
2021-10-15 14:21:11.446 7f28a57fa700  1 -- 172.16.80.86:0/1706554648 <== mon.2 v2:172.16.80.136:3300/0 3 ==== mdsmap(e 251) v1 ==== 693+0+0 (secure 0 0 0) 0x7f289c040a20 con 0x7f2874008600
2021-10-15 14:21:11.446 7f28a57fa700 10 client.1058179.objecter ms_dispatch 0x560568e78330 mdsmap(e 251) v1
2021-10-15 14:21:15.317 7f28a7fff700 10 client.1058179.objecter tick
2021-10-15 14:21:15.565 7f28a6ffd700  1 -- 172.16.80.86:0/1706554648 --> [v2:172.16.80.10:6818/1475563033,v1:172.16.80.10:6819/1475563033] -- client_session(request_renewcaps seq 476) v3 -- 0x7f2884045550 con 0x560568fe8180
2021-10-15 14:21:15.893 7f28ad46b700  1 --2- 172.16.80.86:0/1706554648 >> [v2:172.16.80.10:6818/1475563033,v1:172.16.80.10:6819/1475563033] conn(0x560568fe8180 0x560568fea590 unknown :-1 s=BANNER_CONNECTING pgs=16770 cs=565 l=0 rev1=1 rx=0 tx=0)._handle_peer_banner_payload supported=1 required=0
2021-10-15 14:21:15.893 7f28ad46b700  1 --2- 172.16.80.86:0/1706554648 >> [v2:172.16.80.10:6818/1475563033,v1:172.16.80.10:6819/1475563033] conn(0x560568fe8180 0x560568fea590 crc :-1 s=SESSION_RECONNECTING pgs=16770 cs=565 l=0 rev1=1 rx=0 tx=0).handle_session_reset received session reset full=1
2021-10-15 14:21:15.893 7f28ad46b700  1 --2- 172.16.80.86:0/1706554648 >> [v2:172.16.80.10:6818/1475563033,v1:172.16.80.10:6819/1475563033] conn(0x560568fe8180 0x560568fea590 crc :-1 s=SESSION_RECONNECTING pgs=16770 cs=565 l=0 rev1=1 rx=0 tx=0).reset_session
2021-10-15 14:21:15.894 7f28a57fa700  0 client.1058179 ms_handle_remote_reset on v2:172.16.80.10:6818/1475563033
2021-10-15 14:21:15.894 7f28a57fa700 10 client.1058179.objecter _maybe_request_map subscribing (onetime) to next osd map
2021-10-15 14:21:15.894 7f28a57fa700  1 -- 172.16.80.86:0/1706554648 --> [v2:172.16.80.136:3300/0,v1:172.16.80.136:6789/0] -- mon_subscribe({osdmap=379}) v3 -- 0x7f2890007ca0 con 0x7f2874008600
2021-10-15 14:21:15.894 7f28a57fa700 10 client.1058179.objecter ms_handle_connect 0x560568fe8180
2021-10-15 14:21:15.894 7f28ad46b700  1 --2- 172.16.80.86:0/1706554648 >> [v2:172.16.80.10:6818/1475563033,v1:172.16.80.10:6819/1475563033] conn(0x560568fe8180 0x560568fea590 crc :-1 s=READY pgs=17729 cs=0 l=0 rev1=1 rx=0 tx=0).ready entity=mds.0 client_cookie=13aaa6fc82c7d050 server_cookie=a531fa96d3b029a4 in_seq=0 out_seq=0
2021-10-15 14:21:15.895 7f28a57fa700  1 -- 172.16.80.86:0/1706554648 <== mon.2 v2:172.16.80.136:3300/0 4 ==== osd_map(379..390 src has 1..390) v4 ==== 69665+0+0 (secure 0 0 0) 0x7f289c008390 con 0x7f2874008600
2021-10-15 14:21:15.895 7f28a57fa700 10 client.1058179.objecter ms_dispatch 0x560568e78330 osd_map(379..390 src has 1..390) v4
2021-10-15 14:21:15.895 7f28a57fa700  3 client.1058179.objecter handle_osd_map got epochs [379,390] > 378
2021-10-15 14:21:15.895 7f28a57fa700  3 client.1058179.objecter handle_osd_map decoding incremental epoch 379
2021-10-15 14:21:15.895 7f28a57fa700  3 client.1058179.objecter handle_osd_map decoding incremental epoch 380
2021-10-15 14:21:15.895 7f28a57fa700  3 client.1058179.objecter handle_osd_map decoding incremental epoch 381
2021-10-15 14:21:15.895 7f28a57fa700  3 client.1058179.objecter handle_osd_map decoding incremental epoch 382
2021-10-15 14:21:15.895 7f28a57fa700  3 client.1058179.objecter handle_osd_map decoding incremental epoch 383
2021-10-15 14:21:15.896 7f28a57fa700  3 client.1058179.objecter handle_osd_map decoding incremental epoch 384
2021-10-15 14:21:15.896 7f28a57fa700  3 client.1058179.objecter handle_osd_map decoding incremental epoch 385
2021-10-15 14:21:15.896 7f28a57fa700  3 client.1058179.objecter handle_osd_map decoding incremental epoch 386
2021-10-15 14:21:15.896 7f28a57fa700  3 client.1058179.objecter handle_osd_map decoding incremental epoch 387
2021-10-15 14:21:15.896 7f28a57fa700  3 client.1058179.objecter handle_osd_map decoding incremental epoch 388
2021-10-15 14:21:15.896 7f28a57fa700  3 client.1058179.objecter handle_osd_map decoding incremental epoch 389
2021-10-15 14:21:15.896 7f28a57fa700  3 client.1058179.objecter handle_osd_map decoding incremental epoch 390
2021-10-15 14:21:15.896 7f28a57fa700 20 client.1058179.objecter dump_active .. 0 homeless
2021-10-15 14:21:16.019 7f289bfff700  1 -- 172.16.80.86:0/1706554648 >> [v2:172.16.80.136:3300/0,v1:172.16.80.136:6789/0] conn(0x7f2874008600 msgr2=0x7f2874009770 secure :-1 s=STATE_CONNECTION_ESTABLISHED l=1).mark_down
```

##### 5.2.2.1.2. 服务端

###### 5.2.2.1.2.1. 系统日志

dmesg

```log
ganesha.nfsd[1400]: segfault at 7f5d8b8a19d0 ip 00007f5d9ac6ab81 sp 00007fffc2be41b0 error 4 in libpthread-2.17.so[7f5d9ac5e000+17000]
```

```log
2021-10-15 10:15:20.851 7f5d96ffd700  0 client.1008523  destroyed lost open file 0x7f5da8001a10 on 0x10000000018.head(faked_ino=0 ref=3 ll_ref=
1 cap_refs={4=0,1024=0,4096=0,8192=0} open={2=1} mode=100644 size=1571815424/3145728000 nlink=1 btime=0.000000 mtime=2021-10-15 10:15:19.624848
 ctime=2021-10-15 10:15:19.624848 caps=- objectset[0x10000000018 ts 0/0 objects 314 dirty_or_tx 0] parents=0x10000000000.head["dddccc"] 0x7f5dd
400d4a0)
```

客户端被驱逐之后状态会变成stale

```bash
{
    "id": 1094578,
    "inst": {
        "name": {
            "type": "client",
            "num": 1094578
        },
        "addr": {
            "type": "v1",
            "addr": "172.16.80.86:0",
            "nonce": 1231685619
        }
    },
    "inst_str": "client.1094578 v1:172.16.80.86:0/1231685619",
    "addr_str": "v1:172.16.80.86:0/1231685619",
    "sessions": [
        {
            "mds": 0,
            "addrs": {
                "addrvec": [
                    {
                        "type": "v2",
                        "addr": "172.16.80.10:6818",
                        "nonce": 1475563033
                    },
                    {
                        "type": "v1",
                        "addr": "172.16.80.10:6819",
                        "nonce": 1475563033
                    }
                ]
            },
            "seq": 0,
            "cap_gen": 0,
            "cap_ttl": "2021-10-15 16:28:35.900065",
            "last_cap_renew_request": "2021-10-15 16:27:35.900065",
            "cap_renew_seq": 64,
            "num_caps": 2,
            "state": "open"
        }
    ],
    "mdsmap_epoch": 251
}
```

参数 `client_reconnect_stale = true`

设置该参数之后解决问题，该参数在ceph中的使用如下(只截了一小段上下文详见`src/client/Client.cc`)，当状态为stale的连接使用该参数之后后马上进行重连，如果该参数为false则不会做其他操作。

```c++
void Client::ms_handle_remote_reset(Connection *con)
{
  ...
	case MetaSession::STATE_OPEN:
	  {
	    objecter->maybe_request_map(); /* to check if we are blacklisted */
	    const auto& conf = cct->_conf;
	    if (conf->client_reconnect_stale) {
	      ldout(cct, 1) << "reset from mds we were open; close mds session for reconnect" << dendl;
	      _closed_mds_session(s);
	    } else {
	      ldout(cct, 1) << "reset from mds we were open; mark session as stale" << dendl;
	      s->state = MetaSession::STATE_STALE;
	    }
	  }
	  break;
...
}
```

# 6. 附录

## 6.1. 附录1: gracedb数据结构

- list

```bash
rados -p myfs-data0 -N nfs-ns  ls
conf-my-nfs.a
grace
rec-0000000000000003:my-nfs.b
conf-my-nfs.b
conf-my-nfs.c
rec-0000000000000003:my-nfs.a
rec-0000000000000003:my-nfs.c
```

- grace omap

```bash
rados -p myfs-data0 -N nfs-ns  listomapvals grace
my-nfs.a
value (1 bytes) :
00000000  00                                                |.|
00000001

my-nfs.b
value (1 bytes) :
00000000  00                                                |.|
00000001

my-nfs.c
value (1 bytes) :
00000000  00                                                |.|
00000001
```

```bash
rados -p myfs-data0 -N nfs-ns  listomapvals rec-0000000000000003:my-nfs.b
7013321091993042946
value (52 bytes) :
00000000  3a 3a 66 66 66 66 3a 31  30 2e 32 34 33 2e 30 2e  |::ffff:10.243.0.|
00000010  30 2d 28 32 39 3a 4c 69  6e 75 78 20 4e 46 53 76  |0-(29:Linux NFSv|
00000020  34 2e 31 20 6b 38 73 2d  31 2e 6e 6f 76 61 6c 6f  |4.1 k8s-1.novalo|
00000030  63 61 6c 29                                       |cal)|
00000034
```

```bash
ganesha-rados-grace -p cephfs_data --ns nfs-ns --oid conf-ceph03
cur=7021712291677762853 rec=7305734900415688548
======================================================
```

- rgw-export

```bash
rados get -p cephfs_data -N nfs-ns export-2 /tmp/export-2
cat /tmp/export-2
```

```ini
EXPORT {
    FSAL {
        secret_access_key = "2222";
        user_id = "admin";
        name = "RGW";
        access_key_id = "11111";
    }

    pseudo = "/testrgw";
    squash = "no_root_squash";
    access_type = "RW";
    path = "admin-tpuoosdody";
    export_id = 2;
    transports = "UDP", "TCP";
    protocols = 3, 4;
}
```

- cephfs-export

```bash
rados get -p cephfs_data -N nfs-ns export-1 /tmp/export-1
cat /tmp/export-1
```

```ini
EXPORT {
    FSAL {
        secret_access_key = "AQC5Z1Rh6Nu3BRAAc98ORpMCLu9kXuBh/k3oHA==";
        user_id = "admin";
        name = "CEPH";
        filesystem = "cephfs";
    }

    pseudo = "/test";
    squash = "no_root_squash";
    access_type = "RW";
    path = "/test001";
    export_id = 1;
    transports = "UDP", "TCP";
    protocols = 3, 4;
}
```

# 7. 参考

- [SUSE Installation of NFS Ganesha](https://documentation.suse.com/ses/6/html/ses-all/cha-as-ganesha.html)
- [haproxy-nfs-ha, 只支持主备以及商业的aloha](https://www.haproxy.com/support/technical-notes/an-0052-en-nfs-high-availability/)
- [haproxy-nfs-ha](https://www.loadbalancer.org/blog/highly-available-shared-nfs-server/)
- [ha-nfs-cluster-quick-start-guide](https://microdevsys.com/wp/glusterfs-configuration-and-setup-w-nfs-ganesha-for-an-ha-nfs-cluster-quick-start-guide/)
- [mount hung when use nfs-ganesha+cephfs](https://github.com/nfs-ganesha/nfs-ganesha/issues/414)
- [FSAL Ceph Attr_Expiration_Time multiple MDS corrupt metadata](https://github.com/nfs-ganesha/nfs-ganesha/issues/563)
- [MDS problem slow requests, cache pressure, damaged metadata after upgrading 14.2.7 to 14.2.8](https://tracker.ceph.com/issues/44976)
- [Re: mds client reconnect](https://www.spinics.net/lists/ceph-devel/msg38759.html)
- [CEPH FILE SYSTEM CLIENT EVICTION](https://docs.ceph.com/en/latest/cephfs/eviction/)

[^1]: [Client Recovery in NFS Version 4](https://docs.oracle.com/cd/E19120-01/open.solaris/819-1634/rfsrefer-138/index.html)
[^2]: [nfs-ganesha wiki](https://github.com/nfs-ganesha/nfs-ganesha/wiki)
[^3]: [nfs 服务属性](https://docs.oracle.com/cd/E81227_01/html/E81255/gokih.html)
[^4]: [NFS-Ganesha for Clustered NAS](https://www.snia.org/sites/default/files/Poornima_NFS_GaneshaForClusteredNAS.pdf)
[^5]: [linux VFS](https://www.starlab.io/blog/introduction-to-the-linux-virtual-filesystem-vfs-part-i-a-high-level-tour)
[^6]: [ganesha-rados-cluster-design](https://github.com/nfs-ganesha/nfs-ganesha/blob/next/src/doc/man/ganesha-rados-cluster-design.rst)
[^7]: [gracedb](https://github.com/nfs-ganesha/nfs-ganesha/blob/next/src/doc/man/ganesha-rados-grace.rst)
