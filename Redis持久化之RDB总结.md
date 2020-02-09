1. RDB,AOF之间的优先级关系：

   - 如果服务器同时开启了AOF持久化，那么服务器会有显示使用AOF文件来还原数据

   - 只有在AOF持久化功能处于关闭状态时，服务器 才会使用RDB来还原数据库状态

2. RDB 原理：

   - RDB 生成rdb文件有两种方式，一种是SAVE命令，服务端接收该命令后，不能处理其他命令，直到rdb文件创建位置，另外一种则是BGSAVE，服务端进程派生出一个子进程，由子进程负责rdb文件的创建，服务器进程继续处理其他客户端命令请求

3. RDB 配置(配置在redis.conf中就好，redis.conf使用redis-server启动时默认就有一份，如果想要自定义一些配置，启动服务器的时候使用redis-serve /path/to/redis.conf即可)：

   - save 900 1

   - save 300 10

   - save 60 10000

   - 上面三行配置表示多少秒内redis内存数据至少进行了多少次修改才会触发BGSAVE

   - 服务端会记录dirty(自上次成功save或者bgsave之后，服务器修改了多少次数据库)，lastsave(记录上次成功save或者bgsave的unix时间戳秒)，还是用过serverCon这个周期操作函数每100秒就执行一次，分别检查是否配置文件配置指定的多个条件，是的话就执行bgsave

   - 如果不需要持久化，那么可以注释掉所有的 save 行来停用保存功能。可以直接一个空字符串来实现停用：save ""

   - 其他跟rdb相关的配置：

     - **stop-writes-on-bgsave-error** ：**默认值为yes。当启用了RDB且最后一次后台保存数据失败，Redis是否停止接收数据。这会让用户意识到数据没有正确持久化到磁盘上，否则没有人会注意到灾难（disaster）发生了。如果Redis重启了，那么又可以重新开始接收数据了

     - **rdbcompression** ；**默认值是yes。对于存储到磁盘中的快照，可以设置是否进行压缩存储。如果是的话，redis会采用LZF算法进行压缩。如果你不想消耗CPU来进行压缩的话，可以设置为关闭此功能，但是存储在磁盘上的快照会比较大。

      - **rdbchecksum** ：**默认值是yes。在存储快照后，我们还可以让redis使用CRC64算法来进行数据校验，但是这样做会增加大约10%的性能消耗，如果希望获取到最大的性能提升，可以关闭此功能。

     - **dbfilename** ：**设置快照的文件名，默认是 dump.rdb

     - **dir：****设置快照文件的存放路径，这个配置项一定是个目录，而不能是文件名。默认是和当前配置文件保存在同一目录。

4. RDB文件结构：
   ![image-20200209003913596](/Users/yungkit/Library/Application Support/typora-user-images/image-20200209003913596.png)

上图是一个整体的结构图，各部分的代表含义：

  - REDIS，EOF（实际查看dump.db文件时（od -c dump.db），为377）等大写字符表示常量,

  - db_version则是一个rdb文件协议的版本号，比如 0006, 0007

  - databases则是存储真正的数据库数据的部分（这部分可选，为空则表示数据库为空）：![image-20200209232921191](/Users/yungkit/Library/Application Support/typora-user-images/image-20200209232921191.png)

    首先是SELECTDB表示切换到对应db_number下的key-value，key_value_pairs则是存储具体的string，list, set, hash, sorted set等数据的

- Check_sum是一个8字节长的无符号证书，在服务器载入rdb文件时，通过计算REDIS, db_version, databases, EOF计算出校验和，与该值进行比较，以此来判断rdb文件是否有出错或者损坏的情况



5. RDB 的优势和劣势

   - 优势

   　　1. RDB是一个非常紧凑(compact)的文件，它保存了redis 在某个时间点上的数据集。这种文件非常适合用于进行备份和灾难恢复。

   　　2. 生成RDB文件的时候，redis主进程会fork()一个子进程来处理所有保存工作，主进程不需要进行任何磁盘IO操作。

   　　3. RDB 在恢复大数据集时的速度比 AOF 的恢复速度要快。

   - 劣势

   　　1. RDB方式数据没办法做到实时持久化/秒级持久化。因为bgsave每次运行都要执行fork操作创建子进程，属于重量级操作(内存中的数据被克隆了一份，大致2倍的膨胀性需要考虑)，频繁执行成本过高(影响性能)

   　　2. RDB文件使用特定二进制格式保存，Redis版本演进过程中有多个格式的RDB版本，存在老版本Redis服务无法兼容新版RDB格式的问题(版本不兼容)

   　　3. 在一定间隔时间做一次备份，所以如果redis意外down掉的话，就会丢失最后一次快照后的所有修改(数据有丢失)
