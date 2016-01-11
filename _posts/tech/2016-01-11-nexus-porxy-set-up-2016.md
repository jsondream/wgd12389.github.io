---
layout: post
title: nexus在代理仓库的安装
category: 技术
tags: [technology]
keywords: nexus,mac,porxy
description: redis在mac上的安装和使用
---

1.设置你的代理仓库  
1.1.进入到~/.m2文件夹下，查看`settings.xml`文件(`如果m2文件不存在的话,在你的项目中执行一次maven命令的操作即可创建m2文件`)  
1.1.1如果`settings.xml`文件存在  
将如下的标签和值合入到`settings.xml`  
```xml
   <profiles>
        <profile>
          <id>dev</id>
          <repositories>
             <repository>
            <id>nexus</id>
            <url>http://yourIp:8081/nexus/content/repositories/releases/</url>
            <releases>
                <enabled>true</enabled>
                </releases>
            <snapshots>
             <enabled>true</enabled>
            </snapshots>
             </repository>
          </repositories>

         <pluginRepositories>
                <pluginRepository>
                    <id>nexus</id>
                    <url>http://yourIp:8081/nexus/content/repositories/releases/</url>
                    <releases>
                        <enabled>true</enabled>
                    </releases>
                    <snapshots>
                        <enabled>true</enabled>
                    </snapshots>
                </pluginRepository>
            </pluginRepositories>

        </profile>
    </profiles>

    <activeProfiles>
        <activeProfile>dev</activeProfile>
    </activeProfiles>

    <servers>
       <server>
          <id>releases</id>
          <username>admin</username>
          <password>admin123</password>
       </server>
    	<server>
              <id>snapshots</id>
              <username>admin</username>
              <password>admin123</password>
           </server>
    	<server>
          <id>deploymentRepo</id>
          <username>admin</username>
          <password>admin123</password>
       </server>
    </servers>
```

1.1.2如果`settings.xml`文件不存在则创建该文件  
在该文件中建立  
```xml
<settings>
</settings>
```  
在`settings`标签中写入1.1.1的值  
2.进入到你的maven安装目录下的conf文件夹下  
2.1.修改你的`settings.xml`文件  
在你的`servers`标签下加入如下的内容  
```xml
       <server>
          <id>releases</id>
          <username>admin</username>
          <password>admin123</password>
       </server>
    	<server>
              <id>snapshots</id>
              <username>admin</username>
              <password>admin123</password>
           </server>
    	<server>
          <id>deploymentRepo</id>
          <username>admin</username>
          <password>admin123</password>
       </server>
```  
3.查看是否创建成功  
执行`mvn clean package install`命令  
查看控制台中下载jar包的地址知否变为带有
`http://yourip:8081/nexus/content/repositories/releases/`地址的值的输出
如果带有该地址的下载,则说明下载成功  
如果下载的jar依然使用maven的中央仓库,请联系<a href="mailto:jsondream@gmail.com">王光东</a>



