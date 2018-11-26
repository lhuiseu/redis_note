#1.RedisObject
   ##1.1 分类
        字符串对象
        列表对象
        哈希对象
        集合对象
        有序集合对象
   ##1.2 结构
        type 类型
        encoding 编码
        *ptr 指向底层数据结构的指针 
        refcount 引用技术
        lru 对象空转时长
   ##1.3 引用技术和共享
        redis 基于RedisObject实现了内存回收，引用技术
        redis 初始化的时候，将生成保存了"整数"值的字符串对象，以供共享
        redis不共享其他类型的对象（如字符串等），因为共享前，会判断共享对象和目标对象是否完全一致，虽然可以节省内存，但是增加CPU耗时
        当服务器占用的内存超过maxmemory时，空转时长较高的将会优先被释放
#2、RedisServer
   ##2.1 结构   
        []redisDb 服务器中所有数据库
        int dbnum 数据库量
        
        *saveparam save RDB文件保存的条件
        long long dirty 上一次RDB以后，服务器对数据库状态修改的次数
        time_t lastsave 上一次RDB的时间
        
        sds aof_buffer aof缓冲区
        
        list *clients 一个链表，保存了所有客户端状态。里面存储的是redisClient
        
        redis_client *lua_client lua客户端在服务器运行期间
        
        //如果是从服务器
        char *masterhost
        int masterport 
        
        //保存所有频道的订阅关系,key是频道，value是一个链表，订阅的客户端
        dict *pubsub_channels
        
        //保存所有模式订阅关系，一个链表，每个节点是pubsubPattern结构
        list *pubsub_patterns
        
   #####2.1.1 saveparam结构
        time_t seconds 秒数
        int changes 修改数
        
   ##2.2 备注
        默认Redis客户端的目标数据库为0号数据库。可以通过Select进行切换

#3、RedisClient
   ##3.1 结构
        redisDb *db 客户端当前正在使用的数据库
        
        int fd 客户端正在使用的套接字描述符
        
        一个输入缓冲区
        sds querybuf 输入缓冲区（可动态伸缩，最大不能超过1G）
        
        robj **argv 从输入缓冲区，解析出的命令
        
        struct redisCommand *cmd  通过解析出的命令**argvs，从命令表查询得到redisCommand结构
        
        两个输出缓冲区
        char buf[REDIS_REPLY_CHUNK_BYTES] 固定大小缓冲区
        int bufpos
        
        list *reply 链表串联多个字符串对象
        
        //事务相关
        multiState mstate
        
   ###3.1.1 附属结构
        1.multiState
            //事务队列，FIFO顺序
            multiCmd *commands
            
            //已入队命令计数
            int count
        2.multiCmd
            //参数
            robj **argv
            
            //参数个数
            int argc
            
            //命令指针
            struct redisCommand *cmd
        
        
   ##3.2 创建时间
        客户端使用connect连接的时候，服务端调用 连接事件处理器为 客户端 创建相应的客户端状态
        并将客户端状态添加到服务器状态结构 clients链表末尾
        
        

#4、RedisDb
   ##4.1 结构
        dict *dict 数据库结构其实是维护的一个字典（称为健空间）
        expires *dict 过期字典，保存了所有键的过期时间 
        
        //watch相关
        //key是被watch的数据库键，值是一个链表（监视该键的客户端）
        //当对数据库键对应的值进行变动的时候，会触发touchWatchKey函数，修改监视客户端的标识为（REDIS_DIRTY_CAS）
        //在执行事务EXEC的时候，客户端的标识不能为REDIS_DIRTY_CAS，否则将决绝事务
        dict *watched_keys
   ##4.2 备注
        在对键空间进行读操作的时候，服务器会根据键是否存在，记录键空间的命中（hit）和未命中（miss）次数
        在读取一个键以后，会更新LRU（最后一次使用时间）
        *如果服务器在读取一个键的时候发现键已经过期，服务器会删除这个键，然后执行其他操作*
        *过期字典的键是一个指针，指向键空间的键对象。值是一个long类型的整数*
   ##4.3 键过期的删除策略
   #####.定时删除
        创建一个定时器，过期时立即执行删除。内存友好，CPU不友好。
        依赖定时器，需要用到Redis服务器中的时间事件（实现是通过无序列表，查找一个时间事件的时间复杂度是O(N)）
        不现实的方案，影响服务器的响应时间和吞吐量
   #####.惰性删除
        每次获取键的时候检查，过期则删除，未过期则返回。CPU友好，内存不友好。
        容易产生一些内存泄漏
   #####.定期删除
        定期删除是以上两种方式的综合。需要控制好删除操作执行的时长和频率。以免退化成上面两种形式。
        *Redis采用惰性删除和定期删除的方式来删除过期的键*
   ##4.4 Redis的定时删除实现
        服务器周期性执行serverCron（默认100ms就会执行一次）,函数activeExpireCycle将会调用
        设置每次检查的数据库数量 db_numbers
        设置每个数据库检查的键数量 keys_numbers（每次for range 都是随机选择）
        设置全局变量 检查的db进度数 current_db
   ##4.5 RDB、AOF和复制对过期键的处理
   ##### .注意：RDB和AOF是两种持久化方式
        RDB生成的时候，过期键不会写入
        RDB载入的时候，以主服务器启动时：过期键不会载入。以从服务器启动时：过期键载入
        
        AOF写入的时候，过期键未被删除，不会为这个过期键做任何处理。如果已经因过期被删除，将append一个DEL命令
        AOF重写时，过期的键不会被写入
        
        复制模式下
        主服务删除一个过期键的时候会向从服务器发送DEL命令
        从服务器不会主动删除过期键（读写操作的时候），一切听从主服务器的命令（DEL)，以保持主从一致
#5 RDB 
   ##5.1 概念
       数据库中的数据成为数据库状态
       RDB即将数据库状态落盘
       RDB文件是二进制的，通过它可以还原数据库状态
   ##5.2 命令
       SAVE 进程阻塞直到RDB文件生成OK
       BGSAVE 创建子进程，子进程去生成RDB文件
       RDB文件载入是在服务器启动的时候，Redis没有提供专门的命令。注意：如果服务器开启了AOF持久化功能，服务器启动的时候优先使用AOF文件来恢复数据库状态。
       
       BGSAVE执行期间，客户端执行的SAVE、BGSAVE命令都是被拒绝的，避免父进程和子进程同时执行rdbSave调用，产生竞态操作
       BGSAVE执行期间，客户端执行BGREWRITEAOF,会被延迟到RDB文件生成后执行
       BGREWRITEAOF执行期间，客户端执行BGSAVE，是被拒绝的
       BGSAVE和BGREWRITEAOF都是通过子进程的形式实现
       
       RDB在载入期间，服务器进程是被阻塞的，知道载入完毕
       
       因为BGSAVE是不阻塞的，所以可以通过设置，让Redis自动间隔性保存RDB文件
       
   ##5.3 RDB间歇性保存
       同样是有serverCron定期性执行的，会检测RedisServer中saveparam数组，只要条件满足，就执行BGSAVE命令
   ##5.4 RDB文件结构
   #####5.4.1 总体结构
       "REDIS" 5字节，快速检测是否是RDB文件
       db_version 4字节，表示RDB文件的版本号
       databases 各数据库键值对数据
       EOF 1字节，正文内容结束
       checksum 8字节长的无符号整数，检测RDB文件是否出错或者受损
   #####5.4.2 databases结构
       "SELECTDB" 1字节 标示
       db_number 数据库号码
       key_value_pairs 数据对
   #####5.4.3 key_value_pairs
       "EXPIRETIME_MS" 1字节 标示
       ms 8字节长的带符号整数，以毫秒为单位的UNIX时间戳
       TYPE 类型 编码类型
       KEY 
       VALUE
   #####5.4.4
       Value的结构依赖TYPE
       可以通过**od命令**打印出 rdb文件
       
#6 AOF
   ##6.1 概念
       AOF(append only file)持久化
       AOF是通过保存Redis服务器的写命令来记录数据库状态的
       服务器启动的时候，如果开启了AOF，将执行AOF文件保存的命令来还原服务器关闭之前服务器的状态
   ##6.2 AOF持久化
   ##### 6.2.1 命令追加
       服务器执行一个写命令以后，会以协议格式将被执行的命令写入到aof_buffer缓冲区的末尾
   ##### 6.2.2 AOF文件写入和同步
       **Redis服务进程就是一个事件循环**
       文件事件 负责命令的接受 和 回复 （一个文件事件期间，可能多个命令被执行）
       时间事件 负责serverCron这些定时的任务
       
       
       服务器结束一个事件循环后，将会调用flushAppendOnlyFile函数，判断是否将aof_buffer缓冲区的内容写入和同步到AOF文件
       *注意每次其实都会写入（存储在操作系统的内存缓冲区），但是是否同步（落盘）需要有appendfsync 决定*
       
       appendfsync模式：
       always 每次都要同步，效率慢，安全性高，最多丢失一次事件循环的内容
       second 一秒同步，效率高与always,安全性较弱，最多丢失一秒内的命令
       no 等待操作系统将缓存同步，效率最高，但是安全性最弱
   ##6.3 AOF文件载入和数据还原
   ##### 6.3.1 还原流程
       .创建一个不带网络链接的伪客户端。将所有命令发给服务器去执行
       .从AOF中读取出写命令，并执行，知道所有的执行完毕
   ##6.4 AOF文件重写
   #####6.4.1 重写原由
       随着服务器的运行时间流逝，AOF文件将不断增大。通过重写生成新的AOF文件
   #####6.4.2 重写原理
   **服务器直接读取数据库的状态，用一条命令代替某项数据的历史SET操作**
       为避免执行时，造成客户端输入缓冲区溢出。当处理包含多个元素值的键时，会拆分成多条命名执行
   ##6.5 AOF后台重写
       aof_rewrite函数可以生成一个新的AOF文件，但是这个内部会存在大量的写操作，因此执行这个函数的线程将被长时间的阻塞。
       因此这个操作将在子进程中进行
           
   **子进程不阻塞服务器继续处理客户度的命令**
   **子进程带有服务器进程的副本，使用子进程而不是线程，可以在避免使用锁的情况下，保证数据安全**
       
       但是使用子进程可能使得当前数据库的状态和AOF文件保存的数据库状态不一致
   
   **为了解决上述问题，服务器设置了一个AOF重写缓冲区，当服务器执行一个写命令后，会将写命名发送给AOF缓冲区和AOF重写缓冲区** 
   **当子进程完成了AOF重写工作后，会给服务器进程发送信号，父进程收到信号后，会执行一个信号处理函数。注意这个信号处理的时候，服务器进程是无法处理客户端请求的**
   **（1）将AOF重写缓冲区的内容写入新的AOF文件**
   **（2）对新的AOF文件进行改名，原子性的覆盖现有的AOF文件**
   
   **整个重写过程，只有父进程处理信号的时候是阻塞状态，将AOF重写对性能的影响降到了最低**
  
       
        
        