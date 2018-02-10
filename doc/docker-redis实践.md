# docker安装redis主从与集群

## redis基本概念

- ### Redis 与其他 key – value 缓存产品有以下三个特点：

  1. Redis支持数据的持久化，可以将内存中的数据保持在磁盘中，重启的时候可以再次加载进行使用。
  2. Redis不仅仅支持简单的key-value类型的数据，同时还提供list，set，zset，hash等数据结构的存储。
  3. Redis支持数据的备份，即master-slave模式的数据备份。

  ![](https://raw.githubusercontent.com/AaronFSJ/learning-experience/master/images/redis-advance.png)

概念详情查看<a href="http://blog.jobbole.com/112024/" target="_blank">Redis 核心概念</a>，更多查看<a href="http://blog.jobbole.com/tag/redis/" target=_blank>伯乐在线—>redis</a>



- ### Redis持久化存储(AOF与RDB两种模式)

  - RDB(redis database)

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
    #当snapshot时出现错误无法继续时，是否阻塞客户端“变更操作”
    #默认情况下，如果 redis 最后一次的后台保存失败，redis 将停止接受写操作，
    # 这样以一种强硬的方式让用户知道数据不能正确的持久化到磁盘，
    # 否则就会没人注意到灾难的发生。
    #
    # 如果后台保存进程重新启动工作了，redis 也将自动的允许写操作。
    #
    # 然而你要是安装了靠谱的监控，你可能不希望 redis 这样做，那你就改成 no 好了。
    stop-writes-on-bgsave-error yes  
    ##是否启用rdb文件压缩，默认为“yes”，压缩往往意味着“额外的cpu消耗”，同时也意味这较小的文件尺寸以及较短的网络传输时间  
    rdbcompression yes  
    ```

    snapshot触发的时机，是有“间隔时间”和“变更次数”共同决定，同时符合2个条件才会触发snapshot,否则“变更次数”会被继续累加到下一个“间隔时间”上。snapshot过程中并不阻塞客户端请求。snapshot首先将数据写入临时文件，当成功结束后，将临时文件重名为dump.rdb。

    使用RDB恢复数据： 
    自动的持久化数据存储到dump.rdb后。实际只要重启redis服务即可完成（启动redis的server时会从dump.rdb中先同步数据）

    客户端使用命令进行持久化save存储：

    ```shell
    #前台进行存储
    ./redis-cli -h ip -p port save
    #后台进行存储
    ./redis-cli -h ip -p port bgsave
    ```

    一个是在前台进行存储，一个是在后台进行存储。我的client就在server这台服务器上，所以不需要连其他机器，直接./redis-cli bgsave。由于redis是用一个主线程来处理所有 client的请求，这种方式会阻塞所有client请求。所以不推荐使用。另一点需要注意的是，**每次快照持久化都是将内存数据完整写入到磁盘一次，并不是增量的只同步脏数据**。如果数据量大的话，而且写操作比较多，必然会引起大量的磁盘io操作，可能会严重影响性能。

  - AOF(Append-only file)

    将“操作 + 数据”以格式化指令的方式**追加到操作日志文件的尾部**，在append操作返回后(已经写入到文件或者即将写入)，才进行实际的数据变更（**先将操作+数据append到日志文件，然后再修改数据库的值**），“日志文件”保存了历史所有的操作过程；当server需要数据恢复时，可以直接replay此日志文件，即可还原所有的操作过程。AOF相对可靠，它和mysql中bin.log、apache.log、zookeeper中txn-log简直异曲同工。AOF文件内容是字符串，非常容易阅读和解析。 
    优点：**可以保持更高的数据完整性，如果设置追加file的时间是1s，如果redis发生故障，最多会丢失1s的数据；且如果日志写入不完整支持redis-check-aof来进行日志修复；AOF文件没被rewrite之前（文件过大时会对命令进行合并重写），可以删除其中的某些命令（比如误操作的flushall）。** 
    缺点：**AOF文件比RDB文件大，且恢复速度慢。**

    **AOF数据格式例子**

    ```
    *2          // 接下来的一条命令有2个参数
    $6         // 第一个参数的长度为6
    SELECT      // 第一个参数
    $1         // 第二个参数的长度为1
    0           // 第二个参数
    *3          // 接下来的一条命令有3个参数
    $3         // ...
    SET
    $5
    mystr
    $13
    this is redis
    *5
    $5
    RPUSH
    ```

    ​

    我们可以简单的认为AOF就是日志文件，此文件只会记录“变更操作”(例如：set/del等)，如果server中持续的大量变更操作，将会导致AOF文件非常的庞大，意味着server失效后，数据恢复的过程将会很长；事实上，一条数据经过多次变更，将会产生多条AOF记录，其实只要保存当前的状态，历史的操作记录是可以抛弃的；因为AOF持久化模式还伴生了“AOF rewrite”。 
    AOF的特性决定了它相对比较安全，如果你期望数据更少的丢失，那么可以采用AOF模式。如果AOF文件正在被写入时突然server失效，有可能导致文件的最后一次记录是不完整，你可以通过手工或者程序的方式去检测并修正不完整的记录，以便通过aof文件恢复能够正常；同时需要提醒，**如果你的redis持久化手段中有aof，那么在server故障失效后再次启动前，需要检测aof文件的完整性**。

    AOF默认关闭，开启方法，修改配置文件reds.conf：appendonly yes

    ```shell
    ##此选项为aof功能的开关，默认为“no”，可以通过“yes”来开启aof功能。只有在“yes”下，aof重写/文件同步等特性才会生效  
    appendonly yes  

    ##指定aof文件名称  
    appendfilename appendonly.aof  

    ##指定aof操作中文件同步策略，有三个合法值：always everysec no,默认为everysec  
    appendfsync everysec  
    ##在aof-rewrite期间，appendfsync是否暂缓文件同步，"no"表示“不暂缓”，“yes”表示“暂缓”，默认为“no”  
    no-appendfsync-on-rewrite no  

    ##aof文件rewrite触发的最小文件尺寸(mb,gb),只有大于此aof文件大于此尺寸是才会触发rewrite，默认“64mb”，建议“512mb”  
    auto-aof-rewrite-min-size 64mb  

    ##相对于“上一次”rewrite，本次rewrite触发时aof文件应该增长的百分比。 每一次rewrite之后，redis都会记录下此时“新aof”文件的大小(例如A)，那么当aof文件增长到A*(1 + p)之后触发下一次rewrite，每一次aof记录的添加，都会检测当前aof文件的尺寸。  
    auto-aof-rewrite-percentage 100  
    ```

    AOF是文件操作，对于变更操作比较密集的server，那么必将造成磁盘IO的负荷加重；此外linux对文件操作采取了“延迟写入”手段，即并非每次write操作都会触发实际磁盘操作，而是进入了buffer中，当buffer数据达到阀值时触发实际写入(也有其他时机)，这是linux对文件系统的优化，但是这却有可能带来隐患，如果buffer没有刷新到磁盘，此时物理机器失效(比如断电)，那么有可能导致最后一条或者多条aof记录的丢失。通过上述配置文件，可以得知redis提供了3中aof记录同步选项：

    > - always：每一条aof记录都立即同步到文件，这是最安全的方式，也以为更多的磁盘操作和阻塞延迟，是IO开支较大。
    > - everysec：每秒同步一次，性能和安全都比较中庸的方式，也是redis推荐的方式。如果遇到物理服务器故障，有可能导致最近一秒内aof记录丢失(可能为部分丢失)。
    > - no：redis并不直接调用文件同步，而是交给操作系统来处理，操作系统可以根据buffer填充情况/通道空闲时间等择机触发同步；这是一种普通的文件操作方式。性能较好，在物理服务器故障时，数据丢失量会因OS配置有关。

    其实，我们可以选择的太少，everysec是最佳的选择。如果你非常在意每个数据都极其可靠，建议你选择一款“关系性数据库”吧。 

    Redis Server将所有写入的命令转换成协议文本的方式写入AOF文件，例如：Server收到 set key value的的写入命令，server会进行以下几步操作：

    1. 将命令转换成协议文本，转换后的结果：`*3\r\n$3\r\nSET\r\n$3\r\nKEY\r\n$5\r\nVALUE\r\n`；
    2. 将协议文本追加到aof缓存，也就是aof_buf；
    3. 根据sync策略调用fsync/fdatasync。

    到目前为止已经成功保存数据，如果想要还原AOF，只需要将AOF里命令读出来并重放就可以还原数据库。

    AOF持久化机制存在一个致命的问题，随着时间推移，AOF文件会膨胀，如果频繁写入AOF文件会膨胀到无限大，**当server重启时严重影响数据库还原时间，影响系统可用性。为解决此问题，系统需要定期重写（rewrite）AOF文件**，目前采用的方式是创建一个新的AOF文件，将数据库里的全部数据转换成协议的方式保存到文件中，通过此操作达到减少AOF文件大小的目的，重写后的大小一定是小于等于旧AOF文件的大小。

    重写AOF提供两种方式

    1. REWRITE: 在主线程中重写AOF，会阻塞工作线程，在生产环境中很少使用，处于废弃状态；
    2. BGREWRITE: 在后台（子进程）重写AOF, 不会阻塞工作线程，能正常服务，此方法最常用。

    AOF文件会不断增大，它的大小直接影响“故障恢复”的时间,而且AOF文件中历史操作是可以丢弃的。**AOF rewrite操作就是“压缩”AOF文件的过程，当然redis并没有采用“基于原aof文件”来重写的方式，而是采取了类似snapshot的方式：基于copy-on-write，全量遍历内存中数据，然后逐个序列到aof文件中。因此AOF rewrite能够正确反应当前内存数据的状态，这正是我们所需要的；rewrite过程中，对于新的变更操作将仍然被写入到原AOF文件中，同时这些新的变更操作也会被redis收集起来(buffer，copy-on-write方式下，最极端的可能是所有的key都在此期间被修改，将会耗费2倍内存)，当内存数据被全部写入到新的aof文件之后，收集的新的变更操作也将会一并追加到新的aof文件中，此后将会重命名新的aof文件为appendonly.aof,此后所有的操作都将被写入新的aof文件。**如果在rewrite过程中，出现故障，将不会影响原AOF文件的正常工作，只有当rewrite完成之后才会切换文件，因为rewrite过程是比较可靠的。

    触发rewrite的时机可以通过配置文件来声明，同时redis中可以通过bgrewriteaof指令人工干预。

    ```
    redis-cli -h ip -p port bgrewriteaof
    ```

    因为rewrite操作/aof记录同步/snapshot都消耗磁盘IO，redis采取了“schedule”策略：无论是“人工干预”还是系统触发，snapshot和rewrite需要逐个被执行。

    AOF rewrite过程并不阻塞客户端请求。系统会开启一个子进程来完成。

  - 总结

    AOF和RDB各有优缺点，这是有它们各自的特点所决定：

    > - 1) AOF更加安全，可以将数据更加及时的同步到文件中，但是AOF需要较多的磁盘IO开支，AOF文件尺寸较大，文件内容恢复数度相对较慢。 
    >   *2) snapshot，安全性较差，它是“正常时期”数据备份以及master-slave数据同步的最佳手段，文件尺寸较小，恢复数度较快。

    可以通过配置文件来指定它们中的一种，或者同时使用它们(不建议同时使用)，或者全部禁用，***在架构良好的环境中，master通常使用AOF，slave使用snapshot，主要原因是master需要首先确保数据完整性，它作为数据备份的第一选择；slave提供只读服务(目前slave只能提供读取服务)，它的主要目的就是快速响应客户端read请求；但是如果你的redis运行在网络稳定性差/物理环境糟糕情况下，建议你master和slave均采取AOF，这个在master和slave角色切换时，可以减少“人工数据备份”/“人工引导数据恢复”的时间成本；如果你的环境一切非常良好，且服务需要接收密集性的write操作，那么建议master采取snapshot，而slave采用AOF***。

    ​

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

  - master

    ```shell
    cd /home/volumes/redis/master
    cp /home/soft/redis-4.0.7/redis.conf ./
    ```

  - slave

    ```shell
    cd /home/volumes/redis/slave
    cp /home/soft/redis-4.0.7/redis.conf ./
    #替换slaveof <masterip> <masterport>为slaveof redis-master 6379
    sed -i 's/# slaveof <masterip> <masterport>/slaveof redis-master 6379/g' redis.conf 
    ```

    创建redis主数据库 -v 挂载宿主主机的/home/docker/redis/master/redis.conf到目标主机的/data/conf/redis.conf上

    ***方式一创建***

    ```shell
    docker run -v /home/volumes/redis/master:/data --name redis-master  -d redis redis-server /data/redis.conf
    ```

    进入redis-master主机查询地址（docker inspect redis-master|grep "IPAddress"），例如查到redis-master地址为：172.17.0.3，修改宿主主机对应的挂载文件（/home/volumes/redis/master/redis.conf）的bind地址，否则slave无法通过ip访问到master。

    ```shell
    #bind 127.0.0.1 ::1
    # 127.0.0.1本地lo地址，172.17.0.3本机ip地址
    bind 127.0.0.1 172.17.0.3
    ```

    修改地址后重启

    ```shell
    docker restart redis-master
    ```

    ​

    ***方式二创建***

    ```shell
    #此方式启动没有配置redis.conf，docker创建redis自动会配置好
    docker run  --name redis-master -d redis redis-server
    ```

- 创建从数据库

  ```shell
  #创建slave1
  docker run --link redis-master:redis-master -v /home/volumes/redis/slave:/data --name redis-slave1 -d redis redis-server /data/redis.conf
  #创建slave2
  docker run --link redis-master:redis-master -v /home/volumes/redis/slave:/data --name redis-slave2 -d redis redis-server /data/redis.conf 
  ```

- 查看是否创建成功，docker创建redis 容器，，挂载在/data，当然挂载可以改成别的目录

  ```shell
  docker exec -it redis-master bash
  /usr/local/bin/redis-cli
  #进入redis客户端
  127.0.0.1:6379>info
  ```

  ![](https://raw.githubusercontent.com/AaronFSJ/learning-experience/master/images/redis-info.png)


***以上是最简单的主从复制，但是明显slave1和slave2都用同一份redis.conf启动，因此都明显只绑定了bind 127.0.0.1，代表除了各自的container里用exec才能连接，除本机外其他服务远程不过来，这种明显不能被sentinel监听，只能做备份，如果要被外网访问应各自修改bind 本机IP***

## redis主从优化（备份与恢复配置） 

*在上面也描述了，什么时候用RDB和AOF*，下面将会用master AOF+slave snapshot(RDB)和master snapshot+slave AOF

> 可以通过配置文件来指定它们中的一种，或者同时使用它们(不建议同时使用)，或者全部禁用，在架构良好的环境中，master通常使用AOF，slave使用snapshot，主要原因是master需要首先确保数据完整性，它作为数据备份的第一选择；slave提供只读服务(目前slave只能提供读取服务)，它的主要目的就是快速响应客户端read请求；但是如果你的redis运行在网络稳定性差/物理环境糟糕情况下，建议你master和slave均采取AOF，这个在master和slave角色切换时，可以减少“人工数据备份”/“人工引导数据恢复”的时间成本；如果你的环境一切非常良好，且服务需要接收密集性的write操作，那么建议master采取snapshot，而slave采用AOF

在上面配置的基础上*

```shell
#用inspect查询reids-master的基础信息，查挂载volumes
docker inspect --format "{{.Mounts}}" redis-master 
#结果如下，查到宿主机对应的/var/lib/docker/volumes/affe0a16c1cd701bcbfd0d4614ea1a5dea0725e9582d7772c508f62f0096e06f/_data,对应目标主机的/data,因为在创建redis-master容器时，没有用-v指明挂载，docker会自动生成一个目录
[{affe0a16c1cd701bcbfd0d4614ea1a5dea0725e9582d7772c508f62f0096e06f /var/lib/docker/volumes/affe0a16c1cd701bcbfd0d4614ea1a5dea0725e9582d7772c508f62f0096e06f/_data /data local  true }]
```

- master AOF+slave snapshot

  > 架构良好的环境中，master通常使用AOF，slave使用snapshot，主要原因是master需要首先确保数据完整性，它作为数据备份的第一选择；slave提供只读服务(目前slave只能提供读取服务)，它的主要目的就是快速响应客户端read请求。

  - master 

    从宿主机copy redis-conf到宿主机的挂载目录上，因为创建时指明了挂载目录

    ```shell
    cd /home/volumes/redis/master/
    #cp /home/soft/redis-4.0.7/redis.conf ./
    #从原来上面配置的文件中copy重命名，因为后面还会有rdb的conf，重命名好区分
    cp redis.conf redis-aof.conf
    #编辑redis-aof.conf
    vi redis-aof.conf
    ```

    ***redis-conf（redis-aof.conf）***

    ```shell
    #修改日志存放目录
    #dir ./
    dir /data
    # save ""关闭snapshot功能
    save ""
    #save 900 1
    #save 300 10
    #save 60 10000
    ##此选项为aof功能的开关，默认为“no”，可以通过“yes”来开启aof功能。只有在“yes”下，aof重写/文件同步等特性才会生效  
    appendonly yes  

    ##指定aof文件名称  
    appendfilename appendonly.aof  

    ##指定aof操作中文件同步策略，有三个合法值：always everysec no,默认为everysec  
    appendfsync everysec  
    #AOF采用文件追加的方式，文件会越来越大为避免出现此种情况，新增了重写机制，当AOF文件的大小超过所设定的阈值时，redis就会启动AOF文件的内容压缩，只保留可以恢复数据的最小指令集，可以使用bgrewriteaof
    ##在aof-rewrite期间，appendfsync是否暂缓文件同步，"no"表示“不暂缓”，“yes”表示“暂缓”，默认为“no”  
    no-appendfsync-on-rewrite no  

    ##aof文件rewrite触发的最小文件尺寸(mb,gb),只有大于此aof文件大于此尺寸是才会触发rewrite，默认“64mb”，建议“512mb”  
    auto-aof-rewrite-min-size 64mb  

    ##相对于“上一次”rewrite，本次rewrite触发时aof文件应该增长的百分比。 每一次rewrite之后，redis都会记录下此时“新aof”文件的大小(例如A)，那么当aof文件增长到A*(1 + p)之后触发下一次rewrite，每一次aof记录的添加，都会检测当前aof文件的尺寸。  
    auto-aof-rewrite-percentage 100  
    ```

    ***创建Container***

    ```shell
    docker run -v /home/volumes/redis/master:/data --name redis-aof-master -p 6379:6379 -d redis redis-server /data/redis-aof.conf
    ```

    ***添加-p 6379:6379是为了让docker容器能让宿主机以外的机器或者能够访问，相当于一个外网端口号，如果不开启，除了宿主机，其他别机器无法访问***

    查询redis-master主机地址，(docker inspect redis-aof-master|grep "IPAddress"查询)例如查到redis-sof-master地址为：172.17.0.6，修改宿主主机对应的挂载文件（/home/volumes/redis/master/redis-sof.conf）的bind地址，否则slave无法通过ip访问到master。

    ```shell
    #bind 127.0.0.1 ::1
    # 127.0.0.1本地lo地址，172.17.0.6本机ip地址
    bind 127.0.0.1 172.17.0.6
    ```

    修改地址后重启

    ```shell
    docker restart redis-master
    ```

    ​


  - slave redis-conf

    从宿主机copy redis-conf到宿主机的挂载目录上

    ```shell
    cd /home/volumes/redis/slave/
    #cp /home/soft/redis-4.0.7/redis.conf ./
    #从原来上面配置的文件中copy重命名，重命名好区分
    cp redis.conf redis-rdb.conf
    #编辑redis-rdb.conf
    vi redis-rdb.conf
    ```

    ***redis-rdb-slave1.conf***

    ```shell
    #修改日志存放目录
    #dir ./
    dir /data
    # 开启快照功能，如下为900秒后，至少有一个变更操作，才会snapshot，有“间隔时间”和“变更次数”共同决定，同时符合2个条件才会触发snapshot,否则“变更次数”会被继续累加到下一个“间隔时间”上
    # 如不满足第一个，次数会累计到下一个300秒时间会重新计算？，至少有10次
    save 900 1
    save 300 10
    save 60 10000

    #当snapshot时出现错误无法继续时，是否阻塞客户端“变更操作”
    #默认情况下，如果 redis 最后一次的后台保存失败，redis 将停止接受写操作，
    # 这样以一种强硬的方式让用户知道数据不能正确的持久化到磁盘，
    # 否则就会没人注意到灾难的发生。
    #
    # 如果后台保存进程重新启动工作了，redis 也将自动的允许写操作。
    #
    # 然而你要是安装了靠谱的监控，你可能不希望 redis 这样做，那你就改成 no 好了。
    stop-writes-on-bgsave-error yes

    # 不开启AOF
    appendonly no
    ```

    redis-rdb-slave2.conf***

    ```shell
    #修改日志存放目录
    #dir ./
    dir /data
    # 开启快照功能，如下为900秒后，至少有一个变更操作，才会snapshot，有“间隔时间”和“变更次数”共同决定，同时符合2个条件才会触发snapshot,否则“变更次数”会被继续累加到下一个“间隔时间”上
    # 如不满足第一个，次数会累计到下一个300秒时间会重新计算？，至少有10次
    save 900 1
    save 300 10
    save 60 10000

    #当snapshot时出现错误无法继续时，是否阻塞客户端“变更操作”
    #默认情况下，如果 redis 最后一次的后台保存失败，redis 将停止接受写操作，
    # 这样以一种强硬的方式让用户知道数据不能正确的持久化到磁盘，
    # 否则就会没人注意到灾难的发生。
    #
    # 如果后台保存进程重新启动工作了，redis 也将自动的允许写操作。
    #
    # 然而你要是安装了靠谱的监控，你可能不希望 redis 这样做，那你就改成 no 好了。
    stop-writes-on-bgsave-error yes

    # 不开启AOF
    appendonly no
    ```

    ​

    ***创建slave container***

    ```shell
    #创建slave1
    docker run --link redis-aof-master:redis-aof-master -v /home/volumes/redis/slave:/data --name redis-rdb-slave1 -p 6380:6379 -d redis redis-server /data/redis-rdb-slave1.conf
    #创建slave2
    docker run --link redis-aof-master:redis-aof-master -v /home/volumes/redis/slave:/data -p 6381:6379 --name redis-rdb-slave2 -d redis redis-server /data/redis-rdb-slave2.conf
    ```

    查询两个slave主机查询地址，(docker inspect <container>|grep "IPAddress"查询)例如查到slave1地址为：172.17.0.7,slave2地址为172.17.0.8，修改宿主主机对应的挂载文件的bind地址，否则slave无法被本机以外的服务器调用。

    vim redis-rdb-slave1.conf

    ```shell
    #bind 127.0.0.1 ::1
    # 127.0.0.1本地lo地址，172.17.0.7本机ip地址
    bind 127.0.0.1 172.17.0.7
    ```

    vim redis-rdb-slave2.conf

    ```shell
    #bind 127.0.0.1 ::1
    # 127.0.0.1本地lo地址，172.17.0.8本机ip地址
    bind 127.0.0.1 172.17.0.8
    ```

    修改地址后重启

    ```shell
    docker restart redis-rdb-slave1
    docker restart redis-rdb-slave2
    ```

    ​

- master snapshot+slave AOF（参考上面例子）

  > 如果你的环境一切非常良好，且服务需要接收密集性的write操作，那么建议master采取snapshot，而slave采用AOF

- master AOF + slave AOF（参考上面例子）

  > 但是如果你的redis运行在网络稳定性差/物理环境糟糕情况下，建议你master和slave均采取AOF，这个在master和slave角色切换时，可以减少“人工数据备份”/“人工引导数据恢复”的时间成本；

## redis哨兵sentinel

*当主数据库遇到异常中断服务后，开发者可以通过手动的方式选择一个从数据库来升格为主数据库，以使得系统能够继续提供服务。然而整个过程相对麻烦且需要人工介入，难以实现自动化*。

***哨兵的作用就是监控redis主、从数据库是否正常运行，主出现故障自动将从数据库转换为主数据库***

## 架构

架构如下图，各主从服务器都添加一个哨兵

![](https://raw.githubusercontent.com/AaronFSJ/learning-experience/master/images/redis-sentinel.png)

1. 一个健康的集群部署，至少需要三个Sentinel实例
2. 三个Sentinel实例应该被放在独立的电脑上或虚拟机中(本例所有的sentinel全放在一个服务器上)，比如说不同的物理机或者在不同的可用区域上执行的虚拟机。
3. Sentinel + Redis 分布式系统在失败期间并不确保写入请求被保存，因为Redis使用异步拷贝。可是有很多部署Sentinel的 方式来让窗口把丢失写入限制在特定的时刻，当然也有另外的不安全的方式来部署。
4. 如果你在开发环境中没有经常测试，或者在生产环境中也没有，那就没有高可用的设置是安全的。你或许有一个错误的配置而仅仅只是在很晚的时候才出现（凌晨3点你的主节点宕掉了）。
5. Sentinel，Docker ，其他的网络地址转换表，端口映射 使用应该很小心的使用：Docker执行端口重新映射，破坏Sentinel自动发现另外的Sentinel进程和一个主节点的从节点列表。在文章的稍后部分查看更过关于Sentinel和Docker的信息。

***需要特别注意的是在NAT网络环境中，如果Redis和Sentinel做了端口映射，例如在Docker默认网络模式下，使用-p参数做端口映射，就需要配置一下从服务器redis.conf中的slave-announce-ip 和 slave-announce-port，对应外网的IP和外网端口。Sentinel的配置文件sentinel.conf 也需要配置sentinel announce-ip 和 sentinel announce-port ，对应外网的IP和外网端口。当然，如果Docker配置成host网络模式，就不需要配置了***

在上面的的master AOF+slave snapshot例子上作修改

```shell
cd /home/volumes/redis
mkdir sentinel
#src存在redis-sentinel的前提，是redis源码make了
cd sentinel
cp /home/soft/redis-4.0.7/src/redis-sentinel ./

```



从redis源包里cp sentinel.conf

```shell
cp /home/soft/redis-4.0.7/sentinel.conf ./
vim sentinel.conf
```



```shell
#Sentinel节点的端口
port 26379
#日志存放地址
dir "/data"
#日志文件名
logfile "26379.log"
#后台启动
daemonize yes
bind 0.0.0.0
protected-mode no

#当前Sentinel节点监控 172.17.0.61:6379(master节点地址) 这个2代表判断sentinel主节点失败至少需要2个Sentinel节点节点同意,mymaster是主节点的别名
sentinel monitor mymaster 172.17.0.6 6379 2
#每个Sentinel节点都要定期PING命令来判断Redis数据节点和其余Sentinel节点是否可达，如果超过30000毫秒且没有回复，则判定不可达，默认30000(30秒)
sentinel down-after-milliseconds mymaster 5000
#当Sentinel节点集合对主节点故障判定达成一致时，Sentinel领导者节点会做故障转移操作，选出新的主节点，原来的从节点会向新的主节点发起复制操作，限制每次向新的主节点发起复制操作的从节点个数为1
sentinel parallel-syncs mymaster 1

#故障转移超时时间为180000毫秒
sentinel failover-timeout mymaster 180000
```

```shell
cp sentinel.conf ./sentinel-26380.conf
cp sentinel.conf ./sentinel-26381.conf
```

在上面sentinel.conf的基础上修改，26380

```shell
#Sentinel节点的端口
port 26380
logfile "26380.log"
```

在上面sentinel.conf的基础上修改，26381

```shell
#Sentinel节点的端口
port 26381
logfile "26381.log"
```



本人在docker 又新建了centos镜像sentinel

```shell
docker run -v /home/volumes/redis/sentinel:/data --name sentinel -d centos 
```

创建sentinel 容器后

```shell
docker exec -it sentinel bash
```

进入sentinel容器

```
cd /data
./redis-sentinel sentinel.conf
./redis-sentinel sentinel-26380.conf
./redis-sentinel sentinel-26381.conf
```

启动3个sentinel后，发现/data目录下多了3个日志文件26379.log 、26380.log 、 26381.log，查看任意一个日志文件，同时可以看到各sentinel.conf，下多了一些sentinel进程添加的信息

```shell
sentinel myid f6b7aaac03872419f38793898dbc0f9015122d7b


# Generated by CONFIG REWRITE
sentinel known-slave mymaster 172.17.0.7 6379

sentinel known-slave mymaster 172.17.0.8 6379

sentinel known-sentinel mymaster 172.17.0.9 26381 dd0b449812f17c3d3a0070352db921a4be06d64d

sentinel known-sentinel mymaster 172.17.0.9 26380 b6f979d7b5f986a7ad1e5a9c096bb8196de4c0d9

sentinel current-epoch 3

```

查看其中一个sentinel进程的日志

```shell
tail -f 26379.log
```

**测试**

```
docker stop redis-aof-masetr
```

可以看到

```shell
816:X 10 Feb 09:38:50.980 # +monitor master mymaster 172.17.0.6 6379 quorum 2
816:X 10 Feb 09:38:51.978 * +sentinel sentinel dd0b449812f17c3d3a0070352db921a4be06d64d 172.17.0.9 26381 @ mymaster 172.17.0.6 6379
816:X 10 Feb 09:39:21.756 # +sdown master mymaster 172.17.0.6 6379
816:X 10 Feb 09:39:21.824 # +new-epoch 3
816:X 10 Feb 09:39:21.827 # +vote-for-leader f6b7aaac03872419f38793898dbc0f9015122d7b 3
816:X 10 Feb 09:39:22.873 # +odown master mymaster 172.17.0.6 6379 #quorum 3/2
816:X 10 Feb 09:39:22.873 # Next failover delay: I will not start a failover before Sat Feb 10 09:45:22 2018
816:X 10 Feb 09:39:22.984 # +config-update-from sentinel f6b7aaac03872419f38793898dbc0f9015122d7b 172.17.0.9 26379 @ mymaster 172.17.0.6 6379
816:X 10 Feb 09:39:22.984 # +switch-master mymaster 172.17.0.6 6379 172.17.0.8 6379
816:X 10 Feb 09:39:22.984 * +slave slave 172.17.0.7:6379 172.17.0.7 6379 @ mymaster 172.17.0.8 6379
816:X 10 Feb 09:39:22.984 * +slave slave 172.17.0.6:6379 172.17.0.6 6379 @ mymaster 172.17.0.8 6379
```

+sdown master mymaster 172.17.0.6 6379 监听到172.17.0.6 down ，同样其他2个的sentinel也监听到（查看其他两个日志可知），此时满足2个监听到失败

+vote-for-leader f6b7aaac03872419f38793898dbc0f9015122d7b 3 ，投票的leader(其中一个sentinel进程f6b7aaac03872419f38793898dbc0f9015122d7b)

+odown master mymaster 172.17.0.6 6379 #quorum 3/2，满足3/2个监听到失败

Next failover delay: I will not start a failover before Sat Feb 10 09:45:22 2018，在Sat Feb 10 09:45:22 2018前不会进行failover，为了等待这个时间172.17.0.6能正常连接

+config-update-from sentinel 更新sentinel.conf，变成

```shell
# Generated by CONFIG REWRITE
sentinel known-slave mymaster 172.17.0.7 6379
#本次变更
sentinel known-slave mymaster 172.17.0.6 6379

sentinel known-sentinel mymaster 172.17.0.9 26381 dd0b449812f17c3d3a0070352db921a4be06d64d

sentinel known-sentinel mymaster 172.17.0.9 26380 b6f979d7b5f986a7ad1e5a9c096bb8196de4c0d9

sentinel current-epoch 3
```

***以上sentinel测试成功，很明显，我将所有的sentinel放在同台服务器中，实际生产环境应该sentinel放在不同服务器，防止服务器down,导致所有的sentinel都监听不了***



# redis 配置文件示例

```
# redis 配置文件示例
 
# 当你需要为某个配置项指定内存大小的时候，必须要带上单位，
# 通常的格式就是 1k 5gb 4m 等酱紫：
#
# 1k  => 1000 bytes
# 1kb => 1024 bytes
# 1m  => 1000000 bytes
# 1mb => 1024*1024 bytes
# 1g  => 1000000000 bytes
# 1gb => 1024*1024*1024 bytes
#
# 单位是不区分大小写的，你写 1K 5GB 4M 也行
 
################################## INCLUDES ###################################
 
# 假如说你有一个可用于所有的 redis server 的标准配置模板，
# 但针对某些 server 又需要一些个性化的设置，
# 你可以使用 include 来包含一些其他的配置文件，这对你来说是非常有用的。
#
# 但是要注意哦，include 是不能被 config rewrite 命令改写的
# 由于 redis 总是以最后的加工线作为一个配置指令值，所以你最好是把 include 放在这个文件的最前面，
# 以避免在运行时覆盖配置的改变，相反，你就把它放在后面（外国人真啰嗦）。
#
# include /path/to/local.conf
# include /path/to/other.conf
 
################################ 常用 #####################################
 
# 默认情况下 redis 不是作为守护进程运行的，如果你想让它在后台运行，你就把它改成 yes。
# 当redis作为守护进程运行的时候，它会写一个 pid 到 /var/run/redis.pid 文件里面。
daemonize no
 
# 当redis作为守护进程运行的时候，它会把 pid 默认写到 /var/run/redis.pid 文件里面，
# 但是你可以在这里自己制定它的文件位置。
pidfile /var/run/redis.pid
 
# 监听端口号，默认为 6379，如果你设为 0 ，redis 将不在 socket 上监听任何客户端连接。
port 6379
 
# TCP 监听的最大容纳数量
#
# 在高并发的环境下，你需要把这个值调高以避免客户端连接缓慢的问题。
# Linux 内核会一声不响的把这个值缩小成 /proc/sys/net/core/somaxconn 对应的值，
# 所以你要修改这两个值才能达到你的预期。
tcp-backlog 511
 
# 默认情况下，redis 在 server 上所有有效的网络接口上监听客户端连接。
# 你如果只想让它在一个网络接口上监听，那你就绑定一个IP或者多个IP。
#
# 示例，多个IP用空格隔开:
#
# bind 192.168.1.100 10.0.0.1
# bind 127.0.0.1
 
# 指定 unix socket 的路径。
#
# unixsocket /tmp/redis.sock
# unixsocketperm 755
 
# 指定在一个 client 空闲多少秒之后关闭连接（0 就是不管它）
timeout 0
 
# tcp 心跳包。
#
# 如果设置为非零，则在与客户端缺乏通讯的时候使用 SO_KEEPALIVE 发送 tcp acks 给客户端。
# 这个之所有有用，主要由两个原因：
#
# 1) 防止死的 peers
# 2) Take the connection alive from the point of view of network
#    equipment in the middle.
#
# On Linux, the specified value (in seconds) is the period used to send ACKs.
# Note that to close the connection the double of the time is needed.
# On other kernels the period depends on the kernel configuration.
#
# A reasonable value for this option is 60 seconds.
# 推荐一个合理的值就是60秒
tcp-keepalive 0
 
# 定义日志级别。
# 可以是下面的这些值：
# debug (适用于开发或测试阶段)
# verbose (many rarely useful info, but not a mess like the debug level)
# notice (适用于生产环境)
# warning (仅仅一些重要的消息被记录)
loglevel notice
 
# 指定日志文件的位置
logfile ""
 
# 要想把日志记录到系统日志，就把它改成 yes，
# 也可以可选择性的更新其他的syslog 参数以达到你的要求
# syslog-enabled no
 
# 设置 syslog 的 identity。
# syslog-ident redis
 
# 设置 syslog 的 facility，必须是 USER 或者是 LOCAL0-LOCAL7 之间的值。
# syslog-facility local0
 
# 设置数据库的数目。
# 默认数据库是 DB 0，你可以在每个连接上使用 select <dbid> 命令选择一个不同的数据库，
# 但是 dbid 必须是一个介于 0 到 databasees - 1 之间的值
databases 16
 
################################ 快照 ################################
#
# 存 DB 到磁盘：
#
#   格式：save <间隔时间（秒）> <写入次数>
#
#   根据给定的时间间隔和写入次数将数据保存到磁盘
#
#   下面的例子的意思是：
#   900 秒内如果至少有 1 个 key 的值变化，则保存
#   300 秒内如果至少有 10 个 key 的值变化，则保存
#   60 秒内如果至少有 10000 个 key 的值变化，则保存
#　　
#   注意：你可以注释掉所有的 save 行来停用保存功能。
#   也可以直接一个空字符串来实现停用：
#   save ""
 
save 900 1
save 300 10
save 60 10000
 
# 默认情况下，如果 redis 最后一次的后台保存失败，redis 将停止接受写操作，
# 这样以一种强硬的方式让用户知道数据不能正确的持久化到磁盘，
# 否则就会没人注意到灾难的发生。
#
# 如果后台保存进程重新启动工作了，redis 也将自动的允许写操作。
#
# 然而你要是安装了靠谱的监控，你可能不希望 redis 这样做，那你就改成 no 好了。
stop-writes-on-bgsave-error yes
 
# 是否在 dump .rdb 数据库的时候使用 LZF 压缩字符串
# 默认都设为 yes
# 如果你希望保存子进程节省点 cpu ，你就设置它为 no ，
# 不过这个数据集可能就会比较大
rdbcompression yes
 
# 是否校验rdb文件
rdbchecksum yes
 
# 设置 dump 的文件位置
dbfilename dump.rdb
 
# 工作目录
# 例如上面的 dbfilename 只指定了文件名，
# 但是它会写入到这个目录下。这个配置项一定是个目录，而不能是文件名。
dir ./
 
################################# 主从复制 #################################
 
# 主从复制。使用 slaveof 来让一个 redis 实例成为另一个reids 实例的副本。
# 注意这个只需要在 slave 上配置。
#
# slaveof <masterip> <masterport>
 
# 如果 master 需要密码认证，就在这里设置
# masterauth <master-password>
 
# 当一个 slave 与 master 失去联系，或者复制正在进行的时候，
# slave 可能会有两种表现：
#
# 1) 如果为 yes ，slave 仍然会应答客户端请求，但返回的数据可能是过时，
#    或者数据可能是空的在第一次同步的时候
#
# 2) 如果为 no ，在你执行除了 info he salveof 之外的其他命令时，
#    slave 都将返回一个 "SYNC with master in progress" 的错误，
#
slave-serve-stale-data yes
 
# 你可以配置一个 slave 实体是否接受写入操作。
# 通过写入操作来存储一些短暂的数据对于一个 slave 实例来说可能是有用的，
# 因为相对从 master 重新同步数而言，据数据写入到 slave 会更容易被删除。
# 但是如果客户端因为一个错误的配置写入，也可能会导致一些问题。
#
# 从 redis 2.6 版起，默认 slaves 都是只读的。
#
# Note: read only slaves are not designed to be exposed to untrusted clients
# on the internet. It's just a protection layer against misuse of the instance.
# Still a read only slave exports by default all the administrative commands
# such as CONFIG, DEBUG, and so forth. To a limited extent you can improve
# security of read only slaves using 'rename-command' to shadow all the
# administrative / dangerous commands.
# 注意：只读的 slaves 没有被设计成在 internet 上暴露给不受信任的客户端。
# 它仅仅是一个针对误用实例的一个保护层。
slave-read-only yes
 
# Slaves 在一个预定义的时间间隔内发送 ping 命令到 server 。
# 你可以改变这个时间间隔。默认为 10 秒。
#
# repl-ping-slave-period 10
 
# The following option sets the replication timeout for:
# 设置主从复制过期时间
#
# 1) Bulk transfer I/O during SYNC, from the point of view of slave.
# 2) Master timeout from the point of view of slaves (data, pings).
# 3) Slave timeout from the point of view of masters (REPLCONF ACK pings).
#
# It is important to make sure that this value is greater than the value
# specified for repl-ping-slave-period otherwise a timeout will be detected
# every time there is low traffic between the master and the slave.
# 这个值一定要比 repl-ping-slave-period 大
#
# repl-timeout 60
 
# Disable TCP_NODELAY on the slave socket after SYNC?
#
# If you select "yes" Redis will use a smaller number of TCP packets and
# less bandwidth to send data to slaves. But this can add a delay for
# the data to appear on the slave side, up to 40 milliseconds with
# Linux kernels using a default configuration.
#
# If you select "no" the delay for data to appear on the slave side will
# be reduced but more bandwidth will be used for replication.
#
# By default we optimize for low latency, but in very high traffic conditions
# or when the master and slaves are many hops away, turning this to "yes" may
# be a good idea.
repl-disable-tcp-nodelay no
 
# 设置主从复制容量大小。这个 backlog 是一个用来在 slaves 被断开连接时
# 存放 slave 数据的 buffer，所以当一个 slave 想要重新连接，通常不希望全部重新同步，
# 只是部分同步就够了，仅仅传递 slave 在断开连接时丢失的这部分数据。
#
# The biggest the replication backlog, the longer the time the slave can be
# disconnected and later be able to perform a partial resynchronization.
# 这个值越大，salve 可以断开连接的时间就越长。
#
# The backlog is only allocated once there is at least a slave connected.
#
# repl-backlog-size 1mb
 
# After a master has no longer connected slaves for some time, the backlog
# will be freed. The following option configures the amount of seconds that
# need to elapse, starting from the time the last slave disconnected, for
# the backlog buffer to be freed.
# 在某些时候，master 不再连接 slaves，backlog 将被释放。
#
# A value of 0 means to never release the backlog.
# 如果设置为 0 ，意味着绝不释放 backlog 。
#
# repl-backlog-ttl 3600
 
# 当 master 不能正常工作的时候，Redis Sentinel 会从 slaves 中选出一个新的 master，
# 这个值越小，就越会被优先选中，但是如果是 0 ， 那是意味着这个 slave 不可能被选中。
#
# 默认优先级为 100。
slave-priority 100
 
# It is possible for a master to stop accepting writes if there are less than
# N slaves connected, having a lag less or equal than M seconds.
#
# The N slaves need to be in "online" state.
#
# The lag in seconds, that must be <= the specified value, is calculated from
# the last ping received from the slave, that is usually sent every second.
#
# This option does not GUARANTEES that N replicas will accept the write, but
# will limit the window of exposure for lost writes in case not enough slaves
# are available, to the specified number of seconds.
#
# For example to require at least 3 slaves with a lag <= 10 seconds use:
#
# min-slaves-to-write 3
# min-slaves-max-lag 10
#
# Setting one or the other to 0 disables the feature.
#
# By default min-slaves-to-write is set to 0 (feature disabled) and
# min-slaves-max-lag is set to 10.
 
################################## 安全 ###################################
 
# Require clients to issue AUTH <PASSWORD> before processing any other
# commands.  This might be useful in environments in which you do not trust
# others with access to the host running redis-server.
#
# This should stay commented out for backward compatibility and because most
# people do not need auth (e.g. they run their own servers).
# 
# Warning: since Redis is pretty fast an outside user can try up to
# 150k passwords per second against a good box. This means that you should
# use a very strong password otherwise it will be very easy to break.
# 
# 设置认证密码
# requirepass foobared
 
# Command renaming.
#
# It is possible to change the name of dangerous commands in a shared
# environment. For instance the CONFIG command may be renamed into something
# hard to guess so that it will still be available for internal-use tools
# but not available for general clients.
#
# Example:
#
# rename-command CONFIG b840fc02d524045429941cc15f59e41cb7be6c52
#
# It is also possible to completely kill a command by renaming it into
# an empty string:
#
# rename-command CONFIG ""
#
# Please note that changing the name of commands that are logged into the
# AOF file or transmitted to slaves may cause problems.
 
################################### 限制 ####################################
 
# Set the max number of connected clients at the same time. By default
# this limit is set to 10000 clients, however if the Redis server is not
# able to configure the process file limit to allow for the specified limit
# the max number of allowed clients is set to the current file limit
# minus 32 (as Redis reserves a few file descriptors for internal uses).
#
# 一旦达到最大限制，redis 将关闭所有的新连接
# 并发送一个‘max number of clients reached’的错误。
#
# maxclients 10000
 
# 如果你设置了这个值，当缓存的数据容量达到这个值， redis 将根据你选择的
# eviction 策略来移除一些 keys。
#
# 如果 redis 不能根据策略移除 keys ，或者是策略被设置为 ‘noeviction’，
# redis 将开始响应错误给命令，如 set，lpush 等等，
# 并继续响应只读的命令，如 get
#
# This option is usually useful when using Redis as an LRU cache, or to set
# a hard memory limit for an instance (using the 'noeviction' policy).
#
# WARNING: If you have slaves attached to an instance with maxmemory on,
# the size of the output buffers needed to feed the slaves are subtracted
# from the used memory count, so that network problems / resyncs will
# not trigger a loop where keys are evicted, and in turn the output
# buffer of slaves is full with DELs of keys evicted triggering the deletion
# of more keys, and so forth until the database is completely emptied.
#
# In short... if you have slaves attached it is suggested that you set a lower
# limit for maxmemory so that there is some free RAM on the system for slave
# output buffers (but this is not needed if the policy is 'noeviction').
#
# 最大使用内存
# maxmemory <bytes>
 
# 最大内存策略，你有 5 个选择。
# 
# volatile-lru -> remove the key with an expire set using an LRU algorithm
# volatile-lru -> 使用 LRU 算法移除包含过期设置的 key 。
# allkeys-lru -> remove any key accordingly to the LRU algorithm
# allkeys-lru -> 根据 LRU 算法移除所有的 key 。
# volatile-random -> remove a random key with an expire set
# allkeys-random -> remove a random key, any key
# volatile-ttl -> remove the key with the nearest expire time (minor TTL)
# noeviction -> don't expire at all, just return an error on write operations
# noeviction -> 不让任何 key 过期，只是给写入操作返回一个错误
# 
# Note: with any of the above policies, Redis will return an error on write
#       operations, when there are not suitable keys for eviction.
#
#       At the date of writing this commands are: set setnx setex append
#       incr decr rpush lpush rpushx lpushx linsert lset rpoplpush sadd
#       sinter sinterstore sunion sunionstore sdiff sdiffstore zadd zincrby
#       zunionstore zinterstore hset hsetnx hmset hincrby incrby decrby
#       getset mset msetnx exec sort
#
# The default is:
#
# maxmemory-policy noeviction
 
# LRU and minimal TTL algorithms are not precise algorithms but approximated
# algorithms (in order to save memory), so you can tune it for speed or
# accuracy. For default Redis will check five keys and pick the one that was
# used less recently, you can change the sample size using the following
# configuration directive.
#
# The default of 5 produces good enough results. 10 Approximates very closely
# true LRU but costs a bit more CPU. 3 is very fast but not very accurate.
#
# maxmemory-samples 5
 
############################## APPEND ONLY MODE ###############################
 
# By default Redis asynchronously dumps the dataset on disk. This mode is
# good enough in many applications, but an issue with the Redis process or
# a power outage may result into a few minutes of writes lost (depending on
# the configured save points).
#
# The Append Only File is an alternative persistence mode that provides
# much better durability. For instance using the default data fsync policy
# (see later in the config file) Redis can lose just one second of writes in a
# dramatic event like a server power outage, or a single write if something
# wrong with the Redis process itself happens, but the operating system is
# still running correctly.
#
# AOF and RDB persistence can be enabled at the same time without problems.
# If the AOF is enabled on startup Redis will load the AOF, that is the file
# with the better durability guarantees.
#
# Please check http://redis.io/topics/persistence for more information.
 
appendonly no
 
# The name of the append only file (default: "appendonly.aof")
 
appendfilename "appendonly.aof"
 
# The fsync() call tells the Operating System to actually write data on disk
# instead to wait for more data in the output buffer. Some OS will really flush 
# data on disk, some other OS will just try to do it ASAP.
#
# Redis supports three different modes:
#
# no: don't fsync, just let the OS flush the data when it wants. Faster.
# always: fsync after every write to the append only log . Slow, Safest.
# everysec: fsync only one time every second. Compromise.
#
# The default is "everysec", as that's usually the right compromise between
# speed and data safety. It's up to you to understand if you can relax this to
# "no" that will let the operating system flush the output buffer when
# it wants, for better performances (but if you can live with the idea of
# some data loss consider the default persistence mode that's snapshotting),
# or on the contrary, use "always" that's very slow but a bit safer than
# everysec.
#
# More details please check the following article:
# http://antirez.com/post/redis-persistence-demystified.html
#
# If unsure, use "everysec".
 
# appendfsync always
appendfsync everysec
# appendfsync no
 
# When the AOF fsync policy is set to always or everysec, and a background
# saving process (a background save or AOF log background rewriting) is
# performing a lot of I/O against the disk, in some Linux configurations
# Redis may block too long on the fsync() call. Note that there is no fix for
# this currently, as even performing fsync in a different thread will block
# our synchronous write(2) call.
#
# In order to mitigate this problem it's possible to use the following option
# that will prevent fsync() from being called in the main process while a
# BGSAVE or BGREWRITEAOF is in progress.
#
# This means that while another child is saving, the durability of Redis is
# the same as "appendfsync none". In practical terms, this means that it is
# possible to lose up to 30 seconds of log in the worst scenario (with the
# default Linux settings).
# 
# If you have latency problems turn this to "yes". Otherwise leave it as
# "no" that is the safest pick from the point of view of durability.
 
no-appendfsync-on-rewrite no
 
# Automatic rewrite of the append only file.
# Redis is able to automatically rewrite the log file implicitly calling
# BGREWRITEAOF when the AOF log size grows by the specified percentage.
# 
# This is how it works: Redis remembers the size of the AOF file after the
# latest rewrite (if no rewrite has happened since the restart, the size of
# the AOF at startup is used).
#
# This base size is compared to the current size. If the current size is
# bigger than the specified percentage, the rewrite is triggered. Also
# you need to specify a minimal size for the AOF file to be rewritten, this
# is useful to avoid rewriting the AOF file even if the percentage increase
# is reached but it is still pretty small.
#
# Specify a percentage of zero in order to disable the automatic AOF
# rewrite feature.
 
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
 
################################ LUA SCRIPTING  ###############################
 
# Max execution time of a Lua script in milliseconds.
#
# If the maximum execution time is reached Redis will log that a script is
# still in execution after the maximum allowed time and will start to
# reply to queries with an error.
#
# When a long running script exceed the maximum execution time only the
# SCRIPT KILL and SHUTDOWN NOSAVE commands are available. The first can be
# used to stop a script that did not yet called write commands. The second
# is the only way to shut down the server in the case a write commands was
# already issue by the script but the user don't want to wait for the natural
# termination of the script.
#
# Set it to 0 or a negative value for unlimited execution without warnings.
lua-time-limit 5000
 
################################ REDIS 集群  ###############################
#
# 启用或停用集群
# cluster-enabled yes
 
# Every cluster node has a cluster configuration file. This file is not
# intended to be edited by hand. It is created and updated by Redis nodes.
# Every Redis Cluster node requires a different cluster configuration file.
# Make sure that instances running in the same system does not have
# overlapping cluster configuration file names.
#
# cluster-config-file nodes-6379.conf
 
# Cluster node timeout is the amount of milliseconds a node must be unreachable 
# for it to be considered in failure state.
# Most other internal time limits are multiple of the node timeout.
#
# cluster-node-timeout 15000
 
# A slave of a failing master will avoid to start a failover if its data
# looks too old.
#
# There is no simple way for a slave to actually have a exact measure of
# its "data age", so the following two checks are performed:
#
# 1) If there are multiple slaves able to failover, they exchange messages
#    in order to try to give an advantage to the slave with the best
#    replication offset (more data from the master processed).
#    Slaves will try to get their rank by offset, and apply to the start
#    of the failover a delay proportional to their rank.
#
# 2) Every single slave computes the time of the last interaction with
#    its master. This can be the last ping or command received (if the master
#    is still in the "connected" state), or the time that elapsed since the
#    disconnection with the master (if the replication link is currently down).
#    If the last interaction is too old, the slave will not try to failover
#    at all.
#
# The point "2" can be tuned by user. Specifically a slave will not perform
# the failover if, since the last interaction with the master, the time
# elapsed is greater than:
#
#   (node-timeout * slave-validity-factor) + repl-ping-slave-period
#
# So for example if node-timeout is 30 seconds, and the slave-validity-factor
# is 10, and assuming a default repl-ping-slave-period of 10 seconds, the
# slave will not try to failover if it was not able to talk with the master
# for longer than 310 seconds.
#
# A large slave-validity-factor may allow slaves with too old data to failover
# a master, while a too small value may prevent the cluster from being able to
# elect a slave at all.
#
# For maximum availability, it is possible to set the slave-validity-factor
# to a value of 0, which means, that slaves will always try to failover the
# master regardless of the last time they interacted with the master.
# (However they'll always try to apply a delay proportional to their
# offset rank).
#
# Zero is the only value able to guarantee that when all the partitions heal
# the cluster will always be able to continue.
#
# cluster-slave-validity-factor 10
 
# Cluster slaves are able to migrate to orphaned masters, that are masters
# that are left without working slaves. This improves the cluster ability
# to resist to failures as otherwise an orphaned master can't be failed over
# in case of failure if it has no working slaves.
#
# Slaves migrate to orphaned masters only if there are still at least a
# given number of other working slaves for their old master. This number
# is the "migration barrier". A migration barrier of 1 means that a slave
# will migrate only if there is at least 1 other working slave for its master
# and so forth. It usually reflects the number of slaves you want for every
# master in your cluster.
#
# Default is 1 (slaves migrate only if their masters remain with at least
# one slave). To disable migration just set it to a very large value.
# A value of 0 can be set but is useful only for debugging and dangerous
# in production.
#
# cluster-migration-barrier 1
 
# In order to setup your cluster make sure to read the documentation
# available at http://redis.io web site.
 
################################## SLOW LOG ###################################
 
# The Redis Slow Log is a system to log queries that exceeded a specified
# execution time. The execution time does not include the I/O operations
# like talking with the client, sending the reply and so forth,
# but just the time needed to actually execute the command (this is the only
# stage of command execution where the thread is blocked and can not serve
# other requests in the meantime).
# 
# You can configure the slow log with two parameters: one tells Redis
# what is the execution time, in microseconds, to exceed in order for the
# command to get logged, and the other parameter is the length of the
# slow log. When a new command is logged the oldest one is removed from the
# queue of logged commands.
 
# The following time is expressed in microseconds, so 1000000 is equivalent
# to one second. Note that a negative number disables the slow log, while
# a value of zero forces the logging of every command.
slowlog-log-slower-than 10000
 
# There is no limit to this length. Just be aware that it will consume memory.
# You can reclaim memory used by the slow log with SLOWLOG RESET.
slowlog-max-len 128
 
############################# Event notification ##############################
 
# Redis can notify Pub/Sub clients about events happening in the key space.
# This feature is documented at http://redis.io/topics/keyspace-events
# 
# For instance if keyspace events notification is enabled, and a client
# performs a DEL operation on key "foo" stored in the Database 0, two
# messages will be published via Pub/Sub:
#
# PUBLISH __keyspace@0__:foo del
# PUBLISH __keyevent@0__:del foo
#
# It is possible to select the events that Redis will notify among a set
# of classes. Every class is identified by a single character:
#
#  K     Keyspace events, published with __keyspace@<db>__ prefix.
#  E     Keyevent events, published with __keyevent@<db>__ prefix.
#  g     Generic commands (non-type specific) like DEL, EXPIRE, RENAME, ...
#  $     String commands
#  l     List commands
#  s     Set commands
#  h     Hash commands
#  z     Sorted set commands
#  x     Expired events (events generated every time a key expires)
#  e     Evicted events (events generated when a key is evicted for maxmemory)
#  A     Alias for g$lshzxe, so that the "AKE" string means all the events.
#
#  The "notify-keyspace-events" takes as argument a string that is composed
#  by zero or multiple characters. The empty string means that notifications
#  are disabled at all.
#
#  Example: to enable list and generic events, from the point of view of the
#           event name, use:
#
#  notify-keyspace-events Elg
#
#  Example 2: to get the stream of the expired keys subscribing to channel
#             name __keyevent@0__:expired use:
#
#  notify-keyspace-events Ex
#
#  By default all notifications are disabled because most users don't need
#  this feature and the feature has some overhead. Note that if you don't
#  specify at least one of K or E, no events will be delivered.
notify-keyspace-events ""
 
############################### ADVANCED CONFIG ###############################
 
# Hashes are encoded using a memory efficient data structure when they have a
# small number of entries, and the biggest entry does not exceed a given
# threshold. These thresholds can be configured using the following directives.
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
 
# Similarly to hashes, small lists are also encoded in a special way in order
# to save a lot of space. The special representation is only used when
# you are under the following limits:
list-max-ziplist-entries 512
list-max-ziplist-value 64
 
# Sets have a special encoding in just one case: when a set is composed
# of just strings that happens to be integers in radix 10 in the range
# of 64 bit signed integers.
# The following configuration setting sets the limit in the size of the
# set in order to use this special memory saving encoding.
set-max-intset-entries 512
 
# Similarly to hashes and lists, sorted sets are also specially encoded in
# order to save a lot of space. This encoding is only used when the length and
# elements of a sorted set are below the following limits:
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
 
# HyperLogLog sparse representation bytes limit. The limit includes the
# 16 bytes header. When an HyperLogLog using the sparse representation crosses
# this limit, it is converted into the dense representation.
#
# A value greater than 16000 is totally useless, since at that point the
# dense representation is more memory efficient.
# 
# The suggested value is ~ 3000 in order to have the benefits of
# the space efficient encoding without slowing down too much PFADD,
# which is O(N) with the sparse encoding. The value can be raised to
# ~ 10000 when CPU is not a concern, but space is, and the data set is
# composed of many HyperLogLogs with cardinality in the 0 - 15000 range.
hll-sparse-max-bytes 3000
 
# Active rehashing uses 1 millisecond every 100 milliseconds of CPU time in
# order to help rehashing the main Redis hash table (the one mapping top-level
# keys to values). The hash table implementation Redis uses (see dict.c)
# performs a lazy rehashing: the more operation you run into a hash table
# that is rehashing, the more rehashing "steps" are performed, so if the
# server is idle the rehashing is never complete and some more memory is used
# by the hash table.
# 
# The default is to use this millisecond 10 times every second in order to
# active rehashing the main dictionaries, freeing memory when possible.
#
# If unsure:
# use "activerehashing no" if you have hard latency requirements and it is
# not a good thing in your environment that Redis can reply form time to time
# to queries with 2 milliseconds delay.
#
# use "activerehashing yes" if you don't have such hard requirements but
# want to free memory asap when possible.
activerehashing yes
 
# The client output buffer limits can be used to force disconnection of clients
# that are not reading data from the server fast enough for some reason (a
# common reason is that a Pub/Sub client can't consume messages as fast as the
# publisher can produce them).
#
# The limit can be set differently for the three different classes of clients:
#
# normal -> normal clients
# slave  -> slave clients and MONITOR clients
# pubsub -> clients subscribed to at least one pubsub channel or pattern
#
# The syntax of every client-output-buffer-limit directive is the following:
#
# client-output-buffer-limit <class> <hard limit> <soft limit> <soft seconds>
#
# A client is immediately disconnected once the hard limit is reached, or if
# the soft limit is reached and remains reached for the specified number of
# seconds (continuously).
# So for instance if the hard limit is 32 megabytes and the soft limit is
# 16 megabytes / 10 seconds, the client will get disconnected immediately
# if the size of the output buffers reach 32 megabytes, but will also get
# disconnected if the client reaches 16 megabytes and continuously overcomes
# the limit for 10 seconds.
#
# By default normal clients are not limited because they don't receive data
# without asking (in a push way), but just after a request, so only
# asynchronous clients may create a scenario where data is requested faster
# than it can read.
#
# Instead there is a default limit for pubsub and slave clients, since
# subscribers and slaves receive data in a push fashion.
#
# Both the hard or the soft limit can be disabled by setting them to zero.
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
 
# Redis calls an internal function to perform many background tasks, like
# closing connections of clients in timeout, purging expired keys that are
# never requested, and so forth.
#
# Not all tasks are performed with the same frequency, but Redis checks for
# tasks to perform accordingly to the specified "hz" value.
#
# By default "hz" is set to 10. Raising the value will use more CPU when
# Redis is idle, but at the same time will make Redis more responsive when
# there are many keys expiring at the same time, and timeouts may be
# handled with more precision.
#
# The range is between 1 and 500, however a value over 100 is usually not
# a good idea. Most users should use the default of 10 and raise this up to
# 100 only in environments where very low latency is required.
hz 10
 
# When a child rewrites the AOF file, if the following option is enabled
# the file will be fsync-ed every 32 MB of data generated. This is useful
# in order to commit the file to the disk more incrementally and avoid
# big latency spikes.
aof-rewrite-incremental-fsync yes
```
