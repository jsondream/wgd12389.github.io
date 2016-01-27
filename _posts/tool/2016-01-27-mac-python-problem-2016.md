---
layout: post
title: 那些年,我们用mac遇到的梗
category: 工具
tags: [tool]
keywords: mac,sip,python
description: python安装中遇到的梗
---
   
## scrapy的安装  
### 场景描述  
最近公司业务不是很忙,想从网上找点资料出来,用来.....(你懂得,,,别想歪,我是正经人!)  
由于本人是做java的,之前一直在用jsoup来玩爬虫,听说python的scrapy爬虫简直就是搜易贼(so easy)。  
哪就走起吧.  
### 配置状况  
我用的mac版本是OS X EI capitan。  
### 问题描述  
本机自带了python2.7，直接安装scrapy就可以了，
不过在安装scrapy之前要先确定你的电脑是否已经安装了pip。  
如果没安装pip的话,打开终端(我用的itrem2)，执行以下的命令  
`sudo easy_install pip`  
pip 和 easy_install 都是 Python 的框架管理命令，pip 是对 easy_install的升级。  
安装完`pip`之后我们要开始安装scrapy了,打开终端执行   

>`sudo pip install Scrapy`   

如果执行成功，那么 Scrapy 就安装成功了，但往往事与愿违，你很有可能遇到如下错误：   

>OSError: [Errno 1] Operation not permitted: '/tmp/pip-Tz8iWw-uninstall/System/Library/Frameworks/Python.framework/Versions/2.7/Extras/lib/python/six-1.4.1-py2.7.egg-info'  

### 故障定位  
我google了好久,查了好多原因,试了很多种办法发现都没说道点子上,最后在以为大神的博客里找到了原因   
 
>Because six ships with the system, and almost every popular python project uses it for forwards compatibility, pip tries to upgrade the version it finds first in the python path. Since SIP blocks this, it fails.
Any python dependencies system software has should be hard-coded, and the default path should look in /Library/Python/2.7/site-packages first in order.  
  
[原文传送门](http://www.openradar.me/radar?id=6192110889861120)  
这时候,我们知道了新版的mac系统增加了sip特性,即使使用 sudo 也无法使获得最高权限，无法对 MAC 系统级的目录进行更改  
### 解决问题  
既然我们已经发现问题出现在sip上了,那我们把sip特性关闭了不就完了么,那么我们怎么关闭sip特性呢。  

* 重启 MAC ，在重启的过程中按住 Command+R,进入安全模式
* 在顶部的菜单栏中打开终端 ，输入 csrutil disable 命令关闭 SIP 安全特性(想要在开启sip的话就用csrutil enable命令即可)
* 重启 MAC就ok了  

此时,sip特性已经被我们关闭了,你可以重新安装scrapy试试,打开终端执行   

>`sudo pip install Scrapy`   

在短暂的安装过程等待过后,你原本期望的是安装成功的提示,但是你发现安装又tm的失败了,fuck。  
这时候又发送什么原因了呢,看来下控制台,你发现了如下的错误  
`Scrapy throws ImportError: cannot import name xmlrpc_client`  
这货又是什么梗？  
于是乎,又google了下,发现是six的版本太低了[原文传送门](http://stackoverflow.com/questions/30964836/scrapy-throws-importerror-cannot-import-name-xmlrpc-client)  
那我们更新下six的版本吧,打开控制台输入以下的命令  
 
```
sudo rm -rf /Library/Python/2.7/site-packages/six*  
sudo rm -rf /System/Library/Frameworks/Python.framework/Versions/2.7/Extras/lib/python/six*  
sudo pip install six
```    

ok,我们把six的版本也更新完了,哪这时候我们再试下安装scrapy把.    
这时候会提示你`installation successful`,那就恭喜你成功的解决了sip,并安装了scrapy.  

## python3的安装   
### 需求描述   
大家应该都知道MAC OS X EI Capitan 系统 支持 python的多版本共存,即在我们的环境变量中可以配置python2和python3。  
### 安装过程  
1. 先安装python3,如果你安装了[homebrew](http://brew.sh/),那么你只需要输入一条命令`brew install python3`即可安装python3     
2. 此时你可以输入python3试试,但是你发现这时候系统会提示命令找不到(你都没配置python3,系统找个毛啊)    
3. 这时候我们进行配置了,首先把安装好的 Python3 目录移到原本系统所持有的目录位置,在终端输入以下命令:   
`sudo mv /usr/local/Cellar/python3/3.5.0/Frameworks/Python.framework/Versions/3.5 /System/Library/Frameworks/Python.framework/Versions`   
注意:`/usr/local/Cellar/python3/3.5.0/Frameworks/Python.framework/Versions/3.5`是你的python3的安装路径,一般用homebrew安装的会是这个路径,如果你从官网下载安装的不一定是这个路径,具体路径请参考你的python3的安装路径  
4. 然后修改文件所属的 Group 设置 Group 为 wheel,在终端内输入以下命令`sudo chown -R root:wheel /usr/local/Cellar/python3/3.5.0/Frameworks/Python.framework/Versions/3.5`      
 
5. 重新链接可执行文件	

```
 sudo ln -s /usr/local/Cellar/python3/3.5.0/Frameworks/Python.framework/Versions/3.5/bin/pydoc3.5 /usr/bin/pydoc3  
 sudo ln -s //usr/local/Cellar/python3/3.5.0/Frameworks/Python.framework/Versions/3.5/bin/python3.5 /usr/bin/python3
 sudo ln -s /usr/local/Cellar/python3/3.5.0/Frameworks/Python.framework/Versions/3.5/bin/pythonw3.5 /usr/bin/pythonw3
 sudo ln -s /usr/local/Cellar/python3/3.5.0/Frameworks/Python.framework/Versions/3.5/bin/python3.5m-config /usr/bin/python3-config  
```  

### 查看结果  
在终端里输入 python3 ,会出现如下提示,说明配置成功： 

```
Python 3.5.0 (default, Sep 23 2015, 04:41:38)
[GCC 4.2.1 Compatible Apple LLVM 7.0.0   
(clang-700.0.72)] on darwin  
Type "help", "copyright", "credits" or 
"license" for more information.
```






