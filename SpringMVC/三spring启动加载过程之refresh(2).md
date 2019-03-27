日期：2019-3-27

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

#### 3）prepareBeanFactory:

配置工厂的标准上下文特征，例如上下文的ClassLoader和后处理器。@Qualifier与@Autowired等注解正是在这一步骤中增加的支持  

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    //类加载器设置
   beanFactory.setBeanClassLoader(getClassLoader());
   beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    //org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader资源编辑器
   beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));
   // Configure the bean factory with context callbacks.
   // 下面几行的意思就是，如果某个 bean 依赖于以下几个接口的实现类，在自动装配的时候忽略它们，
   // Spring 会通过其他方式来处理这些依赖。
   beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
   beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
   beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
   beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
   beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
   beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
   beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

   // BeanFactory interface not registered as resolvable type in a plain factory.
   // MessageSource registered (and found for autowiring) as a bean.
    /**
    * 下面几行就是为特殊的几个 bean 赋值，如果有 bean 依赖了以下几个，会注入这边相应的值，
    * 之前我们说过，"当前 ApplicationContext 持有一个 BeanFactory"，这里解释了第一行
    * ApplicationContext 还继承了 ResourceLoader、ApplicationEventPublisher、MessageSource
    * 所以对于这几个依赖，可以赋值为 this，注意 this 是一个 ApplicationContext
    * 那这里怎么没看到为 MessageSource 赋值呢？那是因为 MessageSource 被注册成为了一个普通的 bean
    */
   beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
   beanFactory.registerResolvableDependency(ResourceLoader.class, this);
   beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
   beanFactory.registerResolvableDependency(ApplicationContext.class, this);

   // Register early post-processor for detecting inner beans as ApplicationListeners.
    //添加beanFactory的后置处理器，用于后面的回调
   beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

    
   // Detect a LoadTimeWeaver and prepare for weaving, if found.
    //切面织入标识
   if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
      beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
      // Set a temporary ClassLoader for type matching.
      beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
   }

   // Register default environment beans.
    //没有设置环境，设置环境变量
   if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
      beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
   }
    //检查设置jvm参数变量
   if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
      beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
   }
    //检查设置系统参数变量
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

  属性编辑器，此时的EditorRegistrar对象是beanFactory默认的。如果需要自己定义一个PropertyEditorRegistrar 属性编辑器，可以自定义一个，实现PropertyEditorRegistrar 接口，重写registerCustomEditors（registy）方法，注入CustomEditorConfigurer，spring初始化的时候，会在invokeBeanFactoryPostProcessors调用后置处理器时候调用CustomEditorConfigurer（BeanFactoryPostProcessor接口的实现类）#postProcessBeanFactory方法，

  为beanFactory注册ResourceEditorRegistrar。每个bean都会装载全套的属性编辑器，并在之后使用相应的属性编辑器来格式化属性值。

  ```java
  public class DatePropertyEditorRegistrar implements PropertyEditorRegistrar {
      @Override
      public void registerCustomEditors(PropertyEditorRegistry registry) {
          registry.registerCustomEditor(Date.class, new CustomDateEditor(new SimpleDateFormat("yyyy-MM-dd"), true));
      }
  }
  ```

  ```xml
  <bean class="org.springframework.beans.factory.config.CustomEditorConfigurer">
      <property name="propertyEditorRegistrars">
          <list>
              <bean class="org.springframework.demo.DatePropertyEditorRegistrar"></bean>
          </list>
      </property>
  </bean>
  ```

  ```java
  protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
  		try {
  			Object beanInstance;
  			final BeanFactory parent = this;
  			if (System.getSecurityManager() != null) {
  				beanInstance = AccessController.doPrivileged(new PrivilegedAction<Object>() {
  					@Override
  					public Object run() {
  						return getInstantiationStrategy().instantiate(mbd, beanName, parent);
  					}
  				}, getAccessControlContext());
  			}
  			else {
  				beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
  			}
  			BeanWrapper bw = new BeanWrapperImpl(beanInstance);
              //初始化beanWrapper
  			initBeanWrapper(bw);
  			return bw;
  		}
  		catch (Throwable ex) {
  			throw new BeanCreationException(
  					mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
  		}
  	}
  
  protected void initBeanWrapper(BeanWrapper bw) {
  		bw.setConversionService(getConversionService());
      	//状态自定义的属性编辑器
  		registerCustomEditors(bw);
  	}
  
  protected void registerCustomEditors(PropertyEditorRegistry registry) {
  		PropertyEditorRegistrySupport registrySupport =
  				(registry instanceof PropertyEditorRegistrySupport ? (PropertyEditorRegistrySupport) registry : null);
  		if (registrySupport != null) {
  			registrySupport.useConfigValueEditors();
  		}
  		if (!this.propertyEditorRegistrars.isEmpty()) {
              //遍历用户所有自定义的属性编辑器
  			for (PropertyEditorRegistrar registrar : this.propertyEditorRegistrars) {
  				try {
                      //注册
  					registrar.registerCustomEditors(registry);
  				}
  				catch (BeanCreationException ex) {
  					Throwable rootCause = ex.getMostSpecificCause();
  					if (rootCause instanceof BeanCurrentlyInCreationException) {
  						BeanCreationException bce = (BeanCreationException) rootCause;
  						if (isCurrentlyInCreation(bce.getBeanName())) {
  							if (logger.isDebugEnabled()) {
  								logger.debug("PropertyEditorRegistrar [" + registrar.getClass().getName() +
  										"] failed because it tried to obtain currently created bean '" +
  										ex.getBeanName() + "': " + ex.getMessage());
  							}
  							onSuppressedException(ex);
  							continue;
  						}
  					}
  					throw ex;
  				}
  			}
  		}
  		if (!this.customEditors.isEmpty()) {
  			for (Map.Entry<Class<?>, Class<? extends PropertyEditor>> entry : this.customEditors.entrySet()) {
  				Class<?> requiredType = entry.getKey();
  				Class<? extends PropertyEditor> editorClass = entry.getValue();
  				registry.registerCustomEditor(requiredType, BeanUtils.instantiateClass(editorClass));
  			}
  		}
  	}
  
  // 在实例化lbean之后，属性填充时，找到匹配的类型转化，进行formateorg.springframework.beans.TypeConverterDelegate#doConvertValue(oldValue,newValue,requiredType,editor)
  private Object doConvertValue(Object oldValue, Object newValue, Class<?> requiredType, PropertyEditor editor) {
  		Object convertedValue = newValue;
  
  		if (editor != null && !(convertedValue instanceof String)) {
  			// Not a String -> use PropertyEditor's setValue.
  			// With standard PropertyEditors, this will return the very same object;
  			// we just want to allow special PropertyEditors to override setValue
  			// for type conversion from non-String values to the required type.
  			try {
                  //实际格式化的入口
  				editor.setValue(convertedValue);
                  //获取格式化之后的值
  				Object newConvertedValue = editor.getValue();
  				if (newConvertedValue != convertedValue) {
  					convertedValue = newConvertedValue;
  					// Reset PropertyEditor: It already did a proper conversion.
  					// Don't use it again for a setAsText call.
  					editor = null;
  				}
  			}
  			catch (Exception ex) {
  				if (logger.isDebugEnabled()) {
  					logger.debug("PropertyEditor [" + editor.getClass().getName() + "] does not support setValue call", ex);
  				}
  				// Swallow and proceed.
  			}
  		}
  
  		Object returnValue = convertedValue;
  
  		if (requiredType != null && !requiredType.isArray() && convertedValue instanceof String[]) {
  			// Convert String array to a comma-separated String.
  			// Only applies if no PropertyEditor converted the String array before.
  			// The CSV String will be passed into a PropertyEditor's setAsText method, if any.
  			if (logger.isTraceEnabled()) {
  				logger.trace("Converting String array to comma-delimited String [" + convertedValue + "]");
  			}
  			convertedValue = StringUtils.arrayToCommaDelimitedString((String[]) convertedValue);
  		}
  
  		if (convertedValue instanceof String) {
  			if (editor != null) {
  				// Use PropertyEditor's setAsText in case of a String value.
  				if (logger.isTraceEnabled()) {
  					logger.trace("Converting String to [" + requiredType + "] using property editor [" + editor + "]");
  				}
  				String newTextValue = (String) convertedValue;
  				return doConvertTextValue(oldValue, newTextValue, editor);
  			}
  			else if (String.class == requiredType) {
  				returnValue = convertedValue;
  			}
  		}
  
  		return returnValue;
  	}
  ```



