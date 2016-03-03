---
layout:     post
title:      "也说单例模式"
subtitle:   "设计模式探索"
date:       2016-03-02 20:37:35
author:     "Rorschach"
header-img: "img/post-multi-download.jpg"
tags:
- design pattern
- java
- android
---

## 动机

对于系统中的某些类来说，只有一个实例很重要，例如，一个系统中可以存在多个打印任务，但是只能有一个正在工作的任务；一个系统只能有一个窗口管理器或文件系统；一个系统只能有一个计时工具或ID（序号）生成器。要保证一个类只能有一个实例，就需要我们今天要说的单例模式。

## 定义

1. 某一个类只能有一个实例
2. 这个类能够自行实例化
3. 这个类能够自行向整个系统提供这个实例

## 模式分析

单例模式的目的是保证一个类仅有一个实例，并提供一个访问改实例的全局访问点。单例类拥有一个私有构造函数，确保用户无法通过new关键字直接实例化对象。除此之外，单例中包含一个静态私有成员变量与静态公有的工厂方法，该工厂方法负责检验实例的存在及实例化，然后将实例化的对象存储在静态成员变量中，以确保只有一个实例被创建。

在单例模式的实现过程中，需要注意如下三点：

1. 单例类的构造函数为私有；
2. 提供一个自身的静态私有成员变量；
3. 提供一个公有的静态工厂方法。

## 单例的实现

### 懒汉式

根据上文的分析，我们很容易写出以下的代码

```
public class Sigleton {

    private static Sigleton instance = null;

    private Sigleton(){}

    public static Sigleton getInstance(){
        if(instance != null){
            instance = new Sigleton();
        }

        return instance;
    }
}
```

上面的代码看起来很不错，但是该代码存在很严重的问题：由于单例是全局性的实例，在多线程下，所有全局性的东西都很危险。假设存在多个线程同时调用上述代码，可能在同一时间有多个线程都通过了`instance != null`的检查而创建了多个Singleton的实例，并且有可能造成内存泄露。熟悉多线程的话会立刻想到多线程的同步问题，于是我们写出了如下代码

```
public class Sigleton {

    private static Sigleton instance = null;

    private Sigleton(){}

    public static Sigleton getInstance(){

        if(instance != null){
            synchronized(Sigleton.class){       
                instance = new Sigleton();
            }
        }

        return instance;
    }
}
```

现在看起来是不是不错了，没有问题了吧？等等，真的没问题吗？使用`synchronized`将并行执行的线程变成了串行执行的操作，但是结果和上述代码一样，仍然会出现多个`Sigleton`的实例。因此我们还需要将判断的代码也加入到互斥锁的范围中，代码如下：

```
public class Sigleton {

    private static Sigleton instance = null;

    private Sigleton(){}

    public static Sigleton getInstance(){

        synchronized(Sigleton.class){       
            if(instance != null){
                instance = new Sigleton();
            }
        }

        return instance;
    }
}
```


### Double Check

上述代码，单例的效果能够保证。但是这种方法还是有问题，因为我们原来只需要让new实例的操作上锁，但是读取变量的操作并不需要上锁，上述代码会影响后续操作的响应时间，造成性能上的问题。因此，我们应该在判断之后执行上锁操作

```
public class Sigleton {

    private static Sigleton instance = null;

    private Sigleton(){}

    public static Sigleton getInstance(){

        if(instance != null){
            synchronized(Sigleton.class){       
                if(instance != null){
                    instance = new Sigleton();
                }
            }
        }

        return instance;
    }
}
```

简单的说明一下：

1. 首先判断实例是否存在，若实例存在则直接返回
2. 若实例不存在，同步线程，防止多个线程同时进入代码块，产生多个实例
3. 若被同步的线程中有一个线程产生了实例，则其他线程也没有机会创建

这次的代码看起来应该是没有任何问题了，但是还存在一个问题，因为`instance = new Sigleton()`并不是原子操作，实际上在JVM中，这个操作分为三步：

1. 为`instance`分配内存空间
2. 调用`new Sigleton()`来实例化对象
3. 将`instance`对象指向分配好的内存空间

只有步骤３执行完成后，`instance`才不为空。但是JVM的即时编译器存在指令重排优化，也就是说步骤2和３之间的顺序得不到保证，有可能出现顺序为1-3-2的情况，此时在3-2之间，若存在线程一和线程而，线程一先抢占成功，为`instance`分配内存空间，然后线程二抢占成功，而`instance`确实指向一个内存空间但是却没有被实例化，因此最终会报错。

优化后：

```
public class Sigleton {

    private volatile　static Sigleton instance = null;   //notice here

    private Sigleton(){}

    public static Sigleton getInstance(){

        if(instance != null){
            synchronized(Sigleton.class){       
                if(instance != null){
                    instance = new Sigleton();
                }
            }
        }

        return instance;
    }
}
```

`volatile`的作用：

1. 被其标记的成员不会在多个线程中存在多个副本，直接从内存中读取
2. 禁止指令重排优化

`volatile`关键字只在JDK1.5之后生效。

### 饿汉式

`DoubleCheck`写法既复杂又隐藏很多问题，因此引出更加简单的实现。


```
public class Singleton{

    private static final Singleton instance = new Singleton();     // initialize while class loading
    
    private Singleton(){}

    public static Singleton getInstance(){
        return instance;
    }
}
```


这种写法相对前文中各种实现都更加优雅，在类被第一次加载到内存中是就实例化对象，因此是线程安全的。但是问题就在于这里，实际上我们想在需要时才加载，希望能够控制单例加载的时机，那么是否存在这样的实现呢？


### 静态内部类

```
public class Singleton {

    private static class SingletonHolder {
        private static final Singleton INSTANCE = new Singleton();
    }

    private Singleton (){}

    public static final Singleton getInstance() {
        return SingletonHolder.INSTANCE;
    }
}
```

将`Singleton`实例的创建放置于静态内部类中，当内部类被加载时创建，使用Java的类加载机制保证线程安全。由于`SingletonHolder`是私有的，仍然保证了只有`getInstance()`才能够真正的实例化对象。读取实例的时候不需要同步，没有性能的缺陷，同时不依赖JDK版本，实现相当的优雅。

### 枚举


```
public enum Singleton {
    INSTANCE;
}
```

枚举实例的创建是线程安全的，但是枚举中其他方法的线程安全问题需要程序员自己负责。使用枚举还能防止通过过反射调用私有构造器，简单优雅，值得推荐。

## One More Thing

未完待续