# dubbo服务引入的原理

​	Dubbo 服务引用的时机有两个，第一个是在 Spring 容器调用 ReferenceBean 的 afterPropertiesSet 方法时引用服务，第二个是在 ReferenceBean 对应的服务被注入到其他类中时引用。这两个引用服务的时机区别在于，第一个是饿汉式的，第二个是懒汉式的。默认情况下，Dubbo 使用懒汉式引用服务。如果需要使用饿汉式，可通过配置 <dubbo:reference> 的 init 属性开启。下面我们按照 Dubbo 默认配置进行分析，整个分析过程从 ReferenceBean 的 getObject 方法开始。当我们的服务被注入到其他类中时，Spring 会第一时间调用 getObject 方法，并由该方法执行服务引用逻辑。按照惯例，在进行具体工作之前，需先进行配置检查与收集工作。接着根据收集到的信息决定服务用的方式，有三种，第一种是引用本地 (JVM) 服务，第二是通过直连方式引用远程服务，第三是通过注册中心引用远程服务。不管是哪种引用方式，最后都会得到一个 Invoker 实例。如果有多个注册中心，多个服务提供者，这个时候会得到一组 Invoker 实例，此时需要通过集群管理类 Cluster 将多个 Invoker 合并成一个实例。合并后的 Invoker 实例已经具备调用本地或远程服务的能力了，但并不能将此实例暴露给用户使用，这会对用户业务代码造成侵入。此时框架还需要通过代理工厂类 (ProxyFactory) 为服务接口生成代理类，并让代理类去调用 Invoker 逻辑。避免了 Dubbo 框架代码对业务代码的侵入，同时也让框架更容易使用。

## 服务引入流程中各个方法概要

![1555485982891](d:\user\01385157\Application Data\Typora\typora-user-images\1555485982891.png)

- Reference:get()   主要是获取所有配置类信息，按照优先级对配置信息进行覆盖（refresh()）以及对接口名和applicationConfig的valid字段等的检查，之后调用init（）方法
- Reference：init(...)  主要是将配置信息集中到一个map，生成url，然后调用crateProxy方法
- Reference:   createProxy（...） 主要是根据配置Scope以及ReferenceConfig中的url字段来判断是采用本地连接还是dubbo直连，还是通过register，以及将init（）中的收集的map生成Dubbo的url，然后去调用Protoconfig的refer方法（这里使用dubbo的SPI自适应扩展，会根据方法的参数获取SPI扩展执行）
  - RegisterProtocol：refer（...） 获取url参数设置协议，获取注册中心，调用doRefer（...）
  - RegisterProtocol:   doRefer（...）生成registerDirectory对象，服务的注册与发现与该类关系紧密（下一次分析）构建消费者URl，并向注册中心注册自己，订阅privider，configuration，router的变更
  - DubboProtocol：refer（...） 生成DubboInvoker对象，主要是在getClient中获取通信客户端，根据 connections 数量决定是获取共享客户端还是创建新的客户端实例，默认情况下，使用共享客户端实例。
  - proxyFactory.getProxy(invoker); 将上一步得到的invoker生成代理类，这里也使用了dubbo的自适应SPI扩展机制，最终调用的JavassistProxyFactory：：getProxy(...) ->Proxy.getProxy(...), 在这个方法中会根据接口的方法名，返回值，参数，生成一段代码并以及这段代码的代理类来调用invoker。

## invoke

​	在 Dubbo 中，Invoker 是一个非常重要的模型。在服务提供端，以及服务引用端均会出现 Invoker。Dubbo 官方文档中对 Invoker 进行了说明，这里引用一下。

Invoker 是实体域，它是 Dubbo 的核心模型，其它模型都向它靠扰，或转换成它，它代表一个可执行体，可向它发起 invoke 调用，它有可能是一个本地的实现，也可能是一个远程的实现，也可能一个集群实现。

![img](http://static2.iocoder.cn/images/Dubbo/2018_03_01/04.png)

## dubbo的自适应扩展机制

​	在 Dubbo 中，很多拓展都是通过 SPI 机制进行加载的，比如 Protocol、Cluster、LoadBalance 等。有时，有些拓展并不想在框架启动阶段被加载，而是希望在拓展方法被调用时，根据运行时参数进行加载。这听起来有些矛盾。拓展未被加载，那么拓展方法就无法被调用（静态方法除外）。拓展方法未被调用，拓展就无法被加载。对于这个矛盾的问题，Dubbo 通过自适应拓展机制很好的解决了。自适应拓展机制的实现逻辑比较复杂，首先 Dubbo 会为拓展接口生成具有代理功能的代码。然后通过 javassist 或 jdk 编译这段代码，得到 Class 类。最后再通过反射创建代理类，整个过程比较复杂