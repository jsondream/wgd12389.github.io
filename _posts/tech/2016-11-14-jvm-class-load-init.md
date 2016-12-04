---
layout: post
title: 《深入理解jvm》读书笔记之——类加载机制(类的初始化)
category: 技术
tags: [tech]
keywords: jvm,classloader,clinit
description: 深入理解jvm》读书笔记之——类加载机制(类的初始化)
---

## 类加载的生命周期：   

> 加载 -> 验证 -> 准备 -> 解析 -> 初始化 -> 使用 -> 卸载   

`加载 -> 验证 -> 准备 -> 初始化 -> 卸载 ` 这5个阶段顺序是确定的,klass的加载过程一定会按照这个顺序执行。为了支持java的运行时绑定,***解析阶段***在某些情况下会在***初始化***之后才进行。  

### 类的初始化阶段   

对于`加载`这个阶段是跟具体的虚拟机实现有关,对于整个类加载阶段最重要的就是`初始化`这个阶段.   

#### JVM执行初始化的情况   

对于Hotspot虚拟机而言,遇见以下这5种情况就需要进行`初始化`:   

- 遇到`new、getstatic、putstatic、invokestatic`，如果类还没进行初始化的时候就进行初始化。生成这4种指令最常见的就是：new一个实例化对象、读取或设置一个类的静态字段(被final修饰、已在编译时吧结果放在常量池的静态字段排除)，已经调用一个类的静态方法时。  
- 使用java的反射对类进行反射调用。   
- 初始化类之间，检测父类是否初始化，否则先初始化父类。  
- 虚拟机启动时，用户需要制定一个要执行的主类(包含main方法的类),虚拟机会先初始化这个类  
- 使用jdk7以上动态语言支持时，如果一个methodHandle实例最后的解析结果`REF_getstatic、REF_putstatic、REF_invokestatic`的方法句柄,并且这个方法的句柄对应的类没有初始化的时候。   

这里我们需要注意的点，上面的五种情况指的是主动的引用方式,除了上面5种主动引用之外的被动引用是不会触发`初始化`的.  

***类的被动引用实例:***   

情况一:通过子类来引用父类的静态字段,是只会执行父类的初始化而子类不会初始化的,但是Hotspot虚拟机下会触发子类的加载和验证。   

情况二:声明一个数组类型的类。因为jvm会调用newarray生成一个继承自object的子类，这个类代表了对应的这个类型的数组类型。   

情况三：A引用了B中`fianl`修饰过的静态属性不会导致B的初始化,因为经过编译器的优化,A中引用的这个B的属性元素已经在编译时期存储到了A类下的常量池中，所以其实A下的引用来自于对自身常量池的引用。   

我们这里还需要注意的一点是接口和类不同的就是接口的父接口只有在真正被使用的时候才会被初始化。   

#### 类的初始化之clinit方法   

对于jvm而言，类的初始化也就是执行`clinit`方法，那么什么是`clinit`方法?   

> `clinit`方法是有编译器自动收集类中的所有变量的赋值动作和静态语句块中的语句合并产生的一个用于jvm执行类的初始化的方法。    

需要注意以下几点：    

- `clinit`方法不需要显示的调用父类构造器,虚拟机会保证子类的`clinit`方法执行之前父类的`clinit`方法已经调用完毕,因此虚拟机中第一个被执行`clinit`方法的肯定是`Object`。    
- `clinit`t对于类和接口不是必须的，如果类中没有静态块，也没有对变量的赋值操作，编译器可以不为这个类生产`clinit`方法。  
- 执行接口的`clinit`方法不需要先执行父接口的`clinit`方法，只有当父接口中定义的变量被使用，父接口才会初始化。另外接口的实现类在初始化也一样不会执行接口的`clinit`方法。   
- jvm会保证一个类的`clinit`方法在多线程环境下被正确加锁同步，也就是说类的初始化是线程安全的，同时需要注意的是，如果一个线程执行`clinit`方法时有很耗时的操作，就会阻塞其他也要初始化的这个类的线程。


#### 验证猜想的小技巧   

关于我们文章上述初始化过程中，如何验证，我们可以吧代码在写在类的`static`块里，就能验证我买的猜想了。原理就在上文关于clinit方法中。    
  
