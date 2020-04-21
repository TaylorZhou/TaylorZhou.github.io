---
layout: "post"
title: "Hive hive-jdbc 排错"
date: "2020-04-21 19:15"
categories: Hadoop
description: "Hive hive-jdbc 排错"
tags: Hadoop MapReduce
---

* content
{:toc}

<div class="postImg" style="background-image:url(https://github.com/TaylorZhou/TaylorZhou.github.io/blob/master/assets/blog-image/2020-04-21-1.png?raw=true)"></div>
> “本文记载笔者运行 MapReduce 出错后，如何一步步找出问题及最终解决问题”




## 1. 抛出问题 
我尝试使用hive-jdbc在程序种，远程操纵hive，该操作抛出异常：
```log
2020-04-21 17:04:59,937 ERROR (DirectJDKLog.java:182): http-nio-8288-exec-8 Servlet.service() for servlet [dispatcherServlet] in context with path [/api/v1] threw exception [Handler dispatch failed; nested exception is java.lang.NoClassDefFoundError: org/apache/hive/service/cli/thrift/TCLIService$Iface] with root cause
java.lang.ClassNotFoundException: org.apache.hive.service.cli.thrift.TCLIService$Iface
        at java.net.URLClassLoader.findClass(URLClassLoader.java:382)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:424)
        at org.springframework.boot.loader.LaunchedURLClassLoader.loadClass(LaunchedURLClassLoader.java:93)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
        at org.apache.hive.jdbc.HiveDriver.connect(HiveDriver.java:105)
        at java.sql.DriverManager.getConnection(DriverManager.java:664)
        at java.sql.DriverManager.getConnection(DriverManager.java:247)
```

根据错误定位到HiveDriver类105行的位置  

```java
  public Connection connect(String url, Properties info) throws SQLException {
    return acceptsURL(url) ? new HiveConnection(url, info) : null;
  }
```

发现在HiveConnection中有调用TCLIService.Iface接口,定位到254行的位置  

```java
TCLIService.Iface client = new TCLIService.Client(new TBinaryProtocol(transport));
```

在这里抛了找不到Iface的异常，但是点进去TCLIService类中，确是存在Iface接口，所以说找不到不是因为不存在，而是出现重复的类。  

所以去看下引入的依赖包，如下：
```xml
<dependency>
	<groupId>org.apache.hive</groupId>
	<artifactId>hive-service</artifactId>
	<version>1.1.0-cdh5.4.7</version>
	<exclusions>
		<exclusion>
			<groupId>org.eclipse.jetty.aggregate</groupId>
			<artifactId>*</artifactId>
		</exclusion>
		<exclusion>
			<groupId>org.slf4j</groupId>
			<artifactId>*</artifactId>
		</exclusion>
		<exclusion>
			<groupId>org.apache.thrift</groupId>
			<artifactId>*</artifactId>
		</exclusion>
		<exclusion>
			<groupId>org.apache.hive</groupId>
			<artifactId>hive-exec</artifactId>
		</exclusion>
	</exclusions>
</dependency>
        
<dependency>
    <groupId>org.apache.hive</groupId>
    <artifactId>hive-jdbc</artifactId>
    <version1.1.0-cdh5.4.7</version>
    <exclusions>
        <exclusion>
            <groupId>org.slf4j</groupId>
            <artifactId>*</artifactId>
        </exclusion>
    </exclusions>
</dependency>      

```

发现有hive-jdbc中同样存在hive-service，所以在这里会抛异常，我就先把hive-service从hive-jdbc中移除掉，最终解决问题






