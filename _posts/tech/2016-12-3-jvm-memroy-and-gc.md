---
layout: post
title: 《深入理解jvm》读书笔记之——内存分配和回收策略
category: 技术
tags: [tech]
keywords: jvm,classloader,gc,garbage collector,memory
description: 《深入理解jvm》读书笔记之——内存分配和回收策略
---


## 1、对象优先在eden分配   

jvm给一个对象分配内存会先在eden区域分配，如果内存不足，会发起一次`young GC`.如果回收了之后，内存空间依然不够，就会通过担保机制提前把一些可以转移的对象分配到老年代中，以此保证eden中的内存足够给当前要分配的对象使用。    

## 2、大对象直接进入老年代   

大对象一般指那种很长的字符串以及数组。对于jvm来说，大对象不是一个好的现象，大对象的出现容易导致明明虚拟机内存还有不少空间就可能会提前触发GC（内存空间不连续的时候）。    

设置`-XX:PretenureSizeThreshold`可以设置一个对象大于这个阀值的时候，被直接分配到老年代的区域中。    

## 3、长期存活的对象讲进入老年代   

jvm采用了分带的概念，那么jvm是怎么区分对象具体在哪个代呢？   
JVM给每个对象定义了一个年龄计数器，对象从eden出生后，经过一次young gc仍然存活，能被survivor容纳，就被移动到survivor中，年龄设置为1。之后每次这个对象在survivor中熬过一次young gc之后，年龄都会加1.年龄到了一定的阀值，会晋升到老年代中。我们可以设置JVM参数`-XX:MaxTenuringThreshold`设置晋升到老年代的对象年龄阀值。    
     
## 4、动态对象年龄判定   

如果在survivor中的相同年龄所有对象的大小总和大于survivor空间的一半，年龄大等于该年龄的对象就可以直接进入老年代，不需要等到达到年龄阀值。    

## 5、空间分配担保   

在JVM进行young gc之前，先检查老年代最大可用连续空间是否大于年轻代所有对象总空间：如果条件成立，证明young gc是可以确保安全的；如果不成立，JVM会查看是否允许担保失败，如果允许就会进行young gc。如果不允许，就改为进行一次full gc。   

这里我们说的`是否允许担保失败`我们可以这么理解：在survivor中的一些对象可能会在young gc之后进入老年代（这些对象的大小其实是可以在这次ygc前就知道的），但是如果老年代的空间不足够容纳这些要晋升的对象，这时候进行一次full gc会吧老年代中无用的对象清理，然后给新晋升的这些对象腾出空间。