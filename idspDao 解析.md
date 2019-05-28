# idspDao 个人理解

## 注解标记

​	通过@IdspEntity注解标记相对于的实体类，然后在扫描预定义的包名为

```java
	private static final String RESOURCE_PATTERN = "classpath*:com/sf/**/entity/**/*.class";
	private static final String RESOURCE_PATTERN2 = "classpath*:com/sf/**/model/**/*.class";
```
​	目录下的实体类，通过反射将被注解@IdspEntity的POJO类进行解析（@IdspTable，@IdspId，@IdspTransient...）对类内被相应的注解修饰的属性解析，生成一个table类。将所有table存储在一个列表内（以上过程位于ResolverTableUtil）

## EntityStatement

​	在经过上述的扫描之后我们可以得到项目中entity的一些基础信息（表名，字段信息），之后在IdspEntityStatementBuilder中构建sql语句以及将sql语句设置到Mybatis的SqlSessionFactory的configuration中的statementMap中（sql语句作为这个类的一个属性）并生成对于类型的IdspEntityStatement<T>.

​	通过实现BeanDefinitionParser接口的parse方法（类名：builderBeanDefinitionParser），将idspEntityStatementBuilder初始化并注册到Spring的IOC容器中。实现BeanDefinitionParser接口的parse方法（类名：builderBeanDefinitionParser）将扫描到的每一个实体类名+“_dao” 作为该实体类的entityStatement注册到IOC容器中

​	实现NamespaceHandlerSupport接口将上述builderBeanDefinitionParser以及builderBeanDefinitionParser注册到IOC容器中

​	自定义实现BeanFactoryAware, BeanPostProcessor接口的DaoAnnotationBeanPostProcessor，通过包名以及反射获取泛型类型，将IOC容器中的所有使用@idspEntityStatement以及类型为EntityStateMent的Bean的属性设置为 由getActualTypeArguments方法获取到泛型参数（实体类类型），并将其类名+“_dao”获取到具体entityStateMent的实现类。

