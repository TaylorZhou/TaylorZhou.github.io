---
layout: "post"
title: "Hadoop3 MapReduce 排错"
date: "2020-02-20 19:15"
categories: Hadoop
description: "Hadoop3 MapReduce 排错"
tags: Hadoop MapReduce
---

* content
{:toc}

<div class="postImg" style="background-image:url(https://github.com/TaylorZhou/TaylorZhou.github.io/blob/master/assets/blog-image/2020-02-20-1.png?raw=true)"></div>
> “本文记载笔者运行 MapReduce 出错后，如何一步步找出问题及最终解决问题”




## 1. 抛出问题  
我尝试在新搭建的 Hadoop3 集群上提交简单的 wordcount 作为 MapReduce job。

mapred-site.xml 配置信息如下：
```xml
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
```
该 job 卡住，未能成功运行，输出如下：
```log
2020-02-20 18:58:47,089 INFO client.RMProxy: Connecting to ResourceManager at centos1/192.168.1.6:8032
2020-02-20 18:58:48,225 INFO mapreduce.JobResourceUploader: Disabling Erasure Coding for path: /tmp/hadoop-yarn/staging/hdfs/.staging/job_1582196081257_0002
2020-02-20 18:58:48,383 INFO sasl.SaslDataTransferClient: SASL encryption trust check: localHostTrusted = false, remoteHostTrusted = false
2020-02-20 18:58:48,632 INFO input.FileInputFormat: Total input files to process : 1
2020-02-20 18:58:48,750 INFO sasl.SaslDataTransferClient: SASL encryption trust check: localHostTrusted = false, remoteHostTrusted = false
2020-02-20 18:58:48,800 INFO sasl.SaslDataTransferClient: SASL encryption trust check: localHostTrusted = false, remoteHostTrusted = false
2020-02-20 18:58:48,829 INFO mapreduce.JobSubmitter: number of splits:1
2020-02-20 18:58:49,140 INFO sasl.SaslDataTransferClient: SASL encryption trust check: localHostTrusted = false, remoteHostTrusted = false
2020-02-20 18:58:49,181 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1582196081257_0002
2020-02-20 18:58:49,182 INFO mapreduce.JobSubmitter: Executing with tokens: []
2020-02-20 18:58:49,504 INFO conf.Configuration: resource-types.xml not found
2020-02-20 18:58:49,505 INFO resource.ResourceUtils: Unable to find 'resource-types.xml'.
2020-02-20 18:58:49,625 INFO impl.YarnClientImpl: Submitted application application_1582196081257_0002
2020-02-20 18:58:49,709 INFO mapreduce.Job: The url to track the job: http://centos1:8088/proxy/application_1582196081257_0002/
2020-02-20 18:58:49,709 INFO mapreduce.Job: Running job: job_1582196081257_0002
```

## 2. 解决步骤

检查集群中 ResourceManager 的状态，如果 NM 节点磁盘空间不足， RM 会标记 NM 节点为 "unhealthy" ，导致 NM 节点无法分配 containers。

1. 检查故障节点：http://<active_RM>:8088/cluster/nodes/unhealthy  
错误信息为：local-dirs have errors: [ /data/hadoop/tmp/nm-local-dir : Cannot create directory，笔者以hdfs用户创建了 local-dirs 目录，重新运行 job，依旧卡住，刷新 页面，显示错误信息为： local-dirs have errors: [ /data/hadoop/tmp/nm-local-dir : Directory is not writable: /data/hadoop/tmp/nm-local-dir ，显示没有权限，使用 chmod 赋予权限，再次运行 job，依旧卡住，刷新页面，无错误信息。

2. job的日志显示错误信息如下：


```log
2020-02-20 18:08:32,119 INFO mapreduce.Job: Job job_1582192974317_0003 failed with state FAILED due to: Application application_1582192974317_0003 failed 2 times due to AM Container for appattempt_1582192974317_0003_000002 exited with  exitCode: 1
Failing this attempt.Diagnostics: [2020-02-20 18:08:31.515]Exception from container-launch.
Container id: container_1582192974317_0003_02_000001
Exit code: 1

[2020-02-20 18:08:31.522]Container exited with a non-zero exit code 1. Error file: prelaunch.err.
Last 4096 bytes of prelaunch.err :
Last 4096 bytes of stderr :
Error: Could not find or load main class org.apache.hadoop.mapreduce.v2.app.MRAppMaster

Please check whether your etc/hadoop/mapred-site.xml contains the below configuration:
<property>
  <name>yarn.app.mapreduce.am.env</name>
  <value>HADOOP_MAPRED_HOME=${full path of your hadoop distribution directory}</value>
</property>
<property>
  <name>mapreduce.map.env</name>
  <value>HADOOP_MAPRED_HOME=${full path of your hadoop distribution directory}</value>
</property>
<property>
  <name>mapreduce.reduce.env</name>
  <value>HADOOP_MAPRED_HOME=${full path of your hadoop distribution directory}</value>
</property>
```

修改mapred-site.xml，添加如下配置：

```xml
<property>
  <name>yarn.app.mapreduce.am.env</name>
  <value>HADOOP_MAPRED_HOME=${full path of your hadoop distribution directory}</value>
</property>
<property>
  <name>mapreduce.map.env</name>
  <value>HADOOP_MAPRED_HOME=${full path of your hadoop distribution directory}</value>
</property>
<property>
  <name>mapreduce.reduce.env</name>
  <value>HADOOP_MAPRED_HOME=${full path of your hadoop distribution directory}</value>
</property>

```

再次提交job，成功运行。





