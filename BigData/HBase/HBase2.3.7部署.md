# Table of Contents

* [HBase2.3.7部署](#hbase237部署)
  * [准备安装好Hadoop的集群](#准备安装好hadoop的集群)
  * [安装HBase](#安装hbase)


# HBase2.3.7部署

## 准备安装好Hadoop的集群

## 安装HBase

- 解压Hbase安装包

  ```
  tar -zxvf hbase-2.3.7-bin.tar.gz
  ```

- 配置环境变量

  ```
  export HBASE_HOME=/APP/software/hbase-2.3.7
  export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/sbin:$HADOOP_HOME/bin:$HBASE_HOME/bin
  ```

- 修改配置文件，hbase-env.sh

  ```
  设置JAVA_HOME环境变量
  export JAVA_HOME=/APP/software/jdk1.8.0_281
  设置是否使用自己的zookeeper实例
  export HBASE_MANAGES_ZK=false
  ```

- 修改配置文件，hbase-site.xml

  ```
  <configuration>
    <property>
      <name>hbase.rootdir</name>
      <value>hdfs://mycluster/hbase</value>
    </property>
    <property>
      <name>hbase.cluster.distributed</name>
      <value>true</value>
    </property>
    <property>
      <name>hbase.zookeeper.quorum</name>
      <value>node1,node2,node3</value>
    </property>
  </configuration>
  ```

- 修改regionservers文件，设置regionserver分布的节点

  ```
  node1
  node2
  node3
  ```

- 如果配置Master的高可用，需要在conf目录下创建backup-masters文件

  ```
  master2
  ```

- 拷贝core-site.xml和hdfs-site.xml到conf目录下

- 分发HBase文件

- 执行hbase shell命令，进入到HBase

