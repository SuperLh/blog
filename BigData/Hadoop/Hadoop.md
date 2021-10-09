# Hadoop

## Hadoop的主要模块

- Hadoop Common
  - 基础模块
- Hadoop Distribute File System(HDFS)
  - 分布式文件系统
  
- Hadoop MapReduce

  - 分布式并行己算框架

- Hadoop Yarn
  - 资源管理平台
  
  

# HDFS

## HDFS的存储模型

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




## HDFS的架构设计

- HDFS是一个主从（Master/Slaves）架构
- 由一个NameNode和一些DataNode组成
- 面向文件包含：文件数据（data）和文件元数据（metadata）
- NameNode负责存储和管理文件元数据，并维护了一个层次型的文件目录树
- DataNode负责存储文件数据（block），并提供block的读写
- DataNode与NameNode维持心跳，并汇报自己持有的block信息
- Client和NameNode交互元文件数据和DataNode交互文件block数据



## HDFS的角色功能

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
  
  

## HDFS的元数据持久化

### FSImage+Editslog

- 任何对文件系统元数据产生修改的操作，NameNode都会使用一种称为Editslog的事务日志记录下来
- 使用FSImage存储内存所有的元数据状态
- 使用本地磁盘保存Editslog和FSImage
- Editslog：完整性，数据丢失少，但恢复速度慢，并有体积膨胀的风险
- FSImage：恢复速度快，体积与内存数据相当，但不能实时保存，数据丢失多
- NameNode采用了FSImage+Editslog整合的方案
  - 滚动将增量的Editslog更新到FSImage，以保证更近时点的FSImage和更小体积的Editslog



### FSImage+Edistlog实现方案

- HDFS格式化命令，会产生一个空的FSImage

- 当NameNode启动时，它从硬盘中读取Editslog和FSImage

- 将所有的Editslog中的事务作用在内存中的FSImage

- 并将这个新版本的FSImage从内存中保存到磁盘中

- 然后删除旧的Editslog，因为这个旧的Editslog已经作用在FSImage上了

   ![avatar](pics/fsimage+editslog.png)



### 元数据持久化的SecondaryNameNode

- 在非HA模式下，SecondaryNameNode一般是独立的节点，周期完成对NameNode的Editslog向FSImage合并，减少Editslog的大小，减少NameNode的启动时间
- 根据配置文件设置的时间间隔fs.checkpoint.period，3600s
- 根据配置文件设置Editslog的大小fs.checkpoint.size，规定Editslog文件的最大值，默认是64MB



### 安全模式

- NameNode启动后进入一个称为安全模式的特殊状态
- 处于安全模式的NameNode是不会进行数据块的复制
- NameNode从所有的DataNode接受心跳信号和块状态报告
- 每当NameNode检测确认某个数据块的副本数目达到这个最小值，那么该数据块就会被认为是副本安全的
- 在一定百分比的数据块（可配置）被NameNode检测确认是安全之后（加上额外的30秒等待时间），NameNode会退出安全模式
- 接下来它会确定还有哪些数据块的副本没有达到指定数据，并将这些数据块复制到其他DataNode上



## HDFS读写流程

### HDFS读流程

- Client端加载FileSystem类，建立与NameNode的通信，发送读取数据的请求
- NameNode根据文件快的元数据，判断文件是否存在以及存在哪个DataNode-D1,D2,D3（不同block）
- NameNode回答Client端，不存在文件或者文件所在的DateNode
- Client实现FSDataInputStream类，建立与D1（存放block1）的通信，并请求下载数据
- D1接收到请求，将数据文件从磁盘读取到内存，以Packet为单位，发送给Client
- Client接收到Packet数据，先写到内存缓存，然后写入磁盘
- Block1读取下载完毕后，Client和D2建立通信，下载Block2
- 直到全部下载完成，关闭连接



- 为了降低整体的带宽消耗和读取延时，HDFS会尽量让读取程序读取离他最近的副本
- 如果读取程序的同一个机架上有一个副本，那么就读取该副本
  - Client和NameNode交互文件元数据获取fileBlockLocation
  - NameNode会按距离策略排序返回
  - Client尝试下载Block，并校验数据完整性



### HDFS写流程

- Client端加载FileSystem类，建立与NameNode的通信，发送上传数据请求
- NameNode检查文件是否存在以及要存放的父目录是否存在，然后回复是否可以进行上传
- Client端请求上传第一个Block
- NameNode根据副本数分配对应数量的DataNode节点，返回给Client
- Client加载FSDataOutputStream类，选择一个距离自己最近的DataNode请求建立通信，D1收到请求后调用D2，D2调用D3，然后分别回复Client，并建立数据通道
- Client以Packet形式，向D1传输Block1
- D1接收到Packet到内存，然后拷贝到本地磁盘，写完后内存中的Packet会传递给D2
- D2同样进行拷贝写，然后将内存中的数据传递给D3
- D3作为最后一站，直接将Packet写入到磁盘中
- 第一个Block上传完成后，Client将会再次发送请求上传第二个Block
- 直到所有Block完成上传，关闭管道连接



## Hadoop节点如何进行动态上下线

- 上线
  - 关闭新增节点的防火墙
  - 修改集群节点的host文件，增加新增节点的hostname
  - NameNode增加新增节点的免密码登录
  - 执行hdfs dfsadmin -refreshNodes（刷新操作）
  - 更改slaves节点，增加新增节点
  - 启动DataNode节点
  - 查看NameNode监控页面是否有新增节点
- 下线
  - 修改hdfs-site.xml
  - 配置hdfs.hosts.exclude中需要下架的机器
  - 执行hdfs dfsadmin -refreshNodes（刷新操作）
  - 关闭下架的机器
  - 机器下线完毕后，修改hdfs-site.xml，一处exclude



# MapReduce

## MapReduce的工作原理

![avatar](pics/MapReduce_1.jpg) 



- Map Task
  - 根据InputFormat将输入文件分为多个splits，每个splits会作为一个Map Task的输入
  - 每条数据经过map方法，映射成K-V键值对，相同的Key作为一组，调用一次Reduce方法
- Reduce Task
  - 每组数据在Reduce方法内进行迭代计算，并将最后结果输出至HDFS



## MapReduce的Shuffle过程

![avatar](pics/MapReduce_2.jpg)

- Map端的Shuffle阶段
  - Partition：对于map输出的每一个键值对，系统会计算出相应的partition
  - Collector：环形数据缓冲区，将K-V-P存储于缓冲区，用于磁盘溢写，减少磁盘IO
  - Sort：将环形数据缓冲区的数据按照Partition，Key进行升序排序
  - Spill：将环形数据缓冲区的数据溢写到本地磁盘
  - Merge：将多次Spill后的文件进行归并排序，合并所有输出结果
- Reduce端的Shuffle阶段
  - Copy：复制Map端输出的数据文件
  - Merge Sort：从多个Map端Copy数据，进行归并排序，合并数据，情况可分为可分为内存-内存，内存-磁盘，磁盘-磁盘
