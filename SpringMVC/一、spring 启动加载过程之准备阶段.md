日期： 2019-03-21

编写spring启动入口：

![image](https://upload-images.jianshu.io/upload_images/14771531-ff01e9b568bd67d6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

ClassPathXmlApplicationContext构造器的类加载逻辑：使用给定父级创建新的ClassPathXmlApplicationContext，  从给定的XML文件加载定义的类。

![image](https://upload-images.jianshu.io/upload_images/14771531-ad637bc860389871.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这篇文章主要讲解：super(parent) 和 setConfigLocations(configLocations) 逻辑。

> **1 . supper（parent）的逻辑：调用到父类**

org.springframework.context.support.AbstractApplicationContext() 构造器。

```java
      /**
    *@see org.springframework.context.support.AbstractApplicationContext#setParent(ApplicationContext)
    */
	public AbstractApplicationContext(ApplicationContext parent) {
               //对用户设置的配置文件位置处理，主要是通配符（${ }）的处理
		this();
                //设置父类容器,合并环境变量。主要是系统变量和jvm变量
		setParent(parent);
	}
```
设置父类容器，合并环境变量：在spring作为父容器时，springmvc将spring父容器引用过来，可以获取到spring容器的参数和配置，但是父容器不能访问子容器的配置，原因就在于此。
```java
	public void setParent(ApplicationContext parent) {
		this.parent = parent;
		if (parent != null) {
			Environment parentEnvironment = parent.getEnvironment();
			if (parentEnvironment instanceof ConfigurableEnvironment) {
				getEnvironment().merge((ConfigurableEnvironment) parentEnvironment);
			}
		}
	}
````

接下看看环境变量到底是做什么的：parent.getEnvironment();

```java
	public ConfigurableEnvironment getEnvironment() {
		if (this.environment == null) {
			this.environment = createEnvironment();
		}
		return this.environment;
	}

	protected ConfigurableEnvironment createEnvironment() {
		return new StandardEnvironment();
	}
```



```java
public class StandardEnvironment extends AbstractEnvironment {

	/** System environment property source name: {@value} */
	public static final String SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME = "systemEnvironment";

	/** JVM system properties property source name: {@value} */
	public static final String SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME = "systemProperties";


	/**
	 * 将系统参数变量和jvm参数变量加入缓存map缓存
	 * Customize the set of property sources with those appropriate for any standard
	 * Java environment:
	 * <ul>
	 * <li>{@value #SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME}
	 * <li>{@value #SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME}
	 * </ul>
	 * <p>Properties present in {@value #SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME} will
	 * take precedence over those in {@value #SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME}.
	 * @see AbstractEnvironment#customizePropertySources(MutablePropertySources)
	 * @see #getSystemProperties()
	 * @see #getSystemEnvironment()
	 */
	@Override
	protected void customizePropertySources(MutablePropertySources propertySources) {
		propertySources.addLast(new MapPropertySource(SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME, getSystemProperties()));
		propertySources.addLast(new SystemEnvironmentPropertySource(SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME, getSystemEnvironment()));
	}

}
	//获取系统参数变量
	public Map<String, Object> getSystemEnvironment() {
		if (suppressGetenvAccess()) {
			return Collections.emptyMap();
		}
		try {
			return (Map) System.getenv();
		}
		catch (AccessControlException ex) {
			return (Map) new ReadOnlySystemAttributesMap() {
				@Override
				protected String getSystemAttribute(String attributeName) {
					try {
						return System.getenv(attributeName);
					}
					catch (AccessControlException ex) {
						if (logger.isInfoEnabled()) {
							logger.info("Caught AccessControlException when accessing system environment variable '" +
									attributeName + "'; its value will be returned [null]. Reason: " + ex.getMessage());
						}
						return null;
					}
				}
			};
		}
	}

	//获取jvm参数变量
	@SuppressWarnings({"unchecked", "rawtypes"})
	public Map<String, Object> getSystemProperties() {
		try {
			return (Map) System.getProperties();
		}
		catch (AccessControlException ex) {
			return (Map) new ReadOnlySystemAttributesMap() {
				@Override
				protected String getSystemAttribute(String attributeName) {
					try {
						return System.getProperty(attributeName);
					}
					catch (AccessControlException ex) {
						if (logger.isInfoEnabled()) {
							logger.info("Caught AccessControlException when accessing system property '" +
									attributeName + "'; its value will be returned [null]. Reason: " + ex.getMessage());
						}
						return null;
					}
				}
			};
		}
	}
```

最后，merge方法融合父类容器配置：getEnvironment().merge((ConfigurableEnvironment) parentEnvironment)：

```java
@Override
	public void merge(ConfigurableEnvironment parent) {
		for (PropertySource<?> ps : parent.getPropertySources()) {
			if (!this.propertySources.contains(ps.getName())) {
				this.propertySources.addLast(ps);
			}
		}
        /**
        获取明确配置的profile（key="spring.profiles.active"，默认为key="spring.profiles.default")
        eg 通过配置jvm参数方式设置：-Dspring.profiles.active=dev  
           通过web.xml方式来设置：
        	<context-param>  
                <param-name>spring.profiles.active</param-name>  
                <param-value>dev</param-value>  
            </context-param>  
        
        
        */
		String[] parentActiveProfiles = parent.getActiveProfiles();
		if (!ObjectUtils.isEmpty(parentActiveProfiles)) {
			synchronized (this.activeProfiles) {
				for (String profile : parentActiveProfiles) {
					this.activeProfiles.add(profile);
				}
			}
		}
		String[] parentDefaultProfiles = parent.getDefaultProfiles();
		if (!ObjectUtils.isEmpty(parentDefaultProfiles)) {
			synchronized (this.defaultProfiles) {
				this.defaultProfiles.remove(RESERVED_DEFAULT_PROFILE_NAME);
				for (String profile : parentDefaultProfiles) {
					this.defaultProfiles.add(profile);
				}
			}
		}
	}
```





> **2 . setConfigLocations（String[]）处理配置文件位置**

```java
	/**
	 * Set the config locations for this application context.
	 * <p>If not set, the implementation may use a default as appropriate.
	 * 对配置的配置文件的位置进行通配符的处理，然后，保存在configLocations数组中
	 */
	public void setConfigLocations(String... locations) {
		if (locations != null) {
			Assert.noNullElements(locations, "Config locations must not be null");
			this.configLocations = new String[locations.length];
			for (int i = 0; i < locations.length; i++) {
				//对配置文件路径的${}转义，分隔符为“：”
				this.configLocations[i] = resolvePath(locations[i]).trim();
			}
		}
		else {
			this.configLocations = null;
		}
	}
```

resolvePath（String）方法最后会调用到PropertyPlaceholderHelper#parseStringValue（String , PlaceholderResolver , Set<String>）方法：

1）查找${key}通配符，找到其中key对应value

2）递归查找${key}，替换为value。

3）如果value为null，则查找分隔符":"的位置，截取分割符后面的值作为value

```java
protected String parseStringValue(
      String value, PlaceholderResolver placeholderResolver, Set<String> visitedPlaceholders) {

   StringBuilder result = new StringBuilder(value);
	//查找 “${” 的位置
   int startIndex = value.indexOf(this.placeholderPrefix);
   while (startIndex != -1) {
       // 查找最后一个 “}” 的位置
      int endIndex = findPlaceholderEndIndex(result, startIndex);
      if (endIndex != -1) {
         String placeholder = result.substring(startIndex + this.placeholderPrefix.length(), endIndex);
         String originalPlaceholder = placeholder;
         if (!visitedPlaceholders.add(originalPlaceholder)) {
            throw new IllegalArgumentException(
                  "Circular placeholder reference '" + originalPlaceholder + "' in property definitions");
         }
         // Recursive invocation, parsing placeholders contained in the placeholder key.
          //递归获取${}中的值
         placeholder = parseStringValue(placeholder, placeholderResolver, visitedPlaceholders);
         // Now obtain the value for the fully resolved key...
         String propVal = placeholderResolver.resolvePlaceholder(placeholder);
         if (propVal == null && this.valueSeparator != null) {
             // propVal 为null时，查找分隔符“：”后面的值作为value
            int separatorIndex = placeholder.indexOf(this.valueSeparator);
            if (separatorIndex != -1) {
               String actualPlaceholder = placeholder.substring(0, separatorIndex);
               String defaultValue = placeholder.substring(separatorIndex + this.valueSeparator.length());
               propVal = placeholderResolver.resolvePlaceholder(actualPlaceholder);
               if (propVal == null) {
                  propVal = defaultValue;
               }
            }
         }
         if (propVal != null) {
            // Recursive invocation, parsing placeholders contained in the
            // previously resolved placeholder value.
            propVal = parseStringValue(propVal, placeholderResolver, visitedPlaceholders);
             //将${}获取的value，替换原来的${}
            result.replace(startIndex, endIndex + this.placeholderSuffix.length(), propVal);
            if (logger.isTraceEnabled()) {
               logger.trace("Resolved placeholder '" + placeholder + "'");
            }
            startIndex = result.indexOf(this.placeholderPrefix, startIndex + propVal.length());
         }
         else if (this.ignoreUnresolvablePlaceholders) {
            // Proceed with unprocessed value.
            startIndex = result.indexOf(this.placeholderPrefix, endIndex + this.placeholderSuffix.length());
         }
         else {
            throw new IllegalArgumentException("Could not resolve placeholder '" +
                  placeholder + "'" + " in value \"" + value + "\"");
         }
         visitedPlaceholders.remove(originalPlaceholder);
      }
      else {
         startIndex = -1;
      }
   }

   return result.toString();
}
```