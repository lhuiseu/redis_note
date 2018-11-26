#1 概念

    1.是Redis高可用的解决方案。
    2.有一个或者多个sentinel实例组成的sentinel系统，可以监视多个主服务器以及这些主服务器下面的从服务器
      当某个主服务器下线的时候时候，主动将其下面挂在的从服务器升级为新的主服务器
      
#2 结构
    1.sentinelState 
        //保存所有监视的主服务器，key是主服务器的名字，value是一个sentinelRedisInstance结构的指针
        dict *masters
         
    2.sentinelRedisInstance
        //实例的运行id
        char *runid 
        //实例的地址
        sentinelAddr *addr 
        //主服务器下的所有从服务器，key是ip:port的方式，value也是一个sentinelRedisInstance结构指针
        dict *slaves
        
        //共同监视该主服务器的sentinels。包括自身。key是ip:port。value也是sentinel的实例结构指针
        dict *sentinels
        
        //当前实例的状态，如果主动下线，这个标识可以看的出来
        flag
        
    3.sentinelAddr
        char *ip
        int port
        
#3 初始化
    1. 初始化sentinelState的masters属性。这个是根据配置来初始化的
    2. 创建连接主服务器的网络连接。
       sentinel会创建两个面向主服务器的网络连接
       1）一个是命令连接，这个连接专门用于向主服务器发送命令，并接收命令回复
       2）另一个是订阅连接，这个专门用于订阅主服务器的__sentinel__:hello频道
       
   **这个订阅连接的主要作用，就是监控同一个主服务器的多个sentinel之间相互发现**
       
       因为sentinel需要与多个实例创建多个网络连接，所以sentinel使用的是异步连接
       
#4 获取主服务器的信息
    1.sentinel默认每10s向主服务器发送INFO命令，获取当前的主服务器信息。
      sentinel通过这中方式来获取主服务器下的从服务器的信息。。
    2.获取主服务器信息后，会根据信息是否变动，更新sentinel状态下的主服务器信息和从服务器信息

#5 获取从服务器信息
    1.当sentinel发现主服务器有新的从服务器时，会为这个新的从服务器创建实例结构，同时
      还会创建连接到从服务器的命令连接和订阅连接（所以其实sentinel是与所有的redis实例都连接起来了）
    2.sentinel默认每10s向从服务器发送INFO命令。根据这些，对从服务器实例结构进行更改
    
#6 向主服务器和从服务器发送消息
    1.在默认情况下,sentinel会以每2s一次的频率，通过 命令连接 向所有被监视的主服务器和从服务器发送PUBLISH命令（__sentinel__:hello频道）
    
#7 接受来自来自主服务器和从服务器的频道消息
    1.当sentinel与主服务器／从服务器建立起订阅连接后，sentinel就会通过订阅连接，向服务器发送一下命令：
    subscribe __sentinel__:hello（订阅一直持续）
    
   **也就是说对于每一个与sentinel连接的服务器，sentinel即通过 命令连接 向__sentinel__:hello频道发送信息，
       又通过订阅连接从频道中接受信息**
       
   **对于监视同一个主服务器的多个sentinal实例来说，通过订阅同一个频道，就可以其他sentinal的状态**
   
    2.sentinel为主服务器创建的实例中，存在一个sentinels的字典结构
    
    3.创建连其他sentinel的命令连接
     当sentinel发现了一个新的sentinel时，新创建一个连向新的sentinel的命令连接。
   **最终监视同一个主服务器的多个sentinel将形成相互连接的网络**
   
#8 检测主观下线状态
    1.默认情况下，sentinel会以每秒一次的频率，向与其建立了命令连接的实例（包括主服务器、从服务器、其他sentinel）
      发送PING命令，通过实例返回的PING回复，判断实例是否在线
      
    2.如果多次未收到回复超过设置的（timeout）,sentinel就会将（主服务器|从服务器|其他sentinel）标识为主观下线
    
    3.不同的sentinel判定的超时时间可能不一致，所以主观下线的判定也不一致
    
#9 检测客观下线状态
    1.当sentinel判断主服务器主观下线后，保险期间，该sentinel会向其他的sentinel进行询问，如果接收到足够数量
      的已下线判断后，sentinel将会判定该主服务器为客观下线，并对主服务器执行故障移除操作。
      
    2.不同的sentinel判定客观下线回复的数目可能不一致，所以客观下线的判定也不一致
    
    3.检测客观下线的命令和之后协商领头sentinel的命令一致，但是传参不一致
    
#10 选举领头sentinel
    1.当主服务器被判定为客观下线后，监视这个主服务器的各个sentinel会进行协商，选举产生领头sentinel.
      并有领头sentinel对下线主服务器执行故障转移工作。
    2.领头sentinel的规则和raft很相似哈，此处略，详细见：P239
    
    3.注意：sentinel是先判定为客观下线以后，才会发起领头sentinel的选举
    
#11 故障转移
    1.领头sentinel，从已经下线的主服务器的从服务器中挑选一个服务器，并将其转化为主服务器
      选出来以后执行命令：slaveof no one
      领头sentinel会将这些从服务器保存到一个列表中去，然后执行以下的规则
      规则：
      1）删除已经下线|断线状态的从服务器
      2）删除最近5s未回复sentinel的INFO命令的服务器
      3）删除所有与已下线主服务器连接断开超过down_after_ms*10 ms的从服务器，可以保证数据较新
      4）选出偏移量最大的
      5）选出运行id最小的
      
      当选出来以后，sentinel会以每秒一次的频率（平时是10s）,发送INFO命令，并观察回复中的role是否由原来的slave变成了master
    
    2.让从服务器们复制新的主服务器
      sentinel向从服务器发送：slaveof ip port 
      
    3.将已经下线的主服务器设置为新主服务器的从服务器。
      当原来的主服务器重新上线的时候，sentinel会发送slaveof命令