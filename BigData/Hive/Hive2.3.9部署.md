# Table of Contents

* [Hive2.3.9部署](#hive239部署)
  * [更换yum源](#更换yum源)
  * [安装MySQL（node3）](#安装mysqlnode3)
  * [安装Hive和Hive metaStore](#安装hive和hive-metastore)


# Hive2.3.9部署

## 更换yum源

- 安装wget

  ```
  yum install wget -y
  ```

- 进入yum安装目录

  ```
  /etc/yum.repos.d/
  ```

- 将原有文件进行备份

  ```
  mkdir backup
  mv CentOS-* backup
  ```

- 打开镜像网站 https://mirrors.aliyun.com 进入到系统中

  ```
  wget -O /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
  ```

- 清除yum的已有缓存

  ```
  yum clean all
  ```

- 生成yum的缓存

  ```
  yum makecache
  ```



## 安装MySQL（node3）

- 下载安装rpm包

  ```
  wget http://repo.mysql.com/mysql-community-release-el7-5.noarch.rpm
  rpm -ivh mysql-community-release-el7-5.noarch.rpm
  ```

- yum安装MySQL

  ```
  yum install mysql-server -y
  ```

- 启动MySQL

  ```
  systemctl start mysqld
  ```

- 设置开机启动

  ```
  systemctl enable mysqld
  ```

- 进入到MySQL

  ```
  mysql
  ```

- 修改权限

  ```sql
  --进入数据库
  	use mysql;
  --查看表
  	show tables;
  --查看用户权限
  	select host,user,password from user;
  --赋予权限
  	grant all privileges on *.* to 'root'@'%' identified by 'superman' with grant option
  --删除其余用户
  delete from user where host!='%'
  --刷新权限
  flush privileges;
  ```



## 安装Hive和Hive metaStore

- 选择两台虚拟机，node1作为服务端，node2作为客户端

- 解压Hive

  ```
  tar -zxvf apache-hive-2.3.9-bin.tar.gz
  ```

- 配置环境变量

  ```
  vi /etc/profile
  
  export HIVE_HOME=/APP/software/hive-2.3.9
  export PATH=$PATH:$JAVA_HOME/bin:$ZOOKEEPER_HOME/bin:$HADOOP_HOME/sbin:$HADOOP_HOME/bin:$HIVE_HOME/bin
  
  source /etc/profile
  ```

- 拷贝mysql驱动包到lib目录

  ```
  mv mysql-connector-java-5.1.32-bin.jar $HIVE_HOME/lib
  ```

- 重命名配置文件

  ```
  mv hive-default.xml.template hive-site.xml
  ```

- 修改配置文件（node1）

  ```xml
  <configuration>
      <property>
          <name>hive.metastore.warehouse.dir</name>
          <value>/user/hive/warehouse</value>
      </property>
      <property>
          <name>javax.jdo.option.ConnectionURL</name>
          <value>jdbc:mysql://node3:3306/hive?createDatabaseIfNotExist=true</value>
      </property>
      <property>
          <name>javax.jdo.option.ConnectionDriverName</name>
          <value>com.mysql.jdbc.Driver</value>
      </property>
      <property>
          <name>javax.jdo.option.ConnectionUserName</name>
          <value>root</value>
      </property>
      <property>
          <name>javax.jdo.option.ConnectionPassword</name>
          <value>superman</value>
      </property>
  </configuration>
  ```

- 修改配置文件（node2）

  ```xml
  <configuration>
      <property>
          <name>hive.metastore.warehouse.dir</name>
          <value>/user/hive/warehouse</value>
      </property>
      <property>
          <name>hive.metastore.uris</name>
          <value>thrift://node1:9083</value>
      </property>
  </configuration>
  ```

- 修改guava.jar包，将hadoop中的高版本guava.jar拷贝至Hive目录，删除低版本jar

  ```
  cd $HADOOP_HOME/share/hadoop/common/lib
  cp guava-27.0-jre.jar $HIVE_HOME/lib
  cd $HIVE_HOME/lib
  rm -rf guava-14.0.1.jar
  ```

- 元数据库格式化

  ```
  schematool -dbType mysql -initSchema
  ```

- 开启服务端元数据库服务（node1）

  ```
  hive --service metastore
  ```

- 进入hive cli窗口（node2）

  ```
  hive
  ```



## 安装Hive Server2

- 搭建Hive Server2服务时，需要修改HDFS的超级用户的管理权限，修改core-site.xml

  	<property>
  		<name>hadoop.proxyuser.root.groups</name>	
  		<value>*</value>
  	</property>
  	<property>
  		<name>hadoop.proxyuser.root.hosts</name>	
  		<value>*</value>
  	</property>

- 刷新HDFS配置

  ```python
  hdfs dfsadmin -refreshSuperUserGroupConfiguration
  ```

- 启动hive server2

  ```python
  hiverserver2
  ```



## HiveServer2的访问方式

- beeline

  ```python
  beeline -u jdbc:hive2://<host>:<port>/<db> -n name
  	beeline -u jdbc:hive2://node1:10000 -n root
  ```

- jdbc

  ```java
  import java.sql.Connection;
  import java.sql.DriverManager;
  import java.sql.ResultSet;
  import java.sql.SQLException;
  import java.sql.Statement;
  
  public class HiveJdbcClient {
  
  	private static String driverName = "org.apache.hive.jdbc.HiveDriver";
  
  	public static void main(String[] args) throws SQLException {
  		try {
  			Class.forName(driverName);
  		} catch (ClassNotFoundException e) {
  			e.printStackTrace();
  		}
  		Connection conn = DriverManager.getConnection("jdbc:hive2://node04:10000/default", "root", "");
  		Statement stmt = conn.createStatement();
  		String sql = "select * from psn limit 5";
  		ResultSet res = stmt.executeQuery(sql);
  		while (res.next()) {
  			System.out.println(res.getString(1) + "-" + res.getString("name"));
  		}
  	}
  }
  ```

  



















