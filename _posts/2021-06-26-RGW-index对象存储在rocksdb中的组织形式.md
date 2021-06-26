---
layout: post
title: rgw index对象存储在rocksdb中的组织形式
catalog: true
tag: [Ceph, Rocksdb, Leveldb]
---

<!-- TOC -->

- [0.1. 定位index对象](#01-定位index对象)
- [0.2. 分析rocksdb](#02-分析rocksdb)

<!-- /TOC -->

## 0.1. 定位index对象

- 找到某个桶的bucket index对象

```bash
radosgw-admin bucket stats --bucket test
"id": "c96a8bc3-c206-46a9-9f4a-71f80f7a8e95.24169.1"
```
- 查到他在哪个osd

```bash
ceph osd map default.rgw.buckets.index  .dir.c96a8bc3-c206-46a9-9f4a-71f80f7a8e95.24169.1
osdmap e38 pool 'default.rgw.buckets.index' (6) object '.dir.c96a8bc3-c206-46a9-9f4a-71f80f7a8e95.24169.1' -> pg 6.9c155c0e (6.6) -> up ([0], p0) acting ([0], p0)
```

- 讲对应osd的omap拷贝出来分析

## 0.2. 分析rocksdb

- 找到index相关记录

```bash
ceph-kvstore-tool  rocksdb  . list|grep "c96a8bc3-c206-46a9-9f4a-71f80f7a8e9"
_HOBJTOSEQ_	%25edir%25ec96a8bc3-c206-46a9-9f4a-71f80f7a8e95%25e24169%25e1...head.6.9C155C0E
_HOBJTOSEQ_	c96a8bc3-c206-46a9-9f4a-71f80f7a8e95%25e24169%25e1%25uprog%25ecc...head.7.C6FBF03C
_HOBJTOSEQ_	c96a8bc3-c206-46a9-9f4a-71f80f7a8e95%25e24169%25e1%25uvenv-x86%25etar%25egz...head.7.E71D4EC7
```

三条输出 分别对应2个对象和index本身 第一个为index本身
get 出来看看

```
00000000  02 01 b2 00 00 00 31 00  00 00 00 00 00 00 00 00  |......1.........|
00000010  00 00 00 00 00 00 01 00  00 00 00 00 00 00 02 00  |................|
00000020  01 01 12 00 00 00 01 00  00 00 00 00 00 00 00 00  |................|
00000030  00 00 00 ff ff ff ff ff  fe ff ff ff ff ff ff ff  |................|
00000040  06 03 5c 00 00 00 00 00  00 00 31 00 00 00 2e 64  |..\.......1....d|
00000050  69 72 2e 63 39 36 61 38  62 63 33 2d 63 32 30 36  |ir.c96a8bc3-c206|
00000060  2d 34 36 61 39 2d 39 66  34 61 2d 37 31 66 38 30  |-46a9-9f4a-71f80|
00000070  66 37 61 38 65 39 35 2e  32 34 31 36 39 2e 31 fe  |f7a8e95.24169.1.|
00000080  ff ff ff ff ff ff ff 0e  5c 15 9c 00 00 00 00 00  |........\.......|
00000090  06 00 00 00 00 00 00 00  ff ff ff ff ff ff ff ff  |................|
000000a0  ff 00 01 01 10 00 00 00  00 00 00 00 00 00 00 00  |................|
000000b0  00 00 00 00 00 00 00 00                           |........|
000000b8
```

- key值组成说明

.log 
Record * N 
Record= 7B header(crc *4 + len *2 + type *1 (full/fisrt/middle/last)) + 12B (seq*8+count*4) +  Data ( kType(del/set) * 1 + data len +DATA )
del = 0 +key_len + key_str
set =  1 + key_len+ key_string + value_len+ value_string

```
00000000  38 a9 ba 05 20 00 01 01  00 00 00 00 00 00 00 01  |8... ...........|
00000010  00 00 00 01 06 6b 65 79  31 31 31 0b 76 61 6c 75  |.....key111.valu|
00000020  65 32 32 32 32 32 32 dc  9d b4 3d 14 00 01 02 00  |e222222...=.....|
00000030  00 00 00 00 00 00 01 00  00 00 00 06 6b 65 79 31  |............key1|
00000040  31 31                                             


38 a9 ba 05  -- crc32
20 00 -- 32
01    -kTypeFull

01 00 00 00 00 00 00 00 seq 8
01 00 00 00  count 4

01 kTypeValue  set_key
06 --key len 
6b 65 79 31 31 31 key111
0b value len 11
76 61 6c 75 65 32 32 32 32 32 32     


setset
dc 9d b4 3d
14 00
01 

02 00 00 00 00 00 00 00
01 00  00 00

00  kTypeDelete
06  len 
6b 65 79 31 31 31            
```

- 分析当前实例

```
# crc
02 01 b2 00
# len
00 00
# type
31
# playload
00  00 00 00 00 00 00 00 00
00 00 00 00 00 00 01 00
```

type 16进制为 49

所以对应的index的对象 _USER_0000000000000049_USER_

```bash
ceph-kvstore-tool  rocksdb  . list|grep _USER_0000000000000049_USER_
_USER_0000000000000049_USER_	prog.cc
_USER_0000000000000049_USER_	venv-x86.tar.gz
```

通过index找到了该桶的两个对象

- 通过获取user再能获取到什么信息

```bash
ceph-kvstore-tool  rocksdb  . get _USER_0000000000000049_USER_ venv-x86.tar.gz
00000000  08 03 dd 00 00 00 0f 00  00 00 76 65 6e 76 2d 78  |..........venv-x|
00000010  38 36 2e 74 61 72 2e 67  7a 03 00 00 00 00 00 00  |86.tar.gz.......|
00000020  00 01 05 03 6b 00 00 00  01 0b 8f f2 02 00 00 00  |....k...........|
00000030  00 73 34 60 60 c2 19 02  25 22 00 00 00 34 35 36  |.s4``...%"...456|
00000040  62 32 33 39 30 37 30 37  32 63 62 32 66 32 65 36  |b23907072cb2f2e6|
00000050  38 39 39 39 64 37 30 39  39 30 34 35 34 2d 34 05  |8999d70990454-4.|
00000060  00 00 00 61 64 6d 69 6e  05 00 00 00 61 64 6d 69  |...admin....admi|
00000070  6e 12 00 00 00 61 70 70  6c 69 63 61 74 69 6f 6e  |n....application|
00000080  2f 78 2d 67 7a 69 70 0b  8f f2 02 00 00 00 00 00  |/x-gzip.........|
00000090  00 00 00 00 00 00 00 00  00 00 00 01 01 02 00 00  |................|
000000a0  00 07 03 0e 2d 00 00 00  63 39 36 61 38 62 63 33  |....-...c96a8bc3|
000000b0  2d 63 32 30 36 2d 34 36  61 39 2d 39 66 34 61 2d  |-c206-46a9-9f4a-|
000000c0  37 31 66 38 30 66 37 61  38 65 39 35 2e 32 34 31  |71f80f7a8e95.241|
000000d0  36 39 2e 32 32 00 00 00  00 00 00 00 00 00 00 00  |69.22...........|
000000e0  00 00 00                                          |...|
000000e3
```
按照最后一个字符串可以查到对应的index和文件的每个part

```bash
[root@VM_1_121_centos ~]# rados ls -p default.rgw.buckets.index|grep c206-46a9-9f4a
.dir.c96a8bc3-c206-46a9-9f4a-71f80f7a8e95.24169.1
[root@VM_1_121_centos ~]# rados ls -p default.rgw.buckets.data|grep c206-46a9-9f
c96a8bc3-c206-46a9-9f4a-71f80f7a8e95.24169.1__multipart_venv-x86.tar.gz.2~Hb8FYkvEZUWmUZsOJzpM375GdT7Rd3A.3
c96a8bc3-c206-46a9-9f4a-71f80f7a8e95.24169.1_prog.cc
c96a8bc3-c206-46a9-9f4a-71f80f7a8e95.24169.1__multipart_venv-x86.tar.gz.2~Hb8FYkvEZUWmUZsOJzpM375GdT7Rd3A.1
c96a8bc3-c206-46a9-9f4a-71f80f7a8e95.24169.1__multipart_venv-x86.tar.gz.2~Hb8FYkvEZUWmUZsOJzpM375GdT7Rd3A.2
c96a8bc3-c206-46a9-9f4a-71f80f7a8e95.24169.1__shadow_venv-x86.tar.gz.2~Hb8FYkvEZUWmUZsOJzpM375GdT7Rd3A.3_2
c96a8bc3-c206-46a9-9f4a-71f80f7a8e95.24169.1__shadow_venv-x86.tar.gz.2~Hb8FYkvEZUWmUZsOJzpM375GdT7Rd3A.3_1
c96a8bc3-c206-46a9-9f4a-71f80f7a8e95.24169.1__shadow_venv-x86.tar.gz.2~Hb8FYkvEZUWmUZsOJzpM375GdT7Rd3A.2_1
c96a8bc3-c206-46a9-9f4a-71f80f7a8e95.24169.1__shadow_venv-x86.tar.gz.2~Hb8FYkvEZUWmUZsOJzpM375GdT7Rd3A.1_3
c96a8bc3-c206-46a9-9f4a-71f80f7a8e95.24169.1__multipart_venv-x86.tar.gz.2~Hb8FYkvEZUWmUZsOJzpM375GdT7Rd3A.4
c96a8bc3-c206-46a9-9f4a-71f80f7a8e95.24169.1__shadow_venv-x86.tar.gz.2~Hb8FYkvEZUWmUZsOJzpM375GdT7Rd3A.1_1
c96a8bc3-c206-46a9-9f4a-71f80f7a8e95.24169.1__shadow_venv-x86.tar.gz.2~Hb8FYkvEZUWmUZsOJzpM375GdT7Rd3A.1_2
c96a8bc3-c206-46a9-9f4a-71f80f7a8e95.24169.1__shadow_venv-x86.tar.gz.2~Hb8FYkvEZUWmUZsOJzpM375GdT7Rd3A.3_3
c96a8bc3-c206-46a9-9f4a-71f80f7a8e95.24169.1_venv-x86.tar.gz
c96a8bc3-c206-46a9-9f4a-71f80f7a8e95.24169.1__shadow_venv-x86.tar.gz.2~Hb8FYkvEZUWmUZsOJzpM375GdT7Rd3A.2_3
c96a8bc3-c206-46a9-9f4a-71f80f7a8e95.24169.1__shadow_venv-x86.tar.gz.2~Hb8FYkvEZUWmUZsOJzpM375GdT7Rd3A.2_2
```

最后 omap过大的原因是一个index下存了太多文件导致index对象所在的osd在db中记录也特别多, 最简单的方式是在初始化对象存储的设置好分片和quota, 避免单个分片/桶对象过多，经实践建议

- 单分片对象小于10w
- 单桶设置128分片
- 单桶1280w对象

以上主要在数据异常恢复场景和omap过大导致故障的时候可以有排查处理思路
