---
layout: post
title: parallelStream使用不当引发的血案
category: 技术
tags: [technology]
keywords: parallelStream,concurrentSession,bug
description: java的stream的parallelStream使用不当印发的血案
---
   
# Stream简洁
众所周知,`java8`的新特性中出了`lambda`表达式之外最受人关注的还有`stream`一系列的api。  
`parallelStream`是`stream`中的一个很受开发者喜欢的api,喜欢的同时,如果你使用不当也会造成一些在你看来莫名其妙的问题。  
下面我就跟大家说一下在我是`如何使用不当遇到那个让我感到奇怪的问题`。  
## 问题场景描述:   
1. 我们的系统中使用了一个会话管理器的东西,就是利用ThreadLocal来制造了一个线程变量,存放每次请求的会话线程的线程变量。  
2. 有一个程序变量需要遍历取值,并且需要对其中的值和线程变量的值来做业务判断,进行处理.   
3. 之前使用Stream进行流操作,未发生任何异常.   
## 问题发生情况描述:   
1. 想使用`parallelStream`提升遍历性能,就将`stream`改成了`parallelStream`.  
2. 这时候重启调试之后,请求这个api,总是发生空指针的异常.  
## 问题定位:  
1. 因为使用了lambda表达式,所以控制台只是提示`parallelStream`的遍历这一行报错(这也是使用lambda的不便之处,调错没有之前方便)  
2. 使用debug一步步跟随调试,发现错误定位在了会话管理器获取线程变量这一行   
## 问题思考:  
1. 之前在使用`stream`这个API的时候没有发生问题,便思考到了是`parallelStream`的原因使得程序产生了问题.  
2. 那么`parallelStream`怎么会影响我们的会话管理器取得线程变量呢.   
## 问题解决:    
1. 查看`parallelStream`的源码![parallelStream源码](http://7xpz5v.com1.z0.glb.clouddn.com/parallelStream.png).     
2. parallelStream是创建一个并行的Stream,而且他的并行操作是不具备线程传播性的,所以在使用会话管理器的时候是无法获取值的.  
## 问题总结:   
`parallelStream`是一把双刃利器,他的并行操作可以在很多时候作为提升效率的一把利刃。但是使用的时候仍需要注意一些东西,以免伤到自己。  