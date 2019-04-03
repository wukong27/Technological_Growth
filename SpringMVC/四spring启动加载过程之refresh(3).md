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
   // loadTimeWeaver 方式的切面，为beanFactory设置对应的转化类。在实例化bean的时候，会用到。
   if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
      beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
      beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
   }
}
```

主要的流程在第一步：

```
顺序：1. BeanDefinitionRegistryPostProcessor
        手动设置的 BeanDefinitionRegistryPostProcessor =>排序后的 PriorityOrdered => 排序后的 Ordered =>其他的 BeanDefinitionRegistryPostProcessor
     2. BeanFactoryPostProcessor
        排序后的 PriorityOrdered => 排序后的 Ordered => 其他的 BeanFactoryPostProcessor
```

```java
//PostProcessorRegistrationDelegate#invokeBeanFactoryPostProcessors
	public static void invokeBeanFactoryPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

		// Invoke BeanDefinitionRegistryPostProcessors first, if any.
		//存放所有的 PostProcessor 的名称
		Set<String> processedBeans = new HashSet<String>();

		// DefaultListableBeanFactory 实现了 BeanDefinitionRegistry接口的
		if (beanFactory instanceof BeanDefinitionRegistry) {
			//强转为 BeanDefinitionRegistry
			BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
			// 存放 非BeanDefinitionRegistryPostProcessor
			List<BeanFactoryPostProcessor> regularPostProcessors = new LinkedList<BeanFactoryPostProcessor>();
			//存放 BeanDefinitionRegistryPostProcessor
			List<BeanDefinitionRegistryPostProcessor> registryProcessors = new LinkedList<BeanDefinitionRegistryPostProcessor>();
			for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
				if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
					BeanDefinitionRegistryPostProcessor registryProcessor =
							(BeanDefinitionRegistryPostProcessor) postProcessor;
					//注册到BeanFactory中
					registryProcessor.postProcessBeanDefinitionRegistry(registry);
					registryProcessors.add(registryProcessor);
				}
				else {
					regularPostProcessors.add(postProcessor);
				}
			}

			// Do not initialize FactoryBeans here: We need to leave all regular beans
			// uninitialized to let the bean factory post-processors apply to them!
			// Separate between BeanDefinitionRegistryPostProcessors that implement
			// PriorityOrdered, Ordered, and the rest.
			//用来暂时存放排序后的 BeanDefinitionRegistryPostProcessor，调用过后，就clear
			List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<BeanDefinitionRegistryPostProcessor>();

			// First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
			// 筛选出 PriorityOrdered BeanDefinitionRegistryPostProcessors
			String[] postProcessorNames =
					beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
			for (String ppName : postProcessorNames) {
				if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
					currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					processedBeans.add(ppName);
				}
			}
			sortPostProcessors(currentRegistryProcessors, beanFactory);
			//放入排序之后的 BeanDefinitionRegistryPostProcessor
			registryProcessors.addAll(currentRegistryProcessors);
			//依次调用注册
			invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
			currentRegistryProcessors.clear();

			// Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
			// 筛选出 Ordered BeanDefinitionRegistryPostProcessors
			postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
			for (String ppName : postProcessorNames) {
				if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
					currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					processedBeans.add(ppName);
				}
			}
			sortPostProcessors(currentRegistryProcessors, beanFactory);
			registryProcessors.addAll(currentRegistryProcessors);
			invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
			currentRegistryProcessors.clear();

			// Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
			boolean reiterate = true;
			//在while循环中调用，for循环，会有其他线程同时对beanFactory注册bean么
			while (reiterate) {
				reiterate = false;
				postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
				for (String ppName : postProcessorNames) {
					if (!processedBeans.contains(ppName)) {
						currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
						processedBeans.add(ppName);
						reiterate = true;
					}
				}
				sortPostProcessors(currentRegistryProcessors, beanFactory);
				registryProcessors.addAll(currentRegistryProcessors);
				invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
				currentRegistryProcessors.clear();
			}

			// Now, invoke the postProcessBeanFactory callback of all processors handled so far.
			// 前面调用的是BeanDefinitionRegistryPostProcessor,可能在调用的对beanFactory注册新的
			// 现在调用 BeanFactoryPostProcessor 注册器的后置处理器
			invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
			// 现在调用  其他的 BeanFactoryPostProcessor 注册器的后置处理器
			invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
		}else {
			// Invoke factory processors registered with the context instance.
			// 如果beanFactory不是
			invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
		}
		// Do not initialize FactoryBeans here: We need to leave all regular beans
		// uninitialized to let the bean factory post-processors apply to them!
		// 查找 BeanFactoryPostProcessor 接口下的类
		String[] postProcessorNames =
				beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

		// Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
		// Ordered, and the rest.
		//实现 priorityOrdered 接口的
		List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
		//实现 ordered 接口的
		List<String> orderedPostProcessorNames = new ArrayList<String>();
		// 其他的
		List<String> nonOrderedPostProcessorNames = new ArrayList<String>();
		for (String ppName : postProcessorNames) {
			if (processedBeans.contains(ppName)) {
				// skip - already processed in first phase above
			}
			else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
			}
			else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			else {
				nonOrderedPostProcessorNames.add(ppName);
			}
		}

		//依次调用 后置处理
		// First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
		sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
		invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

		// Next, invoke the BeanFactoryPostProcessors that implement Ordered.
		List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
		for (String postProcessorName : orderedPostProcessorNames) {
			orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
		}
		sortPostProcessors(orderedPostProcessors, beanFactory);
		invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

		// Finally, invoke all other BeanFactoryPostProcessors.
		List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
		for (String postProcessorName : nonOrderedPostProcessorNames) {
			nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
		}
		invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

		// Clear cached merged bean definitions since the post-processors might have
		// modified the original metadata, e.g. replacing placeholders in values...
		beanFactory.clearMetadataCache();
	}
```

在调用后置处理器时，会调用getBean方法，得到后置处理器的实例，所以，这个启动过程，这里是第一次出来实例化。

调用后置处理器的部分，如下。

```java
/**
 * BeanDefinitionRegistryPostProcessor 注册器的后置处理器
 * Invoke the given BeanDefinitionRegistryPostProcessor beans.
 */
private static void invokeBeanDefinitionRegistryPostProcessors(
      Collection<? extends BeanDefinitionRegistryPostProcessor> postProcessors, BeanDefinitionRegistry registry) {

   for (BeanDefinitionRegistryPostProcessor postProcessor : postProcessors) {
      postProcessor.postProcessBeanDefinitionRegistry(registry);
   }
}

/**
 * BeanFactoryPostProcessor 注册器的后置处理器
 * Invoke the given BeanFactoryPostProcessor beans.
 */
private static void invokeBeanFactoryPostProcessors(
      Collection<? extends BeanFactoryPostProcessor> postProcessors, ConfigurableListableBeanFactory beanFactory) {

   for (BeanFactoryPostProcessor postProcessor : postProcessors) {
      postProcessor.postProcessBeanFactory(beanFactory);
   }
}

/**
 * Register the given BeanPostProcessor beans.
 */
private static void registerBeanPostProcessors(
      ConfigurableListableBeanFactory beanFactory, List<BeanPostProcessor> postProcessors) {

   for (BeanPostProcessor postProcessor : postProcessors) {
      beanFactory.addBeanPostProcessor(postProcessor);
   }
}
```

