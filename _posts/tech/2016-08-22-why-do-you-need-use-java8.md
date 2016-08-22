---
layout: post
title: 为什么你要使用java8
category: 技术
tags: [tech]
keywords: java8
description: java8的改进,java8的新特性描述,java8的优点介绍
---


## java8优势  

相信对于java8这个字眼大家都已经不陌生了,但是对于java8的了解和使用很多人还不是很清楚,甚至很多人还在犹豫着要不要用java8,那么我写这篇文章的目的就是告诉你,你一定要使用java8以及你为什么要使用java8.  

### lambda   

在Java7以及之前的代码里，为了实现带一个方法的接口，往往需要定义一个匿名类并复写接口方法，代码显得很臃肿。  
比如我们用来给数组排序的Comparator接口：  

```
String[] str = "aa、bb、cc、dd".split("、");
Arrays.sort(str, new Comparator<String>() {
    @Override
    public int compare(String s1, String s2) {
        return s1.toLowerCase().compareTo(s2.toLowerCase());
    }
});
```  

然而对于这种只有一个方法的接口，在Java8里面，我们可以把它视为一个函数，用lambda表示式简化如下的操作：  

```
String[] str = "aa、bb、cc、dd".split("、");
Arrays.sort(str, (s1, s2) -> {
    return s1.toLowerCase().compareTo(s2.toLowerCase());
});
```  

这样我们的代码看着就简洁了很多,或许单单看这一个例子大家还体会不到lambda那神奇的魔力.  
如果接触过jedis的朋友,相信都知道在使用jedis的时候会有获取连接,进行操作,释放连接这几个步骤。  
大家能看出来,除了进行操作这个步骤是不同的,连接的获取和释放代码是相同的,那么这时候我们就可以考虑用lambda表达式把他封装起来,具体的实现可以去我的github上看,仓库地址[点击这里](https://github.com/wgd12389/redisses/tree/master/redisses-client).  

### stream   

集合类新增的stream()方法用于把一个集合变成Stream，然后，通过filter()、map()等实现Stream的变换。Stream还有一个forEach()来完成每个元素的迭代。

例如我原来要遍历一个list,要对这个list做一个for遍历  

代码是这样的:  

```
        List<String> a = new ArrayList<String>();
        for (String o : a) {
            System.out.println(o)
        }
```

而我用了stream之后,代码却简洁成这个样子：  

```
        List<String> a = new ArrayList<>();
        a.stream().forEach(c-> System.out.println(c));

```

在比如我想取得这个list的的前10项:  

```  
List<String> list = oldList.limit(10).collect(Collectors.toList());
```  

那么如果我想取得数列的第20~30项，可以这样做：  

```
List<String> list = oldList.skip(20).limit(10).collect(Collectors.toList());
```

而且Stream有串行和并行两种，串行Stream上的操作是在一个线程中依次完成，而并行Stream则是在多个线程上同时执行。  
stream提供了parallelStream使用多线程进行操作,加大了运算效率.  

Stream中还有fifter、sorted、Match、map、Reduce这一类的api,大家可以在日后的使用中慢慢去体会他的强大之处.    

### optional   

Optional 不是函数是接口，这是个用来防止NullPointerException异常的辅助类型，这是下一届中将要用到的重要概念，他最初源自于google出的框架包guava之中,于1.8正式引入到jdk中使用  

Optional 被定义为一个简单的容器，其值可能是null或者不是null。在之前一般某个函数应该返回非空对象但是偶尔却可能返回了null，然后这样的结果或许可能会让你的程序出现NullPointerException,会造成程序的崩溃等不可预估的问题,而在Java 8中，不推荐你返回null而是返回Optional。  

```
Optional<String> optional = Optional.of("bam");

optional.isPresent();           // true
optional.get();                 // "bam"
optional.orElse("fallback");    // "bam"

optional.ifPresent((s) -> System.out.println(s.charAt(0)));     // "b"
```

### 构造函数和方法引用  

#### 构造函数  

在java8之前我们新建一个对象都是这样的`User u = new User()`   
而在java8中我们可以这么写代码`User u = User::new `   

### 方法引用  

我经常把方法引用和lambda合在一起使用   
比如我们有一个业务需要把集合里每个元素做处理   
那么我们根据业务规范要把这个处理的事件写成一个业务方法,然后遍历集合元素,并传递元素调用该方法   

一般我们在使用java8以前都是这么写的:  

```
//声明一个名为doSomeThing的方法  
public void doSomeThing(String item){
   // TODO
}


List<String> list = new ArrayList<String>();
for(String item:list){
   doSomeThing(item);
}
``` 

那么在java8中,上面的东西我很简单的就搞定了`list.stream().foreach(item->doSomeThing(item))`  

我用方法引用再简化一下上面的代码,就变成了现在这个样子`list.stream().foreach(this:: doSomeThing)`,很牛逼是不是!  

### hashMap的优化  

- 如果在hash冲突的时候,链表长度大于默认因子数8的时候,会变更为红黑树,利用红黑树快速增删改查的特点提高HashMap的性能,用以提高效率.  
- JDK1.7中rehash的时候,旧链表迁移新链表的时候,如果在新表的数组索引位置相同,则链表元素会倒置,JDK1.8不会倒置.  

详细的优化请参考美团技术团队写的这篇文章*[Java 8系列之重新认识HashMap](http://tech.meituan.com/java-hashmap.html)*    

### concurrentHashMap的优化  

concurrentHashMap变成了cas无锁模式,只有在扩容的时候才会用sync进行锁,cas的无锁模式使得concurrentHashMap在吞吐量上有了一定等级的提升  

### JVM方面的优化  

内存模型换成红黑树,有利于gc,减少内存泄露  
java内存分带进行了改进,取消了永久代,变成了metaSpace  

#### 内存分带的改进:metaSpace  

##### java8内存分带的改进之后带来哪些优势？    

你的metaSpace使用了机器的堆内存,metaSpace是自动扩展的,但是你可以给他设置一个固定的最大容量,但是其实你是无需关系他的大小,让你不用像以前一样担心permGen溢出的问题.     

在永久代被发明的时候那时候还没有spi,osgi这类的动态类加载机制,所以一个类一旦被JVM加载之后他就一直在内存之中,一直到JVM结束运行才会释放(其实这个时候说道释放也没有什么意义了),而现在的阶段类在JVM的生命周期变得不确定了,可以灵活的加载和释放,所以java8中吧永久代变为metaSpace的意义其实也是为了应对现在类机制灵活的变化.  

##### metaSpace的缺点  

上面我们谈到了内存分带改进的好处,但是metaSpace也有他的缺点  

***他的缺点是什么呢？***  

- 拿class实例来讲,无论是永久代还是metaSpace都会存在类加载泄露的风险.唯一的区别是metaSpace的默认设置(自动调整metaSpace空间大小),这个看来是改进的地方,会让你在类加载泄露的问题变得更难发现.  
- 另一方面,如果metaSpace的东西占用的空间更大了之后,他是有可能耗尽操作系统的内存,这种情况的发生比耗尽JVM永久代的后果更严重,虽然你可以设置一个metaSpace的最大值,但是这个最大值的大小又会变成调优的另一个新问题了.  

无论你是在JVM里使用metaSpace还是永久代,如果你正在使用动态类卸载,你应该采取措施来检测和防止类加载泄露.
  
***参考的文章:***  

[What is the difference between PermGen and Metaspace?](http://stackoverflow.com/questions/27131165/what-is-the-difference-between-permgen-and-metaspace)  


