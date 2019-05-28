# dubbo服务目录



dubbo的Directory可以看做是invoker的集合，与普通集合不同的地方在于他内部实现了一个监听器接口，用于销毁和新增它订阅的服务，类似ExtensionLoader，directory也拥有一个泛型参数，每一个service对应一个directory。directory会按照category类进行分类存储，当其接受到服务的变更通知后（接受的url参数），会对invoker的集合进行刷新操作。

## refreshInvoker(List<URL)方法

​	这个方法对传入的参数的空与非空做两种逻辑判断，官方注解如下

- 若为空：变更的规则为覆盖规则（configuator）或者路由规则(router)，invokerurls使用本地缓存
- 若非空，表明其为最新，重新缓存
- 则使用这个url调用toInvokers()方法生成url和invoker的map映射
- 通过比较关闭不用的invoker



## toInvokers()方法

- 校验protocol是否存在（ExtensionLoader.hasExtension（））
- 组合url，将配置按照优先级进行覆盖（overider>-D>provider>consumer）
- protocol.refer 自适应扩展，根据url参数获取对应的实现，执行对应的refer方法


