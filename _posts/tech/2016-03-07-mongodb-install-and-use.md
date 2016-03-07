---
layout: post
title: mongodb的安装和使用
category: 技术
tags: [tech]
keywords: mongodb
description: mongodb的安装和使用
---
# mongodb的安装和使用  

### 在mac上的安装  
使用homebrew来为你的机器架设mongodb  
键入如下命令`brew install mongodb`  
### 配置mongod为全局命令  
编辑你的配置文件,如bash下输入以下命令  
`vi ~/.bash_profile`  
加入如下一行：  
`export PATH="你的mongodb的安装路径/bin:$PATH"`  
保存之后，运行：  
`source ~/.bash_profile`  
### 建立存储空间
做了上述操作之后,就可以直接在任何地方启动 MongoDB 或者进行相关的数据库操作了，使用 mongod 命名启动服务器。  
但是这时候我们却发现一切并没有我们想的那么顺利,出现了如下的错误  
 
```
2016-03-07T14:53:19.382+0800 I STORAGE  [initandlisten] exception in initAndListen: 29 Data directory /data/db not found., terminating  
2016-03-07T14:53:19.383+0800 I CONTROL  [initandlisten] dbexit:  rc: 100
```
现上面这个错误是因为 /data/db 目录不存在，若启动时，不指定任何参数， MongoDB 会默认使用 /data/db 目录存储数据，我们可以使用 --dbpath 来指定其它的路径  
这时候我们就要建议我们的mongodb的存储空间了  
`sudo mkdir /Users/jsondream/mongodb_data`  
然后使用的下面这样的命令启动  
`mongod --dbpath /Users/jsondream/mongodb_data`  
然后就发现我们的mongodb已经成功的启动了  
### mac下的mongodb的客户端管理工具  
 
```
Robomongo 是一个基于 Shell 的跨平台开源 MongoDB 管理工具。
嵌入了JavaScript引擎和 MongoDB mogo 。只要你会使用 mongo
shell ，你就会使用 Robomongo。提供语法高亮、自动完成、差别
视图等。
```  

[下载地址](http://robomongo.org/)  
安装好后建立连接即可使用  
### MongoDB的命令行客户端MongoDB Shell  
MongoDB 自带有 JavaScript Shell ，可以Shell 中使用命令行与 MongoDB 实例交互，Shell非常有用，通过它可以执行管理操作，检查运行实例，亦或是做其它  
#### 运行 shell  
使用 mongo 命令启动 shell：  

```
➜  mongodb_data mongo
MongoDB shell version: 3.0.7
connecting to: test
Welcome to the MongoDB shell.
For interactive help, type "help".
For more comprehensive documentation, see
	http://docs.mongodb.org/
Questions? Try the support group
	http://groups.google.com/group/mongodb-user
Server has startup warnings:
2016-03-07T14:56:37.395+0800 I CONTROL  [initandlisten] ** WARNING: You are running this process as the root user, which is not recommended.
2016-03-07T14:56:37.395+0800 I CONTROL  [initandlisten]
```  
#### 简单的mongo命令  
* 查看当前所在的库  

```
> db
test
```  
test为当前所在的数据库  

* 创建一个集合(相当于传统数据库中的表)  
通过 db.createCollection() 函数可以先创建一个集合：  

```
> db.createCollection("data")
{ "ok" : 1 }
```  

*  插入数据  
insert 可以将一个文档添加到集合中：  
先声明一个数据   

```
> data ={"name":"mongo"}
{
    "name":"mongo"
}
```  
调用`insert`命令插入数据  

```
> db.blog.insert(data)
WriteResult({ "nInserted" : 1 })
```  

*  读取数据  
`find` 与 `findOne` 方法可以用于查询集合里的文档：  
 
```
> db.blog.findOne()
{
    "_id" : ObjectId("55a5b574b705414e188e218a"),
    "name":"mongo"
}
```  

* 更新数据  
使用 `update` 修改数据，它至少接受两个参数，第一个是限定条件，第二个是新的对象。  
假设我们现在要给 data 增加list这个属性：  

```
> data.list = []
[ ]
```  
然后去更新我们的数据   

```
> db.data.update({"name":"mongo"},data)
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
```  

再一次查询我们的data数据  

```
> db.blog.findOne()
{
    "_id" : ObjectId("55a5b574b705414e188e218a"),
    "name":"mongo",
    "list":[]
}
```  

* 删除数据  
使用 remove 可以删除集合中的文档，若没有任何限定参数，它将删除集合中的所有数据。  
也可以像下面这样，删除一个名字叫`mongo`的数据  

```
> db.blog.remove({"name":"mongo"})
WriteResult({ "nRemoved" : 1 })
> db.blog.find()
> 
```  

### 引入mongo的pom依赖  

```
        <dependency>
            <groupId>org.mongodb</groupId>
            <artifactId>mongo-java-driver</artifactId>
            <version>3.0.4</version>
        </dependency>
```
