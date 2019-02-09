# Spring事件

## 事件 

`org.springframework.context.ApplicationEvent`

- `org.springframework.context.event.ApplicationContextEvent`
  - `org.springframework.context.event.ContextClosedEvent`
  - `org.springframework.context.event.ContextRefreshedEvent`
  - `org.springframework.context.event.ContextStoppedEvent`
  - `org.springframework.context.event.ContextStartedEvent`

## 监听器

`org.springframework.context.ApplicationListener`

```java
@FunctionalInterface
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {

	/**
	 * Handle an application event.
	 * @param event the event to respond to
	 */
	void onApplicationEvent(E event);

}
```

## 广播器

`org.springframework.context.event.ApplicationEventMulticaster`

- `org.springframework.context.event.AbstractApplicationEventMulticaster`
  - `org.springframework.context.event.SimpleApplicationEventMulticaster`

```java
public interface ApplicationEventMulticaster {

	/**
	 * Add a listener to be notified of all events.
	 * @param listener the listener to add
	 */
	void addApplicationListener(ApplicationListener<?> listener);

	/**
	 * Add a listener bean to be notified of all events.
	 * @param listenerBeanName the name of the listener bean to add
	 */
	void addApplicationListenerBean(String listenerBeanName);

	/**
	 * Remove a listener from the notification list.
	 * @param listener the listener to remove
	 */
	void removeApplicationListener(ApplicationListener<?> listener);

	/**
	 * Remove a listener bean from the notification list.
	 * @param listenerBeanName the name of the listener bean to add
	 */
	void removeApplicationListenerBean(String listenerBeanName);

	/**
	 * Remove all listeners registered with this multicaster.
	 * <p>After a remove call, the multicaster will perform no action
	 * on event notification until new listeners are being registered.
	 */
	void removeAllListeners();

	/**
	 * Multicast the given application event to appropriate listeners.
	 * <p>Consider using {@link #multicastEvent(ApplicationEvent, ResolvableType)}
	 * if possible as it provides a better support for generics-based events.
	 * @param event the event to multicast
	 */
	void multicastEvent(ApplicationEvent event);

	/**
	 * Multicast the given application event to appropriate listeners.
	 * <p>If the {@code eventType} is {@code null}, a default type is built
	 * based on the {@code event} instance.
	 * @param event the event to multicast
	 * @param eventType the type of event (can be null)
	 * @since 4.2
	 */
	void multicastEvent(ApplicationEvent event, @Nullable ResolvableType eventType);

}
```

### `multicastEvent(ApplicationEvent, ResolvableType)`实现

```java
public void multicastEvent(...) {
    ResolvableType type = (eventType != null ? 
                           eventType : resolveDefaultEventType(event));
    for (final ApplicationListener<?> listener : getApplicationListeners(event, type)){
        Executor executor = getTaskExecutor();
        //异步执行
        if (executor != null) {
            executor.execute(() -> invokeListener(listener, event));
        }
        else {
            //执行
            invokeListener(listener, event);
        }
    }
}

```

**getApplicationListeners(event, type)**

```java
//取出事件对应的监听器ApplicationListener
protected Collection<ApplicationListener<?>> getApplicationListeners(...) {
    ...
    retrieveApplicationListeners(eventType, sourceType, retriever)
    ...
｝
        
private Collection<ApplicationListener<?>> retrieveApplicationListeners(...) {

    List<ApplicationListener<?>> allListeners = new ArrayList<>();
    //待处理的监听器
    Set<ApplicationListener<?>> listeners;
    //待处理的监听器Bean
    Set<String> listenerBeans;
    ...
    return allListeners;
}
//判断是否支持该事件
protected boolean supportsEvent(...) {

    //适配器模式 全部适配为GenericApplicationListener
    GenericApplicationListener smartListener = 
        (listener instanceof GenericApplicationListener ? (GenericApplicationListener) listener : new GenericApplicationListenerAdapter(listener));
    return (smartListener.supportsEventType(eventType) 
            && smartListener.supportsSourceType(sourceType));
}
```

**invokeListener(listener, event)**

```java
protected void invokeListener(ApplicationListener<?> listener, ApplicationEvent event) {
    ErrorHandler errorHandler = getErrorHandler();
    if (errorHandler != null) {
        try {
            //执行 listtener.onApplicationEvent()
            doInvokeListener(listener, event);
        }
        catch (Throwable err) {
            errorHandler.handleError(err);
        }
    }
    else {
        doInvokeListener(listener, event);
    }
}
```

### `addApplicationListener(ApplicationListener<?> listener)`实现

```java
public void addApplicationListener(ApplicationListener<?> listener) {
    synchronized (this.retrievalMutex) {
        // Explicitly remove target for a proxy, if registered already,
        // in order to avoid double invocations of the same listener.
        Object singletonTarget = AopProxyUtils.getSingletonTarget(listener);
        if (singletonTarget instanceof ApplicationListener) {
            this.defaultRetriever.applicationListeners.remove(singletonTarget);
        }
        this.defaultRetriever.applicationListeners.add(listener);
        this.retrieverCache.clear();
    }
}
//defaultRetriever结构
private class ListenerRetriever {

    public final Set<ApplicationListener<?>> applicationListeners = 
        new LinkedHashSet<>();

    public final Set<String> applicationListenerBeans = new LinkedHashSet<>();

    private final boolean preFiltered;
}
```



## 与`ApplicationContext`的关联

- `org.springframework.context.ApplicationEventPublisher`
  - `org.springframework.context.ApplicationContext`

```java
@FunctionalInterface
public interface ApplicationEventPublisher {

	/**
	 * Notify all <strong>matching</strong> listeners registered with this
	 * application of an application event. Events may be framework events
	 * (such as RequestHandledEvent) or application-specific events.
	 * @param event the event to publish
	 * @see org.springframework.web.context.support.RequestHandledEvent
	 */
	default void publishEvent(ApplicationEvent event) {
		publishEvent((Object) event);
	}

	/**
	 * Notify all <strong>matching</strong> listeners registered with this
	 * application of an event.
	 * <p>If the specified {@code event} is not an {@link ApplicationEvent},
	 * it is wrapped in a {@link PayloadApplicationEvent}.
	 * @param event the event to publish
	 * @since 4.2
	 * @see PayloadApplicationEvent
	 */
	void publishEvent(Object event);

}
```

### publishEvent(Object event)实现

```java
protected void publishEvent(Object event, @Nullable ResolvableType eventType) {
   Assert.notNull(event, "Event must not be null");
   if (logger.isTraceEnabled()) {
      logger.trace("Publishing event in " + getDisplayName() + ": " + event);
   }

   // Decorate event as an ApplicationEvent if necessary
   //非ApplicationEvent类型装饰为ApplicationEvent类型
   ApplicationEvent applicationEvent;
   if (event instanceof ApplicationEvent) {
      applicationEvent = (ApplicationEvent) event;
   }
   else {
      applicationEvent = new PayloadApplicationEvent<>(this, event);
      if (eventType == null) {
         eventType = ((PayloadApplicationEvent) applicationEvent).getResolvableType();
      }
   }

   // Multicast right now if possible - or lazily once the multicaster is initialized
   if (this.earlyApplicationEvents != null) {
      //监听器未初始化前，先存起来
      this.earlyApplicationEvents.add(applicationEvent);
   }
   else {
       //获取ApplicationEventMulticaster并发布事件
      getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
   }

   // 父上下文再发布一次
   if (this.parent != null) {
      if (this.parent instanceof AbstractApplicationContext) {
         ((AbstractApplicationContext) this.parent).publishEvent(event, eventType);
      }
      else {
         this.parent.publishEvent(event);
      }
   }
}
```