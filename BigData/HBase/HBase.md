# HBase

## 关系型数据库与非关系型数据库

### 关系型数据库

- 关系型数据库最典型的数据结构是表，由二维表及其之间的联系所组成的一个数据组织

- 优点
  - 易于维护：都是使用表结构，格式一致
  - 使用方便：SQL语言通用，可用于复杂查询
  - 复杂操作：支持SQL，可用于一个表以及多个表之间非常复杂的查询
- 缺点
  - 读写性能比较差，尤其是海量数据的高效率读写
  - 固定的表结构，欠缺灵活度
  - 高并发读写需求，传统关系型数据库，硬盘IO是一个很大的瓶颈



### 非关系型数据库

- 非关系型数据库严格上不是一种数据库，应该是一种数据结构化存储方法的集合，可以是文档或键值对
- 优点
  - 格式灵活：存储数据的格式可以是key，value形式，文档形式，图片形式等等，使用灵活，应用场景广泛，而关系型数据库则支持基础数据类型
  - 速度快：nosql可以使用硬盘或者随机存储器作为载体，而关系型数据库只能使用硬盘
  - 高扩展性
  - 成本低：nosql数据库部署简单，基本都是开源软件
- 缺点
  - 不提供sql支持，学习和使用成本较高
  - 无事务处理
  - 数据结构相对复杂，复杂查询方面稍欠缺



## HBase简介

- HBase的全称是Hadoop DataBase，是一个高可靠性，高性能，面向列，实时读写的分布式数据库
- 利用Hadoop HDFS作为其文件存储系统，利用Hadoop MapReduce来处理HBase中的海量数据，利用ZooKeeper作为其分布式协同服务
- 主要用来存储非结构化和半结构化数据的松散数据



## HBase数据模型

- RowKey
  - 决定一行数据，每行记录的唯一标识
  - 按照字典序排序
  - RowKey最多只能存储64K的字节数据
- Column Family
  - HBase表中的没格列都归属于某个列簇，列簇必须作为表模式定义的一部分预先给出，create 'test','course';
  - 列名以列簇作为前缀，每个列簇可以有多个列成员，如course:math,course:englist，新的列簇成员可以随后按需，动态加入
  - 权限控制，存储以及调优都是在列簇层面进行
  - HBase把同一列簇里面的数据存储在同一个目录下，由几个文件保存
- TimeStamp
  - 在HBase每个cell存储单元对同一份数据有多个版本，根据唯一的时间戳来区分每个版本之间的差异，不同版本的数据按照时间倒序排序
  - 时间戳的类型是64位整型
  - 时间戳可以由HBase在写入数据时自动复制，此时间戳是精确到毫秒的当前系统时间
  - 时间戳也可以由客户显式赋值，如果应用程序要避免数据版本冲突，就必须自己生成具有唯一性的时间戳



- **<font color='red'>由RowKey，column，version确定唯一单元</font>**
- **<font color='red'>cell中的数据是没有类型的，全部是字节数组形式进行存储的</font>**



## HBase架构

![avatar](/pics/hbase架构图.png)

- Client

  - 包含访问HBase的接口并维护cache来加快对HBase的访问

- ZooKeeper

  - 保证任何时候，集群中只有一个活跃Master
  - 存储所有Region的寻址入口
  - 实时监控Region Server的上线和下线信息，并实时通知Master
  - 存储HBase的schema和table元数据

- Master

  - 为Region Server分配Region
  - 负责Region Server的负载均衡
  - 发现失效的Region Server并重新分配其上的Region
  - 管理用户对Table的增删改操作

- Region Server

  - Region Server维护Region，处理对这些Region的IO请求
  - Region Server负责切分在运行过程中变得很大的Region

- Region Server组件介绍

  - Region

    - HBase自动把表水平划分成多个区域（Region），每个Region会保存一个表里某段连续的数据
    - 每个表一开始只有一个Region，随着数据不断插入表，Region不断增大，当增大到一个阈值的时候，Region就会等分两个新的Region（裂变）
    - 当table中的行不断增多，就会有越来越多的Region，这样一张完整的表会被保存在多个Region Server上

  - MemStore与StoreFile

    - 一个Region由多个Store组成，一个Store对应一个列簇

    - Store包括位于内存中的MemStore和位于磁盘的StoreFile，写操作先写入MemStore，当MemStore中的数据达到某个阈值，HRegionServer会启动flashcache进程，写入StoreFile，每次写入形成一个单独的StoreFile

    - 当StoreFile文件的数量增长到一定阈值后，系统会进行合并（minor、major），在合并过程中，会进行版本合并和删除工作（major），形成更大的StoreFile

    - 当一个Region所有的StoreFile的大小和数量都超过一定的阈值以后，会把当前的Region分割为两个，并由HMaster分配到相应的RegionServer服务器，实现负载均衡

    - 客户端检索数据，先在MemStore中找，找不到去BlockCache，再找不到去StoreFile中找

      

    - **<font color='red'>HRegion是HBase中分布式存储和负载均衡的最小单元，最小单元表示不同的HRegion可以分布在不同的Hegion Server上</font>**

    - **<font color='red'>HRegion由一个或者多个Store组成，每个Store保存一个Column Family</font>**

    - **<font color='red'>每个Store又由一个MemStore和0至多个StoreFile组成，StoreFile以HFile格式保存在HDFS上</font>**



## HBase读写流程

### 读流程

- 客户端从ZooKeeper中获取meta表所在的Region Server节点信息
- 客户端访问meta表所在的Region Server，获取到Region所在的Region Server信息
- 客户端访问具体的Region所在的Region Server，找到对应的Region及Store
- 首先从MetaStore中读取数据，如果读取到了那么直接将数据返回，如果没有，则去blockcache读取数据
- 如果blockcache中读取到数据，则直接将数据返回，如果读取不到，则遍历StoreFile文件，查找数据
- 如果从StoreFile中读取不到数据，则返回客户端为空，如果读取到数据，那么先将数据缓存到blockcache中，然后再将数据返回给客户端
- blockcache是内存空间，如果缓存的数据比较多，满了之后会采用LRU策略，将较老的数据删除



### 写流程

- 客户端从ZooKeeper中获取meta表所在的RegionServer节点信息
- 客户端访问meta表所在的Region Server节点，获取到Region所在的Region Server信息
- 客户端访问具体的Region所在的Region Server，找到对应的Region及Store
- 开始写数据，写数据的时候辉县从HLog中写一份数据（方便MetaStore中数据丢失后能够从HLog恢复数据，向HLog中写数据的时候也是优先写入内存，后台会有一个线程，定期异步刷写数据到HDFS，如果HLog的数据也写入失败，那么数据就会发生丢失）
- HLog数据写入完成之后，会先将数据写入到MemStore，MemStore默认大小是64MB，当MemStore满了之后会进行统一的溢写操作，将MemStore中的数据持久化到HDFS中
- 频繁的溢写会导致产生很多的小文件，因此会进行文件的合并，文件在合并的时候有两种，minor和major，minor是小范围文件的合并，major表示将所有的StoreFile文件都合并成一个



## HBase Shell基础操作

### 通用命令

```java
// 展示regionserver的task列表
hbase(main):000:0>processlist
// 展示集群的状态
hbase(main):000:0>status
// table命令的帮助手册
hbase(main):000:0>table_help
// 显示hbase的版本
hbase(main):000:0>version
// 展示当前hbase的用户
hbase(main):000:0>whoami
```



### DDL操作

```java
// 修改表的属性
hbase(main):000:0>alter 't1', NAME => 'f1', VERSIONS => 5
// 创建表
hbase(main):000:0>create 't1', 'cf'
// 查看表描述，只会展示列簇的信息
hbase(main):000:0>describe 'test'
// 禁用表
hbase(main):000:0>disable 'test'
// 禁用所有表
hbase(main):000:0>disable_all
// 删除表
hbase(main):000:0>drop 'test'
// 删除所有表
hbase(main):000:0>drop_all
// 启用表
hbase(main):000:0>enable 'test'
// 启用所有表
hbase(main):000:0>enable_all
// 判断表是否存在
hbase(main):000:0>exists 'test'
// 获取表
hbase(main):000:0>get_table 'test'
// 判断表是否被禁用
hbase(main):000:0>is_disabled 'test'
// 判断表是否被启用
hbase(main):000:0>is_enabled 'test'
// 展示所有表
hbase(main):000:0>list
// 展示表所占用的Region
hbase(main):000:0>list_regions
// 定位某个RowKey所在的行在哪个Region
hbase(main):000:0>locate_region
// 展示所有的过滤器
hbase(main):000:0>show_filters
```

### NameSpace操作

```java
//修改命名空间的属性
hbase(main):000:0>alter_namespace 'my_ns', {METHOD => 'set', 'PROPERTY_NAME' => 'PROPERTY_VALUE'}
//创建命名空间
hbase(main):000:0>create_namespace 'my_ns'
//获取命名空间的描述信息
hbase(main):000:0>describe_namespace 'my_ns'
//删除命名空间
hbase(main):000:0>drop_namespace 'my_ns'
//展示所有的命名空间
hbase(main):000:0>list_namespace
//展示某个命名空间下的所有表
hbase(main):000:0>list_namespace_tables 'my_ns'
```



### DML操作

```java
//向表中追加一个具体的值
hbase(main):000:0>append 't1', 'r1', 'c1', 'value', ATTRIBUTES=>{'mykey'=>'myvalue'}
//统计表的记录条数，默认一千条输出一次
hbase(main):000:0>count 'test'
//删除表的某一个值
hbase(main):000:0>delete 't1', 'r1', 'c1', ts1
//删除表的某一个列的所有值
hbase(main):000:0>deleteall 't1', 'r1', 'c1'
//获取表的一行记录
hbase(main):000:0>get 't1', 'r1'
//获取表的一个列的值的个数
hbase(main):000:0>get_counter 't1', 'r1', 'c1'
//获取表的切片
hbase(main):000:0>get_splits 't1'
//增加一个cell对象的值
hbase(main):000:0>incr 't1', 'r1', 'c1'
//向表中的某一个列插入值
hbase(main):000:0>put 't1', 'r1', 'c1', 'value’, ts1
//扫描表的全部数据
hbase(main):000:0>scan 't1'
//清空表的所有数据
hbase(main):000:0>truncate
```



## HBase的LSM树存储结构

### LSM树的由来

- Hash存储的方式支持增、删、改以及随机读取操作，但是不支持顺序扫描，对应的存储系统为K-V存储系统，对于K-V的插入及查询，要比树的操作块，如果不需要有序的遍历数据
- B+树不仅支持单挑记录的增、删、读、改操作，还支持顺序扫描（B+树的叶子节点之间的指针），对应的存储系统就是关系数据库。但是删除和更新操作比较麻烦
- LSM树（Log-Structed Merge Tree）存储结构和B树存储引擎一样，同样支持增、删、改、读、顺序扫描操作。而且通过批量存储技术规避磁盘随机写入问题。LSM树和B+树相比，LSM树牺牲了部分读性能，用来大幅度提高写性能



### LSM树的设计思想和原理

- 将对数据的修改和增量保持在内存中，达到指定的大小限制后将这些修改操作批量写入磁盘，不过读取的时候比较麻烦，需要合并磁盘中历史数据和内存中最近修改操作，所以写入性能大大提升，读取时可能需要先查看是否命中内存，否则需要访问较多的磁盘文件。极端的说，基于LSM树实现的HBase的写性能比MySQL高了一个数量级，读性能低了一个数量级
- LSM树的原理是把一棵大树拆分成N棵小树，首先写入内存中，随着小树越来越大，内存中的小树会flush到磁盘中，磁盘中的树定期做Merge操作，合并成一颗大树，以优化读性能
- 在HBase中，LSM树的应用流程对应如下
  - 小树先存到内存中，为了防止内存数据丢失，写内存的同时，将数据持久化到硬盘，对应了HBase的MemStore和HLog
  - MemStore上的树达到一定大小之后，需要flush到HRegion磁盘中，一般是Hadoop DataNode，这样MemStore就变成了DataNode上的磁盘文件StoreFile，定期HRegionServer对DataNode的数据做Merge操作，彻底删除无效空间，多棵小树在这个实际合并成大树，增加读性能

## HBase数据读取流程



















