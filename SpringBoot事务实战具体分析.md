首先复习异常：

可检查的异常：Exception下除了RuntimeException以外的所有异常。

不可检查异常：RuntimeException异常及其子类和错误（ERROR）

Spring框架基础事务框架默认只在抛出运行时和不可检查异常时才标识事务回滚，也就是说如果抛出个RuntimeException异常或者及其子类得异常时，Spring默认标识事务回滚。从事务方法中抛出可检查异常时，Spring不会标识为事务回滚。

让可检查异常也回滚：

​	在整个方法前加上@Transactional(rollbackFor=Exception.class)

让不可检查异常不回滚：

​	在整个方法前加上@Transactional(notRollbackFor=RunTimeException.class)

先说几点简单，容易理解得。

1. 数据库存储引擎必须为InnoDB，因为InnoDB支持事务，而myIsam不支持事务。值得注意的是，SpringBoot2.0之后使用JPA新建得数据库默认得为myIsam

2. 注解需要写在public声明的方法上，private、protected方法皆不会生效，也不会报错。最好是写在类上面，或者类声明的方法，不建议写在接口上面。

   解释：SpringAOP实现方式有两种

   - 基于Java本身的JDK动态代理。优点：不依赖第三方Jar包。缺点：只能代理接口，不能代理类
   - 基于CGLIB的动态代理。