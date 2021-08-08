---
layout: post
title: Ubuntu 镜像APT Update无法成功
catalog: true
tag: [Linux,Ubuntu,Docker]
---

<!-- TOC -->

- [1. 问题描述](#1-问题描述)
- [2. 解决方式](#2-解决方式)

<!-- /TOC -->

# 1. 问题描述

|组件|版本|备注|
|Ubuntu Docker Image|20.04||

使用镜像时无论是镜像默认源还是清华源、阿里源都报错

```log
Certificate verification failed: The certificate is NOT trusted. The certificate issuer is unknown.  Could not handshake: Error in the certificate verification
```

google了半天，该试的方法都试了，最后想到直接去 stackoverflow 查下，第一条就是

[Error by trying installing docker repository on linux ubuntu 18.04 LTS](https://stackoverflow.com/questions/54901780/error-by-trying-installing-docker-repository-on-linux-ubuntu-18-04-lts)

计算机使用类常规问题还是问stackoverflow要比google来的快

# 2. 解决方式

```bash
touch /etc/apt/apt.conf.d/99verify-peer.conf \
&& echo >>/etc/apt/apt.conf.d/99verify-peer.conf "Acquire { https::Verify-Peer false }"
```

原因还是网络策略问题，前端可能有代理之类的东西对https进行解密和重新加密。
