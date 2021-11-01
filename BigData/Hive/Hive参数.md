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

