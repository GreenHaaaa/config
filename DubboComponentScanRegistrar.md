DubboComponentScanRegistrar 

注册ServiceAnnotationBeanPostProcessor和ReferenceAnnotationBeanPostProcessor

之前虽然有接触并使用过dubbo进行开发，却没有接触过dubbo 的源码。这次刚刚开始阅读dubbo的源码，时间上较为短暂，再加上dubbo内部的很多细节自己并不了解，因此在阅读源码的过程中总是磕磕绊绊，没有对dubbo整体有一个较为全面的认识，很多地方都理解的不是很到位，大多数都只是在读代码，尝试去理解它（理解的较为困难）。因此本次对dubbo源码的笔记可能条理不是那么的清晰，相信在下一次报告上会有很大的进步。

​	本次对dubbo代码阅读的报告主要涉及到dubbo的xml配置解析过程以及dubbo消费者对服务引入的过程，其中参考了dubbo官网的源码导入以及一些博客上的介绍，在自己debug源码的过程中写了一些个人的理解。

​	