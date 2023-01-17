---
id: 344
title: Dubbo面试大纲
date: '2019-09-07T17:10:43+08:00'
author: Smile
layout: post
guid: 'http://codercoder.cn/?p=344'
image: http://codercoder.cn/wp-content/uploads/2019/09/2019-09-0815.png
permalink: /index.php/2019/09/dubbo-interview/
views:
    - '11277'
    - '11277'
    - '11277'
    - '11277'
    - '11277'
    - '11277'
categories:
    - Tech
tags:
    - dubbo
    - Tech
    - 面试
---

### Dubbo

#### 简介

 本篇文章不是进行详细的Dubbo实现以及原理分析的文章，适用于用过Dubbo，对Dubbo有一定了解准备面试的小伙伴阅读。下面列的一些点，如果能在面试时候说到，那面试官肯定觉得不错了。

##### 服务暴露

1.从xml读取ServiceBean配置，订阅了spring容器上下文刷新事件进行export动作  
2.暴露过程中，如果是延迟暴露，则启动一个线程，延迟delay时长在进行doExport动作  
3.会遍历所有的协议，所有的url进行暴露。（这也就是dubbo支持多协议多注册中心）  
4.接下来就是一系列的构建url操作（获取ip，端口，提供方法等）  
5.将bean进行一个代理（最终调用就不是通过反射调用）换成一个Invoker对象  
6.将Invoker对象去注册中心上进行export动作

- 本地进行暴露，主要就是根据传输层协议扩展点启动一个netty服务，监听一个端口，返回一个exporter对象
- 将提供者url信息注册到zookeeper，并且监听provider下目录数据的变化

##### 服务引用

1.从xml读取ReferenceBean配置，ReferenceBean实现了FactoryBean接口，会调用getObject进行引用ref  
2.通过注册中心想provider目录下的consumers目录下注册本机消费者地址，同时订阅provider下providers、configurators、routers目录数据变化  
3.通过获取到provider的url，将提供者的url列表转换成invoker列表  
– 转换的过程中，每一个url会初始化创建一个netty新连接，以及开启心跳

4.最终将多个invoker，封装成一个invoker对外使用，封装出来的invoker就继承了集群，负载均衡，容错等功能  
5.最后将ReferenceBean进行一个代理，传入上面获取的invoker

##### 服务调用

- 消费方调用provider方法，会进入到invoker.invoke方法，判断是否走mock，以及集群，容错，负载，在经过一系列的filter，最终调用到具体的一个invoker对象。通过ref时候的client连接向服务端发送一个请求。  
  consumer :  
  `MockClusterInvoker->FailoverClusterInvoke->ListenerInvokerWrapper->ProtocolFilterWrapper-><br></br>ConsumerTraceFilter->InvokeResultLogFilter->FutureFilter->MonitorFilter->DubboInvoker`
- 提供方接受到请求后，进行消息解码，消息转换成Invocation对象，从exporterMap中获取到exporter和Invoker对象，根据方法名调用provider方法，得到返回结果返回

#### SPI

1.普通扩展点  
2.自适应扩展点  
3.wrapper扩站点  
4.自动激活扩展点

#### 负载均衡-LoadBalance

- 基于权重的随机（默认负载均衡策略）。首先判断权重是否都相同，如果都相同则直接随机选取，如果权重不一样，则根据权重得到一个总和，遍历来确定某个提供者
- 基于权重的轮询。同样的首先判断权重是否都相同，如果都相同则根据序号取模轮询，如果不同则序号对最大权重取模得到当前权重，遍历获得比当前权重大的集合在轮询
- 最少活跃连接数。为每个提供者维护了一个活跃连接数值，获得最少的活跃连接数列表，然后随机取值。
- 一致性hash。默认一个提供者对应的160个虚拟节点

#### 集群-Cluster

- 失败转移/失败重试（failover）默认策略）。当调用失败后，根据配置重试次数重新选取另一个提供者进行调用，直至成功或者最大重试次数
- 可用（available）。遍历所有提供者，只要一个可用就返回
- 广播（broadcast）。遍历所有提供者，所有都调用一次
- 失败降级（failback）。调用失败降级，后台记录失败请求，定时重发，适用消息通知操作
- 快速失败（failfast）。调用失败后，直接报错
- 失败安全（failsafe）。调用失败后，降级，只记录失败日志，可用户写入审计日志
- 并行调用（forking）。并行调用，只要有一个成功就返回，通常适用于实时性要求较高场景，但会浪费更多服务资源

#### 使用dubbo过程中遇到的坑

- 使用(2.5.3)时候，异步调用存在传递，A调B异步调用，B里面去调用C的时候也变成异步调用，拿到是NULL  
  解决：在B里面手动调用从RpcContext attachements里面删除异步调用key  
  新版本解决这个问题，ContextFilter里面删除了异步调用KEY
- 版本：（2.6.5）dubbo优雅关闭，spring容器关闭了，dubbo可能还未关闭，造成dubbo请求去获取redis连接时候报连接池已经关闭，报错。  
  解决：利用spring关闭提供的Lifecycle接口start, stop钩子，执行时机在spring关闭destroyBean之前，这样等dubbo关闭了spring在关闭  
  利用spring关闭提供的ContextCloseEvent事件  
  ![](http://codercoder.cn/wp-content/uploads/2019/09/2019-09-0815.png)
- 使用(2.5.3)时候，Provider方在ContextFilter之前如果自己设置参数到RpcContext#attachements中，到后面就没有了，经过ContextFilter就被覆盖了

#### 2.6.5相较于2.5.3 diff

1.替换zkClient为Apache Curator  
2.增加QOS模块  
3.外部化配置  
4.ServiceConfig hostToRegistry

#### 不好的点

- 很多兼容的代码，和很多重复的判断，可以去掉精简

#### 使用到设计模式：

1.模版模式  
2.代理模式  
3.装饰器模式

1.SPI ExtensionLoader  
2.ServiceConfig export服务暴露  
2.ReferenceConfig refer服务引用  
3.服务一次调用过程

provider接收请求处理类流程:  
`NettyHandler->MultiMessageHandler->HeartbeatHandler->AllChannelHandler->DecodeHandler->HeaderExchangeHandler->DubboProtocol(requestHandler)`

consumer filter:  
`MockClusterInvoker->FailoverClusterInvoke->ListenerInvokerWrapper->ProtocolFilterWrapper->ConsumerTraceFilter->InvokeResultLogFilter->FutureFilter->MonitorFilter->DubboInvoke`