## ReferenceConfig进行init之前进行的检查

*基于dubbo 2.7.1*

- init（）之前做的检查和配置的设置
  - 接口名非空检查
  - 对必要的配置信息进行检查并且赋值（consumerConfig->ModulConfig->applicationConfig）
  - 获取所有配置类信息，并按照配置优先级进行配置的覆盖（2.7.0 dubbo 配置外部化，refresh（），checkoutDefault（） dubbo 2.6）
  - 接口泛化设置
  - 校验接口名是否是一个接口以及远程服务是否提供该方法
  - 默认加载${user.home}/dubbo-resolve.properties配置文件
  - 检查applicationConfig等设置是否有效
- init()
  - 已经初始化，直接返回
  - 获取包名，版本号以及一些配置信息用于拼接dubbo的url
- createProxy（）
  - 决定引用类型（injvm，直连，register中心）