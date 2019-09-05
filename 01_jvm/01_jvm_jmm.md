## 1.1 Java内存模型（JMM）

> [!COMMENT]
>
> 操作之间的内存可见性。在JMM中，如果一个操作执行的结果需要对另一个操作可见，那么这两个操作之间必须要存在happens-before关系。

> 参考《Java并发编程艺术》第3章Java内存模型

#### 1.1.1 happens-before

- 程序顺序规则：一个线程中的每个操作，happens-before于该线程中任意后续操作。

  ```java
  class HBExample {
      int x = 0;
      boolean v = false;
      
      public void writer() {
          x = 42;
          v = true;
      }
      
      public void reader() {
          if(v == true) {
              // x = 42;
          }
      }
  }
  ```

  

- 监视器锁规则：对一个锁的解锁，happens-before于随后对这个锁的加锁。

  ```java
  class HBExample {
      int x = 0;
      boolean v = false;
      
      public synchronized void writer() {
          x = 42;
          v = true;
      }
      
      public synchronized void reader() {
          if(v == true) {
              // x = 42;
          }
      }
  }
  ```

  

- volatile变量规则：对一个volatile域的写，happens-before于任意后续对这个volatile域的读。

  ```java
  class HBExample {
      int x = 0;
      volatile boolean v = false;
      
      public void writer() {
          x = 42;
          v = true;
      }
      
      public void reader() {
          if(v == true) {
              // x = 42;
          }
      }
  }
  ```

  

- 传递性：如果A happens-before B，且B happens-before C，那么A happens-before C。

- start()规则：如果线程A执行操作ThreadB.start()（启动线程B），那么A线程的ThreadB.start()操作happens-before于线程B中的任意操作。

  ```java
  Thread B = new Thread() -> {
    //主线程调用B.start()之前
    // 所有对共享变量的修改，此处皆可见
    // var == 77
  };
  // 此处对共享变量var修改
  var = 77;
  // 主线程启动子线程
  B.start();
  ```

- join()规则：如果线程A执行操作ThreadB.join()并成功返回，那么线程B中的任意操作happens-before于线程A从ThreadB.join()操作成功返回。

  ```java
  Thread B = new Thread() -> {
     // 此处对共享变量var修改
     var = 66;
  };
  
  // 例如此处对共享变量修改，则这个修改结果对线程B可见
  // 主线程启动子线程
  B.start();
  B.join();
  // 子线程所有对共享变量的修改
  // 在主线程调用B.join()之后皆可见
  // var == 66
  ```

  

#### 1.1.2 重排序

##### 数据依赖性

- 如果两个操作访问同一个变量，且这两个操作中有一个为写操作，此时这两个操作之间就存在数据依赖性。

- 编译器和处理器在重排序时，会遵守数据依赖性，编译器和处理器不会改变存在数据依赖关系的两个操作的执行顺序。

##### as-if-serial语义

> 不管怎重排序，单线程程序的执行结果不会被改变。编译器、runtime和处理器都必须遵守as-if-serial语义。



#### 1.1.3 Volatile

> 理解volatile特性的一个好方法是把对volatile变量的单个读/写，看成是使用同一个锁，对这些单个读/写操作做了同步。

##### volatile写-读建立的happens-before关系

```java
class VolatileExample {
    int a = 0;
    volatile boolean flag = false;
    
    public void writer() {
        a = 1; // 1
        flag = true; // 2
    }
    
    public void reader() {
        if(flag) { // 3
            int i = a; // 4
            ......
        }
    }
}
```

> 假设线程A执行writer()方法之后，线程B执行reader()方法。根据happens-before规则，这个过程建立的happens-before关系可以分为3类：

- 根据程序次序规则，1 happens-before 2; 3 happens-before 4。
- 根据volatile规则，2 happens-before 3。
- 根据happens-before的传递性规则，1 happens-before 4。

##### volatile写-读的内存语义

> 当写一个volatile变量时，JMM会把该线程对应的本地内存中的共享变量值刷新到主内存中。

##### volatile内存语义的实现

> 重排序分为编译器重排序和处理器重排序。为了实现volatile内存语义，JMM会分别限制这两种类型的重排序类型。

- 在每个volatile写操作的前面插入一个StoreStore屏障。
- 在每个volatile写操作的后面插入一个StoreLoad屏障。
- 在每个volatile读操作的后面插入一个LoadLoad屏障。
- 在每个volatile读操作的后面插入一个LoadStore屏障。



#### 1.1.4 双重检查锁定与延迟初始化

> 在Java程序中，有时候可能需要推迟一些高开销的对象初始化操作，并且只有在使用这些对象时才进行初始化。

```java
public class DoubleCheckedLocking { // 1
    private static Instance instance; // 2
    
    public static Instance getInstance() { // 3
        if (instance == null) { // 4：第一次检查
            synchronized (DoubleCheckedLocking.class) { // 5:加锁
                if( instance == null) { // 6:第二次检查
                    instance = new Instance(); // 7:问题的根据在这里
                }
            }
        }
        return instance; // 8
    }
}
```

> 双重检查锁定看起来似乎很完美，但这是一个错误的优化！在线程执行到第4行，代码读取到instance不为null时，instance引用的对象有可能还没有完成初始化。

##### 问题的根源

> 前面的双重检查锁定示例代码的第7行创建一个对象。这一行代码可以分解为如下的3行伪代码。

```java
memory = allocate(); // 1、分配对象的内存空间
ctorInstance(memory); // 2、初始化对象
instance = memory; // 3、设置instance指向刚分配的内存地址
```

> 上面3行伪代码中的2和3之间，可能会被重排序。2和3之间重排序之后执行的时序如下：

```java
memory = allocate(); // 1、分配对象的内存空间
instance = memory; // 3、设置instance指向刚分配的内存地址
				   // 注意，此时对象还没有被初始化！
ctorInstance(memory); // 2、初始化对象

```

##### 基于volatile的解决方案

> 对于前面的基于双重检查锁定来实现延迟初始化的方案，只需要做一点小的修改(把**instance声明为volatile型**)，就可以实现线程安全的延迟初始化。当声明对象的引用为volatile后，伪代码中的2和3之间的重排序，在多线程环境中会被禁止。

##### 基于类初始化的解决方案

> JVM类的初始化阶段（即在Class被加载后，且被线程使用之前），会执行类的初始化。在执行类的初始化期间，JVM会去获取一个锁。这个锁可以同步多个线程对同一个类初始化。

```java
public class InstaceFactory {
    private static class InstanceHolder {
        public static Instance instance = new Instance();
    }
    
    public static Instance getInstance() {
        return InstanceHolder.instance; // 这里导致InstanceHolder类被初始化
    }
}
```

> JVM在类初始化期间会获取这个初始化锁，并且每个线程至少获取一次锁来确保这类已经被初始化过 。