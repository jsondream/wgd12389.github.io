---
layout: post
title: java8利用lambda表达式来封装jedis的连接池资源
category: 技术
tags: [tech]
keywords: lambda,java8,jedis,pool
description: java8利用lambda表达式来封装jedis的连接池资源
---


### lambda表达式  

***什么是lambda？***   
> “Lambda 表达式”(lambda expression)是一个匿名函数，Lambda表达式基于数学中的λ演算得名，直接对应于其中的lambda抽象(lambda abstraction)，是一个匿名函数，即没有函数名的函数。Lambda表达式可以表示闭包（注意和数学传统意义上的不同）。  

### 需求点  

使用过redis的java客户端框架jedis的朋友都知道，jedis里有个连接池的概念。然而从jedis的连接池里获取连接资源要经过以下的步骤:   

> 1. 获取Jedis实例需要从JedisPool中获取；  
> 2. 用完Jedis实例需要返还给JedisPool；  
> 3. 如果Jedis在使用过程中出错，则也需要还给JedisPool；  

注意：***这里面我们的jedis并不能释放使用完成的资源***   

那么我们怎么利用lambda表达式来达到只写业务代码，吧自动获取资源和释放资源来交给上层封装的代码来处理呢？  

### 核心代码封装  

我们知道关于jedis的操作这一块，我们的过程是这样的三步：  

1. 获取jedis连接  
2. 业务操作  
3. 释放资源  

其中1，3是固定的操作，只有2是一个不同类型的抽象动作   
那么我们首先要吧我们的业务封装成一个接口，至于为什么是一个接口呢，那就是因为lambda表达式意味着我们可以在接口做参数的方法部分直接写实现类的代码   

```java
public interface RedisDomainInterface <T> {
    public T domain(Jedis jedis);
}
```  

然后我们需要实现我们的1，3部分的封装，那么我们这部分的代码实现是这样的  

```java

public class RedisClient {

    public static <T extends Object> T domain(RedisDomainInterface<T> interfaces) {
        // 返回值
        T Object;
        // 获取连接池里的连接
        Jedis jedis = RedisPoolClient.getInstance().getJedis();
        try {
            // 业务操作
            Object = interfaces.domain(jedis);
        } finally {
            // 释放链接
            RedisPoolClient.getInstance().returnResource(jedis);
        }
        return Object;
    }
}
```  

那么我们封装好了之后应该怎么用呢?  
比如我们这时候需要给一个userName为xxx的人设置他的标签，那么我们的代码是这个样子的   


```java
    public static String setTag(String userName ,String tag) {
        return RedisClient.domain(jedis -> jedis.set(userName,tag));
    }
```  

简洁了很多，有木有，这下我们只需要关注业务代码就可以了  
