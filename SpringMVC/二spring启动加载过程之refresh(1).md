

日期：2019-03-22

### （一）refresh主流程

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



### （二）refresh分步解析

#### 1）prepareRefresh() 

##### 准备此上下文以进行刷新，设置其启动日期和活动标志以及执行属性源的任何初始化。

```java
/**
 * Prepare this context for refreshing, setting its startup date and
 * active flag as well as performing any initialization of property sources.
 * 准备此上下文以进行刷新，设置其启动日期和活动标志以及执行属性源的任何初始化。
 */
protected void prepareRefresh() {
    //启动时间
   this.startupDate = System.currentTimeMillis();
    //关闭与激活状态
   this.closed.set(false);
   this.active.set(true);

   if (logger.isInfoEnabled()) {
      logger.info("Refreshing " + this);
   }

   // Initialize any placeholder property sources in the context environment
   // 在上下文环境中初始化任何占位符属性源
   //用户可以自定义，继承 对应的ApplicationContext,重写initPropertySources()方法
   initPropertySources();

   // Validate that all properties marked as required are resolvable
   // see ConfigurablePropertyResolver#setRequiredProperties
   // 验证标记为必需的所有属性是否可解析
   //如果设置的property为空，则会报错
   getEnvironment().validateRequiredProperties();

   // Allow for the collection of early ApplicationEvents,
   // to be published once the multicaster is available...
   // 允许收集早期的ApplicationEvents，
   // 一旦多功能车可用就会发布......
   this.earlyApplicationEvents = new LinkedHashSet<ApplicationEvent>();
}
```

 **initPropertySources()**：

初始化配置文件，默认不做任何操作。有些AbstractApplicationContext的子类，做了一些初始化的操作。比如：AbstractRefreshableWebApplicationContext#initPropertySources()

```java
protected void initPropertySources() {
   // For subclasses: do nothing by default.
   // 对于子类：默认情况下不执行任何操作。
}


```

```java
// AbstractRefreshableWebApplicationContext#initPropertySources()
	protected void initPropertySources() {
		ConfigurableEnvironment env = getEnvironment();
		if (env instanceof ConfigurableWebEnvironment) {
			((ConfigurableWebEnvironment) env).initPropertySources(this.servletContext, this.servletConfig);
		}
	}
//((ConfigurableWebEnvironment) env).initPropertySources会调用到WebApplicationContextUtils#initServletPropertySources，主要是替换了property中的servletContextInitParams参数和servletConfigInitParams参数。
public static void initServletPropertySources(
      MutablePropertySources propertySources, ServletContext servletContext, ServletConfig servletConfig) {

   Assert.notNull(propertySources, "'propertySources' must not be null");
   if (servletContext != null && propertySources.contains(StandardServletEnvironment.SERVLET_CONTEXT_PROPERTY_SOURCE_NAME) &&
         propertySources.get(StandardServletEnvironment.SERVLET_CONTEXT_PROPERTY_SOURCE_NAME) instanceof StubPropertySource) {
      propertySources.replace(StandardServletEnvironment.SERVLET_CONTEXT_PROPERTY_SOURCE_NAME,
            new ServletContextPropertySource(StandardServletEnvironment.SERVLET_CONTEXT_PROPERTY_SOURCE_NAME, servletContext));
   }
   if (servletConfig != null && propertySources.contains(StandardServletEnvironment.SERVLET_CONFIG_PROPERTY_SOURCE_NAME) &&
         propertySources.get(StandardServletEnvironment.SERVLET_CONFIG_PROPERTY_SOURCE_NAME) instanceof StubPropertySource) {
      propertySources.replace(StandardServletEnvironment.SERVLET_CONFIG_PROPERTY_SOURCE_NAME,
            new ServletConfigPropertySource(StandardServletEnvironment.SERVLET_CONFIG_PROPERTY_SOURCE_NAME, servletConfig));
   }
}
```



**getEnvironment().validateRequiredProperties()**：

​	验证标记为必需的所有属性是否可解析，如果设置的property为空，则会报错；最后调用到org.springframework.core.env.AbstractPropertyResolver#validateRequiredProperties();

主要是一个非空检验。相当于，StringUtils.isEmpty()的操作。

```java
@Override
public void validateRequiredProperties() {
   MissingRequiredPropertiesException ex = new MissingRequiredPropertiesException();
   for (String key : this.requiredProperties) {
      if (this.getProperty(key) == null) {
         ex.addMissingRequiredProperty(key);
      }
   }
   if (!ex.getMissingRequiredProperties().isEmpty()) {
      throw ex;
   }
}
```

#### **2）obtainFreshBeanFactory()** 

**准备此上下文以进行刷新，设置其启动日期和活动标志以及执行属性源的任何初始化。**

```java
/**
 * Tell the subclass to refresh the internal bean factory.
 * 告诉子类刷新内部bean工厂。
 * @return the fresh BeanFactory instance
 * @see #refreshBeanFactory()
 * @see #getBeanFactory()
 */
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
  //此实现执行此上下文的基础bean工厂的实际刷新，关闭先前的bean工厂（如果有），并初始化上一个生命周期的下一阶段的新bean工厂。
   refreshBeanFactory();
    //获取刷新后的 beanFactory
   ConfigurableListableBeanFactory beanFactory = getBeanFactory();
   if (logger.isDebugEnabled()) {
      logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
   }
   return beanFactory;
}
```



BeanFactory 是spring中最常见的接口之一，很多容器都继承了 这个接口，容易让初学者混淆。但是记住，BeanFactory只是一个规范，这个规范主要规定了获取 bean的口。ApplicationContext也继承了这个接口，不是有时候，并不是为了能够作为一个bean工厂，产生bean，而是方便获取到bean而已。很多继承这个接口的类，并没有具体实现它，而是将实现逻辑交给了org.springframework.beans.factory.support.DefaultListableBeanFactory这样的真正的工厂类。再比如：dubbo中的ReferenceBean，也实现了BeanFatory接口，是为了给调用方提供接口实例。



**refreshBeanFactory()**：

刷新工厂实例。为什么不直接返回，beanfactory实例，而是在使用getBeanFactory()方法去获取。高度封装，让属性的set和get只有一个入口，使得调用方都保持统一，维护和改变都很容易，是值得借鉴的。

```java
/**
 * This implementation performs an actual refresh of this context's underlying
 * bean factory, shutting down the previous bean factory (if any) and
 * initializing a fresh bean factory for the next phase of the context's lifecycle.
    * 此实现执行此上下文的基础bean工厂的实际刷新，关闭先前的bean工厂（如果有），并初始化上一个生命周期的下一阶段的新bean工厂。
 */
@Override
protected final void refreshBeanFactory() throws BeansException {
   // 这里判断是否已经建立了beanFactory，如果存在就销毁并关闭
   if (hasBeanFactory()) {
      destroyBeans();
      closeBeanFactory();
   }
   try {
      // 默认使用DefaultListableBeanFactory
      DefaultListableBeanFactory beanFactory = createBeanFactory();
      //设置ID标识
      beanFactory.setSerializationId(getId());
      //定制bean factory，设置创建beanFactory的许可条件，准入门槛
      customizeBeanFactory(beanFactory);
	  //org.springframework.context.support.AbstractXmlApplicationContext
      loadBeanDefinitions(beanFactory);
      synchronized (this.beanFactoryMonitor) {
         this.beanFactory = beanFactory;
      }
   }
   catch (IOException ex) {
      throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
   }
}

//创建beanFactory，使用的是DefaultListableBeanFactory
protected DefaultListableBeanFactory createBeanFactory() {
        /**
         *getInternalParentBeanFactory（），parent对象是在创建ApplicationContext对象时，传入的
         * @see ClassPathXmlApplicationContext#ClassPathXmlApplicationContext(String[], ApplicationContext)
         * 第二个参数，就是parent对象
         */
		return new DefaultListableBeanFactory(getInternalParentBeanFactory());
	}

//定制beanFactory
protected void customizeBeanFactory(DefaultListableBeanFactory beanFactory) {
		//是否允许使用相同名称重新注册不同的定义，默认为true
		if (this.allowBeanDefinitionOverriding != null) {
		beanFactory.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
		}
		// 是否允许循环引用,默认为true
		if (this.allowCircularReferences != null) {
			beanFactory.setAllowCircularReferences(this.allowCircularReferences);
		}
	}
```

关于循环引用，在单例中是默认是可以进行循环引用的，而在多例中，循环引用是会报错的。

- loadBeanDefinitions(beanFactory):

  解析xml文件，将解析出来的beanDefinition注册到 DefaultListableBeanFactory.beanDefinitionMap中。

```java
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
   // Create a new XmlBeanDefinitionReader for the given BeanFactory.
   // beanFactory的xml文件读取包装类
   XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

   // Configure the bean definition reader with this context's
   // resource loading environment.
   beanDefinitionReader.setEnvironment(this.getEnvironment());
   beanDefinitionReader.setResourceLoader(this);
   beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

   // Allow a subclass to provide custom initialization of the reader,
   // then proceed with actually loading the bean definitions.
   initBeanDefinitionReader(beanDefinitionReader);
   //调用此方法解析xml，获取beanDefinitionHolder
   loadBeanDefinitions(beanDefinitionReader);
}
```