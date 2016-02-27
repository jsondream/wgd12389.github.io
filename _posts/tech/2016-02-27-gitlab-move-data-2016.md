---
layout: post
title: gitlab迁移数据
category: 技术
tags: [tech]
keywords: gitlab,linux权限
description: gitlab迁移数据和服务器遇到的坑
---
   
### 迁移服务器  

1. 镜像系统盘  
   1.1 在腾讯云控制台把原来的gitlab服务器关机  
   1.2 在更多按钮中选择制作镜像  
2. 更换服务器  
	2.1 利用镜像好的文件来吧原来的gitlab全部转移到新的云服务器     
3. 挂载云硬盘到数据盘(具体方法自行查找)   

### 修改仓库(repositories)的位置  

1. 新建新仓库目录  
	`mkdir -p /mnt/application/gitlab/git-data`  
2. 修改配置文件 sudo vi /etc/gitlab/gitlab.rb   
	` git_data_dir "/mnt/application/gitlab/git-data"`      
	如果`git_data_dir`前面带有`#`号表示这个属性为被注释的,把前面的`#`号去掉就好了   
3. 更新gitlab配置   
	`sudo gitlab-ctl reconfigure `  
	
### 迁移旧数据到新仓库位置  
*这时候你就可以访问新的gitlab的网站了,但是你会发现所有的仓库都是空的,这是因为原来的git仓库的数据都没迁移过来,下面我们就要迁移旧仓库的数据到新建的仓库位置下了*  

1. 找到旧仓库的位置,一般都会在这个位置下    
	`/var/opt/gitlab/git-data/`  
2. 用`cp`命令复制旧仓库下的数据到新的仓库下   
3. 这时候重新打开git的网站(记得关闭之后再打开,不然浏览器会有缓存的),就可以看到旧仓库的数据导入到新的仓库下了   

### 修改本地的仓库配置   
1. 找到迁移的项目的本地项目的路径下   
2. 输入`git remote -v `,查看现在的本地仓库所对应的地址   
3. 修改git根目录下`.git`文件下的config文件   
4. 修改`url`参数的值为新的仓库地址   
5. 提交一个代码测试下,卧槽,报错了？什么错误,让我们来看一看  
 
	```   
    remo@remo:/qualcomm/jenkins/r1528_ap/oe-core$ git push  
    Counting objects: 10, done.
    Delta compression using up to 4 threads.
    Compressing objects: 100% (5/5), done.
    Writing objects: 100% (8/8), 676 bytes, done.
    Total 8 (delta 3), reused 0 (delta 0)
    error: insufficient permission for adding an object to repository database ./objects
    fatal: failed to write object
    error: unpack failed: unpack-objects abnormal exit
    To ssh://repogitgerrit@192.168.0.198/home/repogitgerrit/repositories/9x25oecore.git
    ! [remote rejected] master -> master (n/a (unpacker error))
    error: failed to push some refs to 'ssh://repogitgerrit@192.168.0.198/home/repogitgerrit/repositories/9x25oecore.git'
    remo@remo:/qualcomm/jenkins/r1528_ap/oe-core$ ls  
	```  
	
### 解决错误    
现在的错误就是因为linux的用户组权限问题而造成的,因为我们迁移到的云数据盘,git的用户组在云硬盘上没有权限而造成的  
怎么解决我们的问题呢?  
1. 进到你服务器的git仓库下  
	我们刚才将`mnt/application/gitlab/git-data`设置为我们现在的仓库位置了  
2. 执行如下命令   
输入`vim /etc/group`查看用户组  
`chgrp -R git .`  git代表的是用户组为git的组  
`chmod -R g+rwX .`赋给用户组读写可执行的权限        
`find . -type d -exec chmod g+s '{}' +`	  
如果刚才的操作没生效的话,再输入`git config core.sharedRepository`或者`git config core.sharedRepository group`	 
 
3. 提交代码进行测试,ok了  
 
4. [参考链接1](http://stackoverflow.com/questions/6448242/git-push-error-insufficient-permission-for-adding-an-object-to-repository-datab)        [参考链接2](https://github.com/deis/deis/issues/1843)
