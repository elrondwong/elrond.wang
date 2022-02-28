---
layout: post
title: Minio集群部署
catalog: true
tag: [Minio, Ceph, 云原生]
---

<!-- TOC -->

- [1. 概述](#1-概述)
- [2. 对比](#2-对比)
  - [2.1. 软件整体对比[^1] [^2]](#21-软件整体对比1-2)
    - [2.1.1. 语言与代码量及技术栈](#211-语言与代码量及技术栈)
      - [2.1.1.1. 编译时间](#2111-编译时间)
      - [2.1.1.2. 技术栈](#2112-技术栈)
    - [2.1.2. License  [^4]](#212-license--4)
      - [2.1.2.1. LGPL](#2121-lgpl)
      - [2.1.2.2. AGPLv3](#2122-agplv3)
    - [2.1.3. 文档](#213-文档)
    - [2.1.4. 社区支持](#214-社区支持)
  - [2.2. 对象存储功能对比](#22-对象存储功能对比)
    - [2.2.1. 数据存储](#221-数据存储)
  - [2.3. 致命问题](#23-致命问题)
    - [2.3.1. 单盘极端情况下影响整个集群](#231-单盘极端情况下影响整个集群)
    - [2.3.2. 元数据过大导致静默校验卡IO](#232-元数据过大导致静默校验卡io)
    - [2.3.3. 数据平衡过程中性能损耗](#233-数据平衡过程中性能损耗)
    - [2.3.4. 单集群规模受限](#234-单集群规模受限)
    - [2.3.5. 集群联邦](#235-集群联邦)
    - [2.3.6. 数据分布算法](#236-数据分布算法)
- [3. 总结](#3-总结)
  - [3.1. Minio](#31-minio)
    - [3.1.1. 优势](#311-优势)
    - [3.1.2. 劣势](#312-劣势)
  - [3.2. Ceph](#32-ceph)
    - [3.2.1. 优势](#321-优势)
    - [3.2.2. 劣势](#322-劣势)
      - [3.2.2.1. 商业化](#3221-商业化)
- [4. 参考](#4-参考)

<!-- /TOC -->

![CephVsminio](/img/posts/MinioVSCeph/CephVsminio.jpg)

# 1. 概述

> 数据统计时间为2022年3月

Minio作为分布式存储新秀,从2016年发布第一个版本到现在短短6年时间，github start已达到31.9K, 远超2015年发布的另一款分布式存储 seaweedfs 13.9k、及2010年的Ceph 10.2k，一时风头无二；但贡献者Ceph 1172人，而Minio只有337，sweedfs只有146, 社区活跃度来讲离Ceph有不小的差距。国内生产真正大规模使用Minio的比较少见，跟当前License有不小的关系。

从三个维度来对比Ceph和Minio:

- 软件整体
- 对象存储功能
- Ceph在生产中的致命问题

# 2. 对比

## 2.1. 软件整体对比[^1] [^2]

|Project|Ceph|Minio|
|:-:|:-:|:-:|
|语言|C++|Go|
|代码量(行)|1147695|**43818**|
|License|**LGPL**|AGPLv3|
|提供的API|librados (C, C++, Python, Ruby, PHP, C#, Perl), S3, Swift, FUSE|AWS S3 API (Go, Java, Python, .NET, Javascript, Haskell...)|
|高可用|Yes|Yes|
|Shared|Yes|Yes|
|数据冗余方式|**副本/纠删码**|纠删码|
|纠删码算法算法|Pluggable erasure codes(RS/LRC/SHEC)|Reed-Solomon|
|纠删码作用单位|Pool|Object|
|初始版本发布时间|2010|2014|
|部署内存需求|1 per 1TB of storage|**无限制**|
|部署方式|k8s operator(rook)/裸机(ansible/cephadm/ceph-volume/binary)|k8s operator/裸机(docker-compose/ansible/binary/)|
|平台支持|Linux/~~Windows~~/~~MacOS~~|Linux/Windows/MacOS|
|K8S存储支持|CSI/COSI(NAS)|DirectPV(DAS)[^3]|
|数据一致性|强|强|
|文档支持|强|弱|
|社区支持|邮件列表|**slack**|

### 2.1.1. 语言与代码量及技术栈

#### 2.1.1.1. 编译时间

- Ceph 4C8G 编译一次2h30min， 变异过程以及产物需要占用30-50G空间
- Minio 预估下应该在分钟级别(实际没有编译，根据代码量和系统调用预估), 产物minio 106MB

#### 2.1.1.2. 技术栈

- Ceph使用C++开发，功能多、性能好、但代码复杂，功能繁多、涉及技术栈太多
  - C++ / C / Python / Shell
  - S3
  - KV数据库
  - 网络通信
  - 一致性算法 paxos
  - 数据分布算法
    - hash
    - crush
  - 数据冗余算法
    - 副本
    - 纠删码
  - 硬件
    - 服务器
    - 网络
    - Raid
    - 硬盘
    - 交换机
    - ...
  - Linux IO栈
    - VFS
    - 文件系统
    - 块层
    - 驱动层
    - 设备层
  - ...

- Minio 使用Go开发，代码精简易读
  - Go / Shell
  - S3
  - 数据冗余算法
    - 纠删码
  - Linux文件系统
  - ...

### 2.1.2. License  [^4]

最初是Apache License 2.0，商业友好

#### 2.1.2.1. LGPL

修改库代码之后必须开源且协议为LGPL，主要针对库类使用，调用不受限制

#### 2.1.2.2. AGPLv3

只要调用到，调用项目项目协议需要使用AGPL，并且对使用该软件的用户提供源码，软件本身提供了商业授权。不能基于该软件做开发或者对软件本身进行修改而用于商用

### 2.1.3. 文档

Ceph各类架构文档、维护文档齐全，Minio文档缺失严重，例如纠删码块是怎么分布以及组装的、数据如何落盘的流程等等

### 2.1.4. 社区支持

Ceph主要使用邮件列表，Minio使用slack，slack实时性更强、效率更高，设置直接可以对其他人进行实时会话

## 2.2. 对象存储功能对比

|项目|Ceph|Minio|
|:-:|:-:|:-:|
|容灾备份|AA/AP|AA/AP|
|认证/访问控制|支持插件(Active Directory/OpenLDAP/OpenID)/IAM|支持插件(KEYCLOAK/facebook/Google/Okata/Active Directory/OpenLDAP)/IAM|
|TLS加密|Yes|Yes|
|客户端加密|Yes|Yes|
|服务端加密|Yes|Yes|
|对象锁定(WORM)[^5]|Yes(nautilus)[^6]|Yes|
|版本管理|Yes|Yes|
|生命周期管理|Yes|Yes|
|数据分层|Yes|Yes|
|管理接口|Dashboard/API/CLI|Console/API/CLI|
|监控告警|Prometheus|Prometheus|
|事件通知|Yes|Yes|
|可扩展性|Yes|Yes|
|AWS兼容|Yes(不完全兼容)|**Yes**|
|S3 Select|Yes|Yes|
|云网关模式|无|**Google Azure/NAS/S3/HDFS**|
|导出为文件系统|**nfs**|minfs[^7]|
|数据存储|(bluestore)/Linux文件系统(filestore)|linux文件系统|
|元数据存储|kv数据库/(rockdb/leveldb/other)|linux文件系统|

### 2.2.1. 数据存储

- Ceph: 数据存储在filestore/bluestore 元数据存储在kv数据库(filestore元数据一部分存储在xfs拓展属性)
- Minio: 数据存储在linux本地目录, 小文件数据和元数据都存在元数据文件里面`xl.meta`纠删码分布存储, 大文件数据与与元数据分开存储， 没有集中的元数据服务器，理论上集群规模不受中心化有状态组件的影响，可以无限横向扩容，带来的问题是每次io都要从磁盘上拉取到元数据计算组装，然后再操作，好处是对象存储没有修改操作，计算压力就少了很多

## 2.3. 致命问题

|项目|Ceph|Minio|
|:-:|:-:|:-:|
|单盘极端情况下影响整集群|Yes|Yes|
|元数据过大导致数据IO卡住|Yes|No(元数据在本地磁盘一个块一个元数据)|
|扩缩容性能损耗|High|No(Server Pool)、单Set会有重平衡的问题|
|单集群规模受限|单盘性能(水桶效应)|**横向无限扩容(server pool) [^8]|
|集群联邦|No|No[^9](集群联邦功能已弃用)|

### 2.3.1. 单盘极端情况下影响整个集群

当分布式存储系统中一块磁盘时延忽高忽低时，无法触发集群故障处理机制，导致落到这快盘的io在时延高的情况下卡住，这类问题在架构复杂的系统不好定位，造成影响也比较大，Ceph和Minio都存在这样的问题。

### 2.3.2. 元数据过大导致静默校验卡IO

- Ceph的元数据存储在KV数据库，对象存储对象寻址是通过bucket index，如果一个bucket里面文件过多，这个index会非常大，例如有5000w个对象时，没有分片情况下这个index的大小会达到40G，静默校验过不去导致整个集群的IO卡住，且处理需要花费小时级的事件

- Minio元数据以Linux文件的方式存储，每个数据块一个元数据文件，一般情况元数据不会太大，当启用多版本的时候，极端情况，元数据每个版本追加一条，追加到几十万几百万的量级也是会影响到正常IO的，这种情况在正常的使用中不会见到

### 2.3.3. 数据平衡过程中性能损耗

- Ceph官方没有支持多集群，当新加入/删除一个磁盘时，会造成整个pool的数据平衡，平衡期间占用磁盘IO非常大，大规模场景经常因为添加/删除OSD造成满请求，导致业务卡住
- Minio使用Server Pool(类似Ceph的不同Pool使用不同服务器思路)，尽量避免数据均衡，不同Server pool不涉及数据平衡，同一个Pool中的磁盘如果有新增/损坏，就需要纠删码重新计算，数据重新分布，耗费的计算资源非常高，所以不建议在Server pool里面新增/删除磁盘

### 2.3.4. 单集群规模受限

- Ceph 单集群规模受限于单盘的性能，当某一块盘性能差的时候，整个集群的性能会被拖下来，和但集群极端情况下影响整个集群的问题差不多
- Ceph中每个IO请求几乎都要和mon交互，而mon是有状态的，多台mon同时只有一台在进行写操作，当规模特别大或者集群异常的时候，mon处理不过来会导致集群瘫痪
- Minio没有中心化的管控节点，横向扩容只需考虑请求转发和数据平衡，Http请求天然支持负载分发，所以这块不构成问题, 数据平衡上，见数据平衡过程中性能损耗

### 2.3.5. 集群联邦

- Ceph 不支持
- Minio 弃用，现在都使用 Server Pool

### 2.3.6. 数据分布算法

# 3. 总结

## 3.1. Minio

### 3.1.1. 优势

- 短小精悍: 麻雀虽小，五脏俱全，商业化做的非常出色
- 简单方便: 使用Go开发，主要操作对象是文件系统的文件，二次开发门槛低，维护成本低
- 顺势而生: 云原生场景存储的需求方是各Sass应用，主要需求有如下集中
  - 简单 http协议搞定所有交互，部署方便，启动一个二进制即部署完成
  - 轻量 硬件要求低、虚拟机也能部署，主要是为了满足对性能要求不高，但需要有稳定共享存储功能的
  - 便携 不只在数据中心可以部署，在边缘端也可以部署, 也有类似CDN的便端方案[^10]
  - 功能单一但适用度广 S3可以满足AI、区块链、大数据等热门技术的存储需求
- UI友好: 大部分功能在console即可完成，无需太多的后台操作
- 按需订阅: 提供了免费的社区版与收费的订阅服务，解决了重要数据兜底的问题
- 网关模式: 支持多种存储上面跑minio, Azure Blob/NAS/S3/HDFS，将不同的共享文件形式转化为minio提供使用，省去自行实现适配器的工作

### 3.1.2. 劣势

- 文档支持比较差: 架构文档比较少
- 不支持副本模式: 对于数据安全性要求高的需求没法满足
- 性能差: 数据落在本地文件系统, IO链路比较长
- License限制严格: 无法进行商业化的二次开发或者基于Minio做开发

## 3.2. Ceph

### 3.2.1. 优势

- 兼容并包，大而全: 统一存储，集块设备、对象存储、文件系统于一体
- 生态强大: OpenStack、K8S集成
- 性能强: IOPS高
- License比较宽松: 调用Ceph不受限制
- 冗余策略丰富: 支持副本和纠删码插件(当前支持三种)
- 文档齐全
- 社区支持好
- 监控稍完善 -- 硬件健康检查(功能极不完善)、prometheus+grafana集成...

### 3.2.2. 劣势

- 重: 部署需要的硬件资源较多，例如故障场景下，单OSD内存占用会达到8-10G
- 维护难度大: 设计技术栈较多，大量使用的话需要一个专业的存储团队才能支撑
- 使用难度大: 没有针对资源使用者的友好UI，用户只能通过查阅文档/代码使用
- 理想架构与实际环境的矛盾:
  - 对于集群规模与单盘性能的矛盾，暂时没有比较好的解决方案
  - 大规模使用依赖于大量系统参数和ceph参数的调优
  - 块存储如果给虚拟机的操作系统使用，会因为单块盘故障引起大量虚拟机不可预估的宕机，非重启不能解决，即使恢复之后服务器仍可能有"暗病"
- 目前没有比较好的解决方案来支撑云原生场景(轻量、简单)
  - rook 加大了维护难度
    - [容器化非容器化之争邮件列表](https://lists.ceph.io/hyperkitty/list/ceph-users@ceph.io/thread/TTTYKRVWJOR7LOQ3UCQAZQR32R7YADVY/)

#### 3.2.2.1. 商业化

同样的一个功能

- Minio

![Minio对象锁定功能介绍](/img/posts/MinioVSCeph/Minio对象锁定功能介绍.png)

- Ceph

![Ceph对象锁定功能介绍](/img/posts/MinioVSCeph/Ceph对象锁定功能介绍.png)

> 存储系统主要需要关注的是拓展性、易维护性，指标可观测、事件可自动告警
> 对一个系统的了解必须是在大规模、高负载、有过整机房掉电损坏数据之后才能提出深入的、准确的评估，对minio使用较少，可能一些对比与实际表现存在偏差

**只使用对象存储功能，且不做调用开发和二次开发，Minio好用不折腾。**
**如果有Iaas平台、块存储、文件系统需求，需要做商业化的存储产品、团队处于初期，则Ceph无可比拟。**

# 4. 参考

[^1]:https://en.wikipedia.org/wiki/Comparison_of_distributed_file_systems
[^2]:https://github.com/orgs/minio/repositories?page=1&type=all
[^3]:https://github.com/minio/directpv?utm_campaign=COSI&utm_source=minio&utm_medium=blog&utm_content=cosi-091621
[^4]:https://choosealicense.com/
[^5]:https://docs.amazonaws.cn/AmazonS3/latest/userguide/object-lock-overview.html#object-lock-bucket-config
[^6]:https://tracker.ceph.com/issues/37763
[^7]:https://github.com/minio/minfs
[^8]:https://docs.min.io/minio/baremetal/installation/expand-minio-distributed.html#install-the-minio-binary-on-each-node-in-the-new-server-pool
[^9]:https://docs.min.io/docs/minio-federation-quickstart-guide.html
[^10]:https://blog.min.io/the-architects-guide-to-edge-storage/
