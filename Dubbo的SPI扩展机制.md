# Dubbo的SPI扩展机制

​	dubbo的扩展点加载从JDK标准的SPI扩展点发现机制加强而来，jdk标准的SPI会已一次性的实例化扩展点的所有实现，因此对于并不使用的扩展也会初始化，造成资源的浪费，而dubbo的会在运行时根据Url中配置的参数选择相对应的扩展点进行加载，并且使用包装类对扩展点实现了IOC和Aop的功能，使得一个扩展点可以注入多个扩展点。

## 实现原理

- 扩展点自动包装

  - 使用包装类对扩展点进行自动包装。包装类同样实现类扩展点接口（一个扩展点对应一个扩展类），extensionLoader在对扩展点进行加载的过程中，会将存在拷贝构成函数的扩展点实现识别为自动包装类，并返回一个扩展点的包装类，这个包装类对真正的扩展点实现类进行了封装，可以在包装类内部方法中实现一些公共的代码，实现类似AOP的功能。

- 扩展点自动装配（属性注入位于ExtensionLoader.injectExtension(T t)）

  - 在一个扩展点的实现类内，extensionLoader在加载这个扩展点时回扫描扩展点的实现类的set方法，获取方法参数类型和以及使用getExtension方法获取值（这里也能获取到SpringIOC容器中的bean，具体于ExtensionFactory有关，它内部有一个extensionFactory列表，其中包含SPI的和Spring的，因此在getExtension时也会去Spring的IOC容器中获取这个bean）

- ExtensionLoader内部细节

  - @Adapter注解，在dubbo中adaptive注解的用法有两种，一个是加在类的头部，一个是加在方法上，加在类上，说明这个类是扩展(@SPI修饰,可以指定默认值）的手动适应扩展点实现，不会自动生成代码。加在方法上，一般会指定参数（url中的）来说明具体使用哪一个扩展实现类，这里会先自动生成一段代码，名为xxx$Adaptive，它实现扩展的接口，此后在运行过程中通过反射获取url后，根据meta-info/dubbo/internal(或者meta-info/dubbo、meta-info/service)中参数映射到的类的全限定名,通过反射加载这个类,如果没有在url中获取到@Adaptive的参数，就会使用@SPI的中设置的缺省扩展实现。
  - ExtensionLoader 代码细节

    - cacheClasses 该属性用于存储在meta-info/dubbo/internal(或者meta-info/dubbo、meta-info/service)中设置的类信息
    - loadExtensionClasses（）方法
      - 根据扩展（如protocol）的全限定名获取@SPI注解以及参数等信息
      - 去配置目录下通过全限定名获取配置文件信息（key-value）
      - 解析并加载扩展类，接入缓存（cacheClasses）
  - （内部还有 一些诸如type扩展类型，以及一些默认的扩展点实现的缓存，包装类的缓存等信息，以及生成xxx$Adaptive的代码还没有仔细研究，明天再看）

- 扩展点自适应（代码入口为ExtensionLoader:: getAdaptiveExtension）

  - 方法上使用@Adapter，里面的参数用于决定dubbo回去取用哪一个扩展实现类。其获取扩展的主要步骤为

    1. 从缓存中获取这个自适应扩展类
    2. 无扩展类的情况下，生成自适应扩展类的代码，编译，获取到代理类class

    代码生成的步骤主要有：

    1. 检查@Adapter注解(要求必须有一个)
    2. 对于无@Adapter修饰的接口方法，生成的代码为抛出一个异常
    3. 获取url参数，遍历参数列表，确定 URL 参数位置，遍历参数的方法列表，寻找可返回 URL 的 getter 方法
    4. 获取@Adapter的参数
    5. 检测 Invocation 参数、
    6. 生成拓展名获取逻辑
    7. 生成拓展加载与目标方法调用逻辑

  



