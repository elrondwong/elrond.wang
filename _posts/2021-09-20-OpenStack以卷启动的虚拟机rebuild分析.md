---
layout: post
title: OpenStack卷启动的虚拟机rebuild无效
catalog: true
tag: [OpenStack, Ceph]
---

<!-- TOC -->

- [1. 描述](#1-描述)
- [2. 相关资料](#2-相关资料)
- [3. 修复方案](#3-修复方案)
- [4. 分析过程](#4-分析过程)

<!-- /TOC -->
# 1. 描述

卷启动的虚拟机rebuild无效，rebuild卷启动的虚拟机相当于仅仅对虚拟机进行了一次重启。

-  版本信息如下

  |组件|版本|
  |:-:|:-:|
  |openstack|train|
  |nova|20.6.0-1|

看了好几遍rebuild的代码，都没有找到对卷启动的云硬盘重建或者数据清空的操作，最后找了下社区，找到了一些资料，原来社区没实现对卷启动虚拟机的rebuild，怪不得没找到相关的代码逻辑。

# 2. 相关资料

社区要实现一个专门用于卷启动虚拟机重装的api, 还没实现，但是已经有不是非常严谨的修复方案可以先用。

[bp](https://blueprints.launchpad.net/nova/+spec/volume-backed-server-rebuild)
[specs](https://specs.openstack.org/openstack/nova-specs/specs/train/approved/volume-backed-server-rebuild.html)

# 3. 修复方案

将原来的卷删除掉，使用重装的虚拟机创建一个新的卷挂给虚拟机，实现代码 [Replace root volume during rebuild](https://review.opendev.org/c/openstack/nova/+/305079/22/nova/compute/manager.py#2617)

# 4. 分析过程

总体代码流程就是一路rpc调用+检查，到计算节点上执行 `nova.compute.manager.rebuild_instance` -> `_do_rebuild_instance` -> `_rebuild_default_impl`

- 卸载块存储、更新镜像元数据、销毁虚拟机

-> `virt/libvirt/driver.spawn` -- 这一步又个创建镜像，卷启动的虚拟机不会创建镜像，所以rebuild实际上只是 关机->卸载磁盘->开机->挂盘

- 创建新的虚拟机
- 挂载块存储
- 启动
- 重建成功

具体代码流程可以参考这里，[rebuild时序图](https://raw.githubusercontent.com/int32bit/openstack-workflow/master/output/nova/rebuild.png)

修复之后功能OK
