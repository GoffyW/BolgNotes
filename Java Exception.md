## Java异常详解

## 1、体系

Java所有的异常都派生于Throwable这个类

1. Throwable

   1. Error

      Error类是程序无法处理的错误，通常情况下与程序员的编写无关，通常与JVM（Java虚拟机）有关。如系统奔溃，虚拟机错误，动态链接失败

      这里多说一点，主要是关于OutOfMemoryError（内存溢出），这个错误产生的原因

      - 内存中加载的数据量过于庞大，如一次性从数据库中取出过多数据
      - 集合类对象使用完未清空，使得JVM无法回收
      - 代码中存在死循坏或者产生过多的对象实体
      - 使用第三方的项目的BUG
      - 启动参数内存值设定的过小

   2. Exception

      1. 运行时异常（不可检查异常）
         1. 都是RuntimeException类及其子类，如NullPointerException(空指针异常)、IndexOutOfBoundsException(下标越界异常)等，这些异常是不可检查异常，开发人员应该从逻辑角度避免这类异常的发生。它的特点是Java编译器不会检查它，也就是说即使没有try-catch捕获，或者Throws子句抛出，也会顺利编译通过。
      2. 非运行时异常（编译异常，可检查异常）
         1. 可检查异常体现了Java的设计哲学--没有完善错误处理的代码根本就不会执行！
         2. 从语法角度来说，编译异常即程序可以自动监测到，如果不处理，则编译无法通过。如IOException、SQLException等以及用户自定义的Exception异常。

## 2、机制

在Java语言中，对于受检查异常处理机制为抛出异常和捕获异常

1. 抛出异常：

   1. throws声明异常

      1. 使用Throws声明异常的思路为：当前这个方法不知道如何处理这种类型的异常，该异常应该由上一级调用者处理，如果Main方法也不知道如何处理，也可以使用throws声明抛出异常，该异常交给JVM管理，此时JVM会打印此异常的跟踪栈信息，并中止程序运行，这也就是为什么程序遇到异常后自动终止的原因。
      2. Throws声明抛出异常只能在方法签名中使用，可以抛出多个，之间用逗号分隔。
      3. 如果用throws声明抛出异常有两条限制：
         1. 子类方法声明抛出的异常类型应该是父类方法声明抛出的异常的子类或者相同。
         2. 子类方法声明抛出的异常不允许比父类方法声明抛出的异常多

   2. throw抛出异常

      1. throw抛出的不是异常类，而是一个异常实例，且每次只能抛出一个

      2. 如果throw抛出的是受检查异常，则该异常要么处于try块里面，要么处于一个被throws声明抛出的方法里面；如果throw抛出的是运行时异常，则无需上述处理。

      3. ```java
             public static void main(String[] args) {
                 try{
                     //此时在这里要么try显示抛出，要么main方法再次声明抛出
                     throwChecked(-3);
                 }catch (Exception e){
                     System.out.println(e.getMessage());
                 }
                 // 即可以显示抛出，也可无需理会
                 throeRuntime(-3);
             }
         
             public static void throwChecked(int a) throws Exception {
                 if(a>0){
                     throw new Exception("a的值大于0，不符合要求");
                 }
             }
         
             public static void throeRuntime(int a ){
                 if(a>0){
                     throw  new RuntimeException("a的值大于0，不符合要求");
                 }
             }
         ```

## 3、处理规则

大概率实现以下目标：

1. 使程序代码混乱最小化
2. 捕获并保留诊断信息
3. 通知合适的人员
4. 采用合适的方法结束异常活动

大概率遵循以下基本原则：

1. 不要过度使用异常
2. 不要使用过于庞大的try块
3. 避免使用Catch All 语句
4. 不要忽略捕获到的异常