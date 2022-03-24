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

  - 使用方法

    - set

      ```shell
      127.0.0.1:6379> set k1 liuhui
      ```

    - get

      ```shell
      127.0.0.1:6379> get k1
      "liuhui"
      ```

    - append

      ```shell
      127.0.0.1:6379> append k1 niubi
      127.0.0.1:6379> get k1
      "liuhuiniubi"
      ```

    - setrange

      ```shell
      127.0.0.1:6379> setrange k1 6 queshi
      127.0.0.1:6379> get k1
      "liuhuiqueshi"
      ```

    - getrange

      ```
      127.0.0.1:6379> getrange k1 0 5
      "liuhui"

    - strlen

- 数值

- bitmap

