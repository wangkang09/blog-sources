[TOC]

- Spring 在 TransactionDefinition 接口中规定了 7 种类型的事务传播行为
- 事务传播行为是Spring框架独有的事务增强特性，不属于数据库行为
- 通过这 7 种 传播行为，可以使我们更灵活的控制哪些行为在同一个事务内、哪些行为不在同一事务内、各事务之间的关系，从而使得我们的业务逻辑更加灵活



## 1 Spring中七种事务传播行为

- 事务传播行为规定了在一个嵌套方法内部，哪些方法属于同一个事务，哪些方法不属于同一个事务，以及事务之间的关系

| 事务传播行为类型          | 说明                                                       |
| ------------------------- | ---------------------------------------------------------- |
| PROPAGATION_REQUIRED      | 如果存在则加入事务。如果不存在，创建新事务                 |
| PROPAGATION_SUPPORTS      | 如果存在则加入事务。如果不存在，以不可用事务执行           |
| PROPAGATION_MANDATORY     | 如果存在则加入事务。如果不存在，抛出异常                   |
| PROPAGATION_REQUIRES_NEW  | 如果存在则挂起当前事务，并创建新事务。如果不存在则创建事务 |
| PROPAGATION_NESTED        | 如果存在则创建嵌套事务。如果不存在，创建新事务             |
| PROPAGATION_NOT_SUPPORTED | 以不可用事务运行。如果存在事务，则先挂起当前事务           |
| PROPAGATION_NEVER         | 以不可用事务运行。如果存在事务，抛出异常                   |



## 2 定义

- null：表示没有 @Transactional 注解的方法，且不在事务内
- NULL：表示没有 @Transactional 注解的方法，但**在事务内**（其外部有事务，所以加入外部事务）
- required：表示有 @Transactional(propagation = Propagation.REQUIRED) 的方法，但是外部没有事务
- REQUIRED：表示有 @Transactional(propagation = Propagation.REQUIRED) 的方法，但是外部有事务
- 同理其它 6 个传播行为
- 数据库错误：表示有 @Transactional 注解的方法，**执行数据库操作时失败**，但是异常最终被 catch 掉了
- 异常：方法抛出来的、没有被捕获的异常
- 方法被回滚：表示方法内部的数据库操作被回滚
- 事务切面方法：表示加入事务，且加入事务切面（被 @Transactional 注解注释的方法才会加入事务切面）
- 事务方法：表示加入事务的方法（包含事务切面方法）
- 外层事务：表示先加入同一事务的方法
- 真实事务：表示有 @Transactional 注解的方法
- 普通方法：没有 @Transactional 注解的方法



## 3 定理

- null 方法不受事务影响，只要执行成功就不会被回滚（例:插入了一行数据后，执行其它语句立马报错也不会被回滚，因为**null 方法插入后会立马提交**，其 auto-commit 为 true）
- 事务内发生 **异常**，加入这个事务的所有数据库操作都会被回滚



## 4 推理

1. NULL 方法受事务影响，外层事务被回滚，NULL 方法也会被回滚（...方法证明）

2. 如果 NULL 方法发生 *数据库错误*，不影响事务（即是 3）

3. required 内部的 NULL 方法发生 *数据库错误*，required 方法不会被回滚（虽然在同一事务内，但是 NULL 方法的数据库异常不会传递到事务切面，因为 NULL 方法虽然加入了事务，但是没有加入事务切面）

4. required 内部的 REQUIRED 方法发生 *数据库错误*，required 方法被回滚（通过事务切面将回滚状态传递到外层）

5. required 内部的 SUPPORTS 方法发生 *数据库错误*，required 方法被回滚（同理）

6. required 内部的 MANDATORY 方法发生 *数据库错误*，required 方法被回滚（同理）

7. required 内部的 REQUIRES_NEW 方法发生 *数据库错误*，required 方法不会被回滚（不在同一个事务）

8. required 内部的 NESTED 方法发生 *数据库错误*，required 方法不会被回滚（NESTED 发生数据库错误会回滚到保存点，保证了外部事务的有效性）

9. required 内部的 NOT_SUPPORTED 方法发生 *数据库错误*，required 方法不会被回滚（NOT_SUPPORTED 方法 不在事务内）

10. requires_new、nested 情况和 required 相同，因为这两个都只是开启新事务，意思和 required 一模一样

11. supports、not_supported、never 都是以非事务运行和 null 效果一样（null之内的普通方法不能复用session）

12. mandatory 直接抛异常

13. NESTED：--------0------0---------，两个0 之间是嵌套事务，不管是内层嵌套事务还是外层事务，其实是一个事务，只是在这个事务线上创建了一个保存点，嵌套事务内部事情处理完了之后，整个事务状态会回到保存点时的状态，这样保证了即使嵌套内部发生数据库错误(这时仍然会设置事务为 rollback 状态)，因为方法完了之后事务状态会回到保存点，所以 rollback 状态被覆盖了

14. NESTED 和 NULL 的区别
    - NULL 内部的 *事务方法* 属于 NULL 所在事务，而 NESTED 内部的 *事务方法* 属于 NESTED 开启的嵌套事务和外层事务没关系
    2. NULL 内部的  *事务切面方法*  发生  *数据库错误*  会导致外部事务回滚，而 NESTED 内部的*事务切面方法*  发生 *数据库错误* 只会导致 NESTED 开启的嵌套事务回滚，而外层事务不会

15. NESTED 和 REQUIRES_NEW 的区别

    - 外层事务发生 *数据库错误* 会导致 NESTED 方法回滚，而不会导致 REQUIRES_NEW 方法回滚



## 5 证明

### 5.1 事务传播行为证明

- [事务传播行为证明](https://github.com/wangkang09/spring-boot-transaction-study/blob/master/src/test/java/com/wangkang/PropagationTest.java)

### 5.2 推理证明

- [推理证明](https://github.com/wangkang09/spring-boot-transaction-study/blob/master/src/test/java/com/wangkang/ReasoningTest.java)



## 6 事务切面设置的几个核心状态

### 6.1 newTransaction

- **是否是新开启的事务**

- 此属性是直接赋值的，没有判断逻辑
- 所有的小写/REQUIRES_NEW 为 true
- 所有的大写(除了 REQUIRES_NEW) 为 false
- 事务关闭时如果 newTransaction= true 且 actualTransactionActive=true，才会做 connection 的 commit 操作

### 6.2 actualNewSynchronization

- **当前线程是否是新开启同步的**

- 如果当前线程已经开启同步了，返回 false。否则，返回 true

```java
AbstractPlatformTransactionManager#newTransactionStatus
actualNewSynchronization = !TransactionSynchronizationManager.isSynchronizationActive()
```

### 6.3 isSynchronizationActive/actualTransactionActive

- **当前线程是否同步/当前线程的事务是否可用**

- 如果 actualNewSynchronization 为 false 的话
  -  如果 status 中的 transaction 为 null，actualTransactionActive=false。否则为 true
  - isSynchronizationActive = true

```java
protected void prepareSynchronization(DefaultTransactionStatus status, TransactionDefinition definition) {
    if (status.isNewSynchronization()) {
        //关键 1：设置当前线程事务是否活跃
        setActualTransactionActive(status.hasTransaction());
        //关键 2： 设置事务是否已经同步
        //synchronizations.set(new LinkedHashSet<>());
        initSynchronization();
    }
}
```

### 6.4 connectionHolder

- 此属性只有在创建**新的、可用的**事务时，才会设置
- transaction.newConnectionHolder = true
- conHolder.setSynchronizedWithTransaction(true)
- conHolder.setTransactionActive(true)
- connection.setAutoCommit(false)

```java
org.springframework.jdbc.datasource.DataSourceTransactionManager#doBegin
在 doBegin 方法中做了以下操作
	1. 创建 connectionHolder
    2. 将此 connectionHolder 设置到当前事务中，并设置 newConnectionHolder = true
    3. 设置 ConnectionHolder().setSynchronizedWithTransaction(true)  ----> 1
    4. 将此 connectionHolder 中的 connection 设置 auto-commit = false -----> 2 
    5. 设置 事务的 read only 状态（如果指定了 read only 的话）
    6. getConnectionHolder().setTransactionActive(true)             -----> 3 
```

### 6.5 currentTransactionStatus

- 设置当前线程的事务状态

```java
TransactionAspectSupport#prepareTransactionInfo#bindToThread
//获取当前事务的状态：TransactionStatus[transaction,newTransaction,newSynchronization,suspendedResources,readOnly]
//可以看出 之前很多状态信息 都在 TransactionStatus 中保存了
//这个线程中保存了 旧的 TransactionStatus 和当前的 TransactionStatus，旧的用于事务挂起等功能
private void bindToThread() {
    this.oldTransactionInfo = transactionInfoHolder.get();
    transactionInfoHolder.set(this);
}
```



## 7 required/requires_new/nested  浅析

- 此三个状态为 **创建新事务**

- 事务切面后的核心状态信息
  - newTransaction=true（是新事务）
  - actualNewSynchronization=true（新开启的）
  - isSynchronizationActive=true（新开启同步）
  - actualTransactionActive = true（事务可用）
  - 创建并设置 connectionHolder（设置特定的 connectionHolder）

```java
//AbstractPlatformTransactionManager#getTransaction
suspendedResources = suspend(null);//内部不进行任何操作，返回 null
try {
   boolean newSynchronization = 始终为 true;
   //status 的关键属性， newTransaction=true,actualNewSynchronization=true
   //说明是新事务，且新开启事务同步(后面加入事务的就是 false) 
   DefaultTransactionStatus status = newTransactionStatus();
   //创建新的connection，并设置其状态为和事务同步且代表一个活动的事务，并加入到线程中
   //设置事务 read only 标识，如果需要的话 
   doBegin(transaction, definition);
   //设置当前线程的事务状态：hasTransaction=true, isSynchronizationActive()=true
   //这样在创建 session 后，就会将 session 加入当前线程，重复使用(registerSessionHolder()) 
   prepareSynchronization(status, definition);
   return status;
}
catch (RuntimeException | Error ex) {
   //这里的 suspendedResources == null，内部不会做任何操作  
   resume(null, suspendedResources);
   throw ex;
}
```

### 7.1 doBegin

```java
protected void doBegin(Object transaction, TransactionDefinition definition) {
   try { 
      if (!txObject.hasConnectionHolder() ||
            txObject.getConnectionHolder().isSynchronizedWithTransaction()) {
         //因为事务中还没有 connection 所以进入此处获取 con（这里就可以动态切换数据源了） 
         Connection newCon = obtainDataSource().getConnection();
         //将此 connection 设置到事务中，并标记为 新的连接 
         txObject.setConnectionHolder(new ConnectionHolder(newCon), true);
      }
      //标记此 connection 为和事务同步状态
      txObject.getConnectionHolder().setSynchronizedWithTransaction(true);
      con = txObject.getConnectionHolder().getConnection();
      Integer previousIsolationLevel = 事务隔离级别;
      //Switch to manual commit 
	  //设置事务 read only 状态(如果事务标记了 read only，通过注解中的属性标记)
      prepareTransactionalConnection(con, definition);
      //标记此 connection 代表一个活动的事务
      txObject.getConnectionHolder().setTransactionActive(true);
      if (txObject.isNewConnectionHolder()) {
         //可以走到这，将上面创建的 connection 设置到当前线程中，这样统一事务就复用 con 了
         TransactionSynchronizationManager.bindResource(obtainDataSource(), txObject.getConnectionHolder());
      }
   }
   catch (Throwable ex) {
      if (txObject.isNewConnectionHolder()) {
         //发生异常则关闭连接 
         DataSourceUtils.releaseConnection(con, obtainDataSource());
         txObject.setConnectionHolder(null, false);
      }
      throw new CannotCreateTransactionException("Could not open JDBC Connection for transaction", ex);
   }
}
```

### 7.2 prepareSynchronization

```java
protected void prepareSynchronization(DefaultTransactionStatus status, TransactionDefinition definition) {
    if (status.isNewSynchronization()) {//肯定为 true   
        setActualTransactionActive(status.hasTransaction());//肯定为 true
        setCurrentTransactionIsolationLevel(..);
        setCurrentTransactionReadOnly(..);
        setCurrentTransactionName(..);
        //synchronizations.set(new LinkedHashSet<>());
        initSynchronization();
    }
}
```



## 8 support/not_support/never  浅析

- 此三个状态为以 **不可用事务** 执行状态

- 事务切面后的核心状态信息
  - newTransaction=true（是新事务）
  - actualNewSynchronization=true（新开启的）
  - isSynchronizationActive=true（新开启同步）
  - actualTransactionActive = false（事务不可用）
  - 没有创建 conHolder

- 这 3 个方法表示以 非事务执行

  - 这样虽然执行时创建了 session和connection 并加入了线程，当内部 *事务方法* 执行时也会重新创建事务，并覆盖 session/connection 信息
  - **技巧：** 当此3种方法内部有 *普通方法*  时，普通方法会复用 session/connection，减轻了创建 session/connection 的消耗，特别是在高并发的时候
  - 并且没有执行 doBegin() 方法创建并**设置 connection 的 auto-commit 为 false!**，使得数据库操作一执行完就立刻提交了

  ```java
  @Override
  protected boolean isExistingTransaction(Object transaction) {
     DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
     //这里返回 false,因为  isTransactionActive() = false
     //返回 false，就导致新来的 真实事务 方法会走没有事务存在的逻辑，这样就覆盖了非事务绑定的 session 和 connection 信息了 
     return (txObject.hasConnectionHolder() && txObject.getConnectionHolder().isTransactionActive());
  }
  ```

- 和 6 的区别仅于 
  - 6 通过 doBegin() 方法，创建 connection 相关信息
  - 7 中的事务状态 hasTransaction=false,actualTransactionActive=false

```java
//AbstractPlatformTransactionManager#getTransaction
boolean newSynchronization = 这个目前始终为 true;
//设置当前线程的事务状态：hasTransaction=true, isSynchronizationActive()=true
//ActualTransactionActive = false
//这样在创建 session 后，就会将 session 加入当前线程，重复使用(registerSessionHolder()) 
return prepareTransactionStatus(definition, null, true, newSynchronization, debugEnabled, null);

protected final DefaultTransactionStatus prepareTransactionStatus(
    TransactionDefinition definition, @Nullable Object transaction, boolean newTransaction,
    boolean newSynchronization, boolean debug, @Nullable Object suspendedResources) {
    //transaction 为 null，hasTransaction=false, isSynchronizationActive()=true
    DefaultTransactionStatus status = newTransactionStatus(
        definition, transaction, newTransaction, newSynchronization, debug, suspendedResources);
    prepareSynchronization(status, definition);
    return status;
}
```



## 9 REQUIRED/SUPPORTS/ MANDATORY 浅析

- 此三个状态为 **加入已存在事务**

- 事务切面后的核心状态信息
  - newTransaction=false（不是新事务）
  - actualNewSynchronization=false（不是新同步的）
  - 没有设置 isSynchronizationActive（沿用创建事务时的状态，一般 true）
  - 没有设置 actualTransactionActive（沿用创建事务时的状态，一般 true）
  - 没有创建 conHolder（沿用创建事务时创建 conHolder)

```java
//AbstractPlatformTransactionManager#handleExistingTransaction
boolean newSynchronization = 始终是 true；
return prepareTransactionStatus(definition, transaction, false, newSynchronization, debugEnabled, null);

protected final DefaultTransactionStatus prepareTransactionStatus(
    TransactionDefinition definition, @Nullable Object transaction, boolean newTransaction,
    boolean newSynchronization, boolean debug, @Nullable Object suspendedResources) {
    //hasTransaction=true，newTransaction=false,newSynchronization=false
    //以上值说明，当前的事务不是新开启的事务，也不是新同步的事务
    DefaultTransactionStatus status = newTransactionStatus(
        definition, transaction, newTransaction, newSynchronization, debug, suspendedResources);
    //因为 newSynchronization=false，此方法内部不做任何处理
    prepareSynchronization(status, definition);
    return status;
}
```



## 10 NESTED 浅析

- 此状态为 **创建嵌套事务**。和 9 的唯一区别就是创建了一个 savepoint。在嵌套事务内发生的 *数据库错误*，会回滚到这个保存点，所以并不会影响外部事务。**外部事务仍可影响内部事务**

- **事务切面后的核心状态信息**
  - newTransaction=false（不是新事务）
  - actualNewSynchronization=false（不是新同步的）
  - 没有设置 isSynchronizationActive（沿用创建事务时的状态，一般 true）
  - 没有设置 actualTransactionActive（沿用创建事务时的状态，一般 true）
  - 没有创建 conHolder（沿用创建事务时创建 conHolder)

```java
DefaultTransactionStatus status =
    prepareTransactionStatus(definition, transaction, false, false, debugEnabled, null);
//和 9 的唯一区别就是场景 savepoint
status.createAndHoldSavepoint();
```



## 11 REQUIRES_NEW 浅析

- 挂起之前的事务，**重新创建新事务**。完全可以当作 7 来看，只是挂起了之前的事务

- 事务切面后的核心状态信息
  - newTransaction=true（是新事务）
  - actualNewSynchronization=true（新开启的）
  - isSynchronizationActive=true（新开启同步）
  - actualTransactionActive = true（事务可用）
  - 创建并设置 connectionHolder（设置特定的 connectionHolder）

```java
//挂起之前的事务(把之前的事务信息保存到当前事务，当前事务完成后，在设置回去)
SuspendedResourcesHolder suspendedResources = suspend(transaction);
try {
    boolean newSynchronization = true;
    DefaultTransactionStatus status = newTransactionStatus(
        definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
    doBegin(transaction, definition);
    prepareSynchronization(status, definition);
    return status;
}
catch (RuntimeException | Error beginEx) {
    //发生异常也重置挂起的事务
    resumeAfterBeginException(transaction, suspendedResources, beginEx);
    throw beginEx;
}
```



## 12 NOT_SUPPORTED 浅析

- 挂起之前事务，以不可用事务执行。和 8 的唯一区别是保存了之前事务。该事务执行完之后会重新设置之前的事务
- 事务切面后的核心状态信息
  - newTransaction=true（是新事务）
  - actualNewSynchronization=true（新开启的）
  - isSynchronizationActive=true（新开启同步）
  - actualTransactionActive = false（事务不可用）
  - 没有创建 conHolder

```java
Object suspendedResources = suspend(transaction);
boolean newSynchronization = true;
return prepareTransactionStatus(
      definition, null, false, newSynchronization, debugEnabled, suspendedResources);
```



## 13 数据库操作执行时用到和设置的几个状态

### 13.1 获取 session

- session 只要有任何事务，都会加入到线程中，不可用事务也是

```java
//org.mybatis.spring.SqlSessionUtils#getSqlSession#sessionHolder
//如果当前线程中的 sessionHolder 不为 null 且 此 session 和 事务处于同步状态，就从事务中得到 session，否则返回 null   
//目前只要有 holder 则 isSynchronizedWithTransaction 肯定为 true
if (holder != null && holder.isSynchronizedWithTransaction()) {}

//org.mybatis.spring.SqlSessionUtils#getSqlSession#registerSessionHolder
//如果当前线程事务处于同步状态，则将创建的 session 绑定到当前线程中
//只要方法或方法外层有 @Transactional 任意类型注解，此处都为 true
if (TransactionSynchronizationManager.isSynchronizationActive()) {}
```

### 13.2 获取 connection

- 在此处创建的 connection 都不具备手动 commit/rollback 能力
- 只有在 doBegin() 方法中创建并加入事务的才正真具备事务能力

```java
//org.springframework.jdbc.datasource.DataSourceUtils#doGetConnection
//如果当前线程存在 conHolder 且 conHolder 中存在 connection 或者 conholder 已经和事务同步，则直接返回 conHolder 中的 connection
//只要 conHolder 不为 null，后面一般都为 true
if (conHolder != null && (conHolder.hasConnection() || 	conHolder.isSynchronizedWithTransaction())) {
}
//org.springframework.jdbc.datasource.DataSourceUtils#doGetConnection
//只要方法或方法外层有 @Transactional 任意类型注解，此处一般都为 true
if (TransactionSynchronizationManager.isSynchronizationActive()) {}
```



## 14 connection auto-commit 属性介绍

- 总的来说如果 auto-commit 为 true，则数据库操作全部完成时就自动提交	

```java
将此连接的自动提交模式设置为给定状态。
如果连接处于自动提交模式，则其所有SQL语句都将作为单个事务执行和提交。否则，只会在事务完成或失败后统一进行 commit 或 rollback。
默认情况下，新连接处于自动提交模式。自动提交发生在 statement 完成后。
statement 完成的时间取决于SQL语句的类型：
	1. 对于DML语句，如Insert、Update或Delete和DDL语句，该语句在完成执行后即完成。
	2. 对于Select语句，当关联的结果集关闭时，该语句即完成。
	3. 对于CallableStatement对象或返回多个结果的语句，当所有关联的结果集都已关闭，并且已检索到所有更新计数和输出参数时，该语句即完成。
- 总的来说就是数据库操作全部完成时就自动提交	

注意：如果在事务期间调用此方法，并且更改了自动提交模式，则将提交事务。如果调用setAutoCommit并且自动提交模式没有更改，则调用是no-op。
void setAutoCommit(boolean autoCommit) throws SQLException;
```



## 15 善用 PROPAGATION_SUPPORTS  

- 如果存在则加入事务。如果不存在，以非事务执行
- 此 传播类型其实和没有 @Transactional 标记的作用基本一致
  - 如果外部有事务，则 这两种情况都加入事务
  - 如果外部没有事务，则这两种情况都会自动提交
- **区别：** SUPPORTS 方法在外围没有事务时，也会创建 session/connection 并加入到线程，这样在此方法内的其它 *普通方法* 都会重用线程中的 session/connection，避免了频繁创建！



## 参考

spring-boot-starter:2.1.13.RELEASE

mybatis-spring-boot-starter:2.1.1