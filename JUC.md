## JUC

### volatile

volatile时Java提供的轻量级同步机制

保证可见性、不保证原子性、禁止指令重排

- 保证可见性

  当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看到修改的值

- 不保证原子性

  原子性：不可分割，完整性，即某个线程正在做某个具体的业务时，中间不可以被加塞或者分割，需要整体完整，要么同时成功，要么同时失败

- 禁止指令重排

  有序性：在计算机执行程序时，为了提高性能，编译器和处理器常常会对**指令做重排**

  举例：

  声明变量：int a,b,x,y=0;

  | 线程一 | 线程二  |
  | ------ | ------- |
  | x=a    | y=b     |
  | b=1    | a=2     |
  | 结果   | x=0,y=0 |

  如果编译器对这段程序执行指令重排优化后，可能出现一下情况：

  | 线程一 | 线程二  |
  | ------ | ------- |
  | b=1    | a=2     |
  | x=a    | y=b     |
  | 结果   | x=2,y=1 |

  这个结果说明在多线程下，由于编译器优化重排的存在，两个线程中使用的变量是无法保证一致性的

  > volatile实现禁止指令重排，从而避免了多线程环境下程序出现乱序执行的现象

重排序由以下几种机制引起：

1. 编译器优化：对于没有数据依赖关系的操作，编译器在编译的过程中会进行一定程度的重排。

   > 大家仔细看看线程 1 中的代码，编译器是可以将 a = 1 和 x = b 换一下顺序的，因为它们之间没有数据依赖关系，同理，线程 2 也一样，那就不难得到 x == y == 0 这种结果了。

2. 指令重排序：CPU 优化行为，也是会对不存在数据依赖关系的指令进行一定程度的重排。

   > 这个和编译器优化差不多，就算编译器不发生重排，CPU 也可以对指令进行重排，这个就不用多说了。

3. 内存系统重排序：内存系统没有重排序，但是由于有缓存的存在，使得程序整体上会表现出乱序的行为。

   > 假设不发生编译器重排和指令重排，线程 1 修改了 a 的值，但是修改以后，a 的值可能还没有写回到主存中，那么线程 2 得到 a == 0 就是很自然的事了。同理，线程 2 对于 b 的赋值操作也可能没有及时刷新到主存中。

### JMM模型

概念：JMM(Java Memory Model)本身是一种抽象的概念，并不真实存在，它描述的是一组规则或规范，通过这种规范定义了程序中各个变量（包括实例字段，静态字段和构成数组对象的元素）的访问方式

##### JMM同步的规定：

1. 线程解锁前，必须把共享变量的值刷新回主内存（物理内存）
2. 线程枷锁前，必须读取主内存（物理内存）的最新值到自己的工作内存
3. 加锁解锁时同一把锁

描述：

由于JVM运行程序的实体是线程，而每个线程创建时JVM都会为其创建一个工作内存（有的成为栈空间），工作内存是每个线程的私有数据区域，而java内存模型中规定所有变量都存储在**==主内存==**，主内存是贡献内存区域，所有线程都可以访问，**==但线程对变量的操作（读取赋值等）必须在工作内存中进行，首先概要将变量从主内存拷贝到自己的工作内存空间，然后对变量进行操作，操作完成后再将变量写回主内存，==**不能直接操作主内存中的变量，各个线程中的工作内存中存储着主内存的**==变量副本拷贝==**，因此不同的线程件无法访问对方的工作内存，线程间的通信（传值）必须通过主内存来完成

### volatile的使用场景

1. 单例模式：

   当多线程去访问单例模式时，构造方法在一些情况下会被访问多次

```java
public class SingletonDemo {
    private static SingletonDemo instance = null;

    private SingletonDemo() {
        System.out.println(Thread.currentThread().getName() + "\t 构造方法SingletonDemo（）");
    }

    public static SingletonDemo getInstance() {
        if (instance == null) {
            instance = new SingletonDemo();
        }
        return instance;
    }

    public static void main(String[] args) {
        //构造方法只会被执行一次
//        System.out.println(getInstance() == getInstance());
//        System.out.println(getInstance() == getInstance());
//        System.out.println(getInstance() == getInstance());

        //并发多线程后，构造方法会在一些情况下执行多次
        for (int i = 0; i < 10; i++) {
            new Thread(() -> {
                SingletonDemo.getInstance();
            }, "Thread " + i).start();
        }
    }
}
```

解决方法：

1. 单例模式DCL代码

   DCL（Double Check Lock）双端检锁机制，在加锁前后加锁后都进行不次判断

   ```java
   public static SingletonDemo getInstance() {
           if (instance == null) {
               synchronized (SingletonDemo.class) {
                   if (instance == null) {
                       instance = new SingletonDemo();
                   }
               }
           }
           return instance;
       }
   ```

   在这样的操作下，大部分情况，构造方法只会被访问一次，但由于指令重排的存在，其构造方法也是有可能会被访问多次的

### CAS

##### compareAndSet----比较并交换

> AtomicInteger.conpareAndSet(int expect, indt update)

```java
public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
```

第一个参数为拿到的期望值，如果期望值一致，进行Update操作。如果期望值不一致，证明数据被修改，返回false，取消赋值

```java
/**
 * @author GoffyGUO
 */
public class CasDemo {
    public static void main(String[] args) {
        checkCAS();
    }

    public static void checkCAS(){
        AtomicInteger atomicInteger = new AtomicInteger(5);
        System.out.println(atomicInteger.compareAndSet(5, 100)+"\t current data is "+atomicInteger.get());
        System.out.println(atomicInteger.compareAndSet(5, 200)+"\t current data is "+atomicInteger.get());
    }
}
```

```java
D:\JDK1.8\bin\java.exe "-javaagent:D:\IntelliJIDEA\IntelliJ IDEA 
true	 current data is 100
false	 current data is 100

Process finished with exit code 0
```

##### 底层原理

```java
import sun.misc.Unsafe;
public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }

private volatile int value;

/**
     * Creates a new AtomicInteger with the given initial value.
     *
     * @param initialValue the initial value
     */
public AtomicInteger(int initialValue) {
    value = initialValue;
}
```

Unsafe类

- CAS核心类，由于Java无法直接访问底层系统，通过通过本地（native）方法来访问，Unsafe相当于一个后门，基于此类可以直接操作特定内存数据，Unsafe类存在于sun.misc包中，其内部方法操作可以像C的指针一样直接操作内存，因为Java中CAS操作的执行依赖于Unsafe类的方法。**Unsafe类中的所有方法都是native修饰的，也就是说Unsafe类中的方法都直接调用操作系统底层资源执行相应任务**
- valueOffset：表示该变量值在内存中的偏移地址，因为Unsafe就是根据内存偏移地址来获取数据的
- 变量value用volatile修饰，保证多线程之间的可见性

```java
    public final int getAndAddInt(Object var1, long var2, int var4) {
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

        return var5;
    }
```

CAS全称呼Compare-And-Swap，它是一条CPU并发原语

他的功能是判断内存某个位置的值是否为预期值，如果是则更改为新的值，这个过程是原子的。

CAS并发原语体现在JAVA语言中就是sun.misc.Unsafe类中各个方法。调用Unsafe类中的CAS方法，JVM会帮我们实现CAS汇编指令。这是一种完全依赖于硬件的功能，通过他实现了原子操作。由于CAS是一种系统原语，原语属于操作系统用语范畴，是由若干条指令组成的，用于完成某个功能的一个过程，并且原语的执行必须是连续的，在执行过程中不允许被中断，也就是说CAS是一条CPU的原子指令，不会造成数据不一致问题

> var1 AtomicInteger对象本身
>
> var2 该对象的引用地址
>
> var4 需要变动的数据
>
> var5 通过var1 var2找出的主内存中真实的值

用该对象前的值与var5比较；

如果相同，更新var5+var4并且返回true，

如果不同，继续去之然后再比较，直到更新完成

##### CAS缺点

1. **循环时间长，开销大**

   例如getAndAddInt方法执行，有个do while循环，如果CAS失败，一直会进行尝试，如果CAS长时间不成功，可能会给CPU带来很大的开销

2. **只能保证一个共享变量的原子操作**

   对多个共享变量操作时，循环CAS就无法保证操作的原子性，这个时候就可以用锁来保证原子性

3. **ABA问题**

原子类AtomicInteger的ABA问题，原子更新引用

### 1、ABA如何产生

CAS算法实现一个重要前提需要去除内存中某个时刻的数据并在当下时刻比较并替换，那么在这个时间差类会导致数据的变化。

比如**线程1**从内存位置V取出A，**线程2**同时也从内存取出A，并且线程2进行一些操作将值改为B，然后线程2又将V位置数据改成A，这时候线程1进行CAS操作发现内存中的值依然时A，然后线程1操作成功。

> 尽管线程1的CAS操作成功，但是不代表这个过程没有问题

##### 如何解决？

```java
package juc.cas;

import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.ToString;

import java.util.concurrent.atomic.AtomicReference;

public class AtomicRefrenceDemo {
    public static void main(String[] args) {
        User z3 = new User("张三", 22);
        User l4 = new User("李四", 23);
        AtomicReference<User> atomicReference = new AtomicReference<>();
        atomicReference.set(z3);
        System.out.println(atomicReference.compareAndSet(z3, l4) + "\t" + atomicReference.get().toString());
        System.out.println(atomicReference.compareAndSet(z3, l4) + "\t" + atomicReference.get().toString());
    }
}

@Getter
@ToString
@AllArgsConstructor
class User {
    String userName;
    int age;
}

//==========输出==============
true	User(userName=李四, age=23)
false	User(userName=李四, age=23)
```

##### 时间戳（版本控制）

```java
package com.jian8.juc.cas;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicReference;
import java.util.concurrent.atomic.AtomicStampedReference;

/**
 * ABA问题解决
 * AtomicStampedReference
 */
public class ABADemo {
    static AtomicReference<Integer> atomicReference = new AtomicReference<>(100);
    static AtomicStampedReference<Integer> atomicStampedReference = new AtomicStampedReference<>(100, 1);

    public static void main(String[] args) {
        System.out.println("=====以下时ABA问题的产生=====");
        new Thread(() -> {
            atomicReference.compareAndSet(100, 101);
            atomicReference.compareAndSet(101, 100);
        }, "Thread 1").start();

        new Thread(() -> {
            try {
                //保证线程1完成一次ABA操作
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(atomicReference.compareAndSet(100, 2019) + "\t" + atomicReference.get());
        }, "Thread 2").start();
        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("=====以下时ABA问题的解决=====");

        new Thread(() -> {
            int stamp = atomicStampedReference.getStamp();
            System.out.println(Thread.currentThread().getName() + "\t第1次版本号" + stamp);
            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            atomicStampedReference.compareAndSet(100, 101, atomicStampedReference.getStamp(), atomicStampedReference.getStamp() + 1);
            System.out.println(Thread.currentThread().getName() + "\t第2次版本号" + atomicStampedReference.getStamp());
            atomicStampedReference.compareAndSet(101, 100, atomicStampedReference.getStamp(), atomicStampedReference.getStamp() + 1);
            System.out.println(Thread.currentThread().getName() + "\t第3次版本号" + atomicStampedReference.getStamp());
        }, "Thread 3").start();

        new Thread(() -> {
            int stamp = atomicStampedReference.getStamp();
            System.out.println(Thread.currentThread().getName() + "\t第1次版本号" + stamp);
            try {
                TimeUnit.SECONDS.sleep(4);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            boolean result = atomicStampedReference.compareAndSet(100, 2019, stamp, stamp + 1);

            System.out.println(Thread.currentThread().getName() + "\t修改是否成功" + result + "\t当前最新实际版本号：" + atomicStampedReference.getStamp());
            System.out.println(Thread.currentThread().getName() + "\t当前最新实际值：" + atomicStampedReference.getReference());
        }, "Thread 4").start();
    }
}
//===============输出====================
=====以下时ABA问题的产生=====
true	2019
=====以下时ABA问题的解决=====
Thread 3	第1次版本号1
Thread 4	第1次版本号1
Thread 3	第2次版本号2
Thread 3	第3次版本号3
Thread 4	修改是否成功false	当前最新实际版本号：3
Thread 4	当前最新实际值：100
```

### 公平锁、非公平锁，可重入锁（递归锁）自旋锁

1. ##### 公平锁，非公平锁

   1. 是什么：

      公平锁就是先来后到、非公平锁就是允许加塞，`Lock lock = new ReentrantLock(Boolean fair);` 默认非公平。

      - **公平锁**：是指多个线程按照申请锁的顺序来获取锁，类似排队打饭。
      - **非公平锁**：是指多个线程获取锁的顺序并不是按照申请锁的顺序，有可能后申请的线程优先获取锁，在高并发的情况下，有可能会造成优先级反转或者节现象

   2. 两者区别：

      - **公平锁**：Threads acquire a fair lock in the order in which they requested it

        公平锁，就是很公平，在并发环境中，每个线程在获取锁时，会先查看此锁维护的等待队列，如果为空，或者当前线程就是等待队列的第一个，就占有锁，否则就会加入到等待队列中，以后会按照FIFO的规则从队列中取到自己。

      - **非公平锁**：a nonfair lock permits barging: threads requesting a lock can jump ahead of the queue of waiting threads if the lock happens to be available when it is requested.

        非公平锁比较粗鲁，上来就直接尝试占有额，如果尝试失败，就再采用类似公平锁那种方式。

   3. 其他：

      对Java ReentrantLock而言，通过构造函数指定该锁是否公平，默认是非公平锁，非公平锁的优点在于吞吐量比公平锁大

      对Synchronized而言，是一种非公平锁

2. ##### 可重入锁（递归锁）

   1. **递归锁是什么**

      指的时同一线程外层函数获得锁之后，内层递归函数仍然能获取该锁的代码，在同一个线程在外层方法获取锁的时候，在进入内层方法会自动获取锁，也就是说，==线程可以进入任何一个它已经拥有的锁所同步着的代码块==

   2. **ReentrantLock/Synchronized 就是一个典型的可重入锁**

   3. **可重入锁最大的作用是避免死锁**

   CASE

   ```java
   public static void main(String[] args) {
           Phone phone = new Phone();
           new Thread(() -> {
               try {
                   phone.sendSMS();
               } catch (Exception e) {
                   e.printStackTrace();
               }
           }, "Thread 1").start();
           new Thread(() -> {
               try {
                   phone.sendSMS();
               } catch (Exception e) {
                   e.printStackTrace();
               }
           }, "Thread 2").start();
       }
   }
   class Phone{
       public synchronized void sendSMS()throws Exception{
           System.out.println(Thread.currentThread().getName()+"\t -----invoked sendSMS()");
           Thread.sleep(3000);
           sendEmail();
       }
   
       public synchronized void sendEmail() throws Exception{
           System.out.println(Thread.currentThread().getName()+"\t +++++invoked sendEmail()");
       }
   }
   ==========================================================================================
       public class ReentrantLockDemo {
       public static void main(String[] args) {
           Mobile mobile = new Mobile();
           new Thread(mobile).start();
           new Thread(mobile).start();
       }
   }
   class Mobile implements Runnable{
       Lock lock = new ReentrantLock();
       @Override
       public void run() {
           get();
       }
   
       public void get() {
           lock.lock();
           try {
               System.out.println(Thread.currentThread().getName()+"\t invoked get()");
               set();
           }finally {
               lock.unlock();
           }
       }
       public void set(){
           lock.lock();
           try{
               System.out.println(Thread.currentThread().getName()+"\t invoked set()");
           }finally {
               lock.unlock();
           }
       }
   }
   ```

3. ##### 独占锁（写锁）/共享锁（读锁）/互斥锁

   1、**概念**

   - **独占锁**：指该锁一次只能被一个线程所持有，对ReentrantLock和Synchronized而言都是独占锁

   - **共享锁**：只该锁可被多个线程所持有

     **ReentrantReadWriteLock**其读锁是共享锁，写锁是独占锁

   - **互斥锁**：读锁的共享锁可以保证并发读是非常高效的，读写、写读、写写的过程是互斥的

   CASE:

   ```java
   /**
    * 多个线程同时读一个资源类没有任何问题，所以为了满足并发量，读取共享资源应该可以同时进行。
    * 但是
    * 如果有一个线程象取写共享资源来，就不应该自由其他线程可以对资源进行读或写
    * 总结
    * 读读能共存
    * 读写不能共存
    * 写写不能共存
    */
   public class ReadWriteLockDemo {
       public static void main(String[] args) {
           MyCache myCache = new MyCache();
           for (int i = 1; i <= 5; i++) {
               final int tempInt = i;
               new Thread(() -> {
                   myCache.put(tempInt + "", tempInt + "");
               }, "Thread " + i).start();
           }
           for (int i = 1; i <= 5; i++) {
               final int tempInt = i;
               new Thread(() -> {
                   myCache.get(tempInt + "");
               }, "Thread " + i).start();
           }
       }
   }
   
   class MyCache {
       private volatile Map<String, Object> map = new HashMap<>();
       private ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
   
       /**
        * 写操作：原子+独占
        * 整个过程必须是一个完整的统一体，中间不许被分割，不许被打断
        *
        * @param key
        * @param value
        */
       public void put(String key, Object value) {
           rwLock.writeLock().lock();
           try {
               System.out.println(Thread.currentThread().getName() + "\t正在写入：" + key);
               TimeUnit.MILLISECONDS.sleep(300);
               map.put(key, value);
               System.out.println(Thread.currentThread().getName() + "\t写入完成");
           } catch (Exception e) {
               e.printStackTrace();
           } finally {
               rwLock.writeLock().unlock();
           }
   
       }
   
       public void get(String key) {
           rwLock.readLock().lock();
           try {
               System.out.println(Thread.currentThread().getName() + "\t正在读取：" + key);
               TimeUnit.MILLISECONDS.sleep(300);
               Object result = map.get(key);
               System.out.println(Thread.currentThread().getName() + "\t读取完成: " + result);
           } catch (Exception e) {
               e.printStackTrace();
           } finally {
               rwLock.readLock().unlock();
           }
   
       }
   
       public void clear() {
           map.clear();
       }
   }
   ```

4. ##### 自旋锁

   是指尝试获取锁的线程不会立即阻塞，而是==采用循环的方式去尝试获取锁==，这样的好处是减少线程上下文切换的消耗，缺点是循环会消耗CPU

   ```java
   public final int getAndAddInt(Object var1, long var2, int var4) {
           int var5;
           do {
               var5 = this.getIntVolatile(var1, var2);
           } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));
           return var5;
       }
   ```

   手写自旋锁

   ```javascript
   /**
    * 实现自旋锁
    * 自旋锁好处，循环比较获取知道成功位置，没有类似wait的阻塞
    *
    * 通过CAS操作完成自旋锁，A线程先进来调用mylock方法自己持有锁5秒钟，B随后进来发现当前有线程持有锁，不是null，所以只能通过自旋等待，知道A释放锁后B随后抢到
    */
   public class SpinLockDemo {
       public static void main(String[] args) {
           SpinLockDemo spinLockDemo = new SpinLockDemo();
           new Thread(() -> {
               spinLockDemo.mylock();
               try {
                   TimeUnit.SECONDS.sleep(3);
               }catch (Exception e){
                   e.printStackTrace();
               }
               spinLockDemo.myUnlock();
           }, "Thread 1").start();
   
           try {
               TimeUnit.SECONDS.sleep(3);
           }catch (Exception e){
               e.printStackTrace();
           }
   
           new Thread(() -> {
               spinLockDemo.mylock();
               spinLockDemo.myUnlock();
           }, "Thread 2").start();
       }
   
       //原子引用线程
       AtomicReference<Thread> atomicReference = new AtomicReference<>();
   
       public void mylock() {
           Thread thread = Thread.currentThread();
           System.out.println(Thread.currentThread().getName() + "\t come in");
           while (!atomicReference.compareAndSet(null, thread)) {
   
           }
       }
   
       public void myUnlock() {
           Thread thread = Thread.currentThread();
           atomicReference.compareAndSet(thread, null);
           System.out.println(Thread.currentThread().getName()+"\t invoked myunlock()");
       }
   }
   ```

### CountDownLatch/CyclicBarrier/Semaphore

1. #### CountDownLatch（火箭发射倒计时）

   1. 它允许一个或多个线程一直等待，知道其他线程的操作执行完后再执行。例如，应用程序的主线程希望在负责启动框架服务的线程已经启动所有的框架服务之后再执行
   2. CountDownLatch主要有两个方法，当一个或多个线程调用await()方法时，调用线程会被阻塞。其他线程调用countDown()方法会将计数器减1，当计数器的值变为0时，因调用await()方法被阻塞的线程才会被唤醒，继续执行
   3. CASE:

   ```java
   public class CountDownLatchDemo {
       public static void main(String[] args) throws InterruptedException {
   //        general();
           countDownLatchTest();
       }
   
       public static void general(){
           for (int i = 1; i <= 6; i++) {
               new Thread(() -> {
                   System.out.println(Thread.currentThread().getName()+"\t上完自习，离开教室");
               }, "Thread-->"+i).start();
           }
           while (Thread.activeCount()>2){
               try { TimeUnit.SECONDS.sleep(2); } catch (InterruptedException e) { e.printStackTrace(); }
           }
           System.out.println(Thread.currentThread().getName()+"\t=====班长最后关门走人");
       }
   
       public static void countDownLatchTest() throws InterruptedException {
           CountDownLatch countDownLatch = new CountDownLatch(6);
           for (int i = 1; i <= 6; i++) {
               new Thread(() -> {
                   System.out.println(Thread.currentThread().getName()+"\t被灭");
                   countDownLatch.countDown();
               }, CountryEnum.forEach_CountryEnum(i).getRetMessage()).start();
           }
           countDownLatch.await();
           System.out.println(Thread.currentThread().getName()+"\t=====秦统一");
       }
   }
   ```

2. #### CyclicBarrier（集齐七颗龙珠召唤神龙）

   1. 可循环（Cyclic）使用的屏障。让一组线程到达一个屏障（也可叫同步点）时被阻塞，知道最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续干活，线程进入屏障通过CycliBarrier的await()方法

   2. CASE:

      ```java
      public class CyclicBarrierDemo {
          public static void main(String[] args) {
              cyclicBarrierTest();
          }
      
          public static void cyclicBarrierTest() {
              CyclicBarrier cyclicBarrier = new CyclicBarrier(7, () -> {
                  System.out.println("====召唤神龙=====");
              });
              for (int i = 1; i <= 7; i++) {
                  final int tempInt = i;
                  new Thread(() -> {
                      System.out.println(Thread.currentThread().getName() + "\t收集到第" + tempInt + "颗龙珠");
                      try {
                          cyclicBarrier.await();
                      } catch (InterruptedException e) {
                          e.printStackTrace();
                      } catch (BrokenBarrierException e) {
                          e.printStackTrace();
                      }
                  }, "" + i).start();
              }
          }
      }
      ```

3. #### Semaphore信号量

   > 可以代替Synchronize和Lock

   1. **信号量主要用于两个目的，一个是用于多个共享资源的互斥作用，另一个用于并发线程数的控制**
   2. CASE:抢车位

   ```java
   public class SemaphoreDemo {
       public static void main(String[] args) {
           Semaphore semaphore = new Semaphore(3);//模拟三个停车位
           for (int i = 1; i <= 6; i++) {//模拟6部汽车
               new Thread(() -> {
                   try {
                       semaphore.acquire();
                       System.out.println(Thread.currentThread().getName() + "\t抢到车位");
                       try {
                           TimeUnit.SECONDS.sleep(3);//停车3s
                       } catch (InterruptedException e) {
                           e.printStackTrace();
                       }
                       System.out.println(Thread.currentThread().getName() + "\t停车3s后离开车位");
                   } catch (InterruptedException e) {
                       e.printStackTrace();
                   } finally {
                       semaphore.release();
                   }
               }, "Car " + i).start();
           }
       }
   }
   ```

   ### synchronized和lock有什么区别？用新的lock有什么好处？请举例

   1. 区别

      1. 原始构成

         - synchronized时关键字属于jvm

           **monitorenter**，底层是通过monitor对象来完成，其实wait/notify等方法也依赖于monitor对象只有在同步或方法中才能掉wait/notify等方法

           **monitorexit**

         - Lock是具体类，是api层面的锁（java.util.）

      2. 使用方法

         - sychronized不需要用户取手动释放锁，当synchronized代码执行完后系统会自动让线程释放对锁的占用
         - ReentrantLock则需要用户去手动释放锁若没有主动释放锁，就有可能导致出现死锁现象，需要lock()和unlock()方法配合try/finally语句块来完成

      3. 等待是否可中断

         - synchronized不可中断，除非抛出异常或者正常运行完成
         - ReentrantLock可中断，设置超时方法tryLock(long timeout, TimeUnit unit)，或者lockInterruptibly()放代码块中，调用interrupt()方法可中断。

      4. 加锁是否公平

         - synchronized非公平锁

         - ReentrantLock两者都可以，默认公平锁，构造方法可以传入boolean值，true为公平锁，false为非公平锁

      5. 锁绑定多个条件Condition

         - synchronized没有
         - ReentrantLock用来实现分组唤醒需要要唤醒的线程们，可以精确唤醒，而不是像synchronized要么随机唤醒一个线程要么唤醒全部线程

      6. CASE:

      ```java
      /**
       * synchronized和lock区别
       * <p===lock可绑定多个条件===
       * 对线程之间按顺序调用，实现A>B>C三个线程启动，要求如下：
       * AA打印5次，BB打印10次，CC打印15次
       * 紧接着
       * AA打印5次，BB打印10次，CC打印15次
       * 。。。。
       * 来十轮
       */
      public class SyncAndReentrantLockDemo {
          public static void main(String[] args) {
              ShareData shareData = new ShareData();
              new Thread(() -> {
                  for (int i = 1; i <= 10; i++) {
                      shareData.print5();
                  }
              }, "A").start();
              new Thread(() -> {
                  for (int i = 1; i <= 10; i++) {
                      shareData.print10();
                  }
              }, "B").start();
              new Thread(() -> {
                  for (int i = 1; i <= 10; i++) {
                      shareData.print15();
                  }
              }, "C").start();
          }
      }
      class ShareData {
          private int number = 1;//A:1 B:2 C:3
          private Lock lock = new ReentrantLock();
          private Condition condition1 = lock.newCondition();
          private Condition condition2 = lock.newCondition();
          private Condition condition3 = lock.newCondition();
      
          public void print5() {
              lock.lock();
              try {
                  //判断
                  while (number != 1) {
                      condition1.await();
                  }
                  //干活
                  for (int i = 1; i <= 5; i++) {
                      System.out.println(Thread.currentThread().getName() + "\t" + i);
                  }
                  //通知
                  number = 2;
                  condition2.signal();
      } catch (Exception e) {
                  e.printStackTrace();
              } finally {
                  lock.unlock();
              }
          }
          public void print10() {
              lock.lock();
              try {
                  //判断
                  while (number != 2) {
                      condition2.await();
                  }
                  //干活
                  for (int i = 1; i <= 10; i++) {
                      System.out.println(Thread.currentThread().getName() + "\t" + i);
                  }
                  //通知
                  number = 3;
                  condition3.signal();
      
              } catch (Exception e) {
                  e.printStackTrace();
              } finally {
                  lock.unlock();
              }
          }
          public void print15() {
              lock.lock();
              try {
                  //判断
                  while (number != 3) {
                      condition3.await();
                  }
                  //干活
                  for (int i = 1; i <= 15; i++) {
                      System.out.println(Thread.currentThread().getName() + "\t" + i);
                  }
                  //通知
                  number = 1;
                  condition1.signal();
      
              } catch (Exception e) {
                  e.printStackTrace();
              } finally {
                  lock.unlock();
              }
          }
      }
      ```

      