# Hive

## Hive基本SQL操作

### DDL

- 数据库的基本操作

  ```sql
  --展示所有数据库
  	show databases;
  --切换数据库
  	use database_name;
  --创建数据库
  	create database test;
  --删除数据库
  	drop database test;
  ```

- 数据表的基本操作

  ```sql
  --创建普通表
  	create table psn(
  		id int,
  		name string,
  		likes array<string>,
  		address map<string,string>
  	);
      
  --创建自定义行格式的表
  	create table psn2(
  		id int,
  		name string,
  		likes array<string>,
  		address map<string,string>
  	)
  	row format delimited
  	fields terminated by ','
  	collection items terminated by '-'
  	map keys terminated by ':';
  	
  --创建外部表（需要添加external和location）
  	create external table psn4(
  		id int,
  		name string,
  		likes array<string>,
  		address map<string,string>
  	)
  	row format delimited
  	fields terminated by ','
  	collection items terminated by '-'
  	map keys terminated by ':'
  	location '/data';
  	
  --创建分区表
  	create table psn6(
          id int,
          name string,
          likes array<string>,
          address map<string,string>
  	)
  	partitioned by(gender string,age int)
  	row format delimited
  	fields terminated by ','
  	collection items terminated by '-'
  	map keys terminated by ':';	
  
  --增加分区
  	alter table "table_name" add partition("col_name"="col_value");
  --删除分区
  	alter table "table_name" drop partition("col_name"="col_value");
  ```



### DML

- 插入数据

  ```sql
  --加载本地数据到hive表
  	load data local inpath '/data/psn.csv' into table psn;
  	
  --加载HDFS数据到hive表
  	load data inpath '/data/psn.csv' into table psn;
  
  --通过查询语句，插入数据，需要提前创建好表
  	insert overwrite table psn9 select id,name from psn;
  
  	from psn
  		insert overwrite table psn9 select id,name
  		insert overwrite table psn10 select id
  
  --将查询的结果写入到HDFS文件系统
  	insert overwrite directory '/result/result.csv' select * from psn;
  --将查询的结果写入到本地文件系统
  	insert overwrite local directory '/result/result.csv' select * from psn;
  	
  --单条插入数据
  	insert into psn values(1,'liuhui');
  ```

  

- 更新和删除

  - 更新和删除本质上支持，但是需要事务的支持，并且对表有很多限制要求，不建议使用更新和删除操作

  

## Hive内部表和外部表的区别

- 内部表创建的时候数据是存储在hive的默认目录中，外部表的数据是存储在创建表时指定的目录
- 内部表删除的时候，会将元数据和数据全部删除，外部表删除的时候只会删除元数据，不会删除数据



## Hive分区表

- 当创建完分区表后，在保存数据的时候，会在HDFS目录中看到分区会成为一个目录，以多级的目录形式存在
- 插入分区表数据时，必须要指定所有分区列的值
- 多分区表在指定分区时，与顺序无关，只需要对应名称即可



## Hive分区表的修复分区

- 当我们创建完分区表后，然后将数据按照指定分区目录上传到HDFS后，无法查询到数据，此时需要进行修复分区操作

  ```sql
  --修复分区
  	msck repair table "table_name"
  ```



## Hive Serde

- hive主要用来存储结构化数据，如果结构化数据存储的格式嵌套比较复杂，可以使用serde的方式，利用正则表达式匹配的方法来读取数据

- 数据文件

  ```
  192.168.57.4 - - [29/Feb/2019:18:14:35 +0800] "GET /bg-upper.png HTTP/1.1" 304 -
  192.168.57.4 - - [29/Feb/2019:18:14:35 +0800] "GET /bg-nav.png HTTP/1.1" 304 -
  192.168.57.4 - - [29/Feb/2019:18:14:35 +0800] "GET /asf-logo.png HTTP/1.1" 304 -
  192.168.57.4 - - [29/Feb/2019:18:14:35 +0800] "GET /bg-button.png HTTP/1.1" 304 -
  192.168.57.4 - - [29/Feb/2019:18:14:35 +0800] "GET /bg-middle.png HTTP/1.1" 304 -
  192.168.57.4 - - [29/Feb/2019:18:14:36 +0800] "GET / HTTP/1.1" 200 11217
  192.168.57.4 - - [29/Feb/2019:18:14:36 +0800] "GET / HTTP/1.1" 200 11217
  192.168.57.4 - - [29/Feb/2019:18:14:36 +0800] "GET /tomcat.css HTTP/1.1" 304 -
  192.168.57.4 - - [29/Feb/2019:18:14:36 +0800] "GET /tomcat.png HTTP/1.1" 304 -
  192.168.57.4 - - [29/Feb/2019:18:14:36 +0800] "GET /asf-logo.png HTTP/1.1" 304 -
  192.168.57.4 - - [29/Feb/2019:18:14:36 +0800] "GET /bg-middle.png HTTP/1.1" 304 -
  192.168.57.4 - - [29/Feb/2019:18:14:36 +0800] "GET /bg-button.png HTTP/1.1" 304 -
  192.168.57.4 - - [29/Feb/2019:18:14:36 +0800] "GET /bg-nav.png HTTP/1.1" 304 -
  192.168.57.4 - - [29/Feb/2019:18:14:36 +0800] "GET /bg-upper.png HTTP/1.1" 304 -
  192.168.57.4 - - [29/Feb/2019:18:14:36 +0800] "GET / HTTP/1.1" 200 11217
  192.168.57.4 - - [29/Feb/2019:18:14:36 +0800] "GET /tomcat.css HTTP/1.1" 304 -
  192.168.57.4 - - [29/Feb/2019:18:14:36 +0800] "GET /tomcat.png HTTP/1.1" 304 -
  192.168.57.4 - - [29/Feb/2019:18:14:36 +0800] "GET / HTTP/1.1" 200 11217
  192.168.57.4 - - [29/Feb/2019:18:14:36 +0800] "GET /tomcat.css HTTP/1.1" 304 -
  192.168.57.4 - - [29/Feb/2019:18:14:36 +0800] "GET /tomcat.png HTTP/1.1" 304 -
  192.168.57.4 - - [29/Feb/2019:18:14:36 +0800] "GET /bg-button.png HTTP/1.1" 304 -
  192.168.57.4 - - [29/Feb/2019:18:14:36 +0800] "GET /bg-upper.png HTTP/1.1" 304 -
  ```

- 操作方式

  ```sql
  --创建表
  	CREATE TABLE logtbl (
  	    host STRING,
  	    identity STRING,
  	    t_user STRING,
  	    time STRING,
  	    request STRING,
  	    referer STRING,
  	    agent STRING)
  	ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.RegexSerDe'
  	WITH SERDEPROPERTIES ("input.regex" = "([^ ]*) ([^ ]*) ([^ ]*) \\[(.*)\\] \"(.*)\" (-|[0-9]*) (-|[0-		9]*)")
  	STORED AS TEXTFILE;
  
  --加载数据
  	load data local inpath '/data/log.csv' into table logtbl;
  ```



## Hive自定义函数

- 自定义函数包含三种UDF、UDAF、UDTF

  - UDF：一进一出
  - UDAF：多进一出
  - UDTF：一进多出

  

- UDF开发

  - 自定义UDF需要继承org.apache.hadoop.hive.ql.exec.UDF

  - 需要实现evaluate()方法

    ```java
    package com.liuhui
    
    import org.apache.hadoop.hive.ql.exec.UDF;
    import org.apache.hadoop.io.Text;
    
    public class TuoMin extends UDF {
    
    	public Text evaluate(final Text s) {
    		if (s == null) {
    			return null;
    		}
    		String str = s.toString().substring(0, 1) + "****";
    		return new Text(str);
    	}
    }
    ```

    

- 操作步骤（本地临时方法）

  - 该方式创建的函数属于临时函数，当关闭了当前会话之后，函数就无法使用了，因为jar的引用没有了
    - 把UDF的jar包放到本地文件系统
    - 进入到hive的客户端，增加jar包，hive > add jar /APP/liuhui/udf.jar
    - 创建临时函数，hive > create temporay function tuomin as 'com.liuhui.TuoMin'
    - 查询HQL语句，hive > select tuomin(host) from logtbl;

  

- 操作步骤（HDFS，永久方法）

  - 该方法客户端注销以后，创建的UDF能够直接使用，只是在初次使用时进行了一次导入操作
    - 把UDF的jar包放到HDFS
    - 进入到hive的客户端，创建函数
      - hive > create function tuomin as 'com.liuhui.TuoMin' using jar "hdfs://liuhui/jar/udf.har"
    - 查询HQL语句，hive > select tuomin(host) from logtbl;



## Hive参数

- Hive参数的设置方式
  - 在${HIVE_HOME}/conf/hive_site.xml文件中添加参数设置（永久生效，所有hive会话都会加载项应的配置项）
  - 在启动hive cli时，通过--hiveconf key=value进行设置
    - hive --hiveconf hive.cli.print.header=true
    - 只在当前会话有效，退出会话之后参数失效
  - 在进入到hive cli之后，通过set进行设置
  - hive参数初始化设置，在当前用户的家目录下，创建hive.rc文件，在当前文件中设置hive参数的命令，每次进入hive cli时，都会进行加载
    - 当前家目录下还存在.hivehistory文件，此文件保存了hive cli中执行的所有命令



## Hive运行方式

- 命令行方式、控制台方式

  - 命令行中直接输入SQL语句，例如：select * from psn;
  - 命令行中与HDFS交互，例如：dfs ls /
  - 命令行中与Linux交互，例如：!pwd 或者 !ls/

- hive脚本运行（常用）

  ```--sql
  --hive直接执行sql命令，也可以使用;分割多个sql语句
  	hive -e ""
  --hive执行sql命令，并将执行的结果写入文件
  	hive -e "" > result.txt
  --hive静默模式输出，不包含OK、time、token的信息
  	hive -S -e "" > result.txt
  --hive直接执行sql文件
  	hive -f hive-test.sql
  --hive可以从文件读取命令，并且执行初始化操作
  	hive -i hive-init.sql
  --在hive命令行中，可以执行外部文件
  	hive> source hive-test.sql
  ```

- 通过hiveserver2进行jdbc连接

- hive GUI方式





















