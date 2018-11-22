#IO模型

##1 同步阻塞IO
    用户进程调用recvfrom,然后阻塞等待数据
    等待过程：
    1）、准备数据
    2）、将数据从内核拷贝到用户空间
    
    阻塞结束
    
##2 同步非阻塞IO
    用户进程调用recvfrom，如果数据未准备好，直接返回EWOULDBLOCK
    需要定期去轮训是否数据准备好
    等待过程：
    1）、将数据从内核拷贝到用户空间
    
##3 IO多路复用
    用户进程调用select，然后阻塞等待事件OK
    注意：
    select阻塞并非是IO阻塞
    select里面的每一个socket一般都设置成非阻塞模式，select底层通过轮训判断每个socket读写是否就绪
    等待过程：
    1）、多个socket准备数据，任意一个socket准备数据OK
    2）、将数据从内核拷贝到用户空间
    
    注意：
    从流程上看，IO多路复用和同步阻塞相似，甚至还添加了监视socket，效率更差
    最大的优势在于，用户可以在一个线程里面处理多个IO请求。
    用户可以注册多个socket。
    而同步阻塞需要建立多线程才能实现。
    
   ###3.1 Select
    机制问题：
    1）、每次都需要fd_set 文件描述符集合从用户态拷贝到内核态，如何集合很大，开销也很大
    2）、内核需要遍历fd_set，如果集合很大，同样开销很大
    3）、内核对此做了限制，限制fd_set集合大小为1024
    
    底层实现：
    数组
    
   ###3.2 Poll
    改进：
    1）、改变了文件描述符集合的描述方式。fd_set => pollfd。是的pollfd集合的大小远大于1024
    
    底层实现：
    链表
   ###3.3 EPoll
    改进：
    1）、基于事件驱动的IO方式
    2）、没有文件描述符数量上的限制
    3）、使用一个描述符管理多个描述符，将 用户关心的文件描述符事件 存放到 内核的事件表中
        每一个fd，都绑定了一个回调函数
    4）、用户态和内核态的拷贝只有一次。select／poll模式存在将fd_set从用户态拷贝至内核态，以及从内核态拷贝至用户态。
        而epoll通过mmap,内核和用户空间共享一块内存
    5）、获取事件的时候无需遍历整个描述符集合，只要遍历那些 被内核IO事件异步唤醒而加入Ready队列的描述符集合
    
    两种模式：
    水平触发（LT）
    epoll_wait检测到某描述符事件就绪，并通知应用程序，应用程序可以不立即处理，下次epoll_wait的时候，会再次通知
    
    边缘触发（ET）
    只通知一次
    
    底层实现：
    哈希表
    
    注意：
    1）、在并发不高的场景，多线程+阻塞同步I/O的方式性能更好
    2）、当fd少，活跃性高的场景，select／poll的性能可能比epoll要好，毕竟epoll的通知机制需要很多的回调函数
    
    
##4 异步IO
    用户进程调用aio_read，不阻塞
    内核进行读写操作，并将数据从内核态拷贝至用户态
    处理完后通知用户进程整个操作全部完成（绑定回调）
    