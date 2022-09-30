---
layout: post
title: Kubesphere与Jenkins的集成解析
catalog: true
tag: [云原生, Devops, K8s]
---

<!-- TOC -->

- [1. kubesphere的devops模块介绍](#1-kubesphere的devops模块介绍)
- [2. 集成的亮点](#2-集成的亮点)
- [3. 具体集成说明](#3-具体集成说明)
  - [3.1. jenkins镜像构建](#31-jenkins镜像构建)
  - [3.2. jenkins与devops的部署](#32-jenkins与devops的部署)
    - [3.2.1. ks-devops-helm-chart](#321-ks-devops-helm-chart)
      - [3.2.1.1. devops](#3211-devops)
      - [3.2.1.2. jenkins](#3212-jenkins)
        - [3.2.1.2.1. jenkins配置](#32121-jenkins配置)
      - [3.2.1.3. jenkins pod初始化](#3213-jenkins-pod初始化)
      - [3.2.1.4. s2i](#3214-s2i)
  - [3.3. jenkins kubernets动态slave](#33-jenkins-kubernets动态slave)
    - [3.3.1. kubesphere内置jenkins](#331-kubesphere内置jenkins)
      - [3.3.1.1. 配置](#3311-配置)
      - [3.3.1.2. 使用](#3312-使用)
    - [3.3.2. 独立部署的jenkins](#332-独立部署的jenkins)
      - [3.3.2.1. cloud配置kubernetes](#3321-cloud配置kubernetes)
- [4. Q&A](#4-qa)
  - [4.1. 使用maven构建时，maven仓库如何配置](#41-使用maven构建时maven仓库如何配置)
  - [4.2. kubesphere->jenkins->kubernetes，认证是如何实现的](#42-kubesphere-jenkins-kubernetes认证是如何实现的)
    - [4.2.1. kubesphere与jenkin的认证](#421-kubesphere与jenkin的认证)
    - [4.2.2. jenkins使用驱动kubernetes实现动态slave](#422-jenkins使用驱动kubernetes实现动态slave)
- [5. 部署使用问题](#5-部署使用问题)
  - [5.1. 误删 apiserivice](#51-误删-apiserivice)
    - [5.1.1. 解决方式](#511-解决方式)
  - [5.2. kubesphere api服务无法启动](#52-kubesphere-api服务无法启动)
    - [5.2.1. 错误描述](#521-错误描述)
    - [5.2.2. 解决方式](#522-解决方式)
  - [5.3. jnlp 容器无法启动](#53-jnlp-容器无法启动)
- [6. 附录](#6-附录)
  - [6.1. 附录1: 认证备忘](#61-附录1-认证备忘)
- [7. 参考](#7-参考)

<!-- /TOC -->

# 1. kubesphere的devops模块介绍

- kubesphere 使用可插拔的 devops 模块实现devops功能
- devops驱动jenkins实现具体的操作，例如流水线等

devops与kubesphere的关系如下图, 详细的[组件介绍](https://kubesphere.io/docs/v3.3/introduction/architecture/)

![Architecture](https://pek3b.qingstor.com/kubesphere-docs/png/20190810073322.png)

# 2. 集成的亮点

devops与jenkins集成紧密且优雅，从构建、部署到使用维护纯云原生方式实现

- 一键部署
- 一个参数启用devops功能
- 一个k8s集群内即可完成从jenkins、流水线的全生命周期

# 3. 具体集成说明

用户使用kuberspere平台的devops功能时，调用devops-api发送请求，devops收到请求后，部分请求直接调用jenkins进行操作，部分请求通过更新devops-controller监听的资源，通过devops-controller来操作jenkins。
运行流水线阶段，jenkins配置了kubernetes动态slave

- jenkins pod信息(镜像、卷等)发送给k8s
- k8s启动jenkins slave pod并通过远程协议与jenkins master建立连接
- 运行流水线
- 运行完毕之后根据设置删除/保留创建的pod

![kubesphere-devops-3-0](/img/posts/Kubesphere与Jenkins的集成解析/kubesphere-devops-3-0.png)

## 3.1. jenkins镜像构建

jenkins本身是一个Java应用，当前也没有提供官方的云原生方案，kubesphere通过下面几个项目定制了自己的镜像

- [custom-war-packager](https://github.com/jenkinsci/custom-war-packager) 定制自己的jenkins并生成docker镜像或者war镜像
- [formulas](https://github.com/jenkins-zh/jenkins-formulas) 通过formula.yaml定制自己的jenkins，针对中国区优化
- [ks-jenkins](https://github.com/kubesphere/ks-jenkins/blob/master/formula.yaml) 定制了kubesphere自己的jenkins镜像 使用了jcli 集成了cwp

ks-devops项目中的formulas安装了所有需要的jenkins插件主要有

- blueocean 提供了jenkins的restful api
- [kubernetes](https://plugins.jenkins.io/kubernetes/) 提供了动态slave能力
- [kubesphere-token-auth](https://javadoc.jenkins.io/plugin/kubesphere-token-auth/) 集成kubesphere权限体系
- 其他

## 3.2. jenkins与devops的部署

- ks-install(ansible) 生成环境变量，主要有
  - 要不要使用 ksauth
  - 生成 ksauth使用的密码
  - 主要环境变量在 https://github.com/kubesphere/ks-installer/blob/master/roles/ks-devops/templates/ks-devops-values.yaml.j2
- helm部署 devops 和 [jenkins helm项目](https://github.com/kubesphere-sigs/ks-devops-helm-chart)

### 3.2.1. ks-devops-helm-chart

这个项目里面主要有三个chart

#### 3.2.1.1. devops

部署 devops-api 和 devops-controller

> 注意⚠️ 这里有一个cronjob 作用为清理执行过的流水线记录，定期执行 `ks pip gc`

主要部署的资源有

- deployment
  - devops-apiserver
  - devops-controller
- cronjob
  - devops
- configmap
  - devops-config
  - jenkins-agent-config

#### 3.2.1.2. jenkins

##### 3.2.1.2.1. jenkins配置

- maven配置 charts/ks-devops/charts/jenkins/templates/jenkins-agent-config.yaml
- 配置jenkins dynamic slave charts/ks-devops/charts/jenkins/templates/jenkins-casc-config.yml
  - 自动化配置jenkins Configure Clouds
  - 配置k8s认证
  - 配置pod volume image lable等
  - 会将上面的maven配置挂在到maven容器中
- 将maven配置和casc配置以configmap的方式挂在到jenkins容器的 /var/jenkins_home/casc_configs， 从helm的value获取
- value.yaml charts/ks-devops/charts/jenkins/values.yaml中定义了:
  - 环境变量 从value读出配置到容器中，设置了登陆用的用户名密码
  - 初始化脚本 -- 在helm渲染jenkins deploy时挂载到configmap中
    - mariler插件 -- 绑定邮箱
    - kubernetes插件 -- 创建k8s credential
    - RBAC配置

#### 3.2.1.3. jenkins pod初始化

- kubernetes插件配置 charts/ks-devops/charts/jenkins/templates/config.yaml
  - config.xml
    - 创建 role `kubesphere-user` 所有资源只读 并绑定到`authenticated`用户
    - ladp配置，对接kubesphere ladp
    - cloud配置
      - pod template container配置、挂载等等，包括maven配置、docker sock等
  - apply_config.sh jenkins工作目录初始化
    - slave-to-master-security-kill-switch 禁用agent 访问控制机制
    - 拷贝config.xml 到容器 /var/jenkins_home
    - 将用于初始化的groovy文件拷贝到 /var/jenkins_home/init.groovy.d
      - 初始化用于cloud的 credential `Kubernetes service account`
      - Mailer模块初始化
      - RBAC初始化: 创建admin和kubesphere-user对应的role 做绑定
      - Sonarqube初始化
      - 用户初始化，创建admin用户并设置密码

- deployment charts/ks-devops/charts/jenkins/templates/jenkins-master-deployment.yaml
  - 设置环境变量
    - jvm参数
    - admin用户名密码
    - 超时配置
    - 邮箱配置
  - limit **注意: 默认memory为2g，一般是不够用的，跑多个任务就会引起pod crash，所以至少设置成4g**
  - initContainer 运行/var/jenkins_config/apply_config.sh 初始化jenkins配置，如安装插件、配置cloud、rbac等等

到这里 jenkins pod就创建出来了，我们可以直接开始使用jenkins运行流水线了

#### 3.2.1.4. s2i

source to image 这个组件没怎么用

## 3.3. jenkins kubernets动态slave

### 3.3.1. kubesphere内置jenkins

#### 3.3.1.1. 配置

kubesphere通过ks-install和helm都配置好了，无需单独配置

#### 3.3.1.2. 使用

以流水线为例，groovy中添加以下字段会按照 `'base'` 去匹配pod的lable，匹配到了会使用这个label的pod模板启动pod运行流水线，下面有两个pipeline 脚本，第一个是选定了pod的模板的会启动一个pod来执行，第二个any， 如果设置了master节点为 `Only build jobs with label expressions matching this node` 将会启动base pod来运行，如果选择 `Use this node as much as possible` 则会在jenkins自身的容器/服务器上运行, 如果是普通job的话 勾选`Restrict where this project can be run` 且填写 `Label Expression` 选择要运行的label，和pipeline类似

```groovy
pipeline {
    agent {
        node {
            label 'base'
        }
    }
    stages {
        stage('Run shell') {
            steps {
                sh 'echo hello world'
            }
        }
    }
}
```

```groovy
pipeline {
    agent any
    stages {
        stage('Run shell') {
            steps {
                sh 'echo hello world'
            }
        }
    }
}
```

### 3.3.2. 独立部署的jenkins

#### 3.3.2.1. cloud配置kubernetes

Manage Node ——> Configure Clouds

- kubernetes
- Kubernetes URL: kubernetes apiserver地址， 与kubesphere自带的jenkins不同的是使用了集群内部链接
- Kubernetes Namespace: slave pod运行的namespace
- Credentials: 使用secret file，上传 kubeconfig 与kubesphere自带的jenkins不同的是使用了`kubernetes service account`
- 使用websocket通信 -- 使用jenkins tunnel通信
- Jenkins URL: jenkins api地址
- 其他: 其他的pod配置按需配置即可，这里和kubesphere的一样

# 4. Q&A

## 4.1. 使用maven构建时，maven仓库如何配置

pod所使用的maven配置是挂载进去的，可以通过Jenkins->Configuration->Maven Project Configuration 配置

## 4.2. kubesphere->jenkins->kubernetes，认证是如何实现的

### 4.2.1. kubesphere与jenkin的认证

- jenkins插件 `kubesphere-token-auth-plugin` 集成kubesphere的认证体系，在kubesphere调用jenkins时，都需要经过ks-apiserver进行token的review, 通过之后再调用jenkins执行实际动作

![kubesphere-devops-auth](/img/posts/Kubesphere与Jenkins的集成解析/kubesphere-devops-auth.png)

### 4.2.2. jenkins使用驱动kubernetes实现动态slave

- jenkins的deployment中声明了 `serviceAccountName` `devops-jenkins`
- 启动的pod会讲 对应serviceAccount的token写入pod文件系统中 `/var/run/secrets/kubernetes.io/serviceaccount/token`
- jenkins kubernetes插件会去读pod文件系统的token，这样就可以通过 token 来调度k8s资源实现 slave pod的创建删除

如果是外置jenkins则无法通过读取token来连接kubernetes，需要手动创建serviceAccount、clusterRole、clusterRoleBinding, 然后将token以Secret text或者将ca证书以Secret file形式或将kubconfig以Secret file形式写入credentials

# 5. 部署使用问题

## 5.1. 误删 apiserivice

```bash
kubectl delete --all apiservice
```

### 5.1.1. 解决方式

> 参照 https://github.com/kubernetes/kubernetes/issues/75704

- 将kube-apiserver.yaml移到其他文件夹，这时kube-apiserver的pod会down掉

```bash
mv /etc/kubernetes/manifests/kube-apiserver.yaml /etc/kubernetes/
```

- 在其他api正常的节点删除这个pod，再将配置文件移回去,即可恢复

## 5.2. kubesphere api服务无法启动

> kubesphere v3.3.0

### 5.2.1. 错误描述

> install failed, ks-controller CrashLoopBackOff

```log
E1116 00:55:15.113761 1 notification_controller.go:113] get /, Kind= informer error, no matches for kind "Config" in version "notification.kubesphere.io/v2beta1"
F1116 00:55:15.113806 1 server.go:340] unable to register controllers to the manager: no matches for kind "Config" in version "notification.kubesphere.io/v2beta1"
```

### 5.2.2. 解决方式

> 参照 [kubectl apply -f https://raw.githubusercontent.com/kubesphere/notification-manager/master/config/bundle.yaml](https://github.com/kubesphere/kubesphere/issues/4447)

```bash
kubectl apply -f https://raw.githubusercontent.com/kubesphere/notification-manager/master/config/bundle.yaml
```

## 5.3. jnlp 容器无法启动

jnlp是jenkin的远程调用协议

JNLP(JAVA NETWORK LAUNCH PROTOCOL) is used to Connect to/launch your java application( here Jenkins) from a remote location

```bash
[root@k8s-1 ~]# kubectl logs -f  -n kubesphere-devops-worker base-w9dpq jnlp
Warning: SECRET is defined twice in command-line arguments and the environment variable
Warning: AGENT_NAME is defined twice in command-line arguments and the environment variable
Sep 14, 2022 11:29:43 AM hudson.remoting.jnlp.Main createEngine
INFO: Setting up agent: base-w9dpq
Sep 14, 2022 11:29:44 AM hudson.remoting.jnlp.Main$CuiListener <init>
INFO: Jenkins agent is running in headless mode.
Sep 14, 2022 11:29:44 AM hudson.remoting.Engine startEngine
INFO: Using Remoting version: 4.10
Sep 14, 2022 11:29:44 AM org.jenkinsci.remoting.engine.WorkDirManager initializeWorkDir
INFO: Using /home/jenkins/agent/remoting as a remoting work directory
Sep 14, 2022 11:29:44 AM org.jenkinsci.remoting.engine.WorkDirManager setupLogging
INFO: Both error and output logs will be printed to /home/jenkins/agent/remoting
Sep 14, 2022 11:29:44 AM hudson.remoting.jnlp.Main$CuiListener status
INFO: Locating server among [http://172.16.80.38:8080/]
Sep 14, 2022 11:29:44 AM hudson.remoting.jnlp.Main$CuiListener error
SEVERE: http://172.16.80.38:8080/tcpSlaveAgentListener/ is invalid: 404 Not Found
java.io.IOException: http://172.16.80.38:8080/tcpSlaveAgentListener/ is invalid: 404 Not Found
	at org.jenkinsci.remoting.engine.JnlpAgentEndpointResolver.resolve(JnlpAgentEndpointResolver.java:219)
	at hudson.remoting.Engine.innerRun(Engine.java:724)
	at hudson.remoting.Engine.run(Engine.java:540)
```

```log
Sep 14, 2022 11:33:41 AM hudson.remoting.jnlp.Main$CuiListener status
INFO: Locating server among [http://devops-jenkins.kubesphere-devops-system:80/]
Sep 14, 2022 11:33:42 AM org.jenkinsci.remoting.engine.JnlpAgentEndpointResolver resolve
INFO: Remoting server accepts the following protocols: [JNLP4-connect, Ping]
Sep 14, 2022 11:33:42 AM org.jenkinsci.remoting.engine.JnlpAgentEndpointResolver resolve
INFO: Remoting TCP connection tunneling is enabled. Skipping the TCP Agent Listener Port availability check
Sep 14, 2022 11:33:42 AM hudson.remoting.jnlp.Main$CuiListener status
INFO: Agent discovery successful
  Agent address: devops-jenkins-agent.kubesphere-devops-system
  Agent port:    50000
  Identity:      13:ea:2b:ab:b5:16:70:70:89:58:d1:66:2b:62:b1:16
Sep 14, 2022 11:33:42 AM hudson.remoting.jnlp.Main$CuiListener status
INFO: Handshaking
Sep 14, 2022 11:33:42 AM hudson.remoting.jnlp.Main$CuiListener status
INFO: Connecting to devops-jenkins-agent.kubesphere-devops-system:50000
Sep 14, 2022 11:33:42 AM hudson.remoting.jnlp.Main$CuiListener status
INFO: Trying protocol: JNLP4-connect
Sep 14, 2022 11:33:42 AM org.jenkinsci.remoting.protocol.impl.BIONetworkLayer$Reader run
INFO: Waiting for ProtocolStack to start.
Sep 14, 2022 11:33:46 AM hudson.remoting.jnlp.Main$CuiListener status
INFO: Remote identity confirmed: 13:ea:2b:ab:b5:16:70:70:89:58:d1:66:2b:62:b1:16
Sep 14, 2022 11:33:46 AM hudson.remoting.jnlp.Main$CuiListener status
INFO: Connected
Sep 14, 2022 11:33:58 AM org.csanchez.jenkins.plugins.kubernetes.KubernetesSlave$SlaveDisconnector call
INFO: Disabled agent engine reconnects.
```

# 6. 附录

## 6.1. 附录1: 认证备忘

- ks-installer/roles/ks-core/init-token/tasks/main.yaml 生成一个随机值secret
- 部署kubersphere的时候初始化通过 ks-installer/roles/ks-core/init-token/files/jwt-script/jwt.sh 生成了一个jwt token 入参为 上面生成的字符串和'{"email": "admin@kubesphere.io","username": "admin","token_type": "static_token"}'
- 通过生成的token和secret创建secret名为 kubesphere-secret
- 部署devops的时候将填入authentication.jwtSecret devops.password 通过helm部署devops
- 部署jenkin的密码为写死的"P@ssw0rd"
- admin password生成了一个随机的22位字符串写入jenkins pod环境变了并通过读取configmap devops-jenkins启动jenkins

# 7. 参考

[jcli](https://github.com/jenkins-zh/jenkins-cli)
[jcli使用手册](https://www.bookstack.cn/read/jenkins-cli-0.0.29-zh/263348)
[custom-war-packager](https://github.com/jenkinsci/custom-war-packager)
[jenkins kubernetes插件](https://plugins.jenkins.io/kubernetes/)
[KubeSphere DevOps 3.0 流水线开发指南](https://kubesphere.com.cn/forum/d/2393-kubesphere-devops-30)
[Jenkins 基于Kubernetes动态创建pod](https://blog.csdn.net/qq_34556414/article/details/120623844)
[Can I use Jenkins kubernetes plugin when Jenkins server is outside of a kubernetes cluster?](https://stackoverflow.com/questions/40197607/can-i-use-jenkins-kubernetes-plugin-when-jenkins-server-is-outside-of-a-kubernet)
[kubernetes-jenkins-integration](https://stackoverflow.com/questions/48827345/kubernetes-jenkins-integration)