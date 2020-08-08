### 1、事务ACID

1. ##### 原子性：一个事务要么全部执行，要么不执行

2. ##### 一致性：事务的运行并不改变数据库的一致性，例如检查约束，非空约束，主键约束，外键约束

3. ##### 隔离性：两个以上的事务不会出现交错执行的状态

4. ##### 持久性：事务执行成功后，该事务对数据库所做的更改便持久的保存在数据库中，不会无缘无故的回滚，重启电脑也不会

### 2、Spring的事务

根据底层所使用的不同的持久化API或框架，使用如下：（可以简单的理解为Spring集成了这些数据持久化框架）

1. **DataSourceTransactionManagerAutoConfiguration**：适用于使用JDBC和MyBatis进行持久化操作的情况，在定义时需要需要提供底层的数据源作为其属性，即**DataSource**
2. **HibernateTransactionManager**：适用于使用Hibernate进行数据持久化操作的情况，与 HibernateTransactionManager 对应的是 **SessionFactory**
3. **JpaTransactionManager**：适用于使用JPA进行数据持久化操作的情况，与 JpaTransactionManager 对应的是 **EntityManagerFactory**

### 3、主要接口

#### 3.1、PlatformTransactionManager

Spring中事务的主要接口为org.springframework.transaction.PlatformTransactionManager，通过这个接口，Spring为各个平台如JDBC，Hibernate等都提供了对应的事务管理器，但是具体的实现则是各个平台自己的事。主要包含三个主要的方法：

1. ```Java
   TransactionStatus getTransaction(@Nullable TransactionDefinition definition) throws TransactionException; //获取事务的状态
   ```

2. ```java
   void commit(TransactionStatus status) throws TransactionException;
   //提交事务
   ```

3. ```java
   void rollback(TransactionStatus status) throws TransactionException;
   //回滚事务
   ```

   ![image-20200330213537798](C:\Users\GoffyGUO\AppData\Roaming\Typora\typora-user-images\image-20200330213537798.png)



#### 3.2、TransactionStatus

PlatformTransactionManager.getTransaction(…) 方法返回一个TransactionStatus对象，这个对象代表一个新的或者已经存在的事务（如果在当前退栈有一个符合条件的事务），TransactionStatus提供了一个控制事务执行和查询事务状态的方法：

```java
public interface TransactionStatus extends SavepointManager, Flushable {
	boolean isNewTransaction();//是否为新的事务
	boolean hasSavepoint();//是否有恢复点
	void setRollbackOnly();//设置为只回滚
	boolean isRollbackOnly();//是否为只回滚
	@Override
	void flush();
	boolean isCompleted();//是否以完成
```

#### 3.3、TransactionDefinition

org.springframework.transaction.TransactionDefinition，它用于定义一个事务，包含了事务的静态属性，比如：传播行为，隔离级别，超时行为

```java
int getPropagationBehavior();//返回事务的传播行为
int getIsolationLevel();//返回事务的隔离级别
int getTimeout();//返回事务必须在多少秒以内完成
boolean isReadOnly();//事务是否为只读，事务管理器能够根据这个返回值进行优化，确保事务是只读的
```

### 4、原理分析

Spring事务的本质是数据库事务，没有数据库的支持，Spring是无法提供事务的，对于纯JDBC来说，可以按照一下步骤完成事务：

1. 获取连接 Connection con = DriverManager.getConnection()
2. 开启事务con.setAutoCommit(true/false);
3. 执行CRUD
4. 提交事务/回滚事务 con.commit() / con.rollback();
5. 关闭连接 conn.close();

Spring事务帮我们简化了步骤2和4。

spring事务具体分为编程式事务和声明式事务，编程式事务其实就是上面说的5个步骤，使用TransactionTemplate手动管理事务，相当于硬编码。现在基本不用。而声明式事务是通过AOP实现的，又分为基于tx和aop的XML配置开发和基于注解开发。下面详细解说这两个

#### 4.1、基于XML配置





