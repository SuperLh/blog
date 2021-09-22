# Hadoop

## Hadoop的主要模块

- Hadoop Common
  - 基础模块
- Hadoop Distribute File System(HDFS)
  - 分布式文件系统
- Hadoop Yarn
  - 资源管理平台
- Hadoop MapReduce
  - 分布式并行己算框架



## Hadoop的存储模型

- 文件线性按照字节切割成块（block），每块具有offset和id
- 文件与文件的block大小可以不一样
- 一个文件除了最后一个block，其他block大小一致
- block的大小可以依据硬件的I/O特性进行调整
- block被分散存在集群的节点中，具有location
- block具有副本（replication），没有主从概念，副本不能出现在同一个节点
- 文件上传可以指定block大小和副本数，上传后只能修改副本数
- 一次写入，多次读取，不支持修改
- 支持追加数据



## 副本（Block）放置策略

- 第一个副本
  - 放置在上传文件的DataNode，如果是集群外提交，则随机挑选一台磁盘不太满，CPU不太忙的节点
- 第二个副本
  - 放置在于第一个副本不同的机架的节点上
- 第三个副本
  - 与第二个副本相同机架的节点



## HDFS实现机架感知

- 在Hadoop目录创建机架感知的脚本，RackAware.py

  ```python
  #!/usr/bin/python  
  #-*-coding:UTF-8 -*-  
  import sys  
    
  rack = {  
  
          "12.12.3.1":"SW6300-1",  
          "12.12.3.2":"SW6300-1",  
          "12.12.3.3":"SW6300-1", 
  
          "12.12.3.25":"SW6300-2",  
          "12.12.3.26":"SW6300-2",  
          "12.12.3.27":"SW6300-2",  
   
          "12.12.3.49":"SW6300-3",  
          "12.12.3.50":"SW6300-3",  
          "12.12.3.51":"SW6300-3",  
       
          "12.12.3.73":"SW6300-4",  
          "12.12.3.74":"SW6300-4",  
          "12.12.3.75":"SW6300-4",  
  		}  
  if __name__=="__main__":  
      print "/" + rack.get(sys.argv[1],"SW6300-1-2")  
  ```

- 编辑core-site.xml

  ```
  <property>
  	<name>topology.script.file.name</name>
  	<value>/hadoop-2.6.0-cdh5.14.0/hadoop/etc/hadoop/RackAware.py</value>
  </property>
  ```




## Hadoop的架构设计

- HDFS是一个主从（Master/Slaves）架构
- 由一个NameNode和一些DataNode组成
- 面向文件包含：文件数据（data）和文件元数据（metadata）
- NameNode负责存储和管理文件元数据，并维护了一个层次型的文件目录树
- DataNode负责存储文件数据（block），并提供block的读写
- DataNode与NameNode维持心跳，并汇报自己持有的block信息
- Client和NameNode交互元文件数据和DataNode交互文件block数据



## Hadoop的角色功能

- NameNode
  - 完全基于内存存储元数据，目录结构，文件block的映射
  - 需要持久化方案保证数据可靠性
  - 提供副本放置策略
- SecondaryNameNode
  - 辅助NameNode合并FSImage和Editslog
- DataNode
  - 基于本地磁盘存储block（文件的形式）
  - 保存block的校验和数据，保证block的可靠性
  - 与NameNode保持心跳，汇报block列表状态

