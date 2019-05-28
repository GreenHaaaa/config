# URL的生命周期

## 从配置中读取

​	配置的读入是在Spring解析资源过程中，遇到dubbo命名空间的时候dubbonameSpaceHandler注册的dubbo实现的spring提供的扩展接口BeanDefinationParser接口的DubboBeanDefinationParser类进行解析配置，将所有配置信息封装成对应的配置类。然后在ReferenceConfig或者ServiceConfig中对应的get或者export（）方法将配置的Bean转换成dubbo的url。然后将url作为服务导出导入的上下文信息，url贯穿整个流程作为参数的载体。

- DubboNameSpaceHandler注册DubboBeanDefinitionParser
- DubboBeanDefinitionParser解析XML文件内容，生成各个dubo的配置类信息
- ReferenceConfig#get（）方法引入服务的过程中将Bean转换为URL
- 使用URL进行SPI自适应扩展

## 暴露服务

#### 暴露服务

### 只暴露服务端口：

​	在没有注册中心，直接暴露提供者的情况下，`ServiceConfig` 解析出的 URL 的格式为：`dubbo://service-host/com.foo.FooService?version=1.0.0`。

​	基于扩展点自适应机制，通过 URL 的 `dubbo://` 协议头识别，直接调用 `DubboProtocol`的 `export()` 方法，打开服务端口。

### 向注册中心暴露服务：

- 在有注册中心，需要注册提供者地址的情况下，`ServiceConfig` 解析出的 URL 的格式为: `registry://registry-host/org.apache.dubbo.registry.RegistryService?export=URL.encode("dubbo://service-host/com.foo.FooService?version=1.0.0")`，

- 基于扩展点自适应机制，通过 URL 的 `registry://` 协议头识别，就会调用 `RegistryProtocol` 的 `export()` 方法，将 `export` 参数中的提供者 URL，先注册到注册中心。

- 再重新传给 `Protocol` 扩展点进行暴露： `dubbo://service-host/com.foo.FooService?version=1.0.0`，然后基于扩展点自适应机制，通过提供者 URL 的 `dubbo://` 协议头识别，就会调用 `DubboProtocol` 的 `export()` 方法，打开服务端口。

## 引用服务

### 直连引用服务：

- 在没有注册中心，直连提供者的情况下，`ReferenceConfig` 解析出的 URL 的格式为：`dubbo://service-host/com.foo.FooService?version=1.0.0`。

- 基于扩展点自适应机制，通过 URL 的 `dubbo://` 协议头识别，直接调用 `DubboProtocol` 的 `refer()` 方法，返回提供者引用。

### 从注册中心发现引用服务：

- 在有注册中心，通过注册中心发现提供者地址的情况下，`ReferenceConfig` 解析出的 URL 的格式为：`registry://registry-host/org.apache.dubbo.registry.RegistryService?refer=URL.encode("consumer://consumer-host/com.foo.FooService?version=1.0.0")`。

- 基于扩展点自适应机制，通过 URL 的 `registry://` 协议头识别，就会调用 `RegistryProtocol` 的 `refer()` 方法，基于 `refer` 参数中的条件，查询提供者 URL，如： `dubbo://service-host/com.foo.FooService?version=1.0.0`。

- 基于扩展点自适应机制，通过提供者 URL 的 `dubbo://` 协议头识别，就会调用 `DubboProtocol` 的 `refer()` 方法，得到提供者引用。

- 然后 `RegistryProtocol` 将多个提供者引用，通过 `Cluster` 扩展点，伪装成单个提供者引用返回。