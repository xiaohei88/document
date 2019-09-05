##  2.1 Bean生命周期

- AbstractBeanFactory.doGetBean

  - AbstractAutowireCapableBeanFactory.createBean

    - InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation(Class<?> beanClass, String beanName)

  - AbstractAutowireCapableBeanFactory.doCreateBean

  - AbstractAutowireCapableBeanFactory.createBeanInstance

    - SimpleInstantiationStrategy.instantiate

      ```java
      // 调用factoryBean：配置在某个配置类里的Bean
      return factoryMethod.invoke(factoryBean, args);
      ```

    - MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName)

      - AutowiredAnnotationBeanPostProcessor:扫描成员变量带有@Autowired，组装成InjectionMetadata

  - AbstractAutowireCapableBeanFactory.populateBean

    - InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation(Object bean, String beanName)
    - InstantiationAwareBeanPostProcessor.postProcessPropertyValues(PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName)
      - AutowiredAnnotationBeanPostProcessor：从InjectionMetadata拿去信息获取具体的值

  - AbstractAutowireCapableBeanFactory.initializeBean

    - invokeAwareMethods
      - BeanNameAware
      - BeanClassLoaderAware
      - BeanFactoryAware
    - BeanPostProcessor.postProcessBeforeInitialization(Object bean, String beanName)
    - invokeInitMethods
    - BeanPostProcessor.postProcessAfterInitialization(Object bean, String beanName)
      - AnnotationAwareAspectJAutoProxyCreator:切面事务

- AbstractBeanFactory.registerDisposableBeanIfNecessary