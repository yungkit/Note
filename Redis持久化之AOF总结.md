1. **AOF原理**：

AOF持久化是通过保存Redis服务器所执行的命令来记录数据的状态，比如AOF文件的内容：

![image-20200210014201039](/Users/yungkit/Library/Application Support/typora-user-images/image-20200210014201039.png)

抛开\r\n这些来看，很明显我们能看到select 0; set msg hello这样的信息

AOF文件是服务端每次事件循环时将执行的命令写入aof_buf缓冲区中的，然后再根据appendfsync执行的同步策略同步到aof文件中的，整个过程用伪代码表示则是：

![image-20200210014439246](/Users/yungkit/Library/Application Support/typora-user-images/image-20200210014439246.png)

flushAppendOnlyFile则是上面提到的真正落盘的实现，主要又appendfsync设置的值决定（默认为everysec）

![image-20200210014612394](/Users/yungkit/Library/Application Support/typora-user-images/image-20200210014612394.png)

![image-20200210014646045](/Users/yungkit/Library/Application Support/typora-user-images/image-20200210014646045.png)



注意上面提到的写入AOF文件只是程序上的写入，appendfsync控制的是真正持久化为文件



2. **AOF开启方式**

   **打开 redis.conf 修改以下参数：**

   **appendonly yes**    (默认no,关闭)表示是否开启AOF持久化： 

   **appendfilename “appendonly.aof”**  AOF持久化配置文件的名称：

3. **AOF文件载入与数据还原：**

   ​		在RDB和AOF都开启式，优先使用AOF，服务器启动的时候创建一个不带网络连接的伪客户端，从AOF文件中解析出命令，一句一句执行，当这些命令都执行完毕后，服务器的数据就被还原到服务关闭之前的状态了。

   ​		流程示意图：

   ![image-20200210015257171](/Users/yungkit/Library/Application Support/typora-user-images/image-20200210015257171.png)

   

   4. **AOF文件重写：**

      ​		AOF正常是一句一句地如实记录数据库执行过的命令，但实际是没必要的，以为一个key真正记录的数据大概率中间经历过许多次变化，我们只要读取它在数据库中最新的状态即可，所以，所谓的AOF重写并不是真的对AOF文件进行操作，而是服务器起一个子进程把当前服务器数据库中的所有key都遍历一遍，取到它最新的value，构造对应key类型的设置命令即可，比如list类型，只保留一句rpush key item1...itemn即可，如果n的大小超过64，则对应的命令需要分成多句。

      ​		因为AOF重写是在子进程中完成的，这时候主进程还在不断的接收处理命令，所以上面提到AOF文件重写完成后，其内容跟服务器数据库对应的状态大概率是不一致的，这就要就有一定的机制进行保证不会出现这种不一致。

      ​		Redis服务器在子进程执行AOF重写期间，设置了AOF重写缓存区，每一次事件循环除了向原来的AOF缓存区追加执行后的数据（要保证原来的AOF文件处理工作如常进行），还要在AOF重写缓冲区中追加执行后的数据，当子进程完成AOF重写工作之后，就会向父进程发送一个信号，父进程在接收到该信号后，会调用一个信号处理函数，将AOF重写缓存区的所有内容写入到新的AOF文件中，然后对新的AOF文件进行改名，原子地覆盖现有的AOF文件，完成新旧连个AOF文件替换，注意在这时候父进程，也就是我们Redis服务器的进程，是不会接收处理其他命令的。

   
