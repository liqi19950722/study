# Spring事务

## 事务的基本概念

4个属性

原子性A

一致性C

隔离性I

持久性D

事务基本原理：

```java
Connection con = DriverManager.getConnection();
con.setAutoCommit(true/false);
con.commit();/con.rollback();
con.close()
```

## 事务的传播属性

| 名称            | 解释                                                         |
| --------------- | ------------------------------------------------------------ |
| `REQUIRED`      | 支持当前事务，没有则创建一个                                 |
| `REQUIRES_NEW`  | 新建事务，如果当前有事务，挂起，新建一个。相互独立           |
| `SUPPORTS`      | 支持当前事务，没有则以非事务执行                             |
| `MANDATORY`     | 支持当前事务，没有就抛出异常                                 |
| `NOT_SUPPORTED` | 以非事务形式执行，存在事务抛出异常                           |
| `NEVER`         | 以非事务形式执行，有事务抛出异常                             |
| `NESTED`        | 有活动事务存在，运行一个嵌套事务。没有活动事务，按REQUIRED执行，只对事务管理器有效 |

## 事务隔离级别

| 隔离级别           | 问题                           | 描述                                                         |
| ------------------ | ------------------------------ | ------------------------------------------------------------ |
| `Read-Uncommitted` | 脏读                           | 一个事务对数据进行CRUD，另一个事务读取到未提交数据。事务回滚，第二个事务读到脏数据。 |
| `Read-Committed`   | 避免脏读，允许不可重复读和幻读 | 一个事务中发生两次读操作，两次操作之间另一个事务对数据进行修改，导致两次读不一致。 |
| `Repeatable-Read`  | 避免脏读，不可重复读，允许幻读 | 第一个事务对一定范围的数据进行批量修改，第二个事务在这个范围增加一条数据，这时候第一个事务就会丢失对新增数据的修改 |
| `Serializable`     | 串行化执行                     |                                                              |

总结：

隔离级别越高，越能保证数据完整性，对并发性能影响越大。

`Read-commit` Oracle Sql Server

`Repitable-read` Mysql InnoDB

### Spring事务嵌套

事务传播机制：

ServiceA MethodA()内调用ServiceB MethodB()

```java
class ServiceA{
    ServiceB serviceB;
    public void methodA(){
        serviceB.methodB();
    }
}
```

ServiceA.methodA()定义为`REQUIRED`

- ServiceB.methodB()定义为`REQUIRED`

  MethodB与MethodA 共用一个事务。MethodA或MethodB中某一处报错，都执行回滚。

- ServiceB.MethodB()定义为`REQUIRS_NEW`

  MethodB 新起一个事务。MethodB提交 MethodA异常。B 不回滚，A回滚。 MethodB异常，MethodA捕获，A事务可能提交（B的异常A是否回滚）

- ServiceB.MethodB()定义为`NESTED`

  MethodB rollback

  MethodA 两种处理方式 捕获异常： 不捕获异常：

### Spring事务API

#### 核心事务管理器

`org.springframework.transaction.PlatformTransactionManager`

#### 事务状态

`org.springframework.transaction.TransactionStatus`

#### 事务属性

`org.springframework.transaction.TransactionDefinition`

- `org.springframework.transaction.interceptor.TransactionAttribute`



#### 事务执行类

`org.springframework.transaction.interceptor.TransactionInterceptor`

适配器模式 适配了`org.springframework.transaction.interceptor.TransactionAspectSupport`

和`org.aopalliance.intercept.MethodInterceptor`

#### 执行方法

```java
protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
			final InvocationCallback invocation) throws Throwable {

		// 获取事务元信息
		TransactionAttributeSource tas = getTransactionAttributeSource();
        // 解析出事务属性
		final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
    	// 确定事务管理器
		final PlatformTransactionManager tm = determineTransactionManager(txAttr);
		final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

		if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
			// 组装（包装为一个类） 并绑定到当前线程的ThreadLocal中
			TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
			Object retVal = null;
			try {
				// This is an around advice: Invoke the next interceptor in the chain.
				// This will normally result in a target object being invoked.
				retVal = invocation.proceedWithInvocation();
			}
			catch (Throwable ex) {
				// target invocation exception
				completeTransactionAfterThrowing(txInfo, ex);
				throw ex;
			}
			finally {
				cleanupTransactionInfo(txInfo);
			}
			commitTransactionAfterReturning(txInfo);
			return retVal;
		}
    ....
}
```



`org.springframework.transaction.interceptor.TransactionAttributeSource ` 策略模式

- `org.springframework.transaction.annotation.AnnotationTransactionAttributeSource`

事务属性元信息 

`org.springframework.transaction.interceptor.TransactionAspectSupport.TransactionInfo`

把旧事务存起来，新事物执行完后再把旧事务设回

```java
private void bindToThread() {
    // Expose current TransactionStatus, preserving any existing TransactionStatus
    // for restoration after this transaction is complete.
    this.oldTransactionInfo = transactionInfoHolder.get();
    transactionInfoHolder.set(this);
}

private void restoreThreadLocalStatus() {
    // Use stack to restore old transaction TransactionInfo.
    // Will be null if none was set.
    transactionInfoHolder.set(this.oldTransactionInfo);
}
```

