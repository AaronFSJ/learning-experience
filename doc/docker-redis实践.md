# docker安装redis集群

## redis基本概念

- ### Redis 与其他 key – value 缓存产品有以下三个特点：

  1. Redis支持数据的持久化，可以将内存中的数据保持在磁盘中，重启的时候可以再次加载进行使用。
  2. Redis不仅仅支持简单的key-value类型的数据，同时还提供list，set，zset，hash等数据结构的存储。
  3. Redis支持数据的备份，即master-slave模式的数据备份。

  ![](https://raw.githubusercontent.com/AaronFSJ/learning-experience/master/images/redis-advance.png)

概念详情查看<a href="http://blog.jobbole.com/112024/" target="_blank">Redis 核心概念</a>，更多查看<a href="http://blog.jobbole.com/tag/redis/" target=_blank>伯乐在线—>redis</a>

- ### 

- ### Redis持久化存储(AOF与RDB两种模式)

  - RDB

    RDB是在某个时间点将数据写入一个临时文件，持久化结束后，用这个临时文件替换上次持久化的文件，达到数据恢复。 

    优点：**使用单独子进程来进行持久化，主进程不会进行任何IO操作，保证了redis的高性能** 

    缺点：**RDB是间隔一段时间进行持久化，如果持久化之间redis发生故障，会发生数据丢失。所以这种方式更适合数据要求不严谨的时候**

    这里说的这个执行数据写入到临时文件的时间点是可以通过配置来自己确定的，通过配置**redis在n秒内如果超过m个key被修改**这执行一次RDB操作。这个操作就类似于在这个时间点来保存一次Redis的所有数据，一次快照数据。所有这个持久化方法也通常叫做snapshots。

    RDB默认开启，redis.conf中的具体配置参数如下；

    ```shell
    #dbfilename：持久化数据存储在本地的文件
    dbfilename dump.rdb
    #dir：持久化数据存储在本地的路径，如果是在/redis/redis-3.0.6/src下启动的redis-cli，则数据会存储在当前src目录下
    dir ./
    ##snapshot触发的时机，save <seconds> <changes>  
    ##如下为900秒后，至少有一个变更操作，才会snapshot  
    ##对于此值的设置，需要谨慎，评估系统的变更操作密集程度  
    ##可以通过“save “””来关闭snapshot功能  
    #save时间，以下分别表示更改了1个key时间隔900s进行持久化存储；更改了10个key300s进行存储；更改10000个key60s进行存储。
    save 900 1
    save 300 10
    save 60 10000
    ##当snapshot时出现错误无法继续时，是否阻塞客户端“变更操作”，“错误”可能因为磁盘已满/磁盘故障/OS级别异常等  
    stop-writes-on-bgsave-error yes  
    ##是否启用rdb文件压缩，默认为“yes”，压缩往往意味着“额外的cpu消耗”，同时也意味这较小的文件尺寸以及较短的网络传输时间  
    rdbcompression yes  
    ```

    snapshot触发的时机，是有“间隔时间”和“变更次数”共同决定，同时符合2个条件才会触发snapshot,否则“变更次数”会被继续累加到下一个“间隔时间”上。snapshot过程中并不阻塞客户端请求。snapshot首先将数据写入临时文件，当成功结束后，将临时文件重名为dump.rdb。

    使用RDB恢复数据： 
    自动的持久化数据存储到dump.rdb后。实际只要重启redis服务即可完成（启动redis的server时会从dump.rdb中先同步数据）

    客户端使用命令进行持久化save存储：

    ```shell
    ./redis-cli -h ip -p port save
    ./redis-cli -h ip -p port bgsave
    ```

    一个是在前台进行存储，一个是在后台进行存储。我的client就在server这台服务器上，所以不需要连其他机器，直接./redis-cli bgsave。由于redis是用一个主线程来处理所有 client的请求，这种方式会阻塞所有client请求。所以不推荐使用。另一点需要注意的是，每次快照持久化都是将内存数据完整写入到磁盘一次，并不是增量的只同步脏数据。如果数据量大的话，而且写操作比较多，必然会引起大量的磁盘io操作，可能会严重影响性能。

##redis安装

- 拉取镜像

  ```
  docker pull redis
  ```

- 下载对应镜像版本的redis到服务器,用来获取对应的redis.conf，如

  - 查看redis镜像版本

    ```shell
    docker inspect --format "{{.Config}}" redis
    ```

  - 如查到REDIS_VERSION=4.0.7 REDIS_DOWNLOAD_URL=http://download.redis.io/releases/redis-4.0.7.tar.gz

    ```shell
    cd /home/soft
    wget http://download.redis.io/releases/redis-4.0.7.tar.gz
    tax -xvf redis-4.0.7.tar.gz
    ```

- 将下载并解决的redis的redis.conf放到自己习惯的目录下备用，并修改配置

  ```shell
  cd /home/volumes/redis
  cp /home/soft/redis-4.0.7/redis.conf ./
  #替换slaveof <masterip> <masterport>为slaveof redis-master 6379
  sed -i 's/# slaveof <masterip> <masterport>/slaveof redis-master 6379/g' redis.conf 
  ```

- 创建redis主数据库 -v 挂载宿主主机的/home/docker/redis/redis.conf到目标主机的/usr/local/bin/redis.conf上

  ```shell
  docker run --link redis-master:redis-master -v /home/docker/redis/redis.conf:/usr/local/bin/redis.conf --name redis-slave1 -d redis redis-server /usr/local/bin/redis.conf
  ```

- 创建从数据库

  ```shell
  #创建slave1
  docker run --link redis-master:redis-master -v /home/volumes/redis/redis.conf:/usr/local/bin/redis.conf --name redis-slave1 -d redis redis-server /usr/local/bin/redis.conf
  #创建slave2
  docker run --link redis-master:redis-master -v /home/volumes/redis/redis.conf:/usr/local/bin/redis.conf --name redis-slave2 -d redis redis-server /usr/local/bin/redis.conf
  ```

- 查看是否创建成功

  ```shell
  docker exec -it redis-master /bin/bash
  /usr/local/bin/redis-cli
  #进入redis客户端
  127.0.0.1:6379>info
  ```

  ![](https://raw.githubusercontent.com/AaronFSJ/learning-experience/master/images/redis-info.png)

