---
layout: post
title: 我在学习和实践图数据库 Neo4j 的漫漫成长路
date: 2021-02-11 21:11:22
tags: ['neo4j', "optimization"]
---

## 什么是图数据库和Neo4j?

传统关系型数据库通过文档字段来建立数据的关联, 但是这并不利于海量数据背景下的关系推导. 图数据库应运而生, 图数据库的上层就是我们熟悉的图网络结构, 而下层则是对图结构进行了性能优化, 使之能进行快速的关系推导.  图数据库在知识图谱(比如社交关系推导)和图神经网络(GNN)上有很大的应用. 

[Neo4j](https://github.com/neo4j/neo4j)是业界主流的图数据库, 数据库目前排名是19(参见[DB-Engines Ranking](https://db-engines.com/en/ranking)), 以点/关系进行存储, 支持百亿级别的查询. Neo4j分为免费的社区版和付费的企业版. 社区版只能进行单机部署, 企业版可以部署集群且性能上有着诸多优化, 以及解锁了诸多限制. [ONgDB](https://github.com/graphfoundation/ongdb) 是Neo4j付费闭源前的分支版本, 目前的最新版本为3.6.2, 跟Neo4j企业版本3.6功能相差无几. 

## ONgDB因果集群部署

因果集群基于Raft协议开发, Raft是一种更加易于理解的一致性算法, 可支持大规模和多拓扑结构的数据环境. 因果集群主要有以下两大特点:

1. 安全性：核心服务器(Core)为事物平台处理提供了容错平台

2. 可扩展性：只读副本(Read Replica)为图查询提供了一个大规模高可扩展的平台

集群部署会有一些坑点，也许是版本迭代升级出现的新问题，从网上搜集的现有资料并没有提及到这些问题和解决方法。

首先我的使用的版本是`ongdb-enterprise-3.6.2`，这是我之前就下载好存本地的，但此时在ONgDB仓库里将之前发布的版本都删除了（我推测是因为跟Neo4j打官司的原因），只留下约一个月前发布的[1.0.0-alpha01](https://github.com/graphfoundation/ongdb/releases/tag/1.0.0-alpha01)版本，这也是ONgDB正式发布的第一个alpha测试版本，原先虽然名为ONgDB但其实代码里面都是以neo4j来命名，而这个`1.0.0`则将名称都改为了ongdb。并且ongdb和neo4j在配置上也并不完全一致，比如因果集群部分配置就是这样。我在使用1.0.0版本进行部署的时候主节点会卡在`Attempting to connect to the other cluster members`阶段，而ONgDB的文档内容非常少且网站也挂了，确认我配置没有问题后，我选择切回了旧的`ongdb-enterprise-3.6.2`版本。（并且此时ongdb-apoc还不兼容1.0.0版本）

我测试使用的是三台服务器，需要先配置好SSH密钥，让这三台服务器能通过SSH直接登录到彼此服务器。

随后参考Neo4j的集群配置文档，为配置文件`conf/neo4j.conf`改动以下内容（注意：尽量去找配置文件内已有的项进行修改，而不是直接在文件末尾新增，因为多个相同项Neo4j只取第一项作为结果）：

```shell
dbms.default_listen_address=0.0.0.0
# 其他服务器用该地址来连接到节点，设定为节点的IP即可（无需端口）
dbms.default_advertised_address=core03.example.com
# 指定节点是核心节点还是冗余节点
dbms.mode=CORE
# 集群内所有节点的地址，集群节点启动后会不断通过该列表探测活跃的成员，默认端口5000
causal_clustering.initial_discovery_members=core01.example.com:5000,core02.example.com:5000,core03.example.com:5000
# （可选）该选项能避免集群在选举新的主节点前预先选举候选节点，这样就能避免因网络原因无法选举
causal_clustering.enable_pre_voting=true
```

所有节点的配置上其实除了`dbms.default_advertised_address`需要设置为自己的IP地址外，其他的配置都是一致的（当然还有mode你可以自己选定是作为核心节点还是冗余节点）。

配置完后逐个启动各个节点即可，如果没有成功启动，那么很有可能是数据不一致带来的问题，建议先将`data`清空或重命名其他名称，保证集群内各节点的`data`都是一致的，重新启动图数据库。

**一些报错信息及解决办法**

* `Waiting to hear from leader`： 清空`data`或者重命名成其他名称
* `Unable to find transaction 1 in any of my logical logs`： 数据不一致造成的问题，可以尝试清空`data`或者`neo4j-admin unbind --database=graph.db`
* 对于因果集群要想使用neo4j-import导入全量数据，也会存在数据不一致造成启动失败的问题，所以官方给出的实践方案是，在一个节点上成功导入数据后，使用scp或rsync等工具直接导入的数据库传输到集群内其他节点服务器上。当然如果依然有报错情况的话，还需要执行`neo4j-admin unbind --database=graph.db`来解除绑定。

集群成功部署后，就可以通过`call dbms.cluster.overview()`和`call dbms.cluster.routing.getServers()`来查看集群成员角色和请求路由信息。默认第一个启动的节点为作为主节点，其他节点则作为从节点。主节点负责协调集群并接受所有的写入，数据改动会同步到所有从节点上，主节点挂掉则其他从节点就会开始进行选举，投票出新的节点来作为主节点进行服务，集群在可用性上有很大的强化。

集群的连接方式基本也没有多少变化，原先单实例时的协议头为`bolt://`，而启用了路由的协议头则变更为`bolt+routing://`，协议头后面跟任意集群内节点的地址即可。当然集群内各节点也依然可以使用`bolt://`协议头单独用来查询数据（注意可读不一定可写）。另外要注意自己使用的驱动是否支持直接使用`bolt+routing://`协议头，像py2neo就不支持直接改协议头，但是可以在初始化Graph对象时增加`routing=True`选项启用路由模式。

## 关于性能优化的技巧

* 图数据库配置`conf/neo4j.con`, 可以使用`bin/neo4j-admin memrec`来查看推荐的配置.

  ``` bash
  # 堆内存
  dbms.memory.heap.initial_size=16384m
  dbms.memory.heap.max_size=16384m
  # 页面缓存
  dbms.memory.pagecache.size=80g
  ```

* 创建索引来提高检索速度: `create index on :Person(firstname)`, 使用`:schema`可以确认索引状态, 索引状态为ONLINE则表示索引已经生效. 

* 数据预热: 预先进行全图的查询来将图数据载入到缓存里, 来加快检索速度.如`call apoc.warmup.run()`和`match (n) optional match (n)-[r]->() return count(n) + count(r)`

* 根据情况将不同类型不同场景的数据进行拆分, 构建存储到不同的服务器上来减轻图数据库的压力. 

* 对于图数据的详细属性信息可以存储到ElasticSearch做复杂检索, 让图数据库专注于图分析和检索能力上(有插件支持). 可以参考[这篇文章](https://blog.csdn.net/superman_xxx/article/details/106752758?spm=1001.2014.3001.5502).

* 使用`apoc.path.subgraphNodes`来遍历节点到所有关系**远远快于**使用多层关系查询, 因为多层关系查询实际会展开关系层数逐个查询. 

* 如果有确定的目标, 编写cypher语句时先match该目标再匹配路径, 如 `match (n:Person{name: 'Peter'}) return (n)-[]->()`, 而不是匹配路径再过滤路径上节点属性, 如`match p=(n)-[]->() where n.name='Peter' return p `. 后者会扫描全图. 

* 使用`neo4j-import`导入海量数据, 但该工具需要脱机并且只适用于空库, 数据可能还需要预处理生成CSV, 不过它的效率非常值得你这么做. 同时需要注意其导入时是可以用**正则表达式**来匹配多个文件名的（注意**不是通配符**），比如如下：

  ```shell
  bin/neo4j-admin import \
  	--nodes="import/movies4-header.csv,import/movies4-part.*" \
  	--nodes="import/actors4-header.csv,import/actors4-part.*" \
  	--relationships="import/roles4-header.csv,import/roles4-part.*"
  ```

* 双向关系会极大地影响图数据库的遍历性能, 尽可能地不要这样做.如果真的发生了, 也可以参考[这篇文章](https://blog.csdn.net/superman_xxx/article/details/104791282?spm=1001.2014.3001.5502)进行删除. 

* 图数据库里的环路也会极大地影响查询性能, 我解决环路的方法是检测路径中是否存在相同的节点. 使用`apoc.coll.duplicates`可以返回集合中的重复项.

* 超级节点指的是拥有非常多关系/边的一类节点. 超级节点的存在会极大地影响入库/检索/分析的效率. 

  * 在图数据建模的时候就应该确定好实体应该表示成节点还是标签. 比如“国家”这个实体如果设计成节点, 那么就很容易成为超级节点, 但设计成标签则不会带来性能的影响. 
  * 关系结构优化: 将超级节点与其他节点的关系按照时间或者其他的层次关系进行分组, 这样既能提高查询的并发性也可以减少对超级节点的遍历开销. 
  * 标签细分: 比如原来的标签就是**社交媒体**, 那么就可以将其细分到某个**具体的社交平台**.

* 使用`explain`和`profile`来分析cypher语句的性能, 前者不会执行语句而后者会实际执行. 

* 使用带有补全提示的[cycli](https://github.com/nicolewhite/cycli)来帮助更高效地编写cypher语句, 使用[yFiles](https://www.yworks.com/neo4j-explorer/)来对图进行可视化.

* 删除两个节点之间的重复关系: 

  ``` cypher
  MATCH p=(A:Test {name:'A'})-[r]->(B:Test {name:'B'})
  WITH ID(r) AS id,r.name AS name
  WITH name,COLLECT(id) AS relIds
  WITH name,relIds,SIZE(relIds) AS relIdsSize
  WHERE relIdsSize>1
  WITH name,apoc.coll.subtract(relIds, [relIds[0]]) AS deleteRelIds
  WITH name,deleteRelIds
  MATCH ()-[r]-() WHERE ID(r) IN deleteRelIds DELETE r
  ```

  更多条件分支操作可以参考如下

  ``` cypher
  CALL apoc.do.case([
    relationship=1,
    \'MATCH (from:Label {hcode:$fromHcode}),(to:Label {hcode:$toHcode}) 
      MERGE (from)-[:NEXT]->(to)\',
    relationship=-1,
    \'MATCH (from:Label {hcode:$fromHcode}),(to:Label {hcode:$toHcode}) 
      MERGE (from)<-[:NEXT]-(to)\'],
    \'\',
    {fromHcode:fromHcode,toHcode:toHcode}) 
  YIELD value RETURN value
  ```

* 使用`apoc.cypher.parallel`并行执行查询, 例如如下会从名称列表中并行取出每个姓名, 搜索其邻居节点并返回姓名:

  ``` cypher
  CALL apoc.cypher.parallel(
    'MATCH (p:Person{name:$name}) -[:FRIEND_OF]-> (p1) RETURN p1.name AS name', 
    {name:['John','Mary','Peter','Wong','Chen','Lynas','Smith','Anna']},
    'name'
  )
  ```

## 学习资料

在学习图数据库和实践过程中其实积累了不少资料, 都是自己在初学和实践中遇到困难而去搜索的资料. 而其实这方面的资料差不多也就是这些. 基本上遇到了问题都能在这些地方找到答案. 

**主要资料**

这些是学习和操作图数据库所必需了解的知识部分. 

* [The Neo4j Getting Started Guide](https://neo4j.com/docs/getting-started/current/): Neo4j官方的入门指南
* [Cypher Manual](https://neo4j.com/docs/cypher-manual/current/): Cypher是Neo4j的查询语言, 查询语法比较简单直观, 但尽管如此, 如何判断Cypher语句正确/准确, 以及对Cypher语句进行优化是工作一直需要考虑的问题. 
* [The Neo4j Operations Manual](https://neo4j.com/docs/operations-manual/current/): Neo4j给出的操作手册, 可以大致浏览其中的内容, 因为遇到的很多问题可能最终都指向这里的解答. 

**其他资料**

以下资料并非不重要, 而是用于扩展自己的学习面. 

* [Neo4j Documentation](https://neo4j.com/docs/): 这里列出了Neo4j的所有文档. 
* [The Neo4j Python Driver Manual](https://neo4j.com/docs/python-manual/current/), Neo4j官方给出的Python driver手册. 
* [The Py2neo Handbook](https://py2neo.readthedocs.io/en/latest/): Py2neo相比官方给出的Python driver, 简化和封装了跟Neo4j的连接操作. 
* [APOC User Guide 4.1](https://neo4j.com/labs/apoc/4.1/): APOC是neo4j的插件之一, 能够扩充Neo4j的一些能力. 
* [ONgDB - fork of Neo4j Enterprise: Graphs for Everyone](https://github.com/graphfoundation/ongdb): Neo4j的分支版本, 后续多节点部署会考虑使用这个. 
* [Neo4j community](https://community.neo4j.com/): neo4j官方维护的论坛, 应该是最直接寻求答案的地方. 
* [Neo4j 图数据库中文社区](http://neo4j.com.cn/): 国内的Neo4j中文社区. 
* [Tnoy.ma的csdn博客](https://yc-ma.blog.csdn.net/): 里面有许多的文章介绍图数据库, [Yc-Ma Blog](https://crazyyanchao.github.io/blog/archive.html) 疑似是作者的另一博客. 
* [俞博士的csdn博客](https://blog.csdn.net/GraphWay): 作者应该是在Neo4j就职, 介绍的东西还蛮实际的, 并且有不少PPT方便理解.
* [图数据库neo4j因果集群技术分析](https://zhuanlan.zhihu.com/p/83964428): 非常详细地讲述了因果集群.

实际过程中会有许多需要进行谷歌搜索的事情, 大部分参考于stackoverflow的回答. 