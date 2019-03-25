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
         //调用beanFactory的后处理器，这些后处理器是在bean定义中向容器注册的
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

#### 3）prepareBeanFactory:

配置工厂的标准上下文特征，例如上下文的ClassLoader和后处理器。@Qualifier与@Autowired等注解正是在这一步骤中增加的支持  

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    //类加载器设置
   beanFactory.setBeanClassLoader(getClassLoader());
   beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    //org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader
   beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

   // Configure the bean factory with context callbacks.
   beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
   beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
   beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
   beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
   beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
   beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
   beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

   // BeanFactory interface not registered as resolvable type in a plain factory.
   // MessageSource registered (and found for autowiring) as a bean.
   beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
   beanFactory.registerResolvableDependency(ResourceLoader.class, this);
   beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
   beanFactory.registerResolvableDependency(ApplicationContext.class, this);

   // Register early post-processor for detecting inner beans as ApplicationListeners.
   beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

   // Detect a LoadTimeWeaver and prepare for weaving, if found.
   if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
      beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
      // Set a temporary ClassLoader for type matching.
      beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
   }

   // Register default environment beans.
   if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
      beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
   }
   if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
      beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
   }
   if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
      beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
   }
}
```
- **getClassLoader()**:
  - 获取类加载器，如果当前类加载器已经被用户设置好了，就使用用户设置的类加载器。默认的是使用当前线程的类加载器；在tomcat中，没个线程的类加载器，都被设置成了tomcat的appClassLoader，所有，在tomcat的服务中，默认情况下，spring的类加载器是tomcat的webappClassLoader类加载器（**违反了双亲委派机制的类加载器**）。所以，在spring中默认可以动态加载修改过的jsp文件，不需要再启动tomcat。

```java
@Override
public ClassLoader getClassLoader() {
   return (this.classLoader != null ? this.classLoader : ClassUtils.getDefaultClassLoader());
}

public static ClassLoader getDefaultClassLoader() {
		ClassLoader cl = null;
		try {
            //获取线程中环境中的类加载器。
			cl = Thread.currentThread().getContextClassLoader();
		}
		catch (Throwable ex) {
			// Cannot access thread context ClassLoader - falling back...
		}
		if (cl == null) {
			// No thread context class loader -> use class loader of this class.
			//获取当前类的类加载器。
            cl = ClassUtils.class.getClassLoader();
			if (cl == null) {
				// getClassLoader() returning null indicates the bootstrap ClassLoader
				try {
                    //获取jvm的appClassLoader类加载器
					cl = ClassLoader.getSystemClassLoader();
				}
				catch (Throwable ex) {
					// Cannot access system ClassLoader - oh well, maybe the caller can live with null...
				}
			}
		}
		return cl;
	}
```

- ##### **ResourceEditorRegistrar**

  org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#parseBeanDefinitions(Element , BeanDefinitionParserDelegate ) 方法中，判断是否为spring内部类，进而转化为对应的BeanDefinitions，spring内部的类，会用户的对象实例化之前就加载好。（如何判断是否为内部类的，在之后补充。）



