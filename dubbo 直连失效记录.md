@AutoWire 为空，

@Reference 不带url参数 通过注册中心获取一个

@Reference（url="") 获取指定url直连

*均在dubbo：reference中配置url属性*



## 问题原因

在Spring框架内使用注解会将服务的动态代理类在referenceConfigs（ConcurrentHashMap）中按照接口名和reference的代理类存放。不过在生成动态代理类的过程中配置是独立的，即@Reference中的配置和xml中的配置不互通，在Spring为Controller引入的dubbo服务注入属性时（dubbo默认lazy-init），会首先读取@REference的配置，生成动态代理类并且为该属性注入值，XML的配置在配置init=true时也会在项目初始化的过程中生成，但是注解配置的属性注入优先于（快于XML的），因此只要使用@Reference注解，就会按照默认的方式生成服务的动态代理类，因此导致XML的虽然设置了，但是Controller中已经是@Reference生成的代理类了，XML配置的并未生效。（明天研究下XML的配置读取以及注册的过程）