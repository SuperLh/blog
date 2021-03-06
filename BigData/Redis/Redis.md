# Table of Contents

* [Redis](#redis)
  * [简介](#简介)
  * [Redis的优缺点](#redis的优缺点)
  * [Redis为什么速度快](#redis为什么速度快)
  * [Redis数据类型](#redis数据类型)
    * [String](#string)
    * [List](#List)
    * [Set](#Set)
    * [Sorted Set](#sorted-set)
  * [Redis发布订阅](#Redis发布订阅)
  * [Redis事务](#Redis事务)
  * [Redis扩展工具](#Redis扩展工具)
  * [Redis过期时间](#Redis过期时间)
  * [Redis回收策略](#Redis回收策略)
  * [Redis持久化](#Redis持久化)
    * [RDB](#RDB)
    * [AOF](#AOF)
  * [Redis集群](#Redis集群)
  * [Redis可采用的集群思路](#Redis可采用的集群思路)
    * [Redis主从复制](#Redis主从复制)
    * [Redis哨兵机制高可用](#Redis哨兵机制高可用)
  * [Redis缓存问题](#Redis缓存问题)




# Redis

## 简介

- Redis(Remote Dictionary Server)，远程数据字典服务，内存高速缓存数据库，数据模型是key-value
- Redis支持丰富的数据结构类型，比如字符串(string)、散列(hashes)、列表(lists)、集合(sets)、有序集合(sorted sets)



## Redis的优缺点

- **优点**
  - 速度快，数据存储在内存中，类似于HashMap
  - 支持丰富的数据类型
  - 支持事务，操作都是原子性的
  - 丰富的特性，可用于缓存，消息，按照Key设置过期时间，过期后自动删除
  - 支持数据持久化，支持AOF和RDB两种持久化方式
  - 支持主从复制，Master会自动将数据同步到Slave
- **缺点**
  - 单线程
  - 对一致性Hash支持有线
  - 持久化操作需要很大的系统开销
  - 数据库容量受到物理内存的限制
  - 不具备自动容错和自动恢复
  - 主机宕机，宕机前部分数据未同步，会导致数据不一致的问题
  - 较难支持在线扩容



## Redis为什么速度快

- 完全基于内存操作
- 数据结果简单，对数据操作简单
- 采用单线程，避免线程间的上下文切换，不用考虑线程安全和锁
- 使用多路I/O复用模型，非阻塞IO



- <font color='red'>**什么是多路I/O复用模型？**</font>

  - **BIO：同步并阻塞，服务器实现模式为一个连接一个线程**

     ![avatar](pics/bio.png)

    - 操作系统内核提供read(系统调用)，读文件描述符
    - 一个clinet连接就是一个文件描述符ID
    - socket连接是阻塞的，socket产生的文件描述符，如fd8，当数据包没有到达的时候，左边的read fd8不能返回，且阻塞
    - 即有一个clinet连接，就需要开一个进程（或者线程），有数据就处理，没有数据就阻塞

    

    - 缺点
      - 几个连接几个线程，一个CPU在某一时间片上，只能一个进程（线程）进行处理，就算另外一个数据到了，也不能进行处理，造成CPU资源浪费
      - 开线程数过多，成本很高
      
      

  - **NIO：同步非阻塞，服务器实现模式为一个请求一个线程**

     ![avatar](pics/nio.png)

    - linux内核提供的socket是可以非阻塞的

    - 既然socket不阻塞，那么一个进程（线程就够了），在进程里面写循环，即一个一个问fd有没有数据，即轮询，发生在用户空间遍历，取出来自己处理，为同步非阻塞

      

    - 缺点

      - C10K问题：如果有1w个客户端，代表用户进程需要访问1w次内核，用户态和内核态反复切换，成本开销很大
  
      
  
  - SELECT
  
      ![avatar](pics/select.png)
  
     - 假设有1000个fd，进程需要吧1000个fd春给select，内核监控这些fd，发现哪些fd准备好，则返回fd，然后进程再拿准备好的fd调用read
     - 即多路复用，选择谁数据有了，直接执行，减少用户态和内核态的切换
  
     
  
     - 缺点
       - 每次需要把1000个fd传进去，再返回，用户态和内核态需要fd传来传去
  
     
  
  - epoll
  
      ![avatar](pics/epoll.png)
  
     - epoll是一个整体，包含epoll_create,epoll_ctl,epoll_wait三个系统调用
     - 共享空间，进程把fd存放红黑树，内核通过红黑树拿fd去查询哪个IO数据到达，把到达的放到链表里，然后进程从链表去对应的fd
  
     - 大致过程
       - 进程首先调用epoll_create，创建一个epoll文件描述符
       - epoll通过mmap开辟一块共享空间（红黑树+链表），增删改由内核完成，查询则内核和用户进程都可以
       - 进程调用epoll的ctl add/delete sfd，把新来的连接放入红黑树中
       - 进程调用wait()，等待事件驱动
       - 当红黑树中的fd有数据到了，就把他放入链表中，并维护该数据是可写还是可读，wait返回
       - 上层空间通过epoll从链表中去除fd，然后调用read/write读写数据



## Redis数据类型

### String

- 字符串

- 普通字符串类型

  - 正反向索引

    ![avatar](pics/string_index.png)

  - 使用方法

    - set

      ```shell
      # 设置单个元素的值
      127.0.0.1:6379> set k1 liuhui
      ```
    
    - set nx
    
      ```shell
      # 只有当元素不存在时，才能设置成功
      127.0.0.1:6379> set k1 a nx
      OK
      127.0.0.1:6379> set k1 a nx
      (nil)
      ```
    
    - set xx
    
      ```shell
      # 只有当元素存在时，才能设置成功，即更新
      127.0.0.1:6379> set k1 a xx
      (nil)
      127.0.0.1:6379> set k1 a
      OK
      127.0.0.1:6379> set k1 b xx
      OK
      127.0.0.1:6379> get k1
      "b"
      ```

    - get
    
      ```shell
      # 获取单个元素的值
      127.0.0.1:6379> get k1
      "liuhui"
      ```
    
    - getset
    
      ```shell
      # 获取某个元素的当前值，并将新值赋值给他
      127.0.0.1:6379> getset k1 c
      "b"
      127.0.0.1:6379> get k1
      "c"
      ```
    
    - mset
    
      ```shell
      # 设置多个元素的值
      127.0.0.1:6379> mset k1 a k2 b
      OK
      ```
    
    - mget
    
      ```shell
      # 获取多个元素的值
      127.0.0.1:6379> mget k1 k2
      1) "a"
      2) "b"
      ```
    
    - append
    
      ```shell
      # 追加某个元素的值
      127.0.0.1:6379> append k1 niubi
      127.0.0.1:6379> get k1
      "liuhuiniubi"
      ```
    
    - setrange
    
      ```shell
      # 从指定位置开始设置新的值
      127.0.0.1:6379> setrange k1 6 queshi
      127.0.0.1:6379> get k1
      "liuhuiqueshi"
      ```
    
    - getrange
    
      ```shell
      # 截取字符串
      127.0.0.1:6379> getrange k1 0 5
      "liuhui"
      # 获取全部字符串
      127.0.0.1:6379> getrange k1 0 -1
      "liuhuiqueshi"
    
    - strlen
    
      ```shell
      # 获取字符串长度
      127.0.0.1:6379> strlen k1
      (integer) 16
      ```

- 数值

  - 数值类型

  - 使用方法

    - INCR

      ```shell
      # 初始化
      127.0.0.1:6379> set k1 99
      OK
      # 把当前数值增加1
      127.0.0.1:6379> incr k1
      (integer) 100
      # 把当前数值增加10
      127.0.0.1:6379> incrby k1 10
      (integer) 110
      ```

    - DECR

      ```shell
      # 把当前数值减1
      127.0.0.1:6379> decr k1
      (integer) 109
      # 把当前数值减10
      127.0.0.1:6379> decrby k1 10
      (integer) 99
      ```

    - INCRBYFLOAT

      ```shell
      # 把当前数值加0.5
      127.0.0.1:6379> incrbyfloat k1 0.5
      "99.5"
      # 把当前数值减0.5
      # 注意：没有DECRBYFLOAT方法，只能通过INCRBYFLOAT方法，将增加的数值设置为负数
      127.0.0.1:6379> incrbyfloat k1 -0.5
      "99"
      ```

- bitmap

  - 二进制位图

  - 位图存储形式

    ![avatar](pics/bitmap_index.png)

  - 使用方法

    - setbit

      ```shell
      # 设置位图的值，设置k1位图的第1位为1，即0100 0000
      127.0.0.1:6379> setbit k1 1 1
      (integer) 0
      127.0.0.1:6379> get k1
      "@"
      # 设置位图的值，设置k1位图的第7位为1，即0100 0001
      127.0.0.1:6379> setbit k1 7 1
      (integer) 0
      127.0.0.1:6379> get k1
      "A"
      ```

    - strlen

      ```shell
      # 查看位图的长度（字节长度）
      127.0.0.1:6379> strlen k1
      (integer) 1
      # 设置位图的值，设置k1位图的第9位为1，即0100 0001 0100 0000，此时k1的长度应该为2
      127.0.0.1:6379> setbit k1 9 1
      (integer) 0
      127.0.0.1:6379> strlen k1
      (integer) 2
      ```

    - bitpos

      ```shell
      # 查看k1位图，第0个字节，第一个出现1的位置
      127.0.0.1:6379> bitpos k1 1 0 0
      (integer) 1
      # 查看k1位图，第1个字节，第一个出现1的位置
      127.0.0.1:6379> bitpos k1 1 1 1
      (integer) 9
      # 查看k1位图，第0-1个字节，第一个出现1的位置
      127.0.0.1:6379> bitpos k1 1 0 1
      (integer) 1
      ```

    - bitcount

      ```shell
      # 查看k1位图，第0个字节，1出现的次数
      127.0.0.1:6379> bitcount k1 0 0
      (integer) 2
      # 查看k1位图，第1个字节，1出现的次数
      127.0.0.1:6379> bitcount k1 1 1
      (integer) 1
      # 查看k1位图，第0-1个字节，1出现的次数
      127.0.0.1:6379> bitcount k1 0 1
      (integer) 3
      ```

    - bitop

      ```shell
      127.0.0.1:6379> setbit k1 1 1
      (integer) 0
      127.0.0.1:6379> setbit k1 7 1
      (integer) 0
      127.0.0.1:6379> get k1
      "A"
      127.0.0.1:6379> setbit k2 1 1
      (integer) 0
      127.0.0.1:6379> setbit k2 6 1
      (integer) 0
      127.0.0.1:6379> get k2
      "B"
      # 位图操作，k1和k2进行与操作，并赋值给k3
      127.0.0.1:6379> bitop and k3 k1 k2
      (integer) 1
      127.0.0.1:6379> get k3
      "@"
      # 位图操作，k1和k2进行或操作，并赋值给k4
      127.0.0.1:6379> bitop or k4 k1 k2
      (integer) 1
      127.0.0.1:6379> get k4
      "C"
      ```


- bitmap的应用

  - 用户系统：统计用户登录天数

    ```shell
    setbit username1 1 1
    setbit username1 7 1
    setbit username1 354 1
    # 最后两天的登录次数
    bitcount username1 -2 -1

  - 用户系统

    ```shell
    # setbit 日期 用户id 登录标识
    setbit 20200101 1 1
    setbit 20200102 1 1
    setbit 20200102 7 1
    # 日期间进行或操作
    bitop or destkey 20190101 20190102
    # 统计活跃用户数
    bitcount destkey 0 -1
    ```

    

### List

- 存储模型

  ![avatar](pics/redis_list.png)

- 使用方法

  - lpush

    ```shell
    # lpush，从左向右push元素
    127.0.0.1:6379> lpush k1 a b c d e f
    (integer) 6
    ```

  - rpush

    ```shell
    # rpush，从右向左push元素
    127.0.0.1:6379> rpush k2 a b c d e f
    (integer) 6
    ```

  - lrange

    ```shell
    # 取出元素，从第0位，到-1位
    127.0.0.1:6379> lrange k1 0 -1
    1) "f"
    2) "e"
    3) "d"
    4) "c"
    5) "b"
    6) "a"
    127.0.0.1:6379> lrange k2 0 -1
    1) "a"
    2) "b"
    3) "c"
    4) "d"
    5) "e"
    6) "f"
    127.0.0.1:6379> 
    ```

  - lpop

    ```shell
    # 从左边弹出元素
    127.0.0.1:6379> lpop k1
    "f"
    127.0.0.1:6379> lpop k1
    "e"
    127.0.0.1:6379> lpop k1
    "d"
    ```

  - rpop

    ```shell
    # 从右边弹出元素
    127.0.0.1:6379> rpop k1
    "a"
    127.0.0.1:6379> rpop k1
    "b"
    127.0.0.1:6379> rpop k1
    "c"
    ```

  - lpush <-> lpop

    ```shell
    #从左边push元素，从左边pop元素，相当于栈，先进后出
    127.0.0.1:6379> lpush k1 a b c d
    (integer) 4
    127.0.0.1:6379> lpop k1 
    "d"
    127.0.0.1:6379> lpop k1 
    "c"
    127.0.0.1:6379> lpop k1 
    "b"
    127.0.0.1:6379> lpop k1 
    "a"
    ```

  - lpush <-> rpop

    ```shell
    #从左边push元素，从右边pop元素，相当于队列，先进先出
    127.0.0.1:6379> lpush k1 a b c d
    (integer) 4
    127.0.0.1:6379> rpop k1
    "a"
    127.0.0.1:6379> rpop k1
    "b"
    127.0.0.1:6379> rpop k1
    "c"
    127.0.0.1:6379> rpop k1
    "d"
    ```

  - lindex

    ```shell
    #取第N个元素的值
    127.0.0.1:6379> rpush k1 a b c d
    (integer) 4
    127.0.0.1:6379> lindex k1 2
    "c"
    ```

  - lset

    ```shell
    #设置第N个元素的值
    127.0.0.1:6379> rpush k1 a b c d
    (integer) 4
    127.0.0.1:6379> lset k1 2 g
    OK
    127.0.0.1:6379> lrange k1 0 -1
    1) "a"
    2) "b"
    3) "g"
    4) "d"
    ```

  - lrem

    ```shell
    127.0.0.1:6379> rpush k1 a b a c a d a e
    (integer) 8
    127.0.0.1:6379> lrange k1 0 -1
    1) "a"
    2) "b"
    3) "a"
    4) "c"
    5) "a"
    6) "d"
    7) "a"
    8) "e"
    #移除k1前2个值为a的元素
    127.0.0.1:6379> lrem k1 2 a
    (integer) 2
    127.0.0.1:6379> lrange k1 0 -1
    1) "b"
    2) "c"
    3) "a"
    4) "d"
    5) "a"
    6) "e"
    #移除k1后1个值为a的元素
    127.0.0.1:6379> lrem k1 -1 a
    (integer) 1
    127.0.0.1:6379> lrange k1 0 -1
    1) "b"
    2) "c"
    3) "a"
    4) "d"
    5) "e"
    ```

  - linsert

    ```shell
    127.0.0.1:6379> rpush k1 a b c d e 
    (integer) 5
    #在元素c后面添加一个g
    127.0.0.1:6379> linsert k1 after c g
    (integer) 6
    127.0.0.1:6379> lrange k1 0 -1
    1) "a"
    2) "b"
    3) "c"
    4) "g"
    5) "d"
    6) "e"
    #在元素b之前添加一个f
    127.0.0.1:6379> linsert k1 before b f
    (integer) 7
    127.0.0.1:6379> lrange k1 0 -1
    1) "a"
    2) "f"
    3) "b"
    4) "c"
    5) "g"
    6) "d"
    7) "e"
    ```

  - llen

    ```shell
    127.0.0.1:6379> llen k1
    (integer) 7
    ```

  - blpop

    ```shell
    #阻塞直到取到元素
    127.0.0.1:6379> blpop k1 0
    #阻塞直到取到元素，最大时间为10s
    127.0.0.1:6379> blpop k1 10
    ```

  - ltrim

    ```shell
    #移除指定区间外的所有元素
    127.0.0.1:6379> rpush k1 a b c d e f
    (integer) 6
    127.0.0.1:6379> ltrim k1 2 -3
    OK
    127.0.0.1:6379> lrange k1 0 -1
    1) "c"
    2) "d"
    ```



### Hash

- 键值对

- 使用方法

  - hset

    ```shell
    127.0.0.1:6379> hset person name liuhui
    (integer) 1
    ```

  - hget

    ```shell
    127.0.0.1:6379> hget person name
    "liuhui"
    ```

  - hmset

    ```shell
    127.0.0.1:6379> hmset person name liuhui age 18 sex man
    OK
    ```

  - hmget

    ```shell
    127.0.0.1:6379> hmget person name age sex
    1) "liuhui"
    2) "18"
    3) "man"
    ```

  - hkeys

    ```shell
    127.0.0.1:6379> hkeys person
    1) "name"
    2) "age"
    3) "sex"
    ```

  - hvals

    ```shell
    127.0.0.1:6379> hvals person
    1) "liuhui"
    2) "18"
    3) "man"
    ```

  - hgetall

    ```shell
    127.0.0.1:6379> hgetall person
    1) "name"
    2) "liuhui"
    3) "age"
    4) "18"
    5) "sex"
    6) "man"
    ```

  - hincrbyfloat

    ```shell
    127.0.0.1:6379> hincrbyfloat person age 0.5
    "18.5"
    127.0.0.1:6379> hget person age
    "18.5"
    127.0.0.1:6379> hincrbyfloat person age -0.5
    "18"
    127.0.0.1:6379> hget person age
    "18"
    ```

- hincrby/hincrbyfloat的应用

  - 点赞、收藏、商品详情页等



### Set

- 无序且随机

- 使用方法

  - sadd

    ```shell
    127.0.0.1:6379> sadd k1 a b c d e f
    (integer) 6
    ```

  - smembers

    ```shell
    127.0.0.1:6379> smembers k1
    1) "b"
    2) "a"
    3) "d"
    4) "f"
    5) "e"
    6) "c"
    ```

  - srem

    ```shell
    127.0.0.1:6379> srem k1 a
    (integer) 1
    127.0.0.1:6379> smembers k1
    1) "d"
    2) "f"
    3) "e"
    4) "c"
    5) "b"
    ```

  - sinter

    ```shell
    127.0.0.1:6379> sadd k1 a b c d 
    (integer) 4
    127.0.0.1:6379> sadd k2 c d e f
    (integer) 4
    #取k1,k2的交集
    127.0.0.1:6379> sinter k1 k2
    1) "d"
    2) "c"
    #取k1,k2的交集，并存储在k3
    127.0.0.1:6379> sinterstore k3 k1 k2
    (integer) 2
    127.0.0.1:6379> smembers k3
    1) "c"
    2) "d"
    ```

  - sunion

    ```shell
    127.0.0.1:6379> sunion k1 k2
    1) "d"
    2) "e"
    3) "f"
    4) "a"
    5) "c"
    6) "b"
    127.0.0.1:6379> sunionstore k3 k1 k2
    (integer) 6
    127.0.0.1:6379> smembers k3
    1) "d"
    2) "e"
    3) "f"
    4) "a"
    5) "c"
    6) "b"
    ```

  - sdiff

    ```shell
    #注意前后顺序
    127.0.0.1:6379> sdiff k1 k2
    1) "a"
    2) "b"
    127.0.0.1:6379> sdiff k2 k1
    1) "e"
    2) "f"
    127.0.0.1:6379> sdiffstore k3 k1 k2
    (integer) 2
    127.0.0.1:6379> smembers k3
    1) "a"
    2) "b"
    127.0.0.1:6379> sdiffstore k3 k2 k1
    (integer) 2
    127.0.0.1:6379> smembers k3
    1) "e"
    2) "f"
    127.0.0.1:6379> 
    ```

  - spop

    ```shell
    #随机弹出元素
    127.0.0.1:6379> spop k1
    "a"
    127.0.0.1:6379> spop k1
    "c"
    127.0.0.1:6379> spop k1
    "b"
    127.0.0.1:6379> spop k1
    "d"
    127.0.0.1:6379> spop k1
    (nil)
    ```

  - srandom

    ```shell
    #count参数为正：取出一个去重的结果集，不会超过当前集合
    127.0.0.1:6379> sadd k1 a b c d e f
    (integer) 6
    127.0.0.1:6379> srandmember k1 3
    1) "e"
    2) "a"
    3) "b"
    127.0.0.1:6379> srandmember k1 10
    1) "b"
    2) "a"
    3) "d"
    4) "f"
    5) "e"
    6) "c"
    #count参数为负：取出一个带重复的结果集，一定满足数据量，不超过当前set长度也有可能会重复
    127.0.0.1:6379> srandmember k1 -2
    1) "e"
    2) "e"
    127.0.0.1:6379> srandmember k1 -10
     1) "e"
     2) "b"
     3) "a"
     4) "c"
     5) "e"
     6) "a"
     7) "f"
     8) "f"
     9) "e"
    10) "a"
    ```




### Sorted Set

- 有序集合

- 使用方法

  - zadd

    ```shell
    #赋值元素，分值和元素
    127.0.0.1:6379> zadd k1 8 apple 2 banana 3 orange
    (integer) 3
    ```

  - zrange

    ```shell
    #从小到大取出元素
    127.0.0.1:6379> zrange k1 0 -1
    1) "banana"
    2) "orange"
    3) "apple"
    #从小到大取出元素，并显示对应分数
    127.0.0.1:6379> zrange k1 0 -1 withscores
    1) "banana"
    2) "2"
    3) "orange"
    4) "3"
    5) "apple"
    6) "8"
    ```

  - zrevrange

    ```shell
    #从大到小取出元素
    127.0.0.1:6379> zrevrange k1 0 -1
    1) "apple"
    2) "orange"
    3) "banana"
    ```

  - zrangebyscore

    ```shell
    #按照分数取出元素，最小为3，最大为8
    127.0.0.1:6379> zrangebyscore k1 3 8
    1) "orange"
    2) "apple"
    ```

  - zsocre

    ```shell
    #取对应元素的分值
    127.0.0.1:6379> zscore k1 banana
    "2"
    ```

  - zrank

    ```shell
    #取对应元素的排名
    127.0.0.1:6379> zrank k1 apple
    (integer) 2
    ```

  - zincrby

    ```shell
    #对对应元素的分值进行修改，并会重新进行排序
    127.0.0.1:6379> zadd k1 8 apple 2 banana 3 orange
    (integer) 3
    127.0.0.1:6379> zrange k1 0 -1 withscores
    1) "banana"
    2) "2"
    3) "orange"
    4) "3"
    5) "apple"
    6) "8"
    127.0.0.1:6379> zincrby k1 2.5 banana
    "4.5"
    127.0.0.1:6379> zrange k1 0 -1 withscores
    1) "orange"
    2) "3"
    3) "banana"
    4) "4.5"
    5) "apple"
    6) "8"
    ```

  - zunion

    ```shell
    #将k1和k2合并成k3，默认权重为1，聚合公式为sum
    127.0.0.1:6379> zadd k1 60 user1 40 user2 30 user3
    (integer) 3
    127.0.0.1:6379> zadd k2 20 user1 80 user2 40 user3
    (integer) 3
    127.0.0.1:6379> zunionstore k3 2 k1 k2
    (integer) 3
    127.0.0.1:6379> zrange k3 0 -1 withscores
    1) "user3"
    2) "70"
    3) "user1"
    4) "80"
    5) "user2"
    6) "120"
    #将k1和k2合并成k3，设置k1权重为1，k2权重为0.5，默认聚合公式为sum
    127.0.0.1:6379> zadd k1 60 user1 40 user2 30 user3
    (integer) 3
    127.0.0.1:6379> zadd k2 20 user1 80 user2 40 user3
    (integer) 3
    127.0.0.1:6379> zunionstore k3 2 k1 k2 weights 1 0.5
    (integer) 3
    127.0.0.1:6379> zrange k3 0 -1 withscores
    1) "user3"
    2) "50"
    3) "user1"
    4) "70"
    5) "user2"
    6) "80"
    #将k1和k2合并成k3，设置k1权重为1，k2权重为0.5，聚合公式为max
    127.0.0.1:6379> zadd k1 60 user1 40 user2 30 user3
    (integer) 3
    127.0.0.1:6379> zadd k2 20 user1 80 user2 40 user3
    (integer) 3
    127.0.0.1:6379> zunionstore k3 2 k1 k2 weights 1 0.5 aggregate max
    (integer) 3
    127.0.0.1:6379> zrange k3 0 -1 withscores
    1) "user3"
    2) "30"
    3) "user2"
    4) "40"
    5) "user1"
    6) "60"
    ```




## Redis发布订阅

- **使用方法**

  - 发布者

    ```shell
    #client1开启一个redis客户端作为发布者
    127.0.0.1:6379> publish k1 ceshi
    (integer) 0
    ```

  - 订阅者

    ```shell
    #client2开启一个redis客户端作为订阅者
    127.0.0.1:6379> subscribe k1
    Reading messages... (press Ctrl-C to quit)
    1) "subscribe"
    2) "k1"
    3) (integer) 1
    ```

  - 发布者持续发布消息

    ```shell
    127.0.0.1:6379> publish k1 1
    (integer) 1
    127.0.0.1:6379> publish k1 2
    (integer) 1
    127.0.0.1:6379> publish k1 3
    (integer) 1
    ```

  - 订阅者会收到相应的消息

    ```shell
    127.0.0.1:6379> subscribe k1
    Reading messages... (press Ctrl-C to quit)
    1) "subscribe"
    2) "k1"
    3) (integer) 1
    1) "message"
    2) "k1"
    3) "1"
    1) "message"
    2) "k1"
    3) "2"
    1) "message"
    2) "k1"
    3) "3"
    ```

- **发布订阅的使用场景：聊天系统**

  ![avatar](pics/redis_publish_subscibe.png)



## Redis事务

- <font color='red'>**Redis是单进程的，所以哪个事务的exec先进到执行队列中，会先执行谁的事务**</font>

- **使用方法**

  ```shell
  127.0.0.1:6379> multi
  OK
  127.0.0.1:6379(TX)> set k1 a
  QUEUED
  127.0.0.1:6379(TX)> get k1
  QUEUED
  127.0.0.1:6379(TX)> exec
  1) OK
  2) "a"
  ```

- watch

  - watch会监控某一个值，当这个值在事务中发生了变化，结果会返回nil

  - 使用方法

    ```shell
    #watch k1 之后，k1没有变化，结果成功返回
    127.0.0.1:6379> watch k1
    OK
    127.0.0.1:6379> multi
    OK
    127.0.0.1:6379(TX)> get k1
    QUEUED
    127.0.0.1:6379(TX)> exec
    1) "a"
    
    #watch k1 之后，k1发生变化，结果返回nil
    #client1
    127.0.0.1:6379> set k1 a
    OK
    127.0.0.1:6379> watch k1
    OK
    127.0.0.1:6379> multi
    OK
    127.0.0.1:6379(TX)> get k1
    QUEUED
    
    #此时打开client2，改变k1的值
    127.0.0.1:6379> set k1 b
    OK
    
    #client1执行exec操作之后会返回nil
    127.0.0.1:6379(TX)> exec
    (nil)
    ```



## Redis扩展工具

- **布隆过滤器**

- **原理**

  ![avatar](E:\Private\git\blog\BigData\Redis\pics\redis_bloom.png)

- **使用方法**

  - 将现在已有的元素，通过映射函数，向bitmap中进行标记
  - 当有新元素需要向数据库发生查询请求时，先通过映射函数进行匹配
  - 当数据库增加元素时，需要完成元素映射函数对bitmap的标记
  - <font color='red'>**布隆过滤器只能概率解决问题，会大概率减少放行和穿透，成本低**</font>

- **布隆过滤器的开启方式**

  ```shell
  #第一种，启动服务加命令行
  redis-server --loadmodule /APP/redis/redisbloom.so
  
  #第二种，修改配置文件
  ################################## MODULES #####################################
  
  # Load modules at startup. If the server is not able to load modules
  # it will abort. It is possible to use multiple loadmodule directives.
  #
  # loadmodule /path/to/my_module.so
  # loadmodule /path/to/other_module.so
  loadmodule /APP/redis/redisbloom.so
  ```



## Redis过期时间

- Redis作为缓存，可以不考虑数据的完整性，对冷热数据进行缓存

- 使用方法

  ```shell
  #设置一个过期时间为20s的key
  127.0.0.1:6379> set k1 a ex 20
  OK
  
  #查询某个key的剩余过期时间
  127.0.0.1:6379> ttl k1
  (integer) 16
  127.0.0.1:6379> ttl k1
  (integer) 15
  127.0.0.1:6379> ttl k1
  (integer) 14
  
  #给已存在的key设置过期时间
  127.0.0.1:6379> set k1 a
  OK
  127.0.0.1:6379> expire k1 20
  (integer) 1
  127.0.0.1:6379> ttl k1
  (integer) 18
  127.0.0.1:6379> ttl k1
  (integer) 17
  127.0.0.1:6379> ttl k1
  (integer) 16
  
  #重新设置key会覆盖之前key的过期时间
  127.0.0.1:6379> set k1 a ex 30
  OK
  127.0.0.1:6379> ttl k1
  (integer) 28
  127.0.0.1:6379> ttl k1
  (integer) 27
  127.0.0.1:6379> set k1 b
  OK
  127.0.0.1:6379> ttl k1
  (integer) -1
  
  #指定过期时间删除的key
  127.0.0.1:6379> expireat k1 1293840000
  (integer) 0
  ```

- **Redis如何淘汰过期的keys**

  - Redis的key过期有两种方式，被动和主动
    - 被动
      - 当客户端访问一些key时，key会被发现并主动地过期
      
    - 主动
      - Redis每10s会进行随机20个key的过期检测
      
      - 删除这20个key中所有已经过期的keys
      
      - 如果有多于25%的key被删除，那么重复执行此操作
      
        
  

## Redis回收策略

- 当Redis作为缓存来使用时，当新增数据时，让他自动回收数据是件很方便的事情

- LRU是Redis唯一支持的回收方法

- MaxMemory配置指令

  - 用于配置Redis存储数据时指定限制的内存大小，通过redis.conf进行设置

    ```shell
    maxmemory 100mb
    ```

- 回收策略

  - noevicition：当内存达到限制，且客户端常事执行会让更多内存被使用的命令时，返回错误
  - allkeys-lru：尝试回收最久没有使用的key（LRU）
  - volatile-lru：尝试回收最久没有使用的key，但仅限于是设置了过期时间的key
  - allkeys-random：回收随机的key
  - volatile-random：回收随机的key，但仅限于是设置了过期时间的key
  - volatile-ttl：回收设置了过期时间的键，并且优先回收剩余存活时间较短的key
  - allkeys-lfu：尝试回收使用频率最小的key（LFU）
  - volatile-lfu：尝试回收使用频率最小的key（LFU）



## Redis持久化

### RDB

- **基本介绍**

  - RBD是一种时点性的快照文件，将Redis中的内存数据以快照的形式保存到硬盘中

- **优点**

  - 容灾性好，一个文件可以保存到安全的磁盘
  - 性能最大化，fork子进程来完成对数据的保存操作，主进程依旧处理客户端命令，主进程不会进行任何IO操作，保证Redis的性能
  - 过程类似于Java序列化，恢复的速度相对较快

- **缺点**

  - 不支持拉链操作，磁盘中只存在一个dump.rdb，没有历史版本，会对之前的dump.rdb文件进行覆盖
  - 丢失数据相对较多，时点与时点之间数据容易丢失，比如8点备份了一个rdb，在9点刚要备份一个新的rdb时，服务器宕机，8点~9点的数据全部丢失

- **配置**

  ```shell
  # Unless specified otherwise, by default Redis will save the DB:
  #   * After 3600 seconds (an hour) if at least 1 key changed
  #   * After 300 seconds (5 minutes) if at least 100 keys changed
  #   * After 60 seconds if at least 10000 keys changed
  #
  # You can set these explicitly by uncommenting the three following lines.
  #
  
  # 3600s内，至少有1个key发生变化时，进行持久化
  save 3600 1
  # 300s内，至少有100个key发生变化时，进行持久化
  save 300 100
  # 60s内，至少有1000个key发生变化时，进行持久化
  save 60 10000
  
  # 持久化的快照文件名称
  # The filename where to dump the DB
  dbfilename dump.rdb
  
  # 持久化的快照文件目录
  # Note that you must specify a directory here, not a file name.
  dir /APP/software/redis_data/data/6379
  
  # 一些不是很重要的参数
  
  # 持久化出错 是否需要继续工作
  stop-writes-on-bgsave-error yes
   
  # 是否压缩rdb文件 会消耗一些cpu资源
  rdbcompression yes
   
  # 保存rdb文件时，是否校验rdb文件
  rdbchecksum yes
  ```

- **触发方式**

  - 手动触发（场景：服务器关机维护）
    - redis客户端中执行save命令
  - 当save规则满足时
  - 执行flushall
  - 退出redis
  
- **实现原理**

  - 父子进程之间的数据共享
    - 常规思想：在父子进程之间，数据是需要进行隔离的
    - 进阶思想：在父子进程之间，父进程可以让子进程看到数据，比如说，通过export的环境变量，子进程是可以看到父进程的数据
    - 父进程对数据的修改，并不会影响子进程的数据，子进程对数据的修改，也不会影响父进程
  - 写时复制
    - Redis父进程，通过fork一个子进程对数据进行持久化
    - 父进程fork一个子进程时，并不是把所有的内存数据复制了一份，而是把指针复制了一份
      - 比如父进程某个key（k1）的指针指向了内存空间中的某块地址（1），值为3
      - 那么fork的子进程的这个key（k1）也会指向内存空间中的这块地址（1），值为3

    - 当父进程对这个key（k1）进行修改时（修改为4），父进程不会对当前内存空间中的这块地址（1），进行修改
    - 而是创建一个新的值，这个值的内存空间地址（2），值为4，然后将父进程的这个key（k1），指向新的内存空间（2）
    - 这样父进程对数据的修改，并不会对子进程，即RDB持久化进程造成影响
    - 这样子进程进行持久化的RDB文件，就会是某个时刻的Redis内存数据




### AOF

- **基本介绍**

  - 日志形式的持久化形式，将Redis的写操作记录到文件中

- **优点**

  - 丢失数据少
  - 可以在AOP文件没被rewrite之前删除一些误操作，比如flushall

- **缺点**

  - 性能较差，恢复数据的速度较慢
  - 文件会无限叠加，无限变大

- **配置**

  ```shell
  # 开启AOF持久化，默认是RDB
  appendonly yes
  
  # AOF持久化文件名称
  appendfilename "appendonly.aof"
  
  # appendonly.aof文件大小超过基准的百分之多少之后会触发rewrite
  auto-aof-rewrite-percentage 100
  # appendonly.aof文件超过多少字节之后会触发rewrite
  auto-aof-rewrite-min-size 64mb
  # 上述两个配置文件分别会在appendonly.aof文件达到64mb,128mb...之后触发rewrite
  
  # 总是写入aof，并完成磁盘同步
  appendfsync always
  # 每秒写入aof，并完成磁盘同步
  appendfsync everysec
  # 写入aof，不等待磁盘同步
  appendfsync no
  ```

- **过程**

  - Redis4.0之前
    - AOF文件的重写，会删除抵销的命令，合并重复的命令，最后形成的是一个纯指令型的日志文件
  - Redis4.0之后
    - AOF文件的重写，会将老的数据以RDB形式存储到AOF文件中，增量的数据会以命令的形式append到AOF文件中
    - 最后形成的是一个RDB和AOF的混合文件



## Redis集群

## Redis可采用的集群思路

- 同步阻塞
  - 客户端向Redis主节点发送请求，Redis主节点等待子节点全部返回成功后，再返回给客户端
  - 强一致性，但是破坏了可用性
- 异步非阻塞
  - 客户端向Redis主节点发送请求，Redis主节点向子节点发送请求，不等待子节点的返回结果，直接返回给客户端
  - 弱一致性，保护了可用性，但是会丢失数据
- 中间件同步阻塞
  - 客户端向Redis主节点发送请求，Redis主节点发送给Kafka中间件，这一步是同步阻塞
  - Kafka中间件向Redis子节点发送请求
  - 实现了最终一致性，但是有可能会取到不同的数据（主节点已更新，子节点没有更新）
- Redis采用的是异步非阻塞，特点是延迟低，高性能



### Redis主从复制

- 配置文件

  ```shell
  # 当一个slave失去和master的连接，或者同步正在进行中，slave的行为有两种可能：
  # 1) 如果 replica-serve-stale-data 设置为 "yes" (默认值)，slave会继续响应客户端请求，可能是正常数据，也可能是还没获得值的空数据。
  # 2) 如果 replica-serve-stale-data 设置为 "no"，slave会回复"正在从master同步（SYNC with master in progress）"来处理各种请求，除了 INFO 和 SLAVEOF 命令。
  replica-serve-stale-data yes
  
  # slave子节点是否支持写入
  replica-read-only yes
  
  # 同步策略：磁盘或者socker，默认是用磁盘
  repl-diskless-sync no
  
  # 如果非磁盘同步方式开启，可以配置同步延迟时间，以等待master产生子进程通过socket传输RDB数据给slave。
  # 默认值为5秒，设置为0秒则每次传输无延迟。
  repl-diskless-sync-delay 5
  
  # 数据备份的backlog大小
  repl-backlog-size 1mb
  ```

- 使用方法

  - 例如开启3个Redis Server，端口分别为6379、6380、6381
  - 6379为主节点，6380、6381为子节点
  - 在6380、6381的Redis服务中中执行replicaof 127.0.0.1 6379
  - 6380、6381的Redis服务会flushdb，并将6379的数据进行同步
  - 这样就实现了主从复制

- 当其中某个子节点宕机

  - 在宕机的子节点中开启Redis服务
    - redis.server 6380.conf --replicaof 127.0.0.1 6379
  - 可以发现这次开启的Redis服务并没有进行flushdb操作，说明主从复制是支持增量同步的
  - 但是如果Redis持久化选用的方式是AOF，那么开启服务时，还是会flushdb，进行全量同步



### Redis哨兵机制高可用

- Redis集群，如果主节点挂掉之后，其余的子节点只能进行读操作，不能进行写操作，这时我们需要对主节点做一个高可用

- 哨兵

  - 配置文件（vi 26379.conf）

    ```shell
    # 哨兵端口号
    port 26379
    # 监控主节点ip和端口，最后的数量为投票机器数，建议：集群总数的一半+1
    sentinel monitor mymaster 127.0.0.1 6379 2
    ```

  - 使用方法

    ```shell
    # 直接开启哨兵
    redis-sentienl 26379.conf
    # 开启一个Redis服务，告诉他你是哨兵
    redis-server  26379.conf --sentinel
    ```



## Redis缓存问题

- **缓存雪崩**
  - 现象
    - 缓存在同一时间大面积失效，导致所有请求同时落到数据库，造成数据库崩溃
  - 解决方案
    - 设置增加随机值的过期时间
    - 并发量不大的时候，进行加锁排队
- **缓存穿透**
  - 现象
    - 缓存和数据库中都没有该数据，导致所有请求都落在数据库，造成数据库崩溃
  - 解决方案
    - 接口层增加校验
    - 缓存中获取不到的数据，可以将k-v设置成k-null，进行短时间的缓存
    - 采用布隆过滤器
- **缓存击穿**
  - 现象
    - 缓存中没有，但是数据库中存在（一般是缓存时间到期，并且并发量很大），导致同时去数据库中请求，造成数据库崩溃
    - 与缓存雪崩问题不同，缓存击穿是并发查询同一条数据
  - 解决方案：
    - 热点数据设置永不过期
    - 增加互斥锁
- **缓存预热**
  - 现象
    - 系统上线后，将相关的缓存直接加载到缓存系统，避免用户请求时先查询数据，然后再将数据缓存的问题
  - 解决方案
    - 增加缓存刷新页面，手动加载
    - 数据量不大的时候，可以在项目启动的时候，自动进行加载
    - 定时刷新缓存数据



