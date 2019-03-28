日期：2019-03-28

### 备注：refresh主流程

```java
/**
 * @see AbstractApplicationContext#refresh() 
 */
@Override
public void refresh() throws BeansException, IllegalStateException {
   synchronized (this.startupShutdownMonitor) {
      // Prepare this context for refreshing.
      //准备此上下文以进行刷新
      prepareRefresh();

      // Tell the subclass to refresh the internal bean factory.
      //这里是在子类中启动refreshBeanFactort()的地方
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

      // Prepare the bean factory for use in this context.
      // 准备bean工厂以在此上下文中使用。
      prepareBeanFactory(beanFactory);

      try {
         // Allows post-processing of the bean factory in context subclasses.
         // 允许在上下文子类中对bean工厂进行后处理。
         postProcessBeanFactory(beanFactory);

         // Invoke factory processors registered as beans in the context.
         //调用beanFactory的后处理器，这些后处理器是在bean定义中向容器注册的,用户自定义的Spring的各种内置处理，都是在这个地方进行初始化的。比如：自定义ResourceEditorRegistrar；
         //对priorityOrderedPostProcessors，orderedPostProcessors，BeanDefinitionRegistryPostProcessor进行注册，然后，在按顺序实例化，提前实例化这些接口下的bean。
          
         invokeBeanFactoryPostProcessors(beanFactory);

         // Register bean processors that intercept bean creation.
         // 注册bean的后处理器，在bean创建过程中调用
         registerBeanPostProcessors(beanFactory);

         // Initialize message source for this context.
         //对上下文中的消息源进行初始化
         initMessageSource();

         // Initialize event multicaster for this context.
         //初始化上下文中的事件机制
         initApplicationEventMulticaster();

         // Initialize other special beans in specific context subclasses.
         //初始化其他的特殊bean
         onRefresh();

         // Check for listener beans and register them.
         //检查监听bean并且将这些bean向容器注册
         registerListeners();

         // Instantiate all remaining (non-lazy-init) singletons.
         //实例化所有的(non-lazy-init)单利
         finishBeanFactoryInitialization(beanFactory);

         // Last step: publish corresponding event.
               //发布容器事件，结束refresh过程
         finishRefresh();
      }catch (BeansException ex) {
         if (logger.isWarnEnabled()) {
            logger.warn("Exception encountered during context initialization - " +
                  "cancelling refresh attempt: " + ex);
         }

         // Destroy already created singletons to avoid dangling resources.
         destroyBeans();

         // Reset 'active' flag.
         cancelRefresh(ex);

         // Propagate exception to caller.
         throw ex;
      }

      finally {
         // Reset common introspection caches in Spring's core, since we
         // might not ever need metadata for singleton beans anymore...
               /**
                * 重置Spring核心中的常见内省缓存，
                * 从此我们开始可能再也不需要单例bean的元数据
                */
         resetCommonCaches();
      }
   }
}
```

​									refresh主流程

#### 4）postProcessBeanFactory(beanFactory)

​	允许在上下文子类中对bean工厂进行后处理。

​	默认实现为空，时提供给用户的入口，实现此方法，用来在beanFactory实例化之前做一些定制化的处理。

比如：org.springframework.web.context.support.GenericWebApplicationContext#postProcessBeanFactory(beanFactory)，忽略ServletContextAware，注册了一些scope；

```java
@Override
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
   beanFactory.addBeanPostProcessor(new ServletContextAwareProcessor(this.servletContext));
   beanFactory.ignoreDependencyInterface(ServletContextAware.class);

   WebApplicationContextUtils.registerWebApplicationScopes(beanFactory, this.servletContext);
   WebApplicationContextUtils.registerEnvironmentBeans(beanFactory, this.servletContext);
}
```



#### 5）invokeBeanFactoryPostProcessors(beanFactory)

​	调用beanFactory的后处理器，这些后处理器是在bean定义中向容器注册的,为用户提供的一些工具类：比如格式化日期的 ResourceEditorRegistrar;

```java
/**@see AbstractApplicationContext#invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory)
*/
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    //调用beanFactory中注册的后置处理器，按照ordered、PriorityOrdered接口来排序之后，顺序调用
   PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

   // Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
   // (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
   if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
      beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
      beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
   }
}
```