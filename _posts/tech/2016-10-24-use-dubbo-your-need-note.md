---
layout: post
title: 使用dubbo你所需要注意的
category: 技术
tags: [tech]
keywords: 分布式,dubbo
description: 本文介绍了使用dubbo你所需要注意的事项
---


## 采用注解方式注入消费者接口实力空指针   

注解的方式在现在的项目中由于他的简洁性越来越被大众所喜欢,在我们集成dubbox的时候,发现dubbox支持了注解方式,但是在我们在用注解式集成的时候,发现消费者的对象在没有注入进去,一直都是报空指针异常.   

代码如下:   

```java
/**
 * <p>
 * bug反馈业务接口
 * </p>
 *
 * @author wangguangdong
 * @version 1.0
 * @Date 2016年10月12日16:18:30
 */
@AuthAnnotation
@Controller
public class BugReportResource {

    private static final Logger LOG = Logger.getLogger(BugReportResource.class);
    @Reference(version = "1.0.0",interfaceClass = BugReportService.class)
    BugReportService bugReportService;
}
```    

经过楼主辛苦的查找原因,后来终于找到问题所在:     
在spirng进行实例扫描的时候根本无法识别dubbo中的注解@Reference ,同时,在dubbo扫描的时候也无法识别Spring @Controller ,所以两个框架的扫描顺序要排列好,如果先扫了controller,这时候把控制器都实例化好了,再扫dubbo的服务,就会出现空指针,因为在实例化的时候是没有对应的消费者的实例的,所以就会造成无法注入,这也就是为什么在我们调用消费者服务的时候会造成空指针.   

下面是楼主编排成功的代码.   

```xml
	<mvc:annotation-driven />
	
	<!-- 查找xxx路径下所有@Controller 注释类,添加与项目相关的controller -->
	
	<dubbo:annotation package="XXX.XXX.XXX.controller" />
	
	<context:component-scan base-package="XXX.XXX.XXX.controller"/>
```  

然后在查阅资料之后,楼主又发现了另一种解决办法:    
在一个spring对象中注入dubbo消费者实例,然后在controller中注入这个服务实例即可,这种方法不受dubbo和spirng扫描顺序的影响.其实在项目中我们可能也会有这样的设计(有些的架构改进会进行这样的设计,比如我吧所有的服务细粒度化拆分,并作为提供者注册给dubbo的server,然后我在消费者端多架构一个组合服务层(业务编排service层),进行dubbo子服务的组合,再讲组合后的服务注入到controller中供业务侧使用)    

```java
	@Component
	public class DubboSupport
	{
	    @Reference(version = "1.0.0",interfaceClass = BugReportService.class)
    	 BugReportService bugReportService;	    
	    public BugReportService getBugReportService(){
	        return bugReportService;
	    }
	}
```  

## Dubbo超时和重连   

dubbo启动时默认有重试机制和超时机制,某些业务场景下,如果不注意配置超时和重试,可能会引起一些异常。    

> 超时机制的规则是如果在一定的时间内,provider没有返回，则认为本次调用失败    
> 
> 重试机制在出现调用失败时，会再次调用。如果在配置的调用次数内都失败，则认为此次请求异常，抛出异常。(dubbo默认重试2次)  

如果出现超时，通常是业务处理太慢或者发送io阻塞,可在服务提供方执行：jstack PID > jstack.log 分析线程都卡在哪个方法调用上，这里就是慢的原因。如果这个服务接口不能调优性能，请将timeout设大。

#### 超时设置   

DUBBO消费端设置超时时间需要根据业务实际情况来设定，如果设置的时间太短，一些复杂业务需要很长时间完成，导致在设定的超时时间内无法完成正常的业务处理。这样消费端达到超时时间，那么dubbo会进行重试机制，不合理的重试在一些特殊的业务场景下可能会引发很多问题，需要合理设置接口超时时间。   

比如发送邮件，可能就会发出多份重复邮件，执行注册请求时，就会插入多条重复的注册数据。    

**（1）合理配置超时和重连的思路**    

1. 对于核心的服务中心，去除dubbo超时重试机制，并重新评估设置超时时间。  
2. 业务处理代码必须放在服务端，客户端只做参数验证和服务调用，不涉及业务流程处理   

**（2）Dubbo超时和重连配置示例**   

```
	<!-- 服务调用超时设置为6秒,超时不重试--> 
	<dubbo:service interface="com.provider.service.DemoService" ref="demoService"  retries="0" timeout="5000"/>
```  

**（3）Dubbo消费者端统一的超时和重连配置**   

```
	<!--统一的消费者配置-->
    <dubbo:consumer timeout="30000" retries="0" version="1.0.0"/>
```
#### 重连机制   

dubbo在调用服务不成功时，默认会重试2次。Dubbo的路由机制，会把超时的请求路由到其他机器上，而不是本机尝试，所以 dubbo的重试机器也能一定程度的保证服务的质量。但是如果不合理的配置重试次数，当失败时会进行重试多次，这样在某个时间点出现性能问题，调用方再连续重复调用，系统请求变为正常值的retries倍，系统压力会大增，容易引起服务雪崩，需要根据业务情况规划好如何进行异常处理，何时进行重试。   

**在重试发送的时候也可能会出现这样的问题:**    
比如有一个bug反馈,但是因为数据库io瓶颈,这时候这个服务阻塞了,然后过了一会查看数据库里有3条除了id外剩下都一样的数据(id是在服务提供者里生成的,这里只做异常例子举例).   

这就是重试机制下,业务不合理的设计所造成的坑,这时候我们处理的方式有两种:   

1. 合理规划业务(例如id放在服务上游生成,数据库主键唯一的机制)    
2. 吧服务增加幂等性设置(例如接口中增加消息id)   


## 附带上使用dubbo你所需要的mavne依赖  

```xml
		<dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-expression</artifactId>
            <version>4.2.5.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>dubbo</artifactId>
            <version>2.8.4</version>
            <exclusions>
                <exclusion>
                    <groupId>org.springframework</groupId>
                    <artifactId>spring</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>com.101tec</groupId>
                    <artifactId>zkclient</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.apache.zookeeper</groupId>
            <artifactId>zookeeper</artifactId>
            <version>3.4.6</version>
        </dependency>
        <dependency>
            <groupId>com.101tec</groupId>
            <artifactId>zkclient</artifactId>
            <version>0.7</version>
        </dependency>
```
 

