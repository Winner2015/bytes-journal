# 参考

https://blog.csdn.net/wszcy199503/article/details/106356396

https://www.cnblogs.com/wdss/p/11141051.html

https://www.jianshu.com/u/f7daa458b874

https://www.cnblogs.com/cyfonly/category/1210139.html





DubboBootstrap在dubbo中的作用：https://blog.csdn.net/weixin_38308374/article/details/105918415





https://www.cnblogs.com/binarylei/p/14110008.html



# Dubbo中的Spring


# ClassPathBeanDefinitionScanner

lassPathBeanDefinitionScanner作用就是将指定包下的类通过一定规则（被@Component、@Repository、@Controller等注解）过滤后，将Class 信息包装成 BeanDefinition 的形式注册到IOC容器中。


与 **AnnotatedBeanDefinitionReader** 不同的是，ClassPathBeanDefinitionScanner 是基于路径扫描的，比如 @ComponentScan 注解对应路径的扫描就是由它来完成的。




20

30

110  





