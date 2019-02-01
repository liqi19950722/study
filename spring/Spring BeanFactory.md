# Spring IOC

## IOC/DI

- IOC: 控制反转

  把创建对象的工作交给容器实现。

- DI:依赖注入

  - 字段注入
  - 构造注入
  - 方法注入
  - 接口注入`org.springframework.beans.factory.Aware`

## BeanFactory

Bean容器

- org.springframework.beans.factory.BeanFactory`
  - `org.springframework.beans.factory.support.DefaultListableBeanFactory`

Bean定义

- `org.springframework.beans.factory.config.BeanDefinition`
  - `org.springframework.beans.factory.annotation.AnnotatedBeanDefinition`
  - `org.springframework.beans.factory.support.RootBeanDefinition`
  - `org.springframework.beans.factory.support.ChildBeanDefinition`

Bean读取器 -> 针对XML形式

- `org.springframework.beans.factory.support.BeanDefinitionReader`
  - `org.springframework.beans.factory.xml.XmlBeanDefinitionReader`

Bean注册器

- `org.springframework.beans.factory.support.BeanDefinitionRegistry`

### 创建Bean过程

`org.springframework.beans.factory.support.AbstractBeanFactory`

```java
protected <T> T doGetBean(...) throws BeansException {
    //解析Bean名称
    final String beanName = transformedBeanName(name);
    //从单例缓存中查找
    Object sharedInstance = getSingleton(beanName);
    if (sharedInstance != null && args == null) {
        ...
        bean = getObjectForBeanInstance(...);
    ｝
    else{
        ...
        // 从父类BeanFactory中查找
        BeanFactory parentBeanFactory = getParentBeanFactory();
        ...
            //创建实例
        if (mbd.isSingleton()) {
            sharedInstance = getSingleton(beanName, () -> {
				try {
                    //创建Bean
					return createBean(beanName, mbd, args);
				}
				catch (BeansException ex) {
					destroySingleton(beanName);
					throw ex;
				}
			});
			bean = getObjectForBeanInstance(...);
        ｝
        else{
            //多例情况
            //其他情况
            ...
       }
    }
    // TypeConverter.convertIfNecessary(...)
}
```

`org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory`

```java
protected Object createBean(...)throws BeanCreationException {
	...
    //解析Bean的Class
    Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
    ...
    
    // BeanPostProcess  返回一个代理的bean实例
    Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
    
    //创建Bean
    Object beanInstance = doCreateBean(beanName, mbdToUse, args);
    
}
```

```java
protected Object doCreateBean()throws BeanCreationException {
    // 实例化一个Bean.
	BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {
        //创建Bean实例
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    final Object bean = instanceWrapper.getWrappedInstance();
    Class<?> beanType = instanceWrapper.getWrappedClass();
    ...
    // 修改 合并beanDefinition
    applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
    ...
    //填充Bean
    populateBean(beanName, mbd, instanceWrapper);
    //实例化Bean
	exposedObject = initializeBean(beanName, exposedObject, mbd);
    ...
    //注册可销毁的Bean
    registerDisposableBeanIfNecessary(beanName, bean, mbd);
}
```

```java
protected BeanWrapper createBeanInstance(...) {
    ...
    // 用supplier的方式创建
    Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
    if (instanceSupplier != null) {
        return obtainFromSupplier(instanceSupplier, beanName);
    }
	//用工厂方法创建
    if (mbd.getFactoryMethodName() != null) {
        return instantiateUsingFactoryMethod(beanName, mbd, args);
    }
    //带参数构造
    autowireConstructor(...);
    //使用默认构造器
    instantiateBean();
}
```

```java
protected BeanWrapper instantiateBean(...) {
    ...
    //选择创建策略 JDK反射创键或CGLIB创建
    beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
    ...  
    initBeanWrapper(bw);
}

```

```java
protected void populateBean(...) {
    ...
    ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)
    ...
    PropertyValues pvsToUse = ibp.postProcessProperties(...);
    ...
    // 解析并依赖注入   
	applyPropertyValues(beanName, mbd, bw, pvs);
}
```

```java
protected Object initializeBean(...) {
  //Java 安全
  if (System.getSecurityManager() != null) {
      AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
          invokeAwareMethods(beanName, bean);
          return null;
      }, getAccessControlContext());
  }
  else {
      // 织入 BeanNameAware BeanClassLoaderAware BeanFactoryAware
      invokeAwareMethods(beanName, bean);
  }
  Object wrappedBean = bean;
  ...
  // 实例化之前的处理
  wrappedBean = applyBeanPostProcessorsBeforeInitialization(...);
  ...
  // 主要调用 InitializingBean#afterPropertiesSet 或者InitMethod
  invokeInitMethods(beanName, wrappedBean, mbd);
  ...
  // 实例化之后的处理
  wrappedBean = applyBeanPostProcessorsAfterInitialization(...);
   
}
```

### Bean的生命周期

实例化前`InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation`

实例化

实例化后`InstantiationAwareBeanPostProcessor#postProcessAfterInstantiation`

​              `InstantiationAwareBeanPostProcessor#postProcessProperties`

初始化前`BeanPostProcessor#postProcessBeforeInitialization`

初始化`InitializingBean#afterPropertiesSet`

初始化后 `BeanPostProcessor#postProcessAfterInitialization` 

销毁`DisposableBean#destroy`



## ApplicationContext

`org.springframework.context.ApplicationContext`

- `org.springframework.context.support.AbstractApplicationContext`
  - `org.springframework.context.support.ClassPathXmlApplicationContext`
  - `org.springframework.context.annotation.AnnotationConfigApplicationContext`

### 应用上下文启动过程

`org.springframework.context.support.AbstractApplicationContext#refresh`

```java
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        prepareRefresh();
        
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
        
        prepareBeanFactory(beanFactory);
        
        postProcessBeanFactory(beanFactory);

        invokeBeanFactoryPostProcessors(beanFactory);

        registerBeanPostProcessors(beanFactory);

        initMessageSource();

        initApplicationEventMulticaster();

        onRefresh();

        registerListeners();

        finishBeanFactoryInitialization(beanFactory);

        finishRefresh();
       
        //报错就清除所有
        destroyBeans();

        //关闭上下文
        cancelRefresh(ex);

        //重置缓存
        resetCommonCaches();

}
```

**prepareRefresh()**

- 激活上下文
- 创建`Environment`
- 初始化事件

**obtainFreshBeanFactory()**

- 获取一个BeanFactory
  - 对于基于XML的 发生 **载入 注册**    **定位**在创建ApplicatinContext时发生

**prepareBeanFactory()** 

- 准备一些Bean 注册默认的`Environment` Bean

**postProcessBeanFactory();**

- 对于`BeanFactory`的后置处理（扩展处理）

**invokeBeanFactoryPostProcessors()**

- `BeanFactoryPostProcessor#postProcessBeanFactory`

- `BeanDefinitionRegistryPostProcessor#postProcessBeanDefinitionRegistry`

**registerBeanPostProcessors()**

- 注册`org.springframework.beans.factory.config.BeanPostProcessor`

**initMessageSource()**

- 国际化处理

**initApplicationEventMulticaster()**

 初始化事件广播器`org.springframework.context.event.ApplicationEventMulticaster`

**onRefresh()**

- 为子类提供扩展，初始化其他信息

**registerListeners()**

- 注册监听器

**finishBeanFactoryInitialization()**

- 不允许所有BeanDefinitions发生修改

- 初始化单例Bean(non-lazy-init)

**finishRefresh()**

- 初始化生命周期组件

- 发布`ContextRefreshedEvent`事件

### 基于注解和基于XML的不同

获取`org.springframework.beans.factory.config.BeanDefinition`的方式不同

- 注解

  `org.springframework.context.annotation.AnnotatedBeanDefinitionReader`

  `org.springframework.context.annotation.ConfigurationClassBeanDefinitionReader`

  `org.springframework.context.annotation.ClassPathBeanDefinitionScanner`

- XML

  `org.springframework.beans.factory.support.BeanDefinitionReader`

  `org.springframework.beans.factory.xml.BeanDefinitionDocumentReader`

Bean注册流程

1. 获取`BeanDefinition`
2. 把`BeanDefinition` 包装为`BeanDefinitionHolder`
3. 把`BeanDefinitionHolder`由`BeanDefinitionRegistry`注册到`BeanFactory`