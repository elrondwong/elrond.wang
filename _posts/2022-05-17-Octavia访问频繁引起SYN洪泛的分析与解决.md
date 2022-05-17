---
layout: post
title: Octavia访问频繁引起SYN洪泛的分析与解决
catalog: true
tag: [OpenStack]
---
<!-- TOC -->

- [1. 问题](#1-问题)
- [2. 排查](#2-排查)
	- [2.1. gdb定位下代码卡在哪里](#21-gdb定位下代码卡在哪里)
	- [2.2. dmesg 日志](#22-dmesg-日志)
	- [2.3. strace查看系统调用](#23-strace查看系统调用)
	- [2.4. 查看连接状态](#24-查看连接状态)
	- [2.5. 查看socket统计信息](#25-查看socket统计信息)
- [3. 解决](#3-解决)
- [4. 总结](#4-总结)
- [5. 参考](#5-参考)

<!-- /TOC -->

# 1. 问题

octavia-api进程跑一会就卡死，api无响应，telnet也无响应

# 2. 排查

思路主要是

- 查应用代码
- 查系统日志得出触发了syn flooding
- 根据tcp/socket知识找到问题原因

## 2.1. gdb定位下代码卡在哪里

```python
(gdb) py-list
 442
 443                buf.seek(0, 2)  # seek end
 444                self._rbuf = StringIO()  # reset _rbuf.  we consume it via buf.
 445                while True:
 446                    try:
>447                        data = self._sock.recv(self._rbufsize)
 448                    except error, e:
 449                        if e.args[0] == EINTR:
 450                            continue
 451                        raise
 452                    if not data:

(gdb) py-bt
    self.process_request(request, client_address)
#32 Frame 0x7fd926f4de18, for file /usr/lib64/python2.7/SocketServer.py, line 238, in serve_forever (self=<WSGIServer(RequestHandlerClass=<classobj at remote 0x7fd931382598>, base_environ={'CONTENT_LENGTH': '', 'SERVER_NAME': 'controller1', 'GATEWAY_INTERFACE': 'CGI/1.1', 'SCRIPT_NAME': '', 'SERVER_PORT': '9876', 'REMOTE_HOST': ''}, _BaseServer__shutdown_request=False, server_name='controller1', server_address=('172.16.10.10', 9876), application=<RecursiveMiddleware(application=<CORS(application=<HTTPProxyToWSGI(application=<SkippingAuthProtocol(_service_token_roles=set(['service']), _service_token_roles_required=False, _interface='admin', log=<KeywordArgumentAdapter(logger=<Logger(name='keystonemiddleware.auth_token', parent=<Logger(name='keystonemiddleware', parent=<---Type <return> to continue, or q <return> to quit---
RootLogger(name='root', parent=None, handlers=[<WatchedFileHandler(stream=<file at remote 0x7fd92849f660>, level=0, lock=<_RLock(_Verbose__verbose=False, _RLock__owner=None, _RLock__block=<thread.lock at remote 0x7fd92adc3570>, _RLock__count=0) at ...(truncated)
    self._handle_request_noblock()
#36 Frame 0x7fd9284b4608, for file /usr/lib/python2.7/site-packages/octavia/cmd/api.py, line 46, in main (app=<RecursiveMiddleware(application=<CORS(application=<HTTPProxyToWSGI(application=<SkippingAuthProtocol(_service_token_roles=set(['service']), _service_token_roles_required=False, _interface='admin', log=<KeywordArgumentAdapter(logger=<Logger(name='keystonemiddleware.auth_token', parent=<Logger(name='keystonemiddleware', parent=<RootLogger(name='root', parent=None, handlers=[<WatchedFileHandler(stream=<file at remote 0x7fd92849f660>, level=0, lock=<_RLock(_Verbose__verbose=False, _RLock__owner=None, _RLock__block=<thread.lock at remote 0x7fd92adc3570>, _RLock__count=0) at remote 0x7fd92847b650>, encoding=None, dev=2053L, _name=None, ino=201451105, baseFilename='/var/log/octavia/api.log', mode='a', filters=[], formatter=<ContextFormatter(project='octavia', datefmt='%Y-%m-%d %H:%M:%S', version='unknown', _fmt='%(asctime)s.%(msecs)03d %(process)d %(levelname)s %(name)s [%(request_id)s %(user_identity)s] %(inst...(truncated)
    srv.serve_forever()
```

看起来正常监听

## 2.2. dmesg 日志

```log
[Sun May 15 20:16:52 2022] TCP: request_sock_TCP: Possible SYN flooding on port 9876. Sending cookies.  Check SNMP counters.
[Sun May 15 20:22:22 2022] TCP: request_sock_TCP: Possible SYN flooding on port 9876. Sending cookies.  Check SNMP counters.
```

octavia监听端口触发 syn flooding，通过这个信息可以知道问题处在tcp连接

## 2.3. strace查看系统调用

```bash
strace -o output.txt -T -tt -e trace=all -p 101847
```

```log
17:48:20.962664 fcntl(10, F_SETFL, O_RDWR) = 0 <0.000006>
17:48:20.962702 recvfrom(10, "\7\0\0\1\0\0\0\0\0\0\0", 8192, 0, NULL, NULL) = 11 <0.000007>
17:48:20.962738 fcntl(10, F_GETFL)      = 0x2 (flags O_RDWR) <0.000006>
17:48:20.962761 fcntl(10, F_SETFL, O_RDWR) = 0 <0.000006>
17:48:20.962819 select(0, NULL, NULL, NULL, {tv_sec=0, tv_usec=0}) = 0 (Timeout) <0.000006>
17:48:20.969238 sendto(5, "HTTP/1.0 200 OK\r\n", 17, 0, NULL, 0) = 17 <0.000019>
17:48:20.969337 sendto(5, "Date: Mon, 16 May 2022 09:48:20 "..., 37, 0, NULL, 0) = 37 <0.000008>
17:48:20.969401 sendto(5, "Server: WSGIServer/0.1 Python/2."..., 37, 0, NULL, 0) = 37 <0.000006>
17:48:20.969439 sendto(5, "Content-Length: 2934\r\nContent-Ty"..., 122, 0, NULL, 0) = 122 <0.000006>
17:48:20.969473 sendto(5, "{\"loadbalancers\": [{\"provider\": "..., 2934, 0, NULL, 0) = 2934 <0.000013>
17:48:20.969525 stat("/etc/localtime", {st_mode=S_IFREG|0644, st_size=528, ...}) = 0 <0.000030>
17:48:20.969597 write(2, "172.16.204.135 - - [16/May/2022 "..., 92) = 92 <0.000012>
17:48:20.969661 shutdown(5, SHUT_WR)    = 0 <0.000013>
17:48:20.969703 close(5)                = 0 <0.000011>
17:48:20.969744 select(5, [4], [], [], {tv_sec=0, tv_usec=500000}) = 1 (in [4], left {tv_sec=0, tv_usec=499997}) <0.000009>
17:48:20.969786 accept(4, {sa_family=AF_INET, sin_port=htons(49511), sin_addr=inet_addr("172.16.205.227")}, [16]) = 5 <0.000010>
17:48:20.969849 recvfrom(5,
```

卡在revcfrom 接受客户端请求，到这里不太有思路了回到上面的tcp flooding

## 2.4. 查看连接状态

```bash
netstat -anolp|grep 9876 -w
tcp        6      0 172.16.10.10:9876      0.0.0.0:*               LISTEN      168014/python        keepalive (0.03/0/0)
tcp        0      0 172.16.10.10:9876      172.16.83.77:37110      SYN_RECV    -                    on (0.23/0/0)
tcp        0      0 172.16.10.10:9876      172.16.83.77:36202      SYN_RECV    -                    on (0.23/2/0)
tcp        0      0 172.16.10.10:9876      172.16.83.77:37090      SYN_RECV    -                    on (0.00/0/0)
tcp        0      0 172.16.10.10:9876      172.16.83.77:36308      SYN_RECV    -                    on (3.43/3/0)
tcp        0      0 172.16.10.10:9876      172.16.83.77:36302      SYN_RECV    -                    on (3.43/3/0)
tcp        0      0 172.16.10.10:9876      172.16.83.77:36301      SYN_RECV    -                    on (3.43/3/0)
tcp        0      0 172.16.10.10:9876      172.16.83.77:36934      SYN_RECV    -                    on (3.83/2/0)
tcp        0      0 172.16.10.10:9876      172.16.83.77:36348      SYN_RECV    -                    on (4.63/3/0)
tcp        0      0 172.16.10.10:9876      172.16.83.77:36178      SYN_RECV    -                    on (4.63/3/0)
tcp      384      0 172.16.10.10:9876      172.16.83.77:36962      ESTABLISHED -                    off (0.00/0/0)
tcp      384      0 172.16.10.10:9876      172.16.83.77:36732      ESTABLISHED -                    off (0.00/0/0)
tcp        0      0 172.16.10.10:9876      172.16.83.77:59818      TIME_WAIT   -                    timewait (29.91/0/0)
tcp        0      0 172.16.10.10:9876      172.16.83.77:33976      TIME_WAIT   -                    timewait (52.32/0/0)
```

有几个连接是SYN_RECV状态，syn flooding现象，但连接数量不多，再看看线程属性

## 2.5. 查看socket统计信息

```bash
ss -anlop|grep octavia-api                                                                    Tue May 17 11:27:51 2022

tcp    LISTEN     6 	 5    172.16.10.10:9876                  *:*                   users:(("octavia-api",pid=173917,fd=4))
```

第三列、第四列为

- Recv-Q 接收队列 
  - LISTEN 状态 表示当前等待服务器调用accept完成三次握手的Listen backlog值
  - 其他状态 接收队列的字节数
- Send-Q 发送队列一般创建socket监听时，维护一个syn队列 
  - LISTEN状态 表示最大的listen backlog值
  - 其他状态 发送队列中的字节数

![tcp-half-open](https://www.researchgate.net/profile/Puneet-Kumar-10/publication/344490561/figure/fig3/AS:943562816495619@1601974314485/TCP-Half-Open-Connection.png)

客户端通过socket连接向服务端发送syc包时维护一个等待队列
进入半连接状态，如果等待队列满了，服务端就会丢弃，客户端返回连接超时，只要客户端没有收到syn+ack，客户会多次尝试，表现在体验上面就是超时无响应

看到这里在参考下面红帽的链接，就可以知道问题所在: octavia-api在启动监听套接字时backlog太小了，如果一次性的请求大于5个且持续请求就处理不过来，所以这里将octavia-api启动的backlog改大

```bash
# python listen签名
Socket.listen(backlog)
```

# 3. 解决

找到octavia-api启动时的backlog配置

cmd.api.svc.sevrve_forever ->
wsgiref.simple_server.WSGIServer ->
lib64/BaseHTTPServer.HTTPServer ->
lib64/SocketServer.TCPServer ->

```python
request_queue_size = 5
```

简单改的话地方有如下三个，他们之间是继承关系，该其中任何一个都行，但基于最小化改动原则，simple_server 是相比其他两个掉用最少的，所以修改 simple_server

- wsgiref.simple_server.WSGIServer
- lib64/BaseHTTPServer.HTTPServer
- lib64/SocketServer.TCPServer

```python
# /usr/lib64/python2.7/wsgiref/simple_server.py
class WSGIServer(HTTPServer):

    """BaseHTTPServer that implements the Python WSGI protocol"""

    request_queue_size = 64 #新增

    application = None

```

修改完重启 octavia-api 即可，验证使用如下命令

```bash
ss -anlop|grep octavia-api                                                                    Tue May 17 11:27:51 2022

tcp    LISTEN    0 	 64    172.16.10.10:9876                  *:*                   users:(("octavia-api",pid=173917,fd=4))
```

第三列64即生效,这里改大可能会影响系统其他进程，由于操作系统总的队列数是通过如下两个参数限制的,改完可以粗略的计算下，如果把所有ss输出listen状态的加起来小于下面输出则不用改下面两个参数，如果大于则需要适量调整

```
sysctl -n net.core.somaxconn
sysctl -n net.ipv4.tcp_max_syn_backlog
```


# 4. 总结

- api设计时应该考虑多线程，后续octavia-api改写可能可以参考nova-api
- 管控面API调用不应该太频繁，并发量不应该太大

# 5. 参考

- [kernel: Possible SYN flooding on port #. Sending cookies.](https://access.redhat.com/solutions/30453)
- [https://topic.alibabacloud.com/a/ss-command-and-recv-q-and-send-q-states_8_8_31266407.html](https://topic.alibabacloud.com/a/ss-command-and-recv-q-and-send-q-states_8_8_31266407.html)
- [TCP 半连接队列和全连接队列满了会发生什么？又该如何应对？](https://www.cnblogs.com/xiaolincoding/p/12995358.html)
