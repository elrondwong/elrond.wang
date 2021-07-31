---
layout: post
title: OpenStack使用Placement做资源管理时虚拟机无法调度的一次排错记录
catalog: true
tag: [OpenStack,Nova,Placement]
---

<!-- TOC -->

- [1. 场景说明](#1-场景说明)
  - [1.1. 日志](#11-日志)
- [2. 排查过程](#2-排查过程)
  - [2.1. 资源够用吗](#21-资源够用吗)
  - [2.2. Placement Debug](#22-placement-debug)
  - [2.3. API启动脚本](#23-api启动脚本)
  - [2.4. 具体处理过程](#24-具体处理过程)
- [3. 思考](#3-思考)
  - [3.1. 为什么单个资源分配上限没有乘以超分倍数**](#31-为什么单个资源分配上限没有乘以超分倍数)
  - [3.2. 拓展问题](#32-拓展问题)
- [4. 参考文档](#4-参考文档)

<!-- /TOC -->

# 1. 场景说明

> 计算节点能分配的最大虚拟机资源不能大于自身资源 -- 出错时还挂在嘴边，但真真有所体会和认知，往往是在挨了毒打之后

|组件|版本|备注|
|:-:|:-:|:-:|
|OpenStack|train||
|Nova|20.6.0-1|
|Placement|2.0.1-1|
|python2-osc-placement|1.7.0-1|安装之后就可以使用OpenStack命令行操作placement api了，例如resource provider 等命令|
|计算节点CPU个数|48||
|计算节点CPU超分比|4||

环境信息如上，需要申请一台64C128G的虚拟机，申请时秒报错

## 1.1. 日志

> 排错时日志开debug模式
> 如果是高可用模式，则先只保留一个可用的服务做测试，其余的都关闭

- `/var/log/nova/nova-conductor.log`

```log
e0210a948 0fe34a6944b4416195ff5176f681c34f - default default] Failed to schedule instances: NoValidHost_Remote: No valid host was found.
Traceback (most recent call last):

  File "/usr/lib/python2.7/site-packages/oslo_messaging/rpc/server.py", line 235, in inner
    return func(*args, **kwargs)

  File "/usr/lib/python2.7/site-packages/nova/scheduler/manager.py", line 199, in select_destinations
    raise exception.NoValidHost(reason="")

NoValidHost: No valid host was found.
```

- `/var/log/nova/nova-scheduler.log`

```log
20596:2021-07-30 14:46:16.665 175316 INFO nova.scheduler.manager [None req-cb66e713-19a8-45d2-b107-dd31569f4653 b5957a478f0d4c328ef10cee0210a948 0fe34a6944b4416195ff5176f681c34f - default default] Got no allocation candidates from the Placement API. This could be due to insufficient resources or a temporary occurrence as compute nodes start up.
```

- `/var/log/placement/placement.log`

```log
2021-07-30 16:31:39.909 4638 INFO placement.requestlog [req-5081ba30-e270-446b-afc6-ce5fa550088c 563e3a05b24e4360b884ef0ca35a71ef b3d671ef1603499ab9744dad3a66c26d - default default] 172.16.88.187 "GET /resource_providers/49905d0d-8b33-434a-8bc5-6a73898b9922/allocations" status: 200 len: 259 microversion: 1.0\x1b[00m
2021-07-30 16:31:41.237 4637 DEBUG placement.requestlog [req-54b926ab-64fb-404b-b274-6aa22acadce3 - - - - -] Starting request: 172.16.88.187 "GET /allocation_candidates?resources=VCPU%3A64" __call__ /usr/lib/python2.7/site-packages/placement/requestlog.py:61\x1b[00m
2021-07-30 16:31:41.293 4637 DEBUG placement.objects.research_context [req-54b926ab-64fb-404b-b274-6aa22acadce3 b5957a478f0d4c328ef10cee0210a948 0fe34a6944b4416195ff5176f681c34f - default default] getting providers with 64 VCPU __init__ /usr/lib/python2.7/site-packages/placement/objects/research_context.py:126\x1b[00m
2021-07-30 16:31:41.296 4637 DEBUG placement.objects.research_context [req-54b926ab-64fb-404b-b274-6aa22acadce3 b5957a478f0d4c328ef10cee0210a948 0fe34a6944b4416195ff5176f681c34f - default default] found no providers with 64 VCPU __init__ /usr/lib/python2.7/site-packages/placement/objects/research_context.py:130\x1b[00m
```

通过日志可以看出大致的调用链:
nova-api -> nova-conductor -> nova-schduler -> placement
问题在于调用placement api `/allocation_candidates` 返回了200 但是scheduler还是没选到资源，就需要看看他返回了什么信息，辅之以debug信息`found no providers`看下为什么没有prividers

# 2. 排查过程

> 排查问题一般到无法通过日志分析出问题的时候才看代码，直接看不熟的代码耗时比较长

## 2.1. 资源够用吗

```bash
openstack hypervisor list
openstack hypervisor show xxxxx
```

以上两条命令可以看到计算节点的资源情况，最直观的在dashboard 管理员 虚拟机管理器查看即可，比较细致的可以看数据库

> placement库

```sql
/* 12:08:23 PM dev-openstack01 placement */ 
SELECT * FROM `inventories`;
/* output */
/* allocation ratio 列为超分比对应到resource_class_id 0 为cpu 1 为内存 2 为硬盘 */
2021-05-12 03:35:15	2021-05-21 00:48:33	4	2	0	48	0	1	48	1	4
```

总资源查询如上，已用资源查询如下

```sql
/* 12:11:32 PM dev-openstack01 placement */
SELECT * FROM `allocations` ORDER BY `consumer_id` LIMIT 0,1000;
```

consumer_id对应的虚拟机uuid，一般情况每个uuid对应三个资源 cpu内存磁盘，如果有一个uuid对应6个或者更多则说明该虚拟机占着好几个茅坑不拉屎，释放资源即可 具体操作[参照这篇博客](https://www.jianshu.com/p/48e1451cf7dc)

释放资源操作

```bash
PLACEMENT_ENDPOINT=$(openstack endpoint list|grep placement|awk '{print $((NF-1))}'|uniq)
TOKEN=$(openstack token issue -f value -c id)
curl ${PLACEMENT_ENDPOINT}/allocations/01b6377f-5b0c-4273-a24a-921f793b3004 -H "x-auth-token: $TOKEN" -X DELETE
```

实际资源超分之后完全足够，回到日志，继续看为什么没有能调度到节点

## 2.2. Placement Debug

手动调用下 `allocation_candidates`

- 工具 postman
  - url路径 从placement日志中看到的`?resources=VCPU%3A64` 用url解码下 `?resources=VCPU:64`
    完整路径为 `http://172.16.88.191:8778/allocation_candidates?resources=VCPU:64`
  - 需要注意额外加两个 header
    - X-OpenStack-AUTH 值为TOKEN
    - OpenStack-API-Version 值为 `placement 1.10`(默认version是1.0 无法使用allocation_candidates api填一个介于最大1.36最小1.0中间的即可)

或者可以用OpenStack命令行工具，使用命令行工具时需要安装 `python2-osc-placement`

```bash
openstack --debug --os-placement-api-version 1.10 allocation candidate list --resource VCPU=64
```

结果都是为空，这时候需要debug看下代码怎么算的，如果对代码非常熟就可以用pdb定点打断点了，如果不是很熟建议用ide远程调试[Pycharm调试OpenStack](https://www.jb51.net/article/128727.htm)，这样每一步都有详细的输出

## 2.3. API启动脚本

默认部署通过httpd启动 `/etc/httpd/conf.d/00-placement-api.conf`
debug可以直接按启动脚本的关键命令启动 `placement-api` 默认启动端口为8000，建议用endpoint的地址8778

启动命令为`placement-api --port 8778`

## 2.4. 具体处理过程

代码路径 `/usr/lib/python2.7/site-packages/placement`

- 路由

`handle.py`

```python
    '/allocation_candidates': {
        'GET': allocation_candidate.list_allocation_candidates,
    },
```

- controller
  - `handles/allocation_candidate.py`
  
  `list_allocation_candidates` -> `ac_obj.AllocationCandidates.get_by_requests` -> `_get_by_requests`

- 查数据库
  - `objects/research_context.py`
  `res_ctx.RequestGroupSearchContext` -> `get_providers_with_resource`
  直接看注释的mysql命令更好懂

  ```sql
      SELECT rp.id, rp.root_provider_id
    FROM resource_providers AS rp
    JOIN inventories AS inv
     ON rp.id = inv.resource_provider_id
     AND inv.resource_class_id=0
    LEFT JOIN (
     SELECT
       alloc.resource_provider_id,SUM(alloc.used) AS used
     FROM allocations AS alloc
     WHERE alloc.resource_class_id=0
     GROUP BY alloc.resource_provider_id
    ) AS usages
     ON inv.resource_provider_id = usages.resource_provider_id
    WHERE
     used + 64<= ((total - reserved) * inv.allocation_ratio)
     AND inv.min_unit <= 64
     AND inv.max_unit >= 64
     AND (64 % inv.step_size)=0
  ```
  
通过语句关键问题在inv.max_unit>=64 max_unit在数据表inventories中对应resource_class_id=0就是cpu，可以看到最大cpu为48，最大cpu个数即为物理机核数-预留核数
到这里placemnt部分debug完成了，剩下的问题是 **为什么最大值没有乘以超分倍数**

# 3. 思考

## 3.1. 为什么单个资源分配上限没有乘以超分倍数**

举个不太恰当的例子，一座桥上能跑很多很多小型汽车、双层汽车、大卡车等等宽度小于桥面的车，**当宽度大于桥面时就过不去了，车不能折叠**

回到计算机，操作系统分时使用CPU，每个核心在一个时钟周期类只执行一条CPU指令

- 当有48核心时同时能够执行48条指令，如果48八个物理核心虚拟出来了多个小于48和的虚拟核心，每个虚拟核心对应一个物理核心，当不同虚拟机同时执行指令时，需要排队等待获取CPU执行
- 当虚拟机大于48核心时，48个虚拟核心有分配到物理CPU，剩下的CPU如何分配CPU，有没有这么做的必要？没有必要，CPU核数就那么多，实际上虚拟机进程使用CPU和物理机使用CPU是一样的方式，但内存磁盘不同

## 3.2. 拓展问题

内存超分和磁盘超分是怎么实现的
内存、磁盘都是IO设备，通过地址映射使用的，一般情况下每个进程分配的空间不会用到上限，所以挤一挤总能上

- 内存超分
  内存生产环境不建议超分，不仅不能超分而且需要预留，内存不足时会触发物理机的oomkiiler开始杀进程，可能杀掉虚拟机进程，也可能杀掉物理机操作系统关键进程导致物理机操作系统宕机

- 磁盘超分
  有一些存储支持稀疏配置，也就是用多少占多少空间，而不是分配多少占多少空间，这样只要没占的都可以分配出去，造成的问题就是前期分的很爽，没做好控制，后期使用率都上去存储面临爆盘风险，陷入被动扩容。

# 4. 参考文档

- [关于僵尸实例排查参照](https://www.jianshu.com/p/48e1451cf7dc)
- [关于placement架构理解参照](https://blog.csdn.net/Jmilk/article/details/82980378)
- [pycharm远程调试OpenStack](https://www.jb51.net/article/128727.htm)
