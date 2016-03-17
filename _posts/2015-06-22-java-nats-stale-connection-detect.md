---
layout: post
title:  "解决java-nats不能重连的问题"
date:   2015-06-22 13:34:55
categories: java
tags: java network
---
PaaS项目中使用了nats作为Messaging， 客户端都使用java-nats和nats-server进行通信。今天机房发生停电之后，发现很多nats client不能重新连上nats-server.
在nats-server端查看TCP connction, 发现对应客户端的连接不存在，但是到了对应的nats-client (如10.101.10.20)可以看到连接到连接是正常的。
例如0.101.0.11:4222 ESTABLISHED状态。这样nats-server发送到 client的消息超时，实际上连接并不存在。

解决方法： nats-client和server直接增加心跳，如果断连则重连。

问题：什么情况下nats-server (or tcp 被连接方)会将原先的连接删除？是在检查到原先连接不正常的情况下会自动删除旧连接么
解答:  nats-server会每隔2分钟和nats-client发送一个PING包，用来保持回话，如果两次都没有PANG包应答，nats-server认为连接失效，会把连接删除。

注 ： tcp本身协议层面会定时检查连接是否正常，通过设置keepalive_timeout keepalive_intl等参数，系统会扫描正常连接并检查连接是否正常，线上系统默认是7200，即是说2个小时以后才会发现正常连接已经broken。但是这需要在代码中enable KEEPALIVE，linux并不认为keepalive是系统层级的设置，而是面向应用中得socker设置。所以需要应用程序自己 enable KEEPALIVE。并且也可以对当前使用的socket的keepalive_time, interval, probles进行定制
可以参考: http://tldp.org/HOWTO/TCP-Keepalive-HOWTO/addsupport.html

nats-client使用的netty版本默认并不enable KEEPALIVE.

```java
    Bootstrap b = new Bootstrap(); // (1) 
    b.group(workerGroup); // (2)  
    b.channel(NioSocketChannel.class); // (3)  
    b.option(ChannelOption.SO_KEEPALIVE, true); // (4)  
```java


猜测是B1机房断电，导致10.101.0.11到以下几台机器的网络出现故障，从而引起nats-server将原来的连接删除。而nats-client没有引入心跳机制，并不能感应网络故障。

```ruby
    def send_ping
      return if @closing
      if @pings_outstanding > NATSD::Server.ping_max
        error_close UNRESPONSIVE
        return
      end
      queue_data(PING_RESPONSE)
      flush_data
      @pings_outstanding += 1
    end
```

nats-server kill掉，nats-client 收到socker FIN包，nats-client系统发起事件，java-nats(netty)传递给Listener，从而通知NatsImpl重连
socker FIN包收不到的情况： 1)网络丢包 2)nats-server断电 (没有办法做清除)， 3)kernel panic等

在这种情况下，没有办法依赖server端发起的FIN包，达到重连。需要client端自己检测，检测的方法有两种

1）依赖系统TCP协议栈提供的功能，通过设置较短的keepalive值，使得应用程序感知连接有错；这种方法需要改动java-nats的代码 (java-nats默认keepalive=false，即不对连接做检测)。这种情况下，nats-client系统会发起事件清理掉旧连接(selector.CONNECT事件)，从而达到通知NatsImpl重连的作用。

2)  定时从client向nats-server发送心跳,如果检测到连接有问题，则重新连接。

结论是不能完全依赖server端的FIN包达到连接重置的目标。一定要在client端引入心跳机制，自动重连。

```java
    Thread [nioEventLoopGroup-2-3](Suspended (breakpoint at line 171 in NatsImpl$1))    
    NatsImpl$1.operationComplete(ChannelFuture) line: 171    
    NatsImpl$1.operationComplete(Future) line: 168    
    DefaultPromise<V>.notifyListener0(Future, GenericFutureListener) line: 680    
    DefaultPromise<V>.notifyListeners0(Future<?>, DefaultFutureListeners) line: 603    
    DefaultChannelPromise(DefaultPromise<V>).notifyListeners() line: 563    
    DefaultChannelPromise(DefaultPromise<V>).tryFailure(Throwable) line: 424    
    AbstractNioByteChannel$NioByteUnsafe(AbstractNioChannel$AbstractNioUnsafe).fulfillConnectPromise(ChannelPromise, Throwable) line: 268    
    AbstractNioByteChannel$NioByteUnsafe(AbstractNioChannel$AbstractNioUnsafe).finishConnect() line: 284    
    NioEventLoop.processSelectedKey(SelectionKey, AbstractNioChannel) line: 528    
    NioEventLoop.processSelectedKeysOptimized(SelectionKey[]) line: 468    
    NioEventLoop.processSelectedKeys() line: 382    
    NioEventLoop.run() line: 354    
    SingleThreadEventExecutor$2.run() line: 116    
    DefaultThreadFactory$DefaultRunnableDecorator.run() line: 137    
    FastThreadLocalThread(Thread).run() line: 745    
```java

通过设置iptables来测试，模拟nats-server的FIN包被drop掉得情形。

```bash
    iptables -A INPUT -p tcp -s 10.101.0.11/32 --dport 59482 DROP
    net=src+dst    port=src port + dst port
    tcpdump -nXXvv net 10.101.0.11 port 4222
    tcpdump -nXXvv port 4222
```bash

