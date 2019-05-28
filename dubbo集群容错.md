# dubbo集群容错

![](D:\user\01385157\桌面\dubbo服务容错流程.png)

​	dubbo通过invoker封装关于集群路由，负载均衡等方面的功能，在dubbo引入服务时，通过protocol的refer方法创建director对象，并使用director对象作为参数，调用cluster的join()方法创建invoker对象，每一种cluster的实现都对应着有一种invoker。

## Cluster

dubbo提供的几种容错机制

```
Failover Cluster
失败自动切换，当出现失败，重试其它服务器 [1]。通常用于读操作，但重试会带来更长延迟。可通过 retries="2" 来设置重试次数(不含第一次)。

Failfast Cluster
快速失败，只发起一次调用，失败立即报错。通常用于非幂等性的写操作，比如新增记录。

Failsafe Cluster
失败安全，出现异常时，直接忽略。通常用于写入审计日志等操作。

Failback Cluster
失败自动恢复，后台记录失败请求，定时重发。通常用于消息通知操作。

Forking Cluster
并行调用多个服务器，只要一个成功即返回。通常用于实时性要求较高的读操作，但需要浪费更多服务资源。可通过 forks="2" 来设置最大并行数。

Broadcast Cluster
广播调用所有提供者，逐个调用，任意一台报错则报错 [2]。通常用于通知所有提供者更新缓存或日志等本地资源信息。
```

#### invoker执行调用的流程

![](d:\user\01385157\我的文档\My Pictures\dubbo集群容错调用时序图.png)

**Cluster接口**

​	cluster接口对外提供一个join（Directory directory ）方法，用于返回有一个ClusterInvoker对象（上述几种容错策略的实现类）,然后在调用invoke方法是执行上述时序图流程。

### abstractClusterInvoker

​	**invoke（Invocation invocation）**方法是执行dubbo服务调用的入口，具体不同invoker的差异逻辑由抽象方法doInvoke()执行，在invoke方法中主要流程如下：

1. 检查是否容器是否已被销毁

2. 绑定隐藏参数到这次调用

3. 执行list方法，获取当前消费者列表

4. 初始化loadBalance扩展

5. 设置是否异步执行

6. 调用抽象方法doinvoke（Invocation invocation）

   **list（Invocation invocation）**方法的具体实现由directory的实现类提供，这个方法主要是通过路由规则进行过滤，获取符合条件的invoker，具体详情Router的时候在看。

​	**select(LoadBalance loadbalance, Invocation invocation, List<Invoker<T>> invokers, List<Invoker<T>> selected)**，开启粘滞连接具体的loadBanlance的由doSelect方法来执行在这个doSelect方法中，会有以下4种情况：

1. 只用一个invoker，直接方法
2. 2个及以上，使用loadBablance.select()
3. 存在优先选择的invoker或者开启了可用性检查且当前的选出的invoker不可用，重新选择,reselect()
4. 去负载均衡策略取出的invoker（该invoker不可用）的下一个invoker

**reselect()**方法是select方法在开启了可用检查并当前无法获得可用的invoker时被调用的，有一下几个步骤：

1. 判断avilableCheck是否开启，开启就从list中的所有invoker，过滤掉已被选择的后，将所有可用的invoker收集起来，在使用LoadBalance.select()选择一个，反之就不进行可用的判断
2. 当第一步收集的list为空时，在已选择的列表里面使用loadBalance重新选择一个可用的
3. 前两步都没有成立，直接返回null，今天select的第4步

## Directory

​	dubbo的Directory可以看做是invoker的集合，与普通集合不同的地方在于他内部实现了一个监听器接口，用于销毁和新增它订阅的服务，类似ExtensionLoader，directory也拥有一个泛型参数，每一个service对应一个directory。directory会按照category类进行分类存储，当其接受到服务的变更通知后（接受的url参数），会对invoker的集合进行刷新操作。职责主要有以下两点：

1. 维护本地的一个invoker集合（list()方法）

2. 作为监听器，监听注册中心的配置，路由，消费者信息列表（refreshInvoker（）方法）

   **list()**

   主要是通过路由规则进行invoker的筛选

   **notify（）**

   前面提到directory实现了一个监听器方法，用于监听服务提供者的变动以及路由规则，配置信息的变更，这个同步方法就是在监听到信息的变动后执行的一些操作逻辑。具体步骤如下：

   1. 首先将入参按照路由规则，配置规则，消费者信息分类存储
   2. 将路由规则由url封装成Router类
   3. 将配置规则封装为Configurar类
   4. 将消费者URL作为参数调用refreshInvoker方法

   **refreshInvoker(List<URL)方法**

   ​	这个方法对传入的参数的空与非空做两种逻辑判断，官方注解如下

   - 若为空：变更的规则为覆盖规则（configuator）或者路由规则(router)，invokerurlsp使用本地缓存
   - 若非空，表明其为最新，重新缓存
   - 则使用这个url调用同toInvokers()方法生成行的url和invoker的map映射
   - 通过比较关闭不用的invoker

   **toInvokers()方法**

   - 校验protocol是否存在（ExtensionLoader.hasExtension（））
   - 组合url，将配置按照优先级进行覆盖（overider>-D>provider>consumer）
     - 覆盖规则是Dubbo设计的在无需重启应用的情况下，动态调整RPC调用行为的一种能力
   - protocol.refer 自适应扩展，根据url参数获取对应的实现，执行对应的refer方法

   **一个小问题**

   dubbo的registerDirectory在刷新invoker的过程中，为什么在通过比较新旧invoker时进行删除新的invoker里面没有的？
   	首先这里删除肯定是为了使得消费者在调用服务的过程 中不在调用已经被销毁的服务，问题在于这里为什么不直接整个旧的invoker的map删除，而是只删除新的中没有的。
   	个人的理解是，存在以下情况，因为新的invoker中有一些没有进行变动，因此新旧的map中的invoker其实是一个实例，因此在删除的过程中，不能把整个旧的的invoker都进行销毁，并且 不直接clear旧的map的原因是，需要销毁的invoker可能还在被某个地方使用，因此无法被GC，所有需要手动的销毁。

## Router

​	路由规则决定一次 dubbo 服务调用的目标服务器，分为条件路由规则和脚本路由规则，并且支持可扩展 。

## LoadBalance

​	dubbo对集群环境下的负载均衡提供看多种负载均衡策略，默认使用随机方法，在AbstractLoadBalance中实现getWeight方法获取消费者的权重信息。

​	负载均衡策略如下（以下内容来自官网）：

```
#### Random LoadBalance

- **随机**，按权重设置随机概率。
- 在一个截面上碰撞的概率高，但调用量越大分布越均匀，而且按概率使用权重后也比较均匀，有利于动态调整提供者权重。

#### RoundRobin LoadBalance

- **轮询**，按公约后的权重设置轮询比率。
- 存在慢的提供者累积请求的问题，比如：第二台机器很慢，但没挂，当请求调到第二台时就卡在那，久而久之，所有请求都卡在调到第二台上。

#### LeastActive LoadBalance

- **最少活跃调用数**，相同活跃数的随机，活跃数指调用前后计数差。
- 使慢的提供者收到更少请求，因为越慢的提供者的调用前后计数差会越大。

#### ConsistentHash LoadBalance

- **一致性 Hash**，相同参数的请求总是发到同一提供者。
- 当某一台提供者挂时，原本发往该提供者的请求，基于虚拟节点，平摊到其它提供者，不会引起剧烈变动。
- 算法参见：<http://en.wikipedia.org/wiki/Consistent_hashing>
- 缺省只对第一个参数 Hash，如果要修改，请配置 `<dubbo:parameter key="hash.arguments" value="0,1" />`
- 缺省用 160 份虚拟节点，如果要修改，请配置 `<dubbo:parameter key="hash.nodes" value="320" />`
```

