---
title: 单例模式
date: 2022-08-28 20:44:27
categories:
  - 设计模式
---

**单例模式**是**GoF设计模式**之一，属于<strong>创造性设计模式（Creational Design Pattern）</strong>。从定义来看，它是一个非常简单的设计模式，但在实现时依然存在诸多方面的问题(如线程安全和单例破坏等问题)。本文将简述单例模式的原则、实现及一些最佳实践。

<!-- more -->

## 简介

> + 单例模式限制类的实例化，并确保在java虚拟机中只存在该类的一个实例
> + 单例类必须提供全局访问接口来获取类的实例

## 实现

单例模式可以有多种实现方式，但它们都有以下共同的条件：

> + 构造函数私有，以限制通过构造函数构造多个实例
> + 唯一实例的静态私有变量
> + 返回类实例的公共静态方法，这是外部客户端获取单例类实例的全局接口

通常根据是否提前实例化对象，可以分为<strong>[饿汉式单例（Eager initialization）](#eagerInitialization)</strong>和<strong>[懒汉式单例（Lazy Initialization）](#lazyInitialization)</strong>两种实现。

### <span id="eagerInitialization">饿汉式实现</span>

饿汉式单例，即单例实例是在类加载时创建，是线程安全的，但它有一个缺点，即当客户端应用程序不使用该单例时也会创建实例。这种提前创建实例的实现方式仅适用于单例类不占用大量的资源情况下，但在多数情况下，单例类是为了文件系统、数据库连接等资源创建的，除非客户端调用`getInstance()`方法，否则应该避免实例化。

#### 简单实现

```java
package cn.sillyfish.designpattern.creational.singleton;

public class EagerInitializedSingleton {
    private static final EagerInitializedSingleton instance = new EagerInitializedSingleton();
    
    // private constructor to avoid client applications to use constructor
    private EagerInitializedSingleton() {}

    public static EagerInitializedSingleton getInstance() {
        return instance;
    }
}
```

#### 静态代码块实现

通过静态代码块实现饿汉式单例，可以在静态代码块中处理异常：

```java
package cn.sillyfish.designpattern.creational.singleton;

public class StaticBlockSingleton {
    private static final StaticBlockSingleton instance;
    
    private StaticBlockSingleton() {}
    
    // static block initialization for exception handling
    static {
        try {
            instance = new StaticBlockSingleton();
        } catch (Exception e) {
            throw new RuntimeException("Exception occurred in creating singleton instance");
        }
    }
    
    public static StaticBlockSingleton getInstance() {
        return instance;
    }
}
```

#### 枚举类实现

通过枚举类可以实现饿汉式单例，同时由于**枚举类的保护机制**，这种实现能够防止**反射破坏**：

```java
package cn.sillyfish.designpattern.creational.singleton;

public enum EnumSingleton {
    INSTANCE;
    
    public static void doSomething() {
        //do something
    }
}
```

### <span id="lazyInitialization">懒汉式实现</span>

与<strong>[饿汉式单例](#eagerInitialization)</strong>不同，**懒汉式单例**在第一次调用`getInstance()`方法时才会进行实例化。

#### 简单实现

```java
public class LazyInitializedSingleton {
    private static LazyInitializedSingleton instance;
    
    private LazyInitializedSingleton() {}
    
    public static LazyInitializedSingleton getInstance() {
        if (instance == null) {
            instance = new LazyInitializedSingleton();
        }
        return instance;
    }
}
```

这种实现是线程不安全的，在多线程场景下有可能获取到多个单独的实例，违背了单例模式的初衷，可以通过加锁的方式来保证线程安全。

#### synchronized保证线程安全

```java
package cn.sillyfish.designpattern.creational.singleton;

public class ThreadSafeSingleton {
    private static ThreadSafeSingleton instance;
    
    private ThreadSafeSingleton() {}
    
    public static synchronized ThreadSafeSingleton getInstance() {
        if (instance == null) {
            instance = new ThreadSafeSingleton();
        }
        return instance;
    }
}
```

通过`synchronized`加同步锁，可以保证线程安全，但是每次调用`getInstance()`方法都要获取锁，事实上在实例化完成后，无需再加锁同步，这种加锁方式造成了额外开销。可以在获取同步锁前判断是否已经实例化来决定是否获取锁，即采用<strong>[双检锁（DCL Lock）](#dclLock)</strong>的实现方式。

#### <span id="dclLock">双检锁（DCL Lock）实现</span>

```java
package cn.sillyfish.designpattern.creational.singleton;

public class DCLSingleton {
    private static volatile DCLSingleton instance;

    private DCLSingleton() {}

    public static DCLSingleton getInstance() {
        if (instance == null) {
            synchronized (DCLSingleton.class) {
                if (instance == null) {
                    instance = new DCLSingleton();
                }
            }
        }
        return instance;
    }
}
```

通过添加一个外层判断可以在实例化完成时不再获取锁从而避免调用`getInstance()`方法的额外开销，同时内层的判断也是保证线程安全所必须的，需要两次检查，<strong>双检锁（double checked lock, DCL Lock）</strong>因而得名。同时注意到静态变量`instance`使用了`volatile`关键字，因为`instance = new DCLSingleton()`是包含多个步骤的非原子操作：

> 1. 分配内存空间
> 2. 构造初始化
> 3. `instance`变量指向对象（变量赋值）

且底层**指令重排**优化并不保证**2**、**3** 步骤的顺序，即可能按照**1 → 3 → 2**的顺序执行，在多线程场景下另一个线程可能获取到未初始化完全的实例，使用`volatile`关键字便可以避免**指令重排**（**1 → 2 → 3**），保证构造完成后再给变量赋值，从而可以保证线程安全。

#### 静态内部类实现（Bill Pugh Singleton）

利用**静态内部类**在第一次使用时才加载的特性可以达到**懒汉式单例**的效果：

```java
package cn.sillyfish.designpattern.creational.singleton;

public class BillPughSingleton {
    private BillPughSingleton() {}
    
    private static class SingletonHelper {
        private static final BillPughSingleton INSTANCE = new BillPughSingleton();
    }
    
    public static BillPughSingleton getInstance() {
        return SingletonHelper.INSTANCE;
    }
}
```

### 单例与序列化

当单例类实现了`Serializable`接口并将单例实例进行**序列化**，**反序列化**时会得到一个新的单例实例（内存地址不同），破坏了单例，为了避免这种情况，只需要提供`readResolve()`方法，反序列化便会从该方法获取实例，以保证单例：

```java
package cn.sillyfish.designpattern.creational.singleton;

import java.io.Serializable;

public class SerializedSingleton implements Serializable {
    private static final long serialVersionUID = -7604766932017737115L;

    private SerializedSingleton() {}
    
    private static class SingletonHelper {
        private static final SerializedSingleton instance = new SerializedSingleton();
    }
    
    public static SerializedSingleton getInstance() {
        return SingletonHelper.instance;
    }
    
    private Object readResolve() {
    	return getInstance();
    }
}
```

## 参考

【1】[Java Singleton Design Pattern Best Practices with Examples](https://www.digitalocean.com/community/tutorials/java-singleton-design-pattern-best-practices-examples)

【2】[单例、序列化和readResolve()方法 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/136769959)

【3】[Java单例模式之双检锁深入思考 - 腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1161095)

【4】[Java并发：volatile内存可见性和指令重排 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/151443733)
