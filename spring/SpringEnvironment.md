# SpringEnvironment

https://docs.spring.io/spring/docs/5.1.4.RELEASE/spring-framework-reference/core.html#beans-environment

## `Environment`抽象

`org.springframework.core.env.Environment`

- `org.springframework.core.env.ConfigurableEnvironment`

  - `org.springframework.core.env.AbstractEnvironment`

    - `org.springframework.core.env.StandardEnvironment`

      

```java
public interface Environment extends PropertyResolver {

	String[] getActiveProfiles();

	String[] getDefaultProfiles();

	boolean acceptsProfiles(String... profiles);

}
//Environment抽象
public abstract class AbstractEnvironment implements ConfigurableEnvironment {
    protected final Log logger = LogFactory.getLog(getClass());

    private final Set<String> activeProfiles = 
        new LinkedHashSet<>();

    private final Set<String> defaultProfiles = 
        new LinkedHashSet<>(getReservedDefaultProfiles());

    //源
    private final MutablePropertySources propertySources = 
        new MutablePropertySources(this.logger);

    //propertyResolver
    private final ConfigurablePropertyResolver propertyResolver =
        new PropertySourcesPropertyResolver(this.propertySources);
}
```

`AbstractEnvironment`管理`MutablePropertySources`

## `PropertySource` 抽象

`org.springframework.core.env.PropertySource`

```java
public abstract class PropertySource<T> {

	protected final Log logger = LogFactory.getLog(getClass());

	protected final String name;

	protected final T source;
}
```

由`PropertySources`聚合

`org.springframework.core.env.PropertySources`

- `org.springframework.core.env.MutablePropertySources`

```java
public interface PropertySources extends Iterable<PropertySource<?>> {

   /**
    * Return whether a property source with the given name is contained.
    * @param name the {@linkplain PropertySource#getName() name of the property source} to find
    */
   boolean contains(String name);

   /**
    * Return the property source with the given name, {@code null} if not found.
    * @param name the {@linkplain PropertySource#getName() name of the property source} to find
    */
   @Nullable
   PropertySource<?> get(String name);

}
```

`org.springframework.core.env.MutablePropertySources`

```java
public class MutablePropertySources implements PropertySources {
	private final Log logger;
	private final List<PropertySource<?>> propertySourceList = 
        new CopyOnWriteArrayList<>();
    
    //对PropertySource进行，增删改查的操作
    //addFirst()
    //addLast()
    //addBefore()
    //addAfter()
    //remove()
    //replace()
}
```



由`PropertyResolver`处理

`org.springframework.core.env.PropertyResolver`

- `org.springframework.core.env.AbstractPropertyResolver`
  - `org.springframework.core.env.PropertySourcesPropertyResolver`

```java
protected <T> T getProperty(String key, Class<T> targetValueType, boolean resolveNestedPlaceholders) {
   if (this.propertySources != null) {
       //迭代
      for (PropertySource<?> propertySource : this.propertySources) {
         if (logger.isTraceEnabled()) {
            logger.trace("Searching for key '" + key + "' in PropertySource '" +
                  propertySource.getName() + "'");
         }
          //取值
         Object value = propertySource.getProperty(key);
         if (value != null) {
            if (resolveNestedPlaceholders && value instanceof String) {
               value = resolveNestedPlaceholders((String) value);
            }
            logKeyFound(key, propertySource, value);
             //转化
            return convertValueIfNecessary(value, targetValueType);
         }
      }
   }
   if (logger.isDebugEnabled()) {
      logger.debug("Could not find key '" + key + "' in any property source");
   }
   return null;
}
```

`org.springframework.core.env.PropertyResolver#resolvePlaceholders`

解析占位符

```java
private PropertyPlaceholderHelper nonStrictHelper;
private PropertyPlaceholderHelper strictHelper;

public String resolvePlaceholders(String text) {
    if (this.nonStrictHelper == null) {
        this.nonStrictHelper = createPlaceholderHelper(true);
    }
    return doResolvePlaceholders(text, this.nonStrictHelper);
}
```

## 与`ApplicationContext`关联

`org.springframework.core.env.EnvironmentCapable`

- `org.springframework.context.ApplicationContext`

```java
public ConfigurableEnvironment getEnvironment() {
   if (this.environment == null) {
      this.environment = createEnvironment();
   }
   return this.environment;
}
```