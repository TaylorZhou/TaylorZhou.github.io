---
layout: "post"
title: "Ranger集成Solr"
date: "2020-02-18 11:15"
categories: Hadoop
description: "Ranger集成Solr"
tags: Ranger Solr
---

* content
{:toc}

<div class="postImg" style="background-image:url(https://github.com/TaylorZhou/TaylorZhou.github.io/blob/master/assets/blog-image/2020-ranger-solr.png?raw=true)"></div>
> “Apache Ranger使用Apache Solr存储审核日志，并提供通过审核日志进行的UI搜索。在安装RangerAdmin或任何Ranger组件插件之前，必须先安装和配置Solr。”




安装模式：
1. Solr - Standalone  
    Solr 单节点易于安装，且对 Zookeeper 无依赖。如果是测试 Ranger或者非生产环境中，推荐该安装模式。  
2. SolrCloud  
    这是Ranger的首选设置。 SolrCloud是可伸缩的体系结构，可以运行单节点或多节点群集。它具有复制和分片等其他功能，这些功能对于高可用性（HA）和可伸缩性很有用。您需要根据群集大小计划部署。  
3. Self Install  
    如果安装了自己的Solr，则可以使用Ranger提供的schema.xml和托管模式在设置中创建集合

## 1. 有用的链接 

在大规模环境中配置Apache Solr可能具有挑战性。如果您期望大量的审核日志，请查看以下链接：  

* [https://risdenk.github.io/2017/12/18/ambari-infra-solr-ranger.html](https://risdenk.github.io/2017/12/18/ambari-infra-solr-ranger.html)
* [https://community.hortonworks.com/articles/63853/solr-ttl-auto-purging-solr-documents-ranger-audits.html](https://community.hortonworks.com/articles/63853/solr-ttl-auto-purging-solr-documents-ranger-audits.html)

**请注意，如果您使用的是Apache Ambari，则Ambari会维护它自己的solr-config模板。因此，请确保还更新模板，否则，您的手动更改可能会被覆盖。**

## 2. 前提  
1. JDK 1.7 or above. Apache Solr 5.2 or above
2. Solr占用大量内存和CPU。如果您的生产系统具有大量访问请求，请确保运行Solr的服务器具有足够的内存，CPU和磁盘。
3. 由于审核记录会急剧增长，因此计划在Solr将用于存储索引数据的卷中至少有1 TB的可用空间。
4. Solr与32GB RAM配合良好。计划为Solr进程提供尽可能多的内存
5. 最后，SolrCloud支持复制和分片。强烈建议将SolrCloud与至少两个在不同服务器上运行且已启用复制的Solr节点一起使用。
6. 如果使用SolrCloud，则还需要安装和配置ZooKeeper。

## 3. 安装和配置步骤

1. git clone https://github.com/apache/ranger.git
2. cd security-admin/contrib/solr_for_audit_setup  
3. 编辑install.properties（请参阅后续部分中的说明）
4. ./setup.sh
5. 打开 $SOLR_RANGER_HOME/install_notes.txt 以获取更多说明

## 4. Solr 安装

您可以从 [Apache Solr Downloads](http://lucene.apache.org/solr/downloads.html) 下载Solr软件包。确保Solr版本为5.2或更高。您可能还允许Ranger脚本setup.sh为您自动下载，安装和配置Solr。如果要setup.sh安装Solr，请在install.properties中设置以下属性，然后从下一节中选择一个配置选项。

| Property Name | Sample Values | Description |
|  :--- | :---  | :---    |
| SOLR_INSTALL_FOLDER  | /opt/solr | 您要安装Solr的位置。   |
| SOLR_INSTALL | true | 如果将其设置为true，则setup.sh将下载Solr软件包并进行安装。 |
| SOLR_DOWNLOAD_URL | http://apache.cs.utah.edu/lucene/solr/8.4.1/solr-8.4.1-src.tgz | 下载链接 |

## 5. 配置选项

您可以将Solr配置为 Standalone 或 SolrCloud 运行。如果要setup.sh配置为Standalone，请遵循本节 [Standalone Configuration.](#jump1)。如果要为SolrCloud配置，请遵循本节 [SolrCloud Configuration](#jump2)。如果要配置自己的Solr，请参阅本节 [Self Install](#jump3)。

### <span id="jump1">5.1 Configure Standalone Solr</span>

修改install.properties的以下属性


| Property Name | Sample Values | Description |
|  :--- | :---  | :---    |
| SOLR_MAX_MEM  | 2g | 这是分配给Solr的内存   |
| SOLR_RANGER_PORT | 6083 | 您要Solr侦听的端口 |
| SOLR_INSTALL_FOLDER | /opt/solr | Solr的安装位置 |
| SOLR_RANGER_HOME | /opt/solr/ranger_audit_server | 这是与Ranger相关的配置和架构文件将被复制的位置 |
| SOLR_RANGER_DATA_FOLDER | /opt/solr/ranger_audit_server/data | 这是您要在其中存储索引数据的文件夹 |
| SOLR_LOG_FOLDER | /var/log/solr/ranger_audits | Solr日志文件夹 |
| SOLR_USER | solr | 用于运行Solr的Linux用户 |
| SOLR_DEPLOYMENT | standalone | standalone模式运行 |
| JAVA_HOME | /usr/local/jdk1.8.0_161 | JDK位置 |


使用上述值更新install.properties后，运行以下命令：
```sh
./setup.sh
<logs ...>
########## Done ###################
Created file /opt/solr/ranger_audit_server/install_notes.txt with instructions to start and stop
###################################
```
setup.sh成功返回后，打开文件$ SOLR_RANGER_HOME / install_notes.txt以获取有关启动和停止Solr的说明。  
在启动Solr for RangerAudit之后，Solr将收听$ {SOLR_PORT}。例如，通过从浏览器访问http：// $ {SOLR_HOST}：6083来检查Solr。
### <span id="jump2">5.2 Configure SolrCloud</span>

安装和配置SolrCloud需要几个额外的步骤。我们需要以下几点：
1. 将Ranger Audit配置（包括schema.xml）添加到ZooKeeper
2. 在Solr中创建集合。

首先，将install.properties修改为以下属性：  


| Property Name | Sample Values | Description |
|  :--- | :---  | :---    |
| JAVA_HOME | /usr/local/jdk1.8.0_161 | JDK位置 |
| SOLR_USER | solr | 用于运行Solr的Linux用户 |
| SOLR_INSTALL_FOLDER | /opt/solr | Solr的安装位置 |
| SOLR_RANGER_HOME | /opt/solr/ranger_audit_server | 这是与Ranger相关的配置和架构文件将被复制的位置 |
| SOLR_RANGER_PORT | 6083 | 您要Solr侦听的端口 |
| SOLR_DEPLOYMENT | solrcloud | solrcloud模式运行 |
| SOLR_ZK  | ${zk_host}:2181/ranger_audits | 建议提供子文件夹来创建与Ranger Audit相关的配置  |
| SOLR_SHARDS | 1 | 如果您希望分发审核日志，则可以使用多个分片 |
| SOLR_REPLICATION | 1 | 强烈建议至少设置2个节点并复制索引 |
| SOLR_LOG_FOLDER | /var/log/solr/ranger_audits | Solr日志文件夹 |
| SOLR_MAX_MEM | 2g | 分配给Solr的内存 |

使用上述值更新install.properties后，运行以下命令：
```sh
./setup.sh
<logs ...>
########## Done ###################
Created file /opt/solr/ranger_audit_server/install_notes.txt with instructions to start and stop
###################################
```  
setup.sh成功返回后，打开文件$ SOLR_RANGER_HOME / install_notes.txt以执行其他步骤.  
要配置SolrCloud，您需要执行以下操作：
1. 使用./setup.sh脚本，在所有其他节点上安装和配置Solr for Ranger Audits（尚未启动）
2. 执行 /opt/solr/ranger_audit_server/scripts/add_ranger_audits_conf_to_zk.sh (从安装了solr的任何节点仅一次)
3. 在所有节点启动 Solr： /opt/solr/ranger_audit_server/scripts/start_solr.sh
4. 创建Ranger 审计集合: /opt/solr/ranger_audit_server/scripts/create_ranger_audits_collection.sh (从安装了solr的任何节点仅一次)


确保您有足够的磁盘空间用于索引。建议至少有1TB的可用空间。  
在启动Solr for RangerAudit之后，Solr将收听$ {SOLR_PORT}。例如，通过从浏览器访问http：// $ {SOLR_HOST}：6083来检查Solr。

### <span id="jump3">5.3 Self Install</span>  

如果您足够智慧，那么可以自定义安装Solr并对其进行配置。软件包中的conf文件夹包含参考solrconflig.xml和schema.xml。
```sh
cd ranger/security-admin/contrib/solr_for_audit_setup
$SOLR_INSTALL_HOME/bin/solr create_collection -c ranger_audits -d conf -shards 1 -replicationFactor 1
```

## 6. 配置Ranger Admin和Ranger插件  
Ranger Admin和Ranger插件需要URL才能收集Solr。检查install_notes.txt以获取适当的值。示例URL是：  
http://${SOLR_HOST}:6083/solr/ranger_audits  (替换 ${SOLR_HOST} solr 安装的节点。  
**对于Ranger Admin，在install.properties中配置以下属性：**
```ini
#Source for Audit DB
# * audit_db is solr or db
audit_store=solr
# * audit_solr_url URL to Solr. E.g. http://<solr_host>:6083/solr/ranger_audits
audit_solr_urls=http://localhost:6083/solr/ranger_audits
```
**对于所有插件，在install.properties中配置以下属性**
```ini
XAAUDIT.SOLR.ENABLE=true
XAAUDIT.SOLR.URL=http://localhost:6083/solr/ranger_audits
(replace localhost with the Solr host)
```

## 7. 附录
Sample Ranger Admin UI 截图
![](https://github.com/TaylorZhou/TaylorZhou.github.io/blob/master/assets/blog-image/2020-ranger-audit.png?raw=true)
