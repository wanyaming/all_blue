## Redis实战

### 数据库

#### 数据库结构

```c
typedef struct redisDb {
    // ...
    // 保存所有的键值对
    dict *dict;
    // 过期字典，保存着键的过期时间
    dict *expires
    // ...
}
```

有多个db，可以通过select 显示切换。

#### 过期键删除策略



![过期键的结构](https://i.bmp.ovh/imgs/2019/10/818c105ccba15966.png )

* 定时删除

  设置键的同时，创建一个定时器，自动触发删除任务

* 惰性删除

  放任键过期不管，操作的时候再校验

* 定期删除

  定时任务，分多次从数据库的expires字典中随机检查一部分键的过期时间，删除过期键；

  记录每次的进度，分次完成。

实际使用的是惰性删除配合定期删除

#### AOF、RDB和复制功能对过期键的处理

* RDB

  执行SAVE或BGSAVE时，会忽略过期键；

  载入RDB，如果是主服务器，忽略过期键；如果是从服务器，不做处理

* AOF

  数据库中包含的过期键无影响

* 复制

  主服务器删除过期键，从服务器接收删除命令

---

### RDB持久化

* SAVE

  阻塞Redis服务器进程，知道RDB文件创建完毕

* BGSAVE

  派生出一个子进程，负责RDB的创建，服务器进程（父进程）继续处理命令请求；

  与AOF的BGREWRITEAOF不能同时执行

Redis服务器在启动是检测到RDB文件存在，会自动载入，并处于阻塞状态，知道载入完成。

**RDB文件结构**

| REDIS  | db_version | databases 0      | database 11 | EOF      | check_sum    |
| ------ | ---------- | ---------------- | ----------- | -------- | ------------ |
| 文件头 | 文件版本号 | 多个数据库的数据 |             | 结束标志 | 文件校验比对 |

---

### AOF持久化

* 命令追加(append)

  服务器执行完写命令后，追加一条修改日志

* 文件写入于同步

  将aof_buf缓冲区的日志写入到AOF文件，写入频率有`always`（每次同步）、`everysec`（每秒同步）、`no`（由系统决定）

如果要从AOF中还原，会先创建一个不带网络的伪客户端，然后回放写命令。

**AOF重写**

查出数据库当前的状态，记录成一条日志

**BGREWRITEAOF**

* 创建子进程来进行AOF重写
* 期间主进程执行完写命令会同时往AOF缓冲区和AOF重写缓冲区发日志
* 子进程完成AOF重写工作后，向父进程发送一个信号
* 父进程将AOF重写缓冲区的内容写入新AOF
* 替换老的AOF文件

 ![AOF后台重写](https://i.bmp.ovh/imgs/2019/10/7b8ac9d031043316.png) 



### 事件

#### 文件事件

  ![IO多路复用](https://ftp.bmp.ovh/imgs/2019/11/31d823c04cd150c6.jpg ) 

* 文件事件是对套接字的抽象，每当一个套接字准备好执行连接应答（accept）、写入、读取、关闭等操作时，就会产生一个文件写事件
* I/O多路复用程序将所有事件的套接字放到一个队列里，监听它们的可读、可写事件。有序、同步、每次一个套接字向文件事件分派器传送套接字
* 文件事件分派器根据接收的套接字的类型，调用相应的时间处理器

**I/O多路复用程序的实现**

Redis包装了常见的`select`、`epoll`、`evport`、`kqueue`函数库，程序在编译时自动选择系统中性能最高的I/O多路复用函数库来作为底层实现



 ![](https://i.bmp.ovh/imgs/2019/11/9f6474c3e707b9c4.png) 

1. Redis服务器进行初始化的时候，程序会将`连接应答处理器`和`服务器监听套接字`的可读事件关联。当Redis客户端发起连接，`服务器监听套接字`产生可读事件，`连接应答处理器`进行应答，并创建客户端状态，以及`客户端套接字`，并将`客户端套接字`的可读事件与`命令请求处理器`进行关联
2. 客户端向服务器发送一个命令请求，客户端会产生一个可读事件，处理器读取命令内容，然后传给相关程序去执行
3. 客户端尝试读取命令回复时，`客户端套接字`会产生可写事件，与`命令回复处理器`关联。当处理器将命令回复全部写到套接字后，服务器会解除`客户端套接字`的可写事件与`命令回复处理器`之间的关系

#### 时间事件

* id:时间事件的唯一标识，顺序递增
* when:毫秒精度的时间戳，时间事件的执行时间
* timeProc:回调函数

时间处理器执行后，会返回下一次的执行时间，如果返回ae.h/AE_NOMORE,表示不再执行

 ![](https://i.bmp.ovh/imgs/2019/11/9dd46466e1fb0e9b.png) 

如图，时间事件都放在一个无序链表里，新事件插入表头。每当`时间事件执行器(serverCron)`运行时，会遍历整个链表

#### 事件的调度与执行

 ![](https://i.bmp.ovh/imgs/2019/11/15bf27aec4e6b641.png) 

1. 计算最近的时间事件距离执行还有多少毫秒
2. 阻塞执行文件事件，最大时间为1
3. 执行时间事件
4. 文件事件和时间事件的处理都是同步、有序、原子地

### 复制

通过SLAVEOF命令，让当前服务器成为目标服务器的从服务器。

**旧版复制功能**

* 同步 ，适用于全量。如初次复制、断线后重复制。

   ![](https://i.bmp.ovh/imgs/2019/11/6b0ceedf9a659818.png) 

  > SYNC命令是一个非常耗资源的操作
  >
  > 1. 主服务器执行BGSAVE来生成RDB，耗费大量的CPU、内存、磁盘I/O
  > 2. 将RDB文件发送给从服务器，耗费带宽和流量
  > 3. 从服务器载入RDB

* 命令传播

  适用于增量

**新版本复制**

PSYNC整合了全量和增量的功能

 ![](https://i.bmp.ovh/imgs/2019/11/086e838c4ee8b1c3.png) 

增量同步功能由一下结构实现

* 主服务器和从服务器都记录有偏移量，其值为主服务器日志的字节数，根据偏移量来判断主从是否一致

* 主服务器的复制积压缓冲区

   ![](https://i.bmp.ovh/imgs/2019/11/40d711741d8c6c88.png) 

  > 主服务器发送增量命令时，同时会保存一份到`复制积压缓冲区`,从服务器断线重连时，判断增量是否在缓冲区内。在，执行缓冲区的增量命令；不在，执行全量同步。
  >
  > 缓冲区的大小默认为1M

* 服务器运行ID

  从服务器记录主服务器的运行ID，在发生断线重连时，判断运行ID是否一致。是，可以尝试执行增量同步；不是，执行全量同步

 ![](https://i.bmp.ovh/imgs/2019/11/0392d614997c33ae.png) 

### Sentinel

Sentinel本质上是一个运行在特殊模式下的Redis服务器，启动时也是初始化Redis服务器，但不载入RDB或AOF文件；使用了和普通模式不同的命令表，来获取并维护集群中主服务器、从服务器、其它Sentinel的信息

* 订阅连接

  通过订阅主服务器或从服务器消息，来感知其它的Sentinel

* 命令连接

  与其它Sentinel：主动下线，设置领头Sentinel

  与Redis服务器：设置主从

运行流程如下

1. 启动Sentinel，监控整个集群

2. 每十秒向主服务器发送INFO命令，获取主服务器的信息

3. 上步会返回主服务器下的全部从服务器信息，更新Sentinel状态中保存的主服务器实例的slaves字典。当发现有新的从服务器出现是，会创建这个节点的实例，建立连接，并保持INFO通信

4. 订阅服务器的消息频道，来感知其它的Sentinel，维护到状态表

5. 主观下线

   Sentinel每秒一次向其它节点发送PING命令，将错误状态达到一定时间的节点标记为主观下线

6. 客观下线

   监视同一主服务器的半数Sentinel也认为该节点下线了，判定为客观下线，进行故障转移

7. 选举领头Sentinel

   故障转移只能由一个节点发起，在集群环境下，需要选举出一个领头Sentinel。选举规则如下

   > 每个发现主服务器客观下线的Sentinel会将自己设置为主Sentinel,并建议给其它Sentinel
   >
   > Sentinel接收选举后，会拒绝其它建议

8. 故障转移

   根据优先级、偏移量大小，选举一个新主节点；

   修改其它从服务器的复制目标；

   监听旧的主服务器，再其上线时置为从服务器；

***

### 集群

 ![](https://i.bmp.ovh/imgs/2019/11/56e904651844c663.png) 

每个节点都保存着一个clusterState结构，记录了集群的状态，和其它节点的信息。

集群存在16384个slot，只有每个槽都有节点负责时，集群才是上线状态

 ![](https://ftp.bmp.ovh/imgs/2019/11/edb52fc1e81ff026.jpg) 





