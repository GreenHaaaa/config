# dubbo的XML配置

*基于Dubbo2.7.1*

## 基本概念

- DubboBeanDefinitionParser，用于解析dubbo的xml文件配置信息，每一个dubbo.xsd中的<xsd:element name="" />对应一个DubboBeanDefinitionParser实例（DubboBeanDefinitionParser中的一个Class<?>）
- DubboNameSpaceHandler 用于注册DubboBeanDefinitionParser实例（多个）

## 代码解析

### DubboNameSpaceHandler的内部属性

- private final Class<?> beanClass // Bean 对象的类
- private final boolean require  //是否需要自动生成一个ID

