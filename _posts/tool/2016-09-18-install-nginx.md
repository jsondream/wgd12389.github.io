---
layout: post
title: 那些年,我们用mac遇到的梗
category: 工具
tags: [tool]
keywords: mac,sip,python
description: python安装中遇到的梗
---
   
## nginx的安装   

### 安装步骤   

1. 从http://nginx.org/download/上下载相应的版本(或者wget http://nginx.org/download/nginx-1.5.9.tar.gz直接在Linux上用命令下载)   
2. 解压 tar -zxvf nginx-1.5.9.tar.gz    
3. 设置一下配置信息 ./configure --prefix=/usr/local/nginx ，或者不执行此步，直接默认配置   
4. make 编译 （make的过程是把各种语言写的源码文件，变成可执行文件和各种库文件）    
5. make install 安装 （make install是把这些编译出来的可执行文件和库文件复制到合适的地方）    

### 遇到的错误问题   

#### 错误1：   

错误提示：./configure: error: the HTTP cache module requires md5 functions
from OpenSSL library.   You can either disable the module by using
--without-http-cache option, or install the OpenSSL library into the system,
or build the OpenSSL library statically from the source with nginx by using
--with-http_ssl_module --with-openssl=<path> options.

解决办法：`yum -y install pcre-devel`  

#### 错误2：  

错误提示：./configure: error: the HTTP cache module requires md5 functions
from OpenSSL library.   You can either disable the module by using
--without-http-write option, or install the OpenSSL library into the system,
or build the OpenSSL library statically from the source with nginx by using
--with-http_ssl_module --with-openssl=<path> options.

解决办法：`yum -y install zlib-devel`


