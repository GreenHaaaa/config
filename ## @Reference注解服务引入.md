## @Reference注解服务引入

​	*公司使用的dubbo版本似乎简化过不少东西*，与官网的不同，以下为公司项目中的dubbo-sf.2.1.0版本

​	为了接入Spring框架，Dubbo实现了Spring对外提供的BeanDefinitionParser读入Xml配置生成具体配置类（例如dubbo：reference注解对应了一个ReferenceConfig类），并将配置类的具体信息封装为RootBeandefinition并注册到IOC容器中。

​	服务的初始化，初始化主要是在ReferenceConfig的Init（）方法中进行的。

​        ReferenceConfig中的createProxy（）生成代理类

​	实现了Spring框架的BeanPostProcessor接口的AnnotationBean扫描@Reference的注解，refer()方法来，实现对属性的注入。在refer（）方法中会根据接口名称去referenceConfigs中获取对应的ReferenceBean，由于默认lazy=true，因此在第一次获取时将根据配置内容去生成一个ReferenceBean.这里回去调用referencConfig的init方法，而init()方法回去调用createProxy（）生成动态代理类去调用服务提供者。

​	以上为@Reference的注解服务注入流程，具体代码细节下周代码评审时进行分享

​		

```

```

