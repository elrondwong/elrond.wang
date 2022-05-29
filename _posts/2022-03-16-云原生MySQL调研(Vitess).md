---
layout: post
title: 云原生数据库调研(vitess)
catalog: true
tag: [MySQL, 云原生]
---
<!-- TOC -->

- [1. 背景](#1-背景)
  - [1.1. License](#11-license)
  - [1.2. vitess 知人论事](#12-vitess-知人论事)
  - [1.3. 文档概述](#13-文档概述)
- [2. 版本说明](#2-版本说明)
- [3. 架构](#3-架构)
  - [3.1. 组件架构](#31-组件架构)
  - [3.2. 逻辑架构](#32-逻辑架构)
  - [3.3. 典型场景说明](#33-典型场景说明)
    - [3.3.1. 概念说明](#331-概念说明)
    - [3.3.2. 场景说明](#332-场景说明)
    - [3.3.3. mysql使用](#333-mysql使用)
    - [3.3.4. 用户和权限管理](#334-用户和权限管理)
    - [3.3.5. 数据库配置](#335-数据库配置)
    - [3.3.6. 备份与恢复](#336-备份与恢复)
      - [3.3.6.1. 备份后端](#3361-备份后端)
      - [3.3.6.2. 备份方式](#3362-备份方式)
      - [3.3.6.3. 操作](#3363-操作)
        - [3.3.6.3.1. s3](#33631-s3)
        - [3.3.6.3.2. ceph rgw](#33632-ceph-rgw)
    - [3.3.7. 监控与维护](#337-监控与维护)
    - [3.3.8. 数据迁移](#338-数据迁移)
  - [3.4. 安装部署](#34-安装部署)
  - [3.5. 注意点](#35-注意点)
    - [3.5.1. 操作慢一点](#351-操作慢一点)
- [4. 附录](#4-附录)
  - [4.1. 水平分片表现](#41-水平分片表现)
  - [4.2. reshard 报错](#42-reshard-报错)
  - [4.3. 水平的两个分片的数据](#43-水平的两个分片的数据)
  - [4.4. 分片与数据库对应关系](#44-分片与数据库对应关系)
  - [4.5. 元数据存储](#45-元数据存储)
  - [4.6. 后端存储层实际的数据库结构](#46-后端存储层实际的数据库结构)
  - [4.7. 备份失败](#47-备份失败)
- [5. 选型总结](#5-选型总结)
  - [5.1. 优势](#51-优势)
  - [5.2. 劣势](#52-劣势)
  - [5.3. 总结](#53-总结)
- [6. 参考](#6-参考)

<!-- /TOC -->

# 1. 背景

MySQL是业务系统不可或缺的组件。以进程为单位部署MySQL，通过额外安装各种监控组件维护MySQL的方式，维护难度大、标准化低、成本高，一千个团队就有一千种MySQL的部署+维护方式。借助云原生理念和已有的开源软件可以在很大程度上解决上述问题，当前社区相关度比较高的有如下几个项目:

- [Vitess](https://github.com/vitessio/vitess)
- [mysql官方的operator](https://github.com/mysql/mysql-operator) -- 月久失修，当前只有预览版
- [bitpoke的operator](https://github.com/bitpoke/mysql-operator) -- 维护不够勤快
- [percona-xtradb-cluster-operator](https://github.com/percona/percona-xtradb-cluster-operator)
- [RadonDB](https://github.com/radondb/radon#:~:text=RadonDB%20is%20a%20cloud%2Dnative, engine%20for%20trusted%20data%20reliability.) -- 年久失修
- [yugabyte-db](https://github.com/yugabyte/yugabyte-db) -- PostgreSQL

## 1.1. License

- vitess - Apache-2.0
- bitpoke/mysql-operator - Apache-2.0
- percona-xtradb-cluster-operator - Apache-2.0

## 1.2. vitess 知人论事

生于Youtube: YouTube为了解决MySQL可扩展性问题，2010年内部诞生了Vitess
开源于CNCF: 2018年2月成为CNCF孵化项目，2019年11月毕业

## 1.3. 文档概述

上面几个项目中活跃的只有两个，vitess和yugabyte，但yugabyte是PgSQL所以只有一个vitess可用了。vitess概念相关，前人之述备矣，我们主要聚焦如下几点以下:

- 架构落到用户可见层面是怎么样的
- 集群创建
- 对外提供服务，包括对K8S集群内以及集群外的传统应用
- 监控与维护
- 数据备份
- 优劣势

# 2. 版本说明

|组件名称|版本|备注|
|:-:|:-:|:-:|
|k8s|v1.20.4||
|vitess|v13.0.0|

# 3. 架构

## 3.1. 组件架构

在数据库上加了一个逻辑层，逻辑层的状态存储在etcd，物理层则可以无限度切小，物理层的单元为实际的数据库实例，这样瓶颈主要在etcd上面，而不是数据库本身，所以理论上无限横向扩展。
显而易见的缺点是数据库高可用使用了原生的master/slave模式，导致了数据的一致性没有办法得到保证，对于金融等行业使用需慎重。

![architecture](https://vitess.io/docs/13.0/overview/img/architecture.svg)

架构对应到实际的k8S的组件上[CRD部署模式](https://github.com/planetscale/vitess-operator)

```bash
# crd
kubectl get crd |grep vites
vitessbackups.planetscale.com                         2022-03-08T03:47:12Z
vitessbackupstorages.planetscale.com                  2022-03-08T03:47:12Z
vitesscells.planetscale.com                           2022-03-08T03:47:12Z
vitessclusters.planetscale.com                        2022-03-08T03:47:12Z
vitesskeyspaces.planetscale.com                       2022-03-08T03:47:12Z
vitessshards.planetscale.com                          2022-03-08T03:47:12Z
# deployment
kubectl get deployments.apps
NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
example-zone1-vtctld-1d4dcad0   1/1     1            1           4d21h
example-zone1-vtgate-bc6cde92   1/1     1            1           4d21h
vitess-operator                 1/1     1            1           6d2h
# pod
kubectl get po
NAME                                             READY   STATUS    RESTARTS   AGE
example-etcd-faf13de3-1                          1/1     Running   0          41h
example-etcd-faf13de3-2                          1/1     Running   0          41h
example-etcd-faf13de3-3                          1/1     Running   0          41h
example-vttablet-zone1-0118374573-10d08e80       3/3     Running   0          41h
example-vttablet-zone1-0120139806-fed29577       3/3     Running   0          41h
example-vttablet-zone1-2289928654-7de47379       3/3     Running   0          41h
example-vttablet-zone1-2469782763-bfadd780       3/3     Running   0          41h
example-vttablet-zone1-2548885007-46a852d0       3/3     Running   0          41h
example-vttablet-zone1-4277914223-0f04a9a6       3/3     Running   0          41h
example-zone1-vtctld-1d4dcad0-64668cccc8-lgsrg   1/1     Running   2          41h
example-zone1-vtgate-bc6cde92-7565589466-kxzfn   1/1     Running   2          41h
vitess-operator-7794c74b9b-4pzfl                 1/1     Running   0          2d23h
# secret
example-cluster-config        Opaque                                2      4d22h
vitess-operator-token-xr7gg   kubernetes.io/service-account-token   3      6d2h
```

- etcd 对应架构图上的Toploy组件中的元数据存储
- vttablet 对应一个实际的数据库pod，有三个contanier: mysqld, vttablet, mysqld-exporter,分别是存储数据、存储层代理、监控数据
- vtctld 管理接口
- vtgate 处理用户的MySQL请求，把逻辑请求转化成存储层的真实请求，并把请求到的内容汇聚反馈给客户

## 3.2. 逻辑架构

![逻辑架构](https://storage.googleapis.com/cdn.thenewstack.io/media/2018/02/c54187e7-vitess-architecture.png)
![逻辑架构2](/img/posts/云原生数据库调研(vitess)/逻辑架构2.png)

## 3.3. 典型场景说明

### 3.3.1. 概念说明

说明之前有些几个关键概念，这里简要说一下，更详细的可参考:
[官方文档](https://vitess.io/docs/13.0/concepts/)
[中文个人博客](http://masikkk.com/article/Vitess/)

- 集群: 一个Vitess集群
- Cell: 故障域、资源隔离域
- KeySpace: 逻辑数据库，一个KeySpace对应一个逻辑数据库
- Shard: 数据库分片
  - 垂直分片、水平分片都可以做到数据无感知，分片逻辑由Vitess完成
- 用户面可见的MySQL: 即KeySpace
- Vitess存储面的MySQL: 真实的数据库实例，如上面k8s实际组件中的vttablet,每一个pod实际上都是一个数据库实例(注意⚠️:这里的实例是数据库进程不是库)，实际的MySQL实例按照原生的Master/Slave模式达到高可用，这也造成了Vitess数据一致性问题。
- Workflows 可以理解为vitess针对数据库、数据表等等的定时任务/异步任务管理器

一个集群可以有多个Cell，一个Cell可以有多个KeySpace，一个KeySpace可以有多个分片，如果没有分片KeySpace对应后端存储面的数据库实例关系为1:1，如果有分片则1:N

### 3.3.2. 场景说明

- 用户创建一个MySQL(KeySpace)实例

```bash
# 如果创建只创建一个数据库的话可以参照这个yaml，多加一个keyspace就成
# keyspace也是一个cr，github上没有提供单独创建keyspace cr的方式，应该是可以独立创建，后面研究下
# 这里以修改cluster的方式创建
wget https://github.com/vitessio/vitess/blob/v13.0.0/examples/operator/302_new_shards.yaml
# 修改下增加一个keyspace
kubectl apply -f 302_new_shards.yaml
# 创建完会多出两个vttablet pod
```

### 3.3.3. mysql使用

比较规范的使用方式是通过Load Balance，目前已kubectl端口转发来示例，这个参照文档后面的pf脚本操作即可

```bash
mysql -h 172.16.0.60 -P 15306 -u user
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 18
Server version: 5.7.9-vitess-14.0.0-SNAPSHOT Version: 14.0.0-SNAPSHOT (Git revision 8fde81bc42 branch 'main') built on Tue Mar  8 22:54:23 UTC 2022 by vitess@buildkitsandbox using go1.17.6 linux/amd64

Copyright (c) 2000, 2021, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```

这样，我们就可以进行数据表和记录的增删改查了，但无法进行库的创建删除，也无法进行GRANT操作。

### 3.3.4. 用户和权限管理

在创建集群的时候在[operator.yaml](https://github.com/vitessio/vitess/blob/v13.0.0/examples/operator/operator.yaml)中Secret中的 `users.json` 为用户密码信息

权限则只针对于表，粒度没有mysql grant那么细，详见[authorization](https://vitess.io/docs/13.0/user-guides/configuration-advanced/authorization/)

使用例子摘一个

```bash
$ cat > acls_for_keyspace1.json << EOF
{
  "table_groups": [
    {
      "name": "keyspace1acls",
      "table_names_or_prefixes": ["%"],
      "readers": ["myuser1", "vitess"],
      "writers": ["myuser1", "vitess"],
      "admins": ["myuser1", "vitess"]
    }
  ]
}
EOF

$ vttablet -init_keyspace "keyspace1" -table-acl-config=acls_for_keyspace1.json -enforce-tableacl-config -queryserver-config-strict-table-acl ........
```

用户`myuser1和vitess` 操作keyspace1的所有表的所有权限，其他用户无法访问keyspace1的表

多个用户，同一个集群，多个keyspace，通过acl限制用户权限

### 3.3.5. 数据库配置

通过 `vttablet` 启动参数配置，[VTTablet and MySQL](https://vitess.io/docs/13.0/user-guides/configuration-basic/vttablet-mysql/)

crd安装看起来还没有提供修改配置的选项

### 3.3.6. 备份与恢复

#### 3.3.6.1. 备份后端

支持多种存储作为备份后端

- S3
- FILE(本地路径/共享存储)
- GCS
- Ceph

#### 3.3.6.2. 备份方式

- 默认方式: 停止数据库实例，拷贝数据库文件
- Percona’s XtraBackup 在线备份

以XtraBackup为例

> 这快吐槽下，文档太少了，而且没有说明，得看代码才行

#### 3.3.6.3. 操作

[备份恢复文档](https://vitess.io/docs/13.0/user-guides/operating-vitess/backup-and-restore/creating-a-backup/)

主要操作见 [vitess-operator-for-kubernetes](https://vitess.io/blog/2020-11-09-vitess-operator-for-kubernetes/)

这篇文章写的比较详细，后段存储的配置坑比较多，可以参照这里

##### 3.3.6.3.1. s3

> 这里除了aws s3不兼容其他的s3接口的对象存储例如ceph rgw

`cat s3cfg-credentials.ini`

```json
{
  "aws_access_key_id": "11111",
  "aws_secret_access_key": "2222"
}
```

```bash
kubectl create secret generic s3-auth --from-file=s3cfg-credentials.json
```

```bash
kubectl edit vitessclusters.planetscale.com
```

```yaml
spec:
  backup:
    engine: xtrabackup
    locations:
    - s3:
        authSecret:
          key: s3cfg-credentials.ini
          name: s3-auth
        bucket: admin-coeahbbkkw
        endpoint: http://aws.xxx.xxx
        region: us-west-1
  cells:
  - gateway:
```

##### 3.3.6.3.2. ceph rgw

`cat s3cfg-credentials.json`

```yaml
{
  "accessKey" : "11111",
  "secretKey" : "2222",
  "endPoint"  : "172.16.0.10:8080",
  "useSSL"    : false
}
```

```bash
kubectl create secret generic s3-auth --from-file=s3cfg-credentials.json
```

```bash
kubectl edit vitessclusters.planetscale.com
```

```yaml
spec:
  backup:
    engine: xtrabackup
    locations:
    - ceph:
        authSecret:
          key: s3cfg-credentials.json
          name: s3-auth
  cells:
  - gateway:
```

### 3.3.7. 监控与维护

> 可以通过kubectl port forward通过vtgate api访问监控数据
> vtctld和vtgate都是全局数据，vttablet则是单个的tablet监控

- pf脚本改写如下

```bash
#!/bin/sh

kubectl port-forward --address 0.0.0.0 "$(kubectl get service --selector="planetscale.com/component=vtctld" -o name | head -n1)" 15000 15999 &
process_id1=$!
kubectl port-forward --address 0.0.0.0 "$(kubectl get service --selector="planetscale.com/component=vtgate,!planetscale.com/cell" -o name | head -n1)" 15306:3306 &
process_id2=$!
kubectl port-forward --address 0.0.0.0 "$(kubectl get service --selector="planetscale.com/component=vtgate,!planetscale.com/cell" -o name | head -n1)" 15001:15000 &
process_id3=$!
kubectl port-forward --address 0.0.0.0 "$(kubectl get service --selector="planetscale.com/component=vttablet,!planetscale.com/cell" -o name | head -n1)" 15002:15000 &
process_id4=$!
sleep 2
echo "You may point your browser to http://172.16.0.60:15000, use the following aliases as shortcuts:"
echo 'alias vtctlclient="vtctlclient -server=172.16.0.60:15999 -logtostderr"'
echo 'alias mysql="mysql -h 127.0.0.1 -P 15306 -u user"'
echo "Hit Ctrl-C to stop the port forwards"
wait $process_id1
wait $process_id2
wait $process_id3
wait $process_id4
```

- vtctld http://172.16.0.10:15000/debug/status
- vtgate http://172.16.0.10:15001/debug/status
- vttablet http://172.16.0.10:15001/debug/status -- 单体，某一个vttablet的监控数据

用于web界面使用ingress，服务层转发使用LB

### 3.3.8. 数据迁移

将已有的mysql迁移到vitess

[官方文档提供了三种方式](https://vitess.io/docs/13.0/user-guides/migration/migrate-data/)
[平滑方式实践](https://www.cncf.io/wp-content/uploads/2020/08/Migrating-MySQL-to-Vitess-CNCF-Webinar.pdf)

> 由于时间原因没有实践，条件允许的情况下mysql数据导入导出是最简单粗暴的方式。

## 3.4. 安装部署

使用Operator安装

- 下载代码

```bash
git clone https://github.com/vitessio/vitess.git -b v13.0.0
```

- 进入工作目录

```bash
cd examples/operator/
```

> 接下来的操作对照 [examples/operator/README.md](https://github.com/vitessio/vitess/blob/main/examples/operator/README.md)
> 每步操作的目的对照 [官方中文文档(已停止维护)](https://vitess.io/zh/docs/13.0/get-started/kubernetes/)

operator/README.md 命令比较详细，有遗下需要注意的点

## 3.5. 注意点

### 3.5.1. 操作慢一点

- kubectl apply都需要创建资源，每一步等资源创建完成等1分钟再进行后面操作，running之后还有初始化操作
- vtctlclient命令中有几个是一步任务，也等一分钟之后再做下一步操作

# 4. 附录
## 4.1. 水平分片表现

```bash
# kubectl get po
NAME                                             READY   STATUS            RESTARTS   AGE
example-etcd-faf13de3-1                          1/1     Running           1          3m49s
example-etcd-faf13de3-2                          1/1     Running           1          3m49s
example-etcd-faf13de3-3                          1/1     Running           0          3m49s
example-vttablet-zone1-1250593518-17c58396       0/3     PodInitializing   0          14s -- keyspace customer
example-vttablet-zone1-2469782763-bfadd780       3/3     Running           2          3m49s -- commerce
example-vttablet-zone1-2548885007-46a852d0       3/3     Running           2          3m49s -- commerce
example-vttablet-zone1-3778123133-6f4ed5fc       0/3     PodInitializing   0          14s  -- keyspace customer
example-zone1-vtctld-1d4dcad0-64668cccc8-lwvtb   1/1     Running           2          3m49s
example-zone1-vtgate-bc6cde92-7565589466-fsmxf   1/1     Running           2          3m49s
vitess-operator-7794c74b9b-4pzfl                 1/1     Running           0          27h
```

水平分片后

```bash
[root@k8s-1 operator]# kubectl get po
NAME                                             READY   STATUS     RESTARTS   AGE
example-etcd-faf13de3-1                          1/1     Running    1          7m22s
example-etcd-faf13de3-2                          1/1     Running    1          7m22s
example-etcd-faf13de3-3                          1/1     Running    0          7m22s
example-vttablet-zone1-0118374573-10d08e80       0/3     Init:0/2   0          8s  -- commerce
example-vttablet-zone1-0120139806-fed29577       0/3     Init:0/2   0          8s -- commerce 
example-vttablet-zone1-1250593518-17c58396       3/3     Running    1          3m47s
example-vttablet-zone1-2289928654-7de47379       0/3     Init:0/2   0          8s -- customer 1
example-vttablet-zone1-2469782763-bfadd780       3/3     Running    2          7m22s
example-vttablet-zone1-2548885007-46a852d0       3/3     Running    2          7m22s
example-vttablet-zone1-3778123133-6f4ed5fc       3/3     Running    1          3m47s
example-vttablet-zone1-4277914223-0f04a9a6       0/3     Init:0/2   0          8s -- customer 2
example-zone1-vtctld-1d4dcad0-64668cccc8-lwvtb   1/1     Running    2          7m22s
example-zone1-vtgate-bc6cde92-7565589466-fsmxf   1/1     Running    2          7m22s
vitess-operator-7794c74b9b-4pzfl                 1/1     Running    0          27h
```

## 4.2. reshard 报错

```bash
vtctlclient Reshard -source_shards '-' -target_shards '-80,80-' Create customer.cust2cust
Reshard Error: rpc error: code = Unknown desc = validateWorkflowName.VReplicationExec: found previous frozen workflow on tablet 1250593518, please review and delete it first before creating a new workflow
E0309 07:35:23.947853    9395 main.go:76] remote error: rpc error: code = Unknown desc = validateWorkflowName.VReplicationExec: found previous frozen workflow on tablet 1250593518, please review and delete it first before creating a new workflow
```

- 查看workflow

```bash
vtctlclient Workflow customer listall
Following workflow(s) found in keyspace customer: commerce2customer
[root@k8s-1 operator]# vtctlclient Workflow customer.commerce2customer show
{
	"Workflow": "commerce2customer",
	"SourceLocation": {
		"Keyspace": "commerce",
		"Shards": [
			"-"
		]
	},
	"TargetLocation": {
		"Keyspace": "customer",
		"Shards": [
			"-"
		]
	},
	"MaxVReplicationLag": 673,
	"MaxVReplicationTransactionLag": 673,
	"Frozen": true,
	"ShardStatuses": {
		"-/zone1-1250593518": {
			"PrimaryReplicationStatuses": [
				{
					"Shard": "-",
					"Tablet": "zone1-1250593518",
					"ID": 1,
					"Bls": {
						"keyspace": "commerce",
						"shard": "-",
						"filter": {
							"rules": [
								{
									"match": "customer",
									"filter": "select * from customer"
								},
								{
									"match": "corder",
									"filter": "select * from corder"
								}
							]
						}
					},
					"Pos": "af259889-9f79-11ec-8062-cafafa8a2004:1-64",
					"StopPos": "",
					"State": "Stopped",
					"DBName": "vt_customer",
					"TransactionTimestamp": 0,
					"TimeUpdated": 1646810939,
					"TimeHeartbeat": 1646810939,
					"Message": "FROZEN",
					"Tags": "",
					"CopyState": null
				}
			],
			"TabletControls": null,
			"PrimaryIsServing": true
		}
	}
}
```

## 4.3. 水平的两个分片的数据

```bash
[root@k8s-1 ~]# kubectl exec -it example-vttablet-zone1-0120139806-fed29577 -c mysqld -- mysql -S /vt/socket/mysql.sock -u root -e 'select * from vt_customer.corder'
+----------+-------------+----------+-------+
| order_id | customer_id | sku      | price |
+----------+-------------+----------+-------+
|        1 |           1 | SKU-1001 |   100 |
|        2 |           2 | SKU-1002 |    30 |
|        3 |           3 | SKU-1002 |    30 |
|        5 |           5 | SKU-1002 |    30 |
+----------+-------------+----------+-------+
[root@k8s-1 ~]# kubectl exec -it example-vttablet-zone1-0118374573-10d08e80 -c mysqld --  mysql -S /vt/socket/mysql.sock -u root -e 'select * from vt_customer.corder'
+----------+-------------+----------+-------+
| order_id | customer_id | sku      | price |
+----------+-------------+----------+-------+
|        4 |           4 | SKU-1002 |    30 |
+----------+-------------+----------+-------+
```

- 或者按照分片通过mysql客户端查看

```bash
[root@k8s-1 operator]# mysql --table < ../common/select_customer-80_data.sql
Using customer/-80
Customer
+-------------+--------------------+
| customer_id | email              |
+-------------+--------------------+
|           1 | alice@domain.com   |
|           2 | bob@domain.com     |
|           3 | charlie@domain.com |
|           5 | eve@domain.com     |
+-------------+--------------------+
COrder
+----------+-------------+----------+-------+
| order_id | customer_id | sku      | price |
+----------+-------------+----------+-------+
|        1 |           1 | SKU-1001 |   100 |
|        2 |           2 | SKU-1002 |    30 |
|        3 |           3 | SKU-1002 |    30 |
|        5 |           5 | SKU-1002 |    30 |
+----------+-------------+----------+-------+
[root@k8s-1 operator]# mysql --table < ../common/select_customer80-_data.sql
Using customer/80-
Customer
+-------------+----------------+
| customer_id | email          |
+-------------+----------------+
|           4 | dan@domain.com |
+-------------+----------------+
COrder
+----------+-------------+----------+-------+
| order_id | customer_id | sku      | price |
+----------+-------------+----------+-------+
|        4 |           4 | SKU-1002 |    30 |
+----------+-------------+----------+-------+
[root@k8s-1 operator]#
```

## 4.4. 分片与数据库对应关系

```bash
[root@k8s-1 operator]# vtctlclient ListAllTablets
zone1-0118374573 customer 80- primary 10.233.117.118:15000 10.233.117.118:3306 [] 2022-03-09T09:34:58Z
zone1-0120139806 customer -80 primary 10.233.117.117:15000 10.233.117.117:3306 [] 2022-03-09T09:34:51Z
zone1-2289928654 customer -80 replica 10.233.95.107:15000 10.233.95.107:3306 [] <null>
zone1-2469782763 commerce - primary 10.233.95.105:15000 10.233.95.105:3306 [] 2022-03-09T09:34:35Z
zone1-2548885007 commerce - replica 10.233.117.119:15000 10.233.117.119:3306 [] <null>
zone1-4277914223 customer 80- replica 10.233.95.108:15000 10.233.95.108:3306 [] <null>
```

## 4.5. 元数据存储

```bash
[root@k8s-1 operator]# kubectl exec -it example-etcd-faf13de3-1 -- etcdctl get "" --prefix --keys-only | sed '/^\s*$/d'
/vitess/example/global/cells/zone1/CellInfo
/vitess/example/global/cells_aliases/planetscale_operator_default/CellsAlias
/vitess/example/global/elections/vtctld/locks/6605231913564433811
/vitess/example/global/keyspaces/commerce/Keyspace
/vitess/example/global/keyspaces/commerce/VSchema
/vitess/example/global/keyspaces/commerce/shards/-/Shard
/vitess/example/global/keyspaces/customer/Keyspace
/vitess/example/global/keyspaces/customer/VSchema
/vitess/example/global/keyspaces/customer/shards/-80/Shard
/vitess/example/global/keyspaces/customer/shards/80-/Shard
/vitess/example/local/zone1/SrvVSchema
/vitess/example/local/zone1/keyspaces/commerce/SrvKeyspace
/vitess/example/local/zone1/keyspaces/commerce/shards/-/ShardReplication
/vitess/example/local/zone1/keyspaces/customer/SrvKeyspace
/vitess/example/local/zone1/keyspaces/customer/shards/-80/ShardReplication
/vitess/example/local/zone1/keyspaces/customer/shards/80-/ShardReplication
/vitess/example/local/zone1/tablets/zone1-0118374573/Tablet
/vitess/example/local/zone1/tablets/zone1-0120139806/Tablet
/vitess/example/local/zone1/tablets/zone1-2289928654/Tablet
/vitess/example/local/zone1/tablets/zone1-2469782763/Tablet
/vitess/example/local/zone1/tablets/zone1-2548885007/Tablet
/vitess/example/local/zone1/tablets/zone1-4277914223/Tablet
```

## 4.6. 后端存储层实际的数据库结构

```bash
kubectl exec -it example-vttablet-zone1-0120139806-fed29577  -c mysqld --  mysql -S /vt/socket/mysql.sock -u root -e 'show databases;'
+--------------------+
| Database           |
+--------------------+
| information_schema |
| _vt                |
| mysql              |
| performance_schema |
| sys                |
| vt_customer        |
+--------------------+
```

- _vt 存储主从复制元信息

```bash
kubectl exec -it example-vttablet-zone1-0120139806-fed29577  -c mysqld --  mysql -S /vt/socket/mysql.sock -u root -e 'use _vt; show tables;'
+--------------------+
| Tables_in__vt      |
+--------------------+
| copy_state         |
| local_metadata     |
| reparent_journal   |
| resharding_journal |
| schema_migrations  |
| shard_metadata     |
| vreplication       |
| vreplication_log   |
+--------------------+
```

- vt_$KeySpaceName 则存储实际数据

## 4.7. 备份失败

```log
kubectl logs example-test003-x-x-vtbackup-init-8e4ed4a0
ERROR: logging before flag.Parse: E0314 14:41:06.523279       1 syslogger.go:149] can't connect to syslog
E0314 14:41:06.615150       1 vtbackup.go:179] Can't take backup: refusing to upload initial backup of empty database: the shard test003/- already has at least one tablet that may be serving (zone1-1458266976); you must take a backup from a live tablet instead

vtctlclient -logtostderr BackupShard test003/-
I0314 14:36:17.921916    7305 main.go:67] I0314 14:36:17.919405 backup.go:172] I0314 14:36:17.920898 xtrabackupengine.go:311] xtrabackup stderr: 220314 14:36:17 completed OK!
I0314 14:36:17.931230    7305 main.go:67] I0314 14:36:17.928341 backup.go:172] I0314 14:36:17.929707 xtrabackupengine.go:634] Found position: c97b467c-a34a-11ec-96e6-aeab1d724023:1-30,c98247a5-a34a-11ec-bf36-e2945f838fbe:1-73
I0314 14:36:17.931268    7305 main.go:67] I0314 14:36:17.928387 backup.go:172] I0314 14:36:17.929788 xtrabackupengine.go:117] Closing backup file backup.xbstream.gz-000
I0314 14:36:17.931275    7305 main.go:67] I0314 14:36:17.928397 backup.go:172] I0314 14:36:17.929797 xtrabackupengine.go:117] Closing backup file backup.xbstream.gz-001
I0314 14:36:17.931280    7305 main.go:67] I0314 14:36:17.928441 backup.go:172] I0314 14:36:17.929802 xtrabackupengine.go:117] Closing backup file backup.xbstream.gz-002
I0314 14:36:17.931285    7305 main.go:67] I0314 14:36:17.928475 backup.go:172] I0314 14:36:17.929808 xtrabackupengine.go:117] Closing backup file backup.xbstream.gz-003
I0314 14:36:17.931291    7305 main.go:67] I0314 14:36:17.928483 backup.go:172] I0314 14:36:17.929815 xtrabackupengine.go:117] Closing backup file backup.xbstream.gz-004
I0314 14:36:17.931298    7305 main.go:67] I0314 14:36:17.928494 backup.go:172] I0314 14:36:17.929824 xtrabackupengine.go:117] Closing backup file backup.xbstream.gz-005
I0314 14:36:17.931305    7305 main.go:67] I0314 14:36:17.928500 backup.go:172] I0314 14:36:17.929838 xtrabackupengine.go:117] Closing backup file backup.xbstream.gz-006
I0314 14:36:17.931310    7305 main.go:67] I0314 14:36:17.928510 backup.go:172] I0314 14:36:17.929843 xtrabackupengine.go:117] Closing backup file backup.xbstream.gz-007
I0314 14:36:17.931317    7305 main.go:67] I0314 14:36:17.928518 backup.go:172] I0314 14:36:17.929863 xtrabackupengine.go:164] Writing backup MANIFEST
BackupShard Error: rpc error: code = Unavailable desc = error reading from server: EOF
E0314 14:36:18.260100    7305 main.go:76] remote error: rpc error: code = Unavailable desc = error reading from server: EOF
```

备份保存数据会失败，在内置冷备份方式有差不多的问题，暂不确定是否为bug，先做记录

# 5. 选型总结

## 5.1. 优势

覆盖了数据库常用场景，且水平扩展的架构理念值得学习

## 5.2. 劣势

- 文档不全且少，很多时候填个配置需要查看代码才知道
- operator用起来处处受限
- 无法保证数据的强一致性
- 维护复杂
- 监控界面简陋
- 没有用户自助的dashboard
- 权限粒度太小，只能通过表来限制

## 5.3. 总结

- 对于数据量特别大的场景，经常需要通过分库分表来解决性能瓶颈问题，Vitess是不错的选择。
- 对于简单实用，单实例数据库不会有太大压力的场景下，使用Vitess有点杀鸡用牛刀，会带来很多操作以及维护的问题。

# 6. 参考

- [ChubaoFS+Vitess](https://developer.jdcloud.com/article/985)
- [vitess github](https://github.com/vitessio/vitess/)
- [vitess operator](https://github.com/planetscale/vitess-operator)
- [Migrating-MySQL-to-Vitess-CNCF-Webinar](https://www.cncf.io/wp-content/uploads/2020/08/Migrating-MySQL-to-Vitess-CNCF-Webinar.pdf)
- [vitess history](https://vitess.io/docs/13.0/overview/history/)
- [官方文档](https://vitess.io/docs/) -- 原始文档，最具参考意义
- [官方视频](https://www.youtube.com/watch?v=E6H4bgJ3Z6c) -- 最直观的了解方式
- [官方中文文档(已停止维护)](https://vitess.io/zh/docs/13.0/get-started/kubernetes/) -- 安装和最核心的功能在一起，对于首次了解vitess很有帮助
  需对照 [examples/operator/README.md](https://github.com/vitessio/vitess/blob/main/examples/operator/README.md)
- [概念介绍比较全面的个人博客](http://masikkk.com/article/Vitess/) 写的相对比较详细，可做参考。
