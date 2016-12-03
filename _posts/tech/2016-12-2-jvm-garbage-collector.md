---
layout: post
title: 《深入理解jvm》读书笔记之——GC收集器
category: 技术
tags: [tech]
keywords: jvm,classloader,gc,garbage collector
description: 《深入理解jvm》读书笔记之——GC收集器
---


## 1、serial收集器    

`serial`是一个单线程的垃圾收集器，它只会使用一个cpu、一个收集线程工作；它在进行gc的时候，必须暂停其他所有线程到它工作结束(这种暂停往往让人难以接受)。         

对于单个cpu的情况下，`serial`是没有线程交互的开销，效率要好于其他。例如在一个简单的桌面应用，只有几十M的内存，他的停顿时间可能只有几十毫秒，所以一般默认的client模式下都是用的`serial`做默认垃圾收集器。     

![serial收集器 ](http://upload-images.jianshu.io/upload_images/584578-aef2c6f30a246b64.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)     

## 2、parnew收集器     

`parnew`其实就是`serial`的多线程版本，`parnew`在单线程的情况下甚至不如`serial`，`parnew`是除了`serial`之外唯一能和CMS配合的。    

`parnew`默认开启收集线程数和cpu的数量相同，我们可以利用`-XX:ParallelGCThreads`参数来控制它开启的收集线程数。    

![parnew收集器](http://upload-images.jianshu.io/upload_images/584578-4115b8ffcde46666.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)    

## 3、parallel scavenge收集器     
 
`parallel scavenge`主要就是关注吞吐量。    
所谓吞吐量：运行用户代码的世界/(运行用户代码时间+GC花费的时间)。    
parallel scavenge收集器中，提供了2个参数来控制吞吐量：    
- `-XX:GCTimeRatio`：gc时间占用的总比例，也就是吞吐量的倒数。           
- `-XX:MaxGCPauseMillis`：最大的暂停毫秒数（这个数值并非越小越好，如果把他设置小了，系统会根据这个值调整空间的大小，也就会加快GC的频率）       

`parallel scavenge`可以设置开启`-XX:UseAdaptiveSizePolicy`，开启这个参数之后就无需关注新生代大小eden和survivor等比例，晋升老年代对象年龄的这些细节了。    

![parallel scavenge的工作细节](http://upload-images.jianshu.io/upload_images/584578-8f8dd83fe549172d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 4、serial old收集器     

serial收集器的老年代版本，使用标记整理算法，主要有两个作用：   
- jdk5之前和parallel scavenge配合使用     
- 作为cms失败的后备收集方案     

## 5、parallel old收集器     

是parallel收集器的老年代版本，用于和parallel收集器搭配组合的，因为parallel收集器不能和cms组合，但是和serial old收集器效率又太低。    
对吞吐量和CPU敏感度特别关注的应用可以使用`parallel`+`parallel old`的组合。      

## 6、CMS收集器     

**CMS的适用特点：**     
- 希望JAVA垃圾回收器回收垃圾的时间尽可能短；   
- 应用运行在多CPU的机器上，有足够的CPU资源；   
- 有比较多生命周期长的对象；   
- 希望应用的响应时间短。    

**CMS的过程：**    
1. 初始标记：标记一下GC ROOT能直接关联的对象，速度很快，这个阶段是会STW。    
2. 并发标记：GC ROOT的引用链的查找过程，标记能关联到GC ROOT的对象，这一个阶段是不需要STW的。    
3. 重新标记：在并发标记阶段，应用的线程可能产生新的垃圾，所以需要重新标记，这个阶段也是会STW。     
4. 并发清除：这个阶段就是真正的回收垃圾的对象的阶段，无需STW。     

![CMS运行过程](http://img2.imgtn.bdimg.com/it/u=3636182856,626235002&fm=21&gp=0.jpg)    
 
**CMS的缺点：**     
1. 对cpu比较敏感。     
2. 可能出现浮动垃圾：在并发清除阶段，用户还是继续使用的，这时候就会有新的垃圾出现，CMS只能等下一次GC才能清除掉他们。   
3. CMS运行期间预留内存不够的话，就会出现`concurrent Mode Failure`，这时候就会启动serial收集器进行收集。    
4. CMS基于`标记清除`算法实现，会产生内存碎片空间。碎片空间过多就会对大对象的分配空间造成麻烦。为了解决碎片问题，CMS提供一个参数来控制是否在GC的时候整合内存中的碎片，这个碎片整合的操作是无法并发的，会延长STW的时间。    

## 7、G1收集器     

**G1的特点：**    
- 利用多CPU来缩短STW的时间     
- 可以独立管理整个堆（使用分代算法）    
- 整体是基于`标记-整理`，局部使用`复制`算法，不会产生碎片空间    
- 可以预测停顿：G1吧整个堆分成多个Region，然后计算每个Region里面的垃圾大小（根据回收所获得的空间大小和回收所需要的时间的经验值），在后台维护一个优先列表，每次根据允许的收集时间，优先回收价值最大的Region。    

**G1的运行过程：**   
- 初始标记：标记一下GC ROOT能直接关联的对象，速度很快，这个阶段是会STW。     
- 并发标记：在GC ROOT中运用可达性分析算法，找出存活的对象，耗时较长，但是无需STW。    
- 最终标记：修正并发标记期间用户线程对垃圾对象的修改，需要停顿线程，但是可以并行执行。     
- 筛选回收：先计算回收各个Region的价值，然后根据用户需求来进行回收。    

![G1运行示意图](http://img3.imgtn.bdimg.com/it/u=3460360757,1903828970&fm=21&gp=0.jpg)    