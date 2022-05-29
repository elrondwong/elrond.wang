---
layout: post
title: 云原生数据MySQL调研(percona与bitpoke)
catalog: true
tag: [MySQL, 云原生]
---
<!-- TOC -->

- [1. 概述](#1-概述)
- [2. bitpoke-operator](#2-bitpoke-operator)
  - [2.1. 架构](#21-架构)
    - [2.1.1. k8s服务实体](#211-k8s服务实体)
  - [2.2. 典型场景](#22-典型场景)
    - [2.2.1. 用户申请一个实例](#221-用户申请一个实例)
    - [2.2.2. 用户连接使用](#222-用户连接使用)
    - [2.2.3. 用户认证和权限管理](#223-用户认证和权限管理)
    - [2.2.4. 数据库配置](#224-数据库配置)
    - [2.2.5. 备份与恢复](#225-备份与恢复)
      - [2.2.5.1. 设置集群备份配置](#2251-设置集群备份配置)
    - [2.2.6. 创建备份后端secret](#226-创建备份后端secret)
    - [2.2.7. 创建备份cr](#227-创建备份cr)
  - [2.3. 部署](#23-部署)
  - [2.4. 监控与维护](#24-监控与维护)
  - [2.5. 优劣势总结](#25-优劣势总结)
- [3. percona-xtradb-cluster-operator](#3-percona-xtradb-cluster-operator)
  - [3.1. 架构](#31-架构)
  - [3.2. 系统架构](#32-系统架构)
    - [3.2.1. K8S服务实体](#321-k8s服务实体)
  - [3.3. 优劣势总结](#33-优劣势总结)
- [4. 附录](#4-附录)
  - [4.1. 数据库半同步支持](#41-数据库半同步支持)
    - [4.1.1. bitpoke](#411-bitpoke)
  - [服务提供方式](#服务提供方式)
  - [节点亲和/优先级配置](#节点亲和优先级配置)
    - [bitpoke](#bitpoke)
  - [数据存储PVC配置](#数据存储pvc配置)
    - [bitpoke](#bitpoke-1)
  - [修改密码](#修改密码)
    - [bitpoke](#bitpoke-2)
- [4. 参考](#4-参考)

<!-- /TOC -->
# 1. 概述

- [bitpoke的operator](https://github.com/bitpoke/mysql-operator) -- 维护不够勤快
- [percona-xtradb-cluster-operator](https://github.com/percona/percona-xtradb-cluster-operator)
- [mysql官方的operator](https://github.com/mysql/mysql-operator) -- 月久失修，当前只有预览版，不做考虑

这三个都是直接管理MySQL实例，相对简单，一个集群即用户可见的一个数据库实例集群。
而不像Vitess是一个MySQL平台，对实际的数据库实例又抽象了一层，比较复杂。

文档从如下几个方面对bitpoke-operator和pxc-operator进行描述:

- 架构落到用户可见层面是怎么样的
- 集群创建
- 对外提供服务，包括对K8S集群内以及集群外的传统应用
- 监控与维护
- 数据备份
- 优劣势

# 2. bitpoke-operator

## 2.1. 架构

网上没有相关的架构图，相对简单，使用原生的mysql master/slave 模式高可用，使用 orchestrator 实现master/slave的切换，使用jobs+xtrabackup 实现数据库的定时在线备份

![mysql-operator技术架构](/img/posts/云原生数据MySQL调研(percona与bitpoke)/mysql-operator技术架构.png)

### 2.1.1. k8s服务实体

```bash
# crd
mysqlbackups.mysql.presslabs.org                      2022-03-15T03:26:43Z
mysqlclusters.mysql.presslabs.org                     2022-03-15T03:26:43Z
mysqldatabases.mysql.presslabs.org                    2022-03-15T03:26:43Z
mysqlusers.mysql.presslabs.org                        2022-03-15T03:26:43Z
# statefulset
my-cluster-mysql   2/2     45h
mysql-operator     1/1     2d3h
# pod
my-cluster-backup-backup-pdh55                       0/1     Completed          0          3h12m -- 备份
# mysql sidecar metrics-exporter pt-heartbeat
my-cluster-mysql-0                                   4/4     Running            0          3h10m -- 副本
my-cluster-mysql-1                                   4/4     Running            0          3h12m -- 副本
# operator orchestrator
mysql-operator-0                                     2/2     Running            2          3h21m -- operator
# job
my-cluster-backup-backup   1/1           76s        3h13m
# configMap
# my-cluster-mysql 记录了MySQL实例的my.cnf，默认值在 pkg/controller/mysqlcluster/internal/syncer/config_map.go，在mysqlCluster的CR中修改的值会覆盖默认值，每次修改都会重启pod
my-cluster-mysql                 2      12d
mysql-operator-leader-election   0      15d
mysql-operator-orc               2      15d
```

## 2.2. 典型场景

> 以下前提都是operator已经部署 在 `examples` 下操作

### 2.2.1. 用户申请一个实例

申请数据库实例，即创建一个数据库集群

> v1.0之前默认MySQL版本是5.7.31 如果需要部署 MySQL 8.0，修改 `example-cluster.yaml` 中 `mysqlVersion: "8.0"`
> 注意⚠️: `#image: percona:8.0` 保持注释状态
> 密码设置在 `example-cluster-secret.yaml`中， 当前不支持通过crd修改，只能通过mysql客户端修改

```bash
kubectl apply -f example-cluster-secret.yaml
kubectl apply -f example-cluster.yaml
```

在这个crd里面可以定义

- 副本数
- 镜像版本
- 备份/恢复后端
- mysql配置
- mysql数据存储
- ...

### 2.2.2. 用户连接使用

以service提供服务

```bash
my-cluster-mysql                ClusterIP   10.233.55.139   <none>        3306/TCP                       5h7m
my-cluster-mysql-master         ClusterIP   10.233.62.8     <none>        3306/TCP,8080/TCP              5h7m
my-cluster-mysql-replicas       ClusterIP   10.233.61.220   <none>        3306/TCP,8080/TCP              5h7m
mysql                           ClusterIP   None            <none>        3306/TCP,9125/TCP              5h7m
mysql-operator                  ClusterIP   10.233.25.23    <none>        80/TCP,9125/TCP                5h8m
mysql-operator-orc              ClusterIP   None            <none>        80/TCP,10008/TCP               5h8m
```

连接master即可

```bash
mysql -h 10.233.62.8 -P 3306 -pnot-so-secure
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 13667
Server version: 5.7.31-34-log Percona Server (GPL), Release 34, Revision 2e68637

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
```

连接上去之后和进程版mysql操作没有不一样的地方，例如

- 用户管理
- 库管理
- 表管理
- ...

对外暴露，loadbalance方案还是merge中 [Allow additional service configuration for MysqlCluster](https://github.com/bitpoke/mysql-operator/pull/747) -- 20220322
nodePort方式，需自定义operator [mysql-cluster custom service](https://github.com/bitpoke/mysql-operator/pull/794/files) -- 20220322 merge中，未支持数据库实例级别的nodePort

当前如果需要暴露端口的话，crd是未支持的 需要自己创建service用nodePort暴露

### 2.2.3. 用户认证和权限管理

与进程版mysql操作一致

mysqlusers 也是一个cr，也可以通过定义mysqlusers完成权限管理和用户认证

```bash
kubectl apply -f example-user.yaml
```

### 2.2.4. 数据库配置

在创建集群的时候设置在 cluster的MysqlCluster crd中

### 2.2.5. 备份与恢复

> 多副本模式下，数据库为原生的master/slave模式

备份后端支持, 具体实现为rclone, mysql的备份是通过xtrabackup来实现的

- gcs
- s3
- blob

以s3为例

> 设置定时备份，与备份路径，桶需要提前创建

#### 2.2.5.1. 设置集群备份配置

```yaml
#kubectl edit mysqlclusters.mysql.presslabs.org  my-cluster
# 每分钟备份一次测试
  uid: 4b38cbf7-4d42-4e1e-ae8e-2c3350202346
spec:
  backupSchedule: '* * * * * *'
  backupSecretName: my-cluster-backup-secret
  backupURL: s3://test001/
  initBucketSecretName: my-cluster-backup-secret
  initBucketURL: s3://test001/backup.xbackup.gz
  podSpec:
```

### 2.2.6. 创建备份后端secret

> secret data value都需要base64编码以下

`example-backup-secret.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-cluster-backup-secret
type: Opaque
data:
    AWS_ACCESS_KEY_ID: MTExxMTE=
    AWS_SECRET_ACCESS_KEY: MjaIyMg==
    AWS_REGION: dXMtZWFzdC000x
    S3_PROVIDER: Q0VQSAA==
    S3_ENDPOINT: aHR0cDafaovLzE3Mi4xNi44MC44Njo4MDgw
```

```bash
kubectl apply -f example-backup-secret.yaml
```

### 2.2.7. 创建备份cr

`example-backup.yaml`

```yaml
apiVersion: mysql.presslabs.org/v1alpha1
kind: MysqlBackup
metadata:
  name: my-cluster-backup

spec:
  clusterName: my-cluster
  backupSecretName: my-cluster-backup-secret
```

```bash
kubectl apply -f example-backup.yaml
```

这里定时任务的实现是 jobs.batch

## 2.3. 部署

[getting-started](https://www.bitpoke.io/docs/mysql-operator/getting-started/)

helm一条命令搞定部署

## 2.4. 监控与维护

```bash
# orchestrator
nohup kubectl port-forward --address 0.0.0.0 service/mysql-operator 8082:80 &
# operator prometheus
nohup kubectl port-forward --address 0.0.0.0 service/mysql-operator 9125:9125 &
# mysql实例 prometheus
nohup kubectl port-forward --address 0.0.0.0 service/mysql 9126:9125 &
```

访问 http://172.16.0.60:8082/ 即可看到orchestrator的dashboard

orchestrator 可以自动实现master/slave的监视与切换，而且提供了可视化界面

![orchestrator](/img/posts/云原生数据MySQL调研(percona与bitpoke)/orchestrator.png)

可以通过可视化界面对数据库副本进行简单的管理和观测

- mysql每个副本pod都有一个mysql-exporter，使用了promethues官方的mysql-exporter，暴露mysql基础数据, grafana也提供了dashboard id `7362`
- operator-exporter 暴露了一些集群的数据，这里有个[issue](https://github.com/bitpoke/mysql-operator/issues/609)讨论如何设置servicemonitor的

## 2.5. 优劣势总结

简单易用，可观测性较差，适用于小规模的业务和测试用

# 3. percona-xtradb-cluster-operator

## 3.1. 架构

## 3.2. 系统架构

![replication](https://www.percona.com/doc/kubernetes-operator-for-pxc/_images/replication.png)

ProxySQL + galera 多主，多读写，强一致性

> 按官方安装指导安装完实际上是 HAProxy + galera

这里具体用 HAProxy 还是 ProxySQL 可选

[HAProxy](https://www.percona.com/doc/kubernetes-operator-for-pxc/haproxy-conf.html)
[ProxySQL](https://www.percona.com/doc/kubernetes-operator-for-pxc/proxysql-conf.html)

### 3.2.1. K8S服务实体

```bash
# crd
perconaxtradbbackups.pxc.percona.com                  2022-03-16T01:51:06Z
perconaxtradbclusterbackups.pxc.percona.com           2022-03-16T01:51:05Z
perconaxtradbclusterrestores.pxc.percona.com          2022-03-16T01:51:05Z
perconaxtradbclusters.pxc.percona.com                 2022-03-16T01:51:05Z
# svc
cluster1-haproxy                  ClusterIP   10.233.0.237    <none>        3306/TCP,3309/TCP,33062/TCP,33060/TCP   12m
cluster1-haproxy-replicas         ClusterIP   10.233.54.153   <none>        3306/TCP                                12m
cluster1-pxc                      ClusterIP   None            <none>        3306/TCP,33062/TCP,33060/TCP            12m
cluster1-pxc-unready              ClusterIP   None            <none>        3306/TCP,33062/TCP,33060/TCP            12m
percona-xtradb-cluster-operator   ClusterIP   10.233.14.161   <none>        443/TCP                                 12m
# statefulset
cluster1-haproxy   3/3     29h
cluster1-pxc       3/3     29h
# deployment
percona-xtradb-cluster-operator   1/1     1            1           17m
# po
cluster1-haproxy-0                                 2/2     Running   1          17m
cluster1-haproxy-1                                 2/2     Running   0          12m
cluster1-haproxy-2                                 2/2     Running   0          11m
cluster1-pxc-0                                     3/3     Running   0          17m
cluster1-pxc-1                                     3/3     Running   0          12m
cluster1-pxc-2                                     3/3     Running   0          9m
percona-client                                     1/1     Running   0          14m
percona-xtradb-cluster-operator-69dc6dd7cc-8h78q   1/1     Running   0          18m
```

其他功能都大同小异，这里不展开说明

## 3.3. 优劣势总结

- 数据写入强一致性，单个副本/两副本故障，用户无感知，数据量大的情况下整集群故障之后极难恢复
- 对数据有强一致性需求可选该方案

OpenStack集群用的便是 HAProxy + Galera，数据量大、掉电之后恢复难度很大

# 4. 附录

## 4.1. 数据库半同步支持

### 4.1.1. bitpoke

参照 https://github.com/bitpoke/mysql-operator/issues/519

```yaml
apiVersion: mysql.presslabs.org/v1alpha1
kind: MysqlCluster
metadata:
  name: example-cluster
spec:
  mysqlConf:
    plugin-load: semisync_master.so:semisync_slave.so
    rpl-semi-sync-master-enabled: 1
    rpl-semi-sync-master-timeout: 1000
    rpl-semi-sync-master-wait-for-slave-count: 1
    rpl-semi-sync-slave-enabled: 1
    slave-compressed-protocol: "off"
```

```bash
show variables like '%Rpl%';
+-------------------------------------------+------------+
| Variable_name                             | Value      |
+-------------------------------------------+------------+
| rpl_read_size                             | 8192       |
| rpl_semi_sync_master_enabled              | ON         |
| rpl_semi_sync_master_timeout              | 1000       |
| rpl_semi_sync_master_trace_level          | 32         |
| rpl_semi_sync_master_wait_for_slave_count | 1          |
| rpl_semi_sync_master_wait_no_slave        | ON         |
| rpl_semi_sync_master_wait_point           | AFTER_SYNC |
| rpl_semi_sync_slave_enabled               | ON         |
| rpl_semi_sync_slave_trace_level           | 32         |
| rpl_stop_slave_timeout                    | 31536000   |
+-------------------------------------------+------------+
```

[启用半同步之后orchestrator的强制半同步false解释](https://github.com/openark/orchestrator/issues/690)

## 服务提供方式

默认都是以service clusterip方式提供，外部无法访问，外部如果需要访问的话需要修改operator代码或者单处创建service

## 节点亲和/优先级配置

### bitpoke

参照 [example-cluster](https://github.com/bitpoke/mysql-operator/blob/master/examples/example-cluster.yaml)

```yaml
  ## Specify additional pod specification
  # podSpec:
  #   imagePullSecrets: []
  #   labels: {}
  #   annotations: {}
  #   affinity:
  #     podAntiAffinity:
  #       preferredDuringSchedulingIgnoredDuringExecution:
  #         weight: 100
  #         podAffinityTerm:
  #           topologyKey: "kubernetes.io/hostname"
  #           labelSelector:
  #             matchlabels: <cluster-labels>
  #   backupAffinity: {}
  #   backupNodeSelector: {}
  #   backupPriorityClassName:
  #   backupTolerations: []
  #   # Override the default preStop hook with a custom command/script
  #   mysqlLifecycle:
  #     preStop:
  #       exec:
  #         command:
  #           - /scripts/demote-if-master
  #   nodeSelector: {}
  #   resources:
  #     requests:
  #       memory: 1G
  #       cpu:    200m
  #   tolerations: []
  #   priorityClassName:
  #   serviceAccountName: default
  #   # Use a initContainer to fix the permissons of a hostPath volume.
  #   initContainers:
  #     - name: volume-permissions
  #       image: busybox
  #       securityContext:
  #         runAsUser: 0
  #       command:
  #         - sh
  #         - -c
  #         - chmod 750 /data/mysql; chown 999:999 /data/mysql
  #       volumeMounts:
  #         - name: data
  #           mountPath: /data/mysql
```

## 数据存储PVC配置

### bitpoke

```yaml
  volumeSpec:
    persistentVolumeClaim:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
      storageClassName: csi-cephfs-sc
```

## 修改密码

### bitpoke

> root密码在创建集群时设置过之后就无法修改，其他用户在创建mysqluser cr之后就无法修改，后来需要修改用户需通过mysql客户端

这里有个[issue](https://github.com/bitpoke/mysql-operator/issues/75) 需要去watch secret改变而更新MySQL密码，但迟迟没有实现

```bash
kubectl exec -it my-cluster-mysql-0  -c mysql -- mysql --defaults-file=/etc/mysql/client.conf
mysql> UPDATE mysql.user SET Password=PASSWORD('new password') WHERE User='root';
mysql> FLUSH PRIVILEGES;
mysql> quit;
```

# 4. 参考

- [hoosing-a-kubernetes-operator-for-mysql](https://portworx.com/blog/choosing-a-kubernetes-operator-for-mysql/)
- [bitpoke文档](https://www.bitpoke.io/docs/mysql-operator/backups/)
- [pxc部署](https://www.percona.com/doc/kubernetes-operator-for-pxc/kubernetes.html)
- [pxc架构](https://www.percona.com/doc/kubernetes-operator-for-pxc/architecture.html)
- [ProxySQL](https://github.com/sysown/proxysql)
