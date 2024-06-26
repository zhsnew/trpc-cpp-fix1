[English](../en/architecture_design.md)

# 前言

本篇文章主要对 tRPC-Cpp 具体架构设计进行介绍, 基于框架 v1.0.0 编写.

主要内容分为如下几个部分:

- 介绍下 tRPC-Cpp 的整体架构设计和代码目录结构, 方便大家先有个大概的认识;
- 介绍下 tRPC-Cpp 的server和client工作流程, 方便大家从全局把握server和client工作原理;
- 介绍下 tRPC-Cpp 的插件化设计, 方便大家了解插件化的具体实现机制;

# 架构设计

tRPC-Cpp 整体架构设计如下:

![architecture_design](../images/arch_design.png)

总体架构由"**框架核心**"和"**插件**"两部分组成. 如上图所示, 虚线框内为tRPC, 其中中间的红色实线框为框架核心, 蓝色框为插件部分.

其中框架核心又可以分三层:

- **运行层**: 由线程模型和IO模型组成, 负责框架任务的调度和IO的处理, 其中线程模型目前支持: 普通线程模型（分为io和handle分离或者合并的线程模型）、M:N协程模型（fiber线程模型）, IO模型目前支持: 用于网络IO的Reactor模型和用于磁盘IO得AsyncIO模型（基于io-uring, 目前只支持在合并线程模型使用）;

- **通信层**: 负责数据的传输和协议的编解码. 框架内置支持tcp/udp/unix-socket等通信协议, 传输协议采用基于proto的tRPC协议来承载RPC调用, 支持通过codec插件来使用其它传输协议;

- **调用层**: 封装服务和服务代理实体, 提供RPC调用接口, 支持业务用同步、异步、单向以及流式调用等方式进行服务间调用;

此外框架还提供了admin管理接口, 方便用户或者运营平台可以通过调用admin接口对服务进行管理。 管理接口包括更新配置、查看版本、修改日志级别、查看框架运行时信息等功能，同时框架也支持用户自定义管理接口，以满足业务定制化需求.

tRPC-Cpp 框架的主要目录结构设计如下:

![dir_structure](../images/dir_structure.png)

主要由"**框架核心主体实现**"、"**服务治理插件实现**"、"**辅助工具实现**"三部分的目录组成.

框架核心主体实现主要包括以下几个的模块:

- server：提供了一个服务实现，支持多 service 启动、注册、取消注册、平滑退出等;
- client：提供了一个并发安全的通用的 client 实现，主要负责rpc调用、服务发现、负载均衡、熔断、编解码、自定义拦截器相关的操作, 各部分均支持插件式扩展;
- codec：提供了编解码相关的接口, 允许框架扩展业务协议、序列化方式、数据压缩方式等;
- transport: 提供了网络传输的能力, 支持tcp/udp/ssl/unix-socket等传输方式;
- runtime: 提供了框架运行环境的实现, 对线程模型和io模型进行了封装实现, 支持m:n协程(fiber)模型、io与handle分离和模型线程模型;
- filter：提供了自定义拦截器的定义，允许通过扩展 filter 的方式来丰富处理能力，如 tracing、metrics、logreplay 等等;

服务治理插件实现主要包括以下几个的模块:

- naming：提供了服务注册(registry)、服务发现(selector)、负载均衡(loadbalance)、熔断(circuitbreaker)等能力封装, 用于对接各种名字服务系统;
- config：提供了配置读取相关的接口, 支持读取本地配置文件、远程配置中心配置等，允许插件式扩展支持不同格式的配置文件、不同的配置中心，支持 reload、watch 配置更新;
- metrics：提供了监控上报的能力, 支持常见的单维上报，如 counter、gauge 等, 也支持多维上报, 允许通过扩展 Sink 接口实现对接不同的监控系统;
- logging：提供了通用的日志采集接口, 允许通过插件的方式来扩展日志实现, 允许日志输出到远程；
- tracing：提供了分布式跟踪能力，允许通过插件的方式上报到调用链系统；

## 交互流程

tRPC-Cpp 整体交互流程如下：
![interaction_process](../images/interaction_process.png)

## 服务端

### 启动

Server 启动过程, 大致包括以下流程:

1. 继承 `TrpcApp` 的业务子类实例化并启动;
2. 读取框架配置文件 (通过--config指定), 这里的配置包含了global、server、service、client、plugin等的配置信息;
    1. 读取global配置信息, 初始化并启动框架的运行环境runtime, runtime包括线程模型和网络模型等;
    2. 读取server/service、client配置, 初始化 TrpcServer 和 TrpcClient;
    3. 读取plugin插件配置完成各种插件的初始化逻辑;
3. 运行 `TrpcApp` 业务子类的 `Initilize` 方法, 完成业务代码的初始化;
    1. 业务自身各种初始化逻辑;
    2. 调用 `RegisterService` 完成服务注册, 开始网络请求的监听;
4. 服务此时就已经正常启动了，后续等待 client 建立连接请求.

### 请求处理

请求处理过程, 大致包括以下流程:

1. server transport 调用 Acceptor 等待 client 建立连接;
2. client 发起建立连接请求，server transport Accept 返回一个连接 tcpconn;
3. server transport 根据当前的runtime环境(是否是fiber runtime), 来决定连接的分发;
    1. 如果是fiber runtime, 那么选择一个fiber调度组, 由具体fiber调度组上的 Fiber Reactor 来处理;
    2. 如果不是, 那么选择一个io线程, 由具体io线程的 Reactor 来处理;
4. 开始收包的逻辑，server transport 根据编解码协议、压缩方式、序列化方式不停地读取请求, 每个请求会创建一个 msg, 里面包括请求的上下文 ServerContext, 并请求 msg 交给上层 service 处理;
5. service 会根据 msg 内部context的 rpc 名称，找到对应的注册的处理函数，调用对应的处理函数;
6. 调用对应的处理函数之前, 其实还要过一个埋点的 filterchain，filterchain 执行到最后就是我们注册的 rpc 的处理函数;
7. 将处理结果进行序列化、压缩、编解码, 然后回包给 client, 其中回包方式分同步和异步;
    1. 同步方式是rpc处理函数执行完后, 直接处理结果, 并回包给 client, 默认方式;
    2. 异步方式是rpc处理函数执行完后, 不回包给 client, 而是由业务自己异步处理完后, 主动调用框架的回包即可进行回包;

### 退出

服务退出过程, 大致包括以下流程:

1. 向名字服务反注册所有service，避免下次流量路由到此节点;
2. 禁用连接可读事件，包括 a.禁用监听socket的可读事件，避免创建新连接；b.禁用已连接socket的可读事件，避免收到该socket的新请求;
3. 等待服务端接收的请求都处理完成（为避免因等待无法退出，默认最多等待5秒钟，可通过yaml中的server::max_wait_time配置）;
4. 关闭服务端网络连接;
5. 调用 `TrpcApp` 业务子类的 `Destroy` 方法, 停止业务创建的动态资源（比如: 起的线程）;
6. 停止插件创建的动态资源（比如: 插件内部起的线程）;
7. 停止框架运行环境runtime;
8. 释放框架运行环境 runtime 内部的资源;
9. 释放框架运行环境 TrpcServer 内部的资源;
10. 释放框架运行环境 TrpcClient 内部的资源;
11. 程序退出;

## 客户端

1. 发送请求时，先组装各种调用参数;
2. 执行 client filter 前置逻辑;
3. 服务发现找到被调服务名对应的一组 ip:port 列表;
4. 通过负载均衡算法和熔断策略，找到合适的一个 ip:port 准备发起请求;
5. 对发送数据进行序列化、压缩、编码逻辑;
6. 然后准备建立到 ip:port 的连接, 这个时候会先检查 ClientTransport 中是否存在对应可用的连接, 没有就创建;
7. 获取到连接之后，开始发送数据，并等待接收（如果是连接复用模式，可能会在同一条连接上并发发送多个请求，请求响应通过 seqno 关联）;
8. 接收到数据，解码、解压缩、反序列化逻辑，递交给上层处理;
9. 执行 client filter 后置逻辑;

## 插件化设计

tRPC 的插件化架构设计主要是基于 **基于接口机制的插件工厂** 和 **基于AOP思想的拦截器** 来实现的.

### 插件工厂

tRPC核心框架是采用基于接口编程的思想, 通过把框架功能抽象成一系列的插件组件, 注册到插件工厂, 并由插件工厂实例化插件. tRPC框架负责串联这些插件组件, 拼装出完整的功能. 我们可以把插件模型分为以下部分:

1. 框架设计层: 框架只定义标准接口, 没有任何插件实现, 与平台完全解耦;
2. 插件实现层: 将插件按框架标准接口封装起来即可实现框架插件;
3. 用户使用层: 业务开发只需要引入自己需要的插件即可, 按需索取, 拿来即用;

典型对接名字服务系统的插件实现方式如下:
![plugin_factory](../images/plugin_factory.png)

### 拦截器

为了使框架有更强的可扩展性, 框架支持了拦截器filter, 它借鉴了java面向切面(AOP)的编程思想.

具体的实现方式是通过在框架请求处理流程中设置埋点逻辑, 然后通过在埋点地方插入一系列的filter, 来实现一些业务个性化的功能, 比如：metrics监控、日志收集、链路跟踪、过载保护、参数预处理等这些跟接口请求相关的问题

拦截器filter工作流程如下图:
![fitler](../images/filter.png)

拦截器的最终目的是让业务逻辑与框架
进行解耦,  并允许各自进行内聚性开发. 可以在不修改源代码的情况下, 通过配置即可添加或者更换不同的插件实现.
