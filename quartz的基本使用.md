# quartz的基本使用

## 基础名词

- JobDetail Job的信息
  - name job的名字
  - group job的所属的组名
  - key 一个job的唯一标志符，在没有设置的时候值为：name+group
  - jobDataMap 一个map 可以存储用户自定义的数据等（job的传入参数）
  - JobClass  job要做的事情的类，真正执行的类
  - Durability 标识在这个job在没有触发器的时候是否存储
  - afterPropertiesSet JobDetailFactoryBean的方法，根据目前内部的一些jobdetail属性生成并记录jobdetail
- Schedule job调度执行规则
  - cron 调度的规则表达式（几天一次，几小时一次等，有特定的语法）
  - misfire 错失触发策略 因为某种原因在该触发的时候没有触发
- trigger 触发器
  - startTIme和endTime 指定一个区间，区间外的时间不触发时间
  - 由jobDetail和schedule组合，注册在Scheduler中
- scheduler 调度执行器

## 使用流程

1. 定义一个Job接口的实现类，实现execute(JobExecutionContext context)方法
2. 为定义好的job实现类设置name group 以及在JobDataMap里面设置参数等信息，生成一个jobDetail类
3. 根据使用需求生成一个Schedule，内包含调度规则等
4. 将上述JobDetail以及schedule构建一个trigger
5. 将trigger注册到scheduler中

## Spring 调度任务的使用

​	项目中使用Spring提供的ScheduledExecutorFactoryBean调度执行transferKafkaTask，transferKafkaTask里面包好了多个quartz的job