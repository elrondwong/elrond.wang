

[TOC]
# 版本

|      软件名      |     版本     |   备注   |
| :--------------: | :----------: | :------: |
|      docker      |   19.03.14   |          |
|    kubernetes    |    1.19.7    |          |
|       ceph       |   14.2.21    | nautilus |
|     ceph-csi     | release-v3.3 |          |
| external-storage |    master    |          |

# 块存储

## 准备

- 创建ceph pool

```bash
ceph osd pool create k8s 64 64
rbd pool init k8s
```

- 获取ceph keyring

```bash
ceph auth get client.admin
# 输出如下
key = AQDk18FgMo7NABAA4ufuz3O6/0lE4vsVgHs1yQ==
```

- 获取clusterid

```bash
ceph -s
# 输出如下
    id:     8cfb6405-d75e-466a-8abf-51ba0480d783
...
```

- 获取mon信息

```bash
ceph mon stat
# 输出如下
e2: 3 mons at {ceph01=[v2:172.16.81.237:3300/0,v1:172.16.81.237:6789/0],ceph02=[v2:172.16.81.238:3300/0,v1:172.16.81.238:6789/0],ceph03=[v2:172.16.81.239:3300/0,v1:172.16.81.239:6789/0]}, election epoch 10, leader 0 ceph01, quorum 0,1,2 ceph01,ceph02,ceph03
```

- 每个客户端安装`ceph-common`

```bash
yum install ceph-common -y 
```

## csi模式--当前使用

### 配置configmap

- 生成configmap

  修改字段
  - clusterID
  - monitors
```yaml
cat <<EOF > csi-config-map.yaml
---
apiVersion: v1
kind: ConfigMap
data:
  config.json: |-
    [
      {
        "clusterID": "8cfb6405-d75e-466a-8abf-51ba0480d783",
        "monitors": [
          "172.16.81.237:6789",
          "172.16.81.238:6789",
          "172.16.81.239:6789"
        ]
      }
    ]
metadata:
  name: ceph-csi-config
EOF
```

- 导入configmap

```bash
kubectl apply -f csi-config-map.yaml
```

- 配置kms

```yaml
cat <<EOF > csi-kms-config-map.yaml
---
apiVersion: v1
kind: ConfigMap
data:
  config.json: |-
    {}
metadata:
  name: ceph-csi-encryption-kms-config
EOF
```

```bash
kubectl apply -f csi-kms-config-map.yaml
```

### 配置secret

- 生成secret
  修改字段：
  - userID
  - userKey
```yaml
cat <<EOF > csi-rbd-secret.yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: csi-rbd-secret
  namespace: default
stringData:
  userID: admin
  userKey: AQDk18FgMo7NABAA4ufuz3O6/0lE4vsVgHs1yQ==
EOF
```

- 导入secret

```bash
kubectl apply -f csi-rbd-secret.yaml
```

### 配置rbac


```bash
kubectl apply -f https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/csi-provisioner-rbac.yaml
kubectl apply -f https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/csi-nodeplugin-rbac.yaml
```

### 配置provisioner和node plugins

> 注意⚠️：国内无法访问 `k8s.gcr.io`需要将下面两个文件中的 `k8s.gcr.io/sig-storage` 替换为 `quay.io/cephcsi`

```bash
wget https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/csi-rbdplugin-provisioner.yaml
wget https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/csi-rbdplugin.yaml
for i in $(grep -rn 'k8s.gcr.io' deploy|awk -F':' '{print $1}'|uniq);do sed -i 's/k8s.gcr.io\/sig-storage/quay.io\/cephcsi/g' $i;done
kubectl apply -f  csi-rbdplugin-provisioner.yaml
kubectl apply -f csi-rbdplugin.yaml
```

### 配置storageclass

- 创建配置
  需要修改的参数：
  - pool

```yaml
cat <<EOF > csi-rbd-sc.yaml
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: csi-rbd-sc
provisioner: rbd.csi.ceph.com
parameters:
   clusterID: b9127830-b0cc-4e34-aa47-9d1a2e9949a8
   pool: k8s
   imageFeatures: layering
   csi.storage.k8s.io/provisioner-secret-name: csi-rbd-secret
   csi.storage.k8s.io/provisioner-secret-namespace: default
   csi.storage.k8s.io/controller-expand-secret-name: csi-rbd-secret
   csi.storage.k8s.io/controller-expand-secret-namespace: default
   csi.storage.k8s.io/node-stage-secret-name: csi-rbd-secret
   csi.storage.k8s.io/node-stage-secret-namespace: default
reclaimPolicy: Delete
allowVolumeExpansion: true
mountOptions:
- discard
EOF
```

- 导入

```bash
kubectl apply -f csi-rbd-sc.yaml
```

## external-storage模式 -- 版本陈旧不再使用

### 配置provisioner

```yaml
cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rbd-provisioner
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rbd-provisioner
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: rbd-provisioner
    spec:
      containers:
      - name: rbd-provisioner
        image: "quay.io/external_storage/rbd-provisioner:latest"
        env:
        - name: PROVISIONER_NAME
          value: ceph.com/rbd
EOF
```

```bash
kubectl apply -f  deployment.yaml
```

> 使用的ceph 14但容器里面的ceph-common版本为13，需要将其升级为14才可以正常创建pvc

### 创建clusterstorage

需要修改的参数

- monitors
- name
- pool
- userId

```yaml
cat <<EOF > class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rbd
provisioner: kubernetes.io/rbd
parameters:
  monitors: 172.16.81.237:6789,172.16.81.238:6789,172.16.81.239:6789
  adminId: admin
  adminSecretName: ceph-secret
  adminSecretNamespace: kube-system
  pool: k8s
  userId: admin
  userSecretName: ceph-secret
  userSecretNamespace: default
  fsType: ext4
  imageFormat: "2"
  imageFeatures: "layering"
```

```bash
kubectl apply -f class.yaml
```

### 创建secret

需要修改的参数:

- key: 执行 `ceph auth get-key client.admin | base64` 获取输出

```yaml
cat <<EOF > secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret
  namespace: kube-system
type: "kubernetes.io/rbd"
data:
  # ceph auth get-key client.admin | base64
  key: QVFEazE4RmdNbzdOQUJBQTR1ZnV6M082LzBsRTR2c1ZnSHMxeVE9PQ==
```

```bash
kubectl apply -f secret.yaml
```

## 测试

### 创建pvc

```yaml
cat <<EOF > raw-block-pvc.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: raw-block-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Block
  resources:
    requests:
      storage: 1Gi
  storageClassName: csi-rbd-sc
EOF
```

```bash
kubectl apply -f raw-block-pod.yaml
```

验证是否创建成功

```bash
kubectl get pvc raw-block-pvc
# 输出如下即正常
raw-block-pvc   Bound    pvc-a0fb907f-e067-443c-89fe-127a794c8f1b   1Gi        RWO            csi-rbd-sc     13h
```

### 创建pod使用刚创建的sc

```yaml
cat <<EOF > raw-block-pvc.yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-raw-block-volume
spec:
  containers:
    - name: fc-container
      image: fedora:26
      command: ["/bin/sh", "-c"]
      args: ["tail -f /dev/null"]
      volumeDevices:
        - name: data
          devicePath: /dev/xvda
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: raw-block-pvc
EOF
```

```bash
kubectl apply -f raw-block-pvc.yaml
```

验证创建是否成功

```bash
kubectl get po rbd-provisioner-7f85d94d97-mkzdf
# 输出如下
NAME                               READY   STATUS    RESTARTS   AGE
rbd-provisioner-7f85d94d97-mkzdf   1/1     Running   0          17h
```

验证是否挂载到容器

```bash
kubectl exec -it pod-with-raw-block-volume lsblk
# 输出如下
# rbd0 即为刚申请的pvc
rbd0   252:0    0    1G  0 disk
```

# 对象存储

对象存储直接使用七层协议 http https即可，无需做复杂的csi

# 文件系统

## 准备工作

### 创建一个文件系统

- 创建两个pool 一个存文件系统数据另一个存文件系统元数据

```bash
ceph osd pool create cephfs_metadata 32 32
ceph osd pool create cephfs_data 256 256
```

- 创建fs

> 后面会用到fs_name

```bash
# ceph fs new <fs_name> <metadata> <data>
ceph fs new cephfs cephfs_metadata cephfs_data
```

### 获取集群信息

- 获取clusterid

```bash
ceph -s
# 输出如下
    id:     8cfb6405-d75e-466a-8abf-51ba0480d783
```

- 获取secret

```bash
ceph auth get client.admin
# 输出如下
[client.admin]
	key = AQDk18FgMo7NABAA4ufuz3O6/0lE4vsVgHs1yQ==
```

- 获取mon信息

```bash
ceph mon stat
# 输出如下
e2: 3 mons at {ceph01=[v2:172.16.81.237:3300/0,v1:172.16.81.237:6789/0],ceph02=[v2:172.16.81.238:3300/0,v1:172.16.81.238:6789/0],ceph03=[v2:172.16.81.239:3300/0,v1:172.16.81.239:6789/0]}, election epoch 10, leader 0 ceph01, quorum 0,1,2 ceph01,ceph02,ceph03
```

## 部署ceph-csi-cephfs

- [服务部署配置](https://github.com/ceph/ceph-csi/tree/devel/deploy/cephfs/kubernetes)

- [客户端使用配置](https://github.com/ceph/ceph-csi/tree/devel/examples/cephfs)

- [部署说明](https://github.com/ceph/ceph-csi/blob/devel/docs/deploy-cephfs.md)

> 部署时可以将整个git项目下载下来在项目的基础上修改`git clone git@github.com:ceph/ceph-csi.git -b release-v3.3`

下载下来之后进入 `deploy/cephfs/kubernetes`

## configmap

configmap只给出了模板需要自己填写下

修改：

- clusterID
- monitors

```yaml
cat <<EOF > csi-config-map.yaml
---
apiVersion: v1
kind: ConfigMap
data:
  config.json: |-
    [
      {
        "clusterID": "8cfb6405-d75e-466a-8abf-51ba0480d783",
        "monitors": [
          "172.16.81.237:6789",
          "172.16.81.238:6789",
          "172.16.81.239:6789"
        ]
      }
    ]
metadata:
  name: ceph-csi-config
EOF
```



```bash
kubectl apply -f csi-config-map.yaml
```

### rbac

```
kubectl create -f csi-provisioner-rbac.yaml
kubectl create -f csi-nodeplugin-rbac.yaml
```

### provisioner

部署provisioner的时候需要启动容器,默认的镜像时由于`[kubernetes-csi](https://github.com/kubernetes-csi)`中provisioner使用的镜像地址为 `k8s.gcr.io/sig-storage/csi-provisioner` ，这个地址在中国大陆无法访问，需要手动修改 `k8s.gcr.io/sig-storage` 为 `quay.io/k8scsi` 

```bash
sed -i 's/k8s.gcr.io\/sig-storage/quay.io\/k8scsi/g' csi-cephfsplugin-provisioner.yaml
sed -i 's/k8s.gcr.io\/sig-storage/quay.io\/k8scsi/g' csi-cephfsplugin.yaml
kubectl create -f csi-cephfsplugin-provisioner.yaml
kubectl create -f csi-cephfsplugin.yaml
```

### 确认部署成功

```bash
kubectl get po|grep cephfs
# 输出如下
csi-cephfsplugin-47z57                          3/3     Running   0          66m
csi-cephfsplugin-7ttvw                          3/3     Running   0          65m
csi-cephfsplugin-8g9c2                          3/3     Running   0          66m
csi-cephfsplugin-provisioner-55859c9ff7-g7nx4   6/6     Running   0          64m
csi-cephfsplugin-provisioner-55859c9ff7-hqn2r   6/6     Running   0          64m
csi-cephfsplugin-provisioner-55859c9ff7-vlc6b   6/6     Running   0          64m
```

> 到这里 `deploy/cephfs/kubernetes`部分我们部署完了，切个目录`examples/cephfs/`开始部署客户端
>
> 到这里 `deploy/cephfs/kubernetes`部分我们部署完了，切个目录`examples/cephfs/`开始部署客户端

## 客户端配置

### secret

修改:

- userID
- userKey
- adminID
- adminKey

```yaml
cat <<EOF > secret.yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: csi-cephfs-secret
  namespace: default
stringData:
  # Required for statically provisioned volumes
  userID: admin
  userKey: AQDk18FgMo7NABAA4ufuz3O6/0lE4vsVgHs1yQ==

  # Required for dynamically provisioned volumes
  adminID: admin
  adminKey: AQDk18FgMo7NABAA4ufuz3O6/0lE4vsVgHs1yQ==
```

```bash
kubectl apply -f secret.yaml
```

### storageclass

修改

- clusterID
- fsName

```yaml
cat <<EOF > storageclass.yaml
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: csi-cephfs-sc
provisioner: cephfs.csi.ceph.com
parameters:
  clusterID: 8cfb6405-d75e-466a-8abf-51ba0480d783
  fsName: cephfs
  csi.storage.k8s.io/provisioner-secret-name: csi-cephfs-secret
  csi.storage.k8s.io/provisioner-secret-namespace: default
  csi.storage.k8s.io/controller-expand-secret-name: csi-cephfs-secret
  csi.storage.k8s.io/controller-expand-secret-namespace: default
  csi.storage.k8s.io/node-stage-secret-name: csi-cephfs-secret
  csi.storage.k8s.io/node-stage-secret-namespace: default
reclaimPolicy: Delete
allowVolumeExpansion: true
mountOptions:
  - debug
EOF
```



```bash
kubectl apply -f storageclass.yaml
```

## 测试

### 创建pvc

```bash
kubectl apply -f pvc.yaml
# 稍等片刻查看
kubectl get pvc csi-cephfs-pvc
# 输出如下
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS    AGE
csi-cephfs-pvc   Bound    pvc-cb6988b1-3dde-4227-80ff-5f549709b768   1Gi        RWX            csi-cephfs-sc   62m
```

### 创建pod

```bash
kubectl apply -f pod.yaml
# 查看
kubectl exec -it csi-cephfs-demo-pod df
# 输出如下 可以看出以cephfs方式挂载
# 且目录为/volumes/csi/csi-vol-0af68b6a-cfff-11eb-b335-fad30d8a412c/c721c454-23d6-4d73-8a4f-6d051e322487
172.16.81.237:6789,172.16.81.238:6789,172.16.81.239:6789:/volumes/csi/csi-vol-0af68b6a-cfff-11eb-b335-fad30d8a412c/c721c454-23d6-4d73-8a4f-6d051e322487   1048576       0   1048576   0% /var/lib/www
```


# 参考

- [ceph-csi](https://github.com/ceph/ceph-csi)
- [external-storage](https://github.com/kubernetes-retired/external-storage)
- [Block Devices and Kubernetes](https://docs.ceph.com/en/latest/rbd/rbd-kubernetes/)

# Troubleshooting

## pvc 一直在pending状态--csi容器无法调度

### 现象

```bash
kubectl get pvc
NAME            STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
raw-block-pvc   Pending                                      csi-rbd-sc     8m17s
```

### 排查过程

- describe pvc

```bash
kubectl describe pvc raw-block-pvc
# 关键错误
waiting for a volume to be created, either by external provisioner "rbd.csi.ceph.com" or manually created by system administrator
```

- 查看 rbdplugin pod

```bash
kubectl get po
NAME                                        READY   STATUS    RESTARTS   AGE
csi-rbdplugin-provisioner-9db69594c-gkvfv   0/7     Pending   0          56m
csi-rbdplugin-provisioner-9db69594c-s5sxk   0/7     Pending   0          56m
csi-rbdplugin-provisioner-9db69594c-zc9lt   0/7     Pending   0          56m
```

- describe po

```bash
kubectl describe po csi-rbdplugin-provisioner-9db69594c-gkvfv
# 关键错误
0/3 nodes are available: 3 node(s) had taint {node-role.kubernetes.io/master: }, that the pod didn't tolerate
```

原因是三个节点都没有去污点 所以没有节点可以调度

### 解决方式

去污点

```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
```

## pvc 一直在pending状态 -- provisioner版本太低

- 查看pvc

```bash
kubectl describe pvc claim1
# 关键错误
Failed to provision volume with StorageClass "rbd": failed to create rbd image: executable file not found in $PATH, command output:
```

- 查看provisioner

```bash
kubectl logs rbd-provisioner-7f85d94d97-mkzdf
# 错误信息
W0617 11:31:58.969042       1 controller.go:746] Retrying syncing claim "default/claim1" because failures 3 < threshold 15
E0617 11:31:58.969097       1 controller.go:761] error syncing claim "default/claim1": failed to provision volume with StorageClass "rbd": failed to create rbd image: exit status 13, command output: did not load config file, using default settings.
2021-06-17 11:31:55.915 7fc8835c1900 -1 Errors while parsing config file!
2021-06-17 11:31:55.915 7fc8835c1900 -1 parse_file: cannot open /etc/ceph/ceph.conf: (2) No such file or directory
2021-06-17 11:31:55.915 7fc8835c1900 -1 parse_file: cannot open /root/.ceph/ceph.conf: (2) No such file or directory
2021-06-17 11:31:55.915 7fc8835c1900 -1 parse_file: cannot open ceph.conf: (2) No such file or directory
2021-06-17 11:31:55.916 7fc8835c1900 -1 Errors while parsing config file!
2021-06-17 11:31:55.916 7fc8835c1900 -1 parse_file: cannot open /etc/ceph/ceph.conf: (2) No such file or directory
2021-06-17 11:31:55.916 7fc8835c1900 -1 parse_file: cannot open /root/.ceph/ceph.conf: (2) No such file or directory
2021-06-17 11:31:55.916 7fc8835c1900 -1 parse_file: cannot open ceph.conf: (2) No such file or directory
2021-06-17 11:31:55.953 7fc8835c1900 -1 auth: unable to find a keyring on /etc/ceph/ceph.client.admin.keyring,/etc/ceph/ceph.keyring,/etc/ceph/keyring,/etc/ceph/keyring.bin,: (2) No such file or directory
2021-06-17 11:31:58.958 7fc8835c1900 -1 monclient: get_monmap_and_config failed to get config
2021-06-17 11:31:58.958 7fc8835c1900 -1 auth: unable to find a keyring on /etc/ceph/ceph.client.admin.keyring,/etc/ceph/ceph.keyring,/etc/ceph/keyring,/etc/ceph/keyring.bin,: (2) No such file or directory
rbd: couldn't connect to the cluster!
```

通过上面的错误信息没有找到头绪

- 登入provisioner pod进行debug

将ceph配置和keyring拷贝到pod里面 执行ceph -s 报错

```log
-1 monclient: get_monmap_and_config failed to get config
```

开启ceph debug

`ceph -s --debug-auth=20/20` 看到结果是认证错误，在查看ceph版本
rbd-provisioner 用的ceph版本是13 应该是版本不同消息结构不同导致的，所以升级到14
升级到14之后解决问题

# 总结

官方文档已非常详尽，主要遇到两个问题

- External-storage 方式太久没人维护镜像中ceph-common版本停留在13无法和高版本的服务端认证成功，需要手动将容器的ceph-common升级到14

  > - 同样的ceph-csi如果没有正常工作，当检查了一切配置确保正常之后, 我们需要要去检查版本，但是csi使用的provisioner镜像不能通过命令行去debug(现在还没找到方法)，可以通过dockerfile去对版本，基础镜像配置在git项目中的`build.env`目录 `BASE_IMAGE`配置和`CEPH_VERSION` 如果不匹配则去quay.io找相对应的版本

- csi方式部署时由于镜像地址被墙，需要想其他办法，例如文中提到的

  - 换镜像地址
  - 网上很多人使用docker proxy解决
  - 自己打镜像也成

  更换镜像地址是最简单的