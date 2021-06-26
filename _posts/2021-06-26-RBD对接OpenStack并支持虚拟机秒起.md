---
layout: post
title: OpenStack集成Ceph
catalog: true
tag: [Ceph, OpenStack]
---

<!-- TOC -->

- [1. 版本说明](#1-版本说明)
- [2. 前期准备](#2-前期准备)
  - [2.1. 创建keyring](#21-创建keyring)
  - [2.2. 创建pool](#22-创建pool)
  - [2.3. 安装依赖包](#23-安装依赖包)
- [3. glance 对接](#3-glance-对接)
  - [3.1. glance-api.conf](#31-glance-apiconf)
- [4. 对接nova](#4-对接nova)
  - [4.1. libvirt配置](#41-libvirt配置)
  - [4.2. nova conf配置](#42-nova-conf配置)
- [5. 对接cinder](#5-对接cinder)
  - [5.1. `/etc/cinder/cinder.conf`](#51-etccindercinderconf)
  - [5.2. virsh secret(所有计算节点)](#52-virsh-secret所有计算节点)
  - [5.3. 配置cinder type](#53-配置cinder-type)
- [6. 测试](#6-测试)
  - [6.1. 上传镜像](#61-上传镜像)
  - [6.2. 以镜像创建虚拟机](#62-以镜像创建虚拟机)
  - [6.3. 秒起测试](#63-秒起测试)
    - [6.3.1. 感官上很快一台虚拟机在数秒之内创建完成](#631-感官上很快一台虚拟机在数秒之内创建完成)
    - [6.3.2. 查看虚拟机系统盘rbd卷是否有父卷](#632-查看虚拟机系统盘rbd卷是否有父卷)
- [7. Thoubleshooting](#7-thoubleshooting)
  - [7.1. 由于原本安装了高版本的librbd1以及依赖包，导致需要安装ceph-common时版本冲突无法安装](#71-由于原本安装了高版本的librbd1以及依赖包导致需要安装ceph-common时版本冲突无法安装)

<!-- /TOC -->

# 1. 版本说明

> glance对接cinder
> nova对接cinder
> cinder 对接ceph rbd

|组件名称|组件版本|rpm版本|
|:-:|:-:|:-:|
|openstack|Train|
|ceph|nautilus|14.2.21|
|glance|19.0.4|openstack-glance-19.0.4-1.el7.noarch|
|cinder|15.6.0|openstack-cinder-15.6.0-1.el7.noarch|
|nova|20.6.0|openstack-nova-api-20.6.0-1.el7.noarch|

# 2. 前期准备

## 2.1. 创建keyring

```bash
cd /etc/ceph/ceph.conf
cp  ceph.client.admin.keyring  ceph.client.cinder.keyring
ceph auth import -i  ceph.client.cinder.keyring
# 然后将/etc/ceph/ceph.client.cinder.keyring 拷贝到每个客户端主机上
```

## 2.2. 创建pool

> pg 算法查看 https://ceph.io/pgcalc/
> 原则上每个pg数据量在4g左右较为合理，建议按这个标准来算
> pg太多会导致集群cpu性能损耗过大，io响应变长
> pg太少会导致单个pg数据量过大 osd数据极不均衡

```bash
# glance池存放镜像不多pg个数适当小一点
ceph osd pool create images 32 32
# volumes存储占比比较大所有适当给大点
ceph osd pool create volumes 512 512
```

## 2.3. 安装依赖包

最好安装和ceph集群相同版本的ceph-common 版本差别过大的话cephx认证消息组织可能不一样导致认证不过去无法连接到集群

```bash
yum intall ceph-common -y
```

# 3. glance 对接

## 3.1. glance-api.conf

```ini
[DEFAULT]
# 虚拟机秒起设置
show_image_direct_url = True
show_multiple_locations = True
# ------------
limit_param_default = 10000
[glance_store]
default_store = cinder
stores = cinder
cinder_store_auth_address = http://controller1:5000/v3
cinder_store_user_name = cinder
cinder_store_password = cinder
cinder_store_project_name = service
cinder_os_region_name = RegionOne
```

重启服务

```bash
systemctl restart openstack-glance-registry.service openstack-glance-api.service
# 检查服务
systemctl status openstack-glance-registry.service openstack-glance-api.service
```

# 4. 对接nova

## 4.1. libvirt配置

- `/etc/libvirt/qemu.conf`

```ini
max_processes = 131072
max_files = 32768
```

- /etc/libvirt/libvirt.conf

```ini
listen_tls = 0
listen_tcp = 1
auth_tcp = "none"
log_level = 3
log_outputs = "3:file:/var/log/libvirt/libvirtd.log"
```

```bash
systemctl restart libvirtd
```

## 4.2. nova conf配置

- `/etc/nova/nova.conf`

```ini
[cinder]
os_region_name = RegionOne
auth_url = http://controller1:5000/v3
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = cinder

[libvirt]
virt_type = kvm
images_rbd_ceph_conf = /etc/ceph/ceph.conf
os_region_name = RegionOne
```

```bash
systemctl restart openstack-nova-compute
```

# 5. 对接cinder

## 5.1. `/etc/cinder/cinder.conf`

```ini
[DEFAULT]
# 虚拟机秒起设置
allowed_direct_url_schemes = cinder
# ----------
enabled_backends=lvm,ceph-hdd
default_volume_type = ceph-hdd
[ceph-hdd]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
rbd_pool = volumes
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_flatten_volume_from_snapshot = false
rbd_max_clone_depth = 5
rbd_store_chunk_size = 4
rados_connect_timeout = -1
glance_api_version = 2
rbd_user = cinder
# 这个uuid可以随机生成，但要和secret的保持一致
rbd_secret_uuid = aa03e7e8-6fcc-443f-94aa-ac169bfd0fd5
volume_backend_name=ceph-hdd
```

## 5.2. virsh secret(所有计算节点)

-  创建secret配置
>  注意⚠️: <uuid>xx</uuid> 中的xx要和cinder配置中的一致

`virsh.xml`

```xml
<secret ephemeral='no' private='no'>
  <uuid>aa03e7e8-6fcc-443f-94aa-ac169bfd0fd5</uuid>
  <usage type='ceph'>
     <name>client.cinder secret</name>
  </usage>
</secret>
```

- 导入secret

```bash
virsh secret-define --file virsh.xml
```

- 设置secret key

```bash
# 获取value
ceph auth get client.cinder
# 输出如下
[client.cinder]
	key = AQDk18FgMo7NABAA4ufuz3O6/0lE4vsVgHs1yQ==
...
# 设置value
virsh secret-set-value aa03e7e8-6fcc-443f-94aa-ac169bfd0fd5 AQDk18FgMo7NABAA4ufuz3O6/0lE4vsVgHs1yQ==
```

## 5.3. 配置cinder type

```bash
openstack volume type create ceph-hdd --property volume_backend_name='ceph-hdd'
```

# 6. 测试

## 6.1. 上传镜像

> 注意  nova对接cinder之后只能上传raw格式
> TIP 镜像格式转换命令

```bash
# qcow2 to raw
qemu-img convert -f qcow2 -O raw  ubuntu-20.04-server-cloudimg-amd64.img ubuntu-20.04.raw
```

上传

```bash
glance image-create --name ubuntu --file ubuntu-20.04.raw --container-format bare --disk-format raw --visibility public --progress
```

## 6.2. 以镜像创建虚拟机

如果成功则对接完成

## 6.3. 秒起测试
### 6.3.1. 感官上很快一台虚拟机在数秒之内创建完成

### 6.3.2. 查看虚拟机系统盘rbd卷是否有父卷

有输出就是秒起

```bash
# 589cf738-176c-4ba0-a935-de2dfce8de80 用虚拟机uuid代替
rbd info volumes/volume-$(nova volume-attachments 589cf738-176c-4ba0-a935-de2dfce8de80|grep vda|awk '{print $2}')|grep parent
```

# 7. Thoubleshooting

## 7.1. 由于原本安装了高版本的librbd1以及依赖包，导致需要安装ceph-common时版本冲突无法安装

- 禁用掉原来的ceph仓库
- 增加新的ceph仓库

`cat /etc/yum.repos.d/ceph.repo`

```ini
[Ceph]
name=Ceph packages for $basearch
baseurl=http://download.ceph.com/rpm-nautilus/el7/$basearch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc
priority=1

[Ceph-noarch]
name=Ceph noarch packages
baseurl=http://download.ceph.com/rpm-nautilus/el7/noarch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc
priority=1

[ceph-source]
name=Ceph source packages
baseurl=http://download.ceph.com/rpm-nautilus/el7/SRPMS
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc
priority=1
```

- 降级包

```bash
yum downgrade librbd1.x86_64 librados2 python3-rados librbd1 python3-rbd python3-rados librados2 -y
```

- 安装ceph-common

```bash
yum install ceph-common -y
```