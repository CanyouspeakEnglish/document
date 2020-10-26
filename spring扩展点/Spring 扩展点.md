#### Spring 扩展点

- ##### BeanDefinitionRegistryPostProcessor 

   1 org.mybatis.spring.mapper.MapperScannerConfigurer#postProcessBeanDefinitionRegistry

  ​    在没有接口实现的情况下找到接口方法和sql之前的关系 并注册beanDefinition

  2org.springframework.context.annotation.ConfigurationClassPostProcessor#processConfigBeanDefinitions @Configuration 构建并验证configuration模型 并注册

- ### BeanFactoryPostProcessor

​       1 org.springframework.beans.factory.config.PropertyResourceConfigurer#postProcessBeanFactory 

​                ${} 获取环境变量

​				postProcessBeanFactory方法在BeanFactory初始化后，所有的bean定义都被加载，但是没有bean会被实例化时，允许重写或添加属性



   2 org.springframework.context.support.PropertySourcesPlaceholderConfigurer#postProcessBeanFactory 

​     BeanDefinition生成后，可能某些参数是${key}，这个实现类就是把前边这种参数转换成xxx.properties中key所对应的值