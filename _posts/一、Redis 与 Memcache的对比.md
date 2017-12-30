## 一、Redis 与 Memcache的对比
### 1.1 单进程单线程与单进程多线程
Redis是单进程单线程的工作模式，所有的请求都被排队处理处理，因此缓存数据没有互斥的需求，而Memcached是单进程多线程的工作模式，请求到达时，主线程会将请求分发给多个工作线程，因此必须要做数据的互斥。

在处理请求的能力上，两者是不相上下的，理论上在一台多核的机器上，Memecached的get操作的吞吐量会较Redis高。

* 纯内存操作
* 避免了加锁、锁等待、释放锁操作
* 避免了多线程上下文切换
* NIO

### 1.2 丰富与简单的数据结构
![](https://ws2.sinaimg.cn/large/006tNc79ly1fmwxvtai5fj31kw0tn76q.jpg)
Redis 有丰富的原生数据结构，包括字符串，链表，集合，有序集合，哈希表，二进制数组等，可见 Redis 能适用于更多的场景，可以当作一个数据结构数据库。

Memcached 在这方面较 Redis 逊色，只能做简单的 key/value 存储。

### 1.3 是否支持数据同步
Redis 原生支持主从复制，可以实现一主多从的场景，提高了可用性，而Memcached是不支持的

### 1.4 是否支持数据持久化
Redis 原生支持 RDB 和 AOF 两种持久化方式。Memcached 原生并不支持持久化

### 1.5 是否支持事务
Redis 支持事务，但不是真正的事务，不支持事务回滚，违反了事务的原子性

## 二、缓存的意义
![](https://ws2.sinaimg.cn/large/006tNc79ly1fmz3zm6i4sj30su0eqdm0.jpg)
根据 80/20 法则，百分之八十的业务访问集中在百分之二十的数据上，把热点数据放在cache中，来缓解B系统的压力，提高A系统的响应时间，同时还能节省资源，比如：在没有cache时，可能B系统需要10台机器才能抗住A系统的巨大访问量，现在有了cache可能B系统只需要5台就抗住A系统的压力。

## 三、Redis的网络模型
![](https://ws3.sinaimg.cn/large/006tNc79ly1fmz4l2tg7kj31160wsae3.jpg)
Redis的网格模型其实就是单线程版的Rector模型，利用I/O多路复用器（select、poll、epoll、kqueue）管理和监听多个套接字，当套接字变得可写（Writable）、可读(Readable)、有新连接（Acccptable)时，I/O多路复用器会结束阻塞状态，并返回一个可操作的套接字列表，然后通过这个列表，以有序、同步、每次一个套接字的方式向文件事件分派器传送套接字，文件事件分派器维护了事件与对应的事件处理器之间的关系，通过套接字和对应的事件，可以获取到事件处理器并执行。

## 四、网络模型代码
### 4.1 从Main函数开始说起
```
int main(int argc, char **argv) {
   ......
   // 初始化服务器配置，主要是填充 redisServer 结构体中的各种参数，从redis.conf文件中读取配置，并做初始化，比如配置：监听端口号、是否开启集群模式、是否禁用持久化、开启aof还是rdb持久化、初始化kv数据库的数量
   initServerConfig();
   ......
   // 初始化服务器
   initServer();
   ......
   // 进入事件循环
   aeMain(server.el);
}
```
main函数省略了部分代码，基本步骤如下：

* 初始化服务器配置，配置从redis.conf文件中读取得到
* 初始化服务器，主要是初始化数据库并分配内存、创建服务器接收连接的套接字等
* 进行入事件循环，利用多路复用器处理多个IO事件

### 4.2 服务器初始化代码
```
void initServer() {
    .....

    // 设置 syslog
    if (server.syslog_enabled) {
        openlog(server.syslog_ident, LOG_PID | LOG_NDELAY | LOG_NOWAIT,
            server.syslog_facility);
    }

    // 初始化并创建数据结构,比如客户端列表、从库列表等等
    server.current_client = NULL;
    server.clients = listCreate();
    server.clients_to_close = listCreate();
    server.slaves = listCreate();
    server.monitors = listCreate();
    server.slaveseldb = -1; /* Force to emit the first SELECT command. */
    server.unblocked_clients = listCreate();
    server.ready_keys = listCreate();
    server.clients_waiting_acks = listCreate();
    server.get_ack_from_slaves = 0;
    server.clients_paused = 0;

    // 创建共享对象
    createSharedObjects();
    adjustOpenFilesLimit();
    // 创建一个EventLoop对象，以便于后续的事件管理，整个Redis就维护这一个事件循环对象
    server.el = aeCreateEventLoop(server.maxclients+REDIS_EVENTLOOP_FDSET_INCR);
    // 创建数据库对象：一个redis可以维护多个db，db之间是相互隔离
    server.db = zmalloc(sizeof(redisDb)*server.dbnum);

   // 打开 TCP 监听端口，用于等待客户端的命令请求，该套接字在后面会被加入到EventLoop中进行维护
    if (server.port != 0 &&
        listenToPort(server.port,server.ipfd,&server.ipfd_count) == REDIS_ERR)
        exit(1);

  
    // 打开 UNIX 本地端口
    if (server.unixsocket != NULL) {
        unlink(server.unixsocket); /* don't care if this fails */
        server.sofd = anetUnixServer(server.neterr,server.unixsocket,
            server.unixsocketperm, server.tcp_backlog);
        if (server.sofd == ANET_ERR) {
            redisLog(REDIS_WARNING, "Opening socket: %s", server.neterr);
            exit(1);
        }
        anetNonBlock(NULL,server.sofd);
    }

    /* Abort if there are no listening sockets at all. */
    if (server.ipfd_count == 0 && server.sofd < 0) {
        redisLog(REDIS_WARNING, "Configured to not listen anywhere, exiting.");
        exit(1);
    }

    /* Create the Redis databases, and initialize other internal state. */
    // 创建并初始化数据库结构
    for (j = 0; j < server.dbnum; j++) {
        server.db[j].dict = dictCreate(&dbDictType,NULL);
        server.db[j].expires = dictCreate(&keyptrDictType,NULL);
        server.db[j].blocking_keys = dictCreate(&keylistDictType,NULL);
        server.db[j].ready_keys = dictCreate(&setDictType,NULL);
        server.db[j].watched_keys = dictCreate(&keylistDictType,NULL);
        server.db[j].eviction_pool = evictionPoolAlloc();
        server.db[j].id = j;
        server.db[j].avg_ttl = 0;
    }
	 ....
  
    // 为 serverCron() 创建时间事件
    if(aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL) == AE_ERR) {
        redisPanic("Can't create the serverCron time event.");
        exit(1);
    }

    // 为 TCP 连接关联连接应答（accept）处理器,用于接受并应答客户端的 connect() 调用
    for (j = 0; j < server.ipfd_count; j++) {
        if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
            acceptTcpHandler,NULL) == AE_ERR)
            {
                redisPanic(
                    "Unrecoverable error creating server.ipfd file event.");
            }
    }

    // 为本地套接字关联应答处理器
    if (server.sofd > 0 && aeCreateFileEvent(server.el,server.sofd,AE_READABLE,
        acceptUnixHandler,NULL) == AE_ERR) redisPanic("Unrecoverable error creating server.sofd file event.");

    /* Open the AOF file if needed. */
    // 如果 AOF 持久化功能已经打开，那么打开或创建一个 AOF 文件
    if (server.aof_state == REDIS_AOF_ON) {
        server.aof_fd = open(server.aof_filename,
                               O_WRONLY|O_APPEND|O_CREAT,0644);
        if (server.aof_fd == -1) {
            redisLog(REDIS_WARNING, "Can't open the append-only file: %s",
                strerror(errno));
            exit(1);
        }
    }

    ....
}
```
首先初始化数据库对象，根据配置不同可初始化多个数据库对象，不同数据库对象之前可以进行隔离，同步客户端可以在不同的数据库之前相互切换，创建完数据库对象后，还需要为对象里的各个属性字段进行初化始化，比如创建保存KV的dict属性、创建保存过期键的expires属性等；其次，创建一个事件循环对象，整个Redis所有事件循环都交由该对象进行进行管理，比如：它维护了一个events事件列表来管理Redis服务的所有套接字事件，维护一个fired属性来表示可操作的套接字列表等等；在创建完EventLoop对象后，系统去创建一个TCP连接套接字，用于等待客户端的连接请求，同时创建一个对应的事件并绑定相应的处理器(这是就是一个连接应答处理器)，并接该事件交由EventLoop来维护，最后创建一个时间事件处理器（即定时任务serverCron）并交由EventLoop维护。简而言之，这一步只是去初始化Redis的各种对象，为后续的事件循环执行做准备

### 4.3 事件循环
```
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
    // 进入事件循环可能会进入睡眠状态。在睡眠之前，执行预设置
    // 的函数aeSetBeforeSleepProc()。
    if (eventLoop->beforesleep != NULL)
        eventLoop->beforesleep(eventLoop);
    // AE_ALL_EVENTS 表示处理所有的事件
    aeProcessEvents(eventLoop, AE_ALL_EVENTS);
  }
}
  
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    int processed = 0, numevents;
 
    /* Nothing to do? return ASAP */
    if (!(flags & AE_TIME_EVENTS) && !(flags & AE_FILE_EVENTS)) return 0;
 
    // 判断是否需要进行阻塞以及需要阻塞时阻塞时间的长短
    if (eventLoop->maxfd != -1 ||
        ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
        int j;
        aeTimeEvent *shortest = NULL;
        struct timeval tv, *tvp;
 
        // 获取最近的时间事件
        if (flags & AE_TIME_EVENTS && !(flags & AE_DONT_WAIT))
            shortest = aeSearchNearestTimer(eventLoop);
        if (shortest) {
            // 如果时间事件存在的话
            // 那么根据最近可执行时间事件和现在时间的时间差来决定文件事件的阻塞时间
            long now_sec, now_ms;
 
            /* Calculate the time missing for the nearest
             * timer to fire. */
            // 计算距今最近的时间事件还要多久才能达到
            // 并将该时间距保存在 tv 结构中
            aeGetTime(&now_sec, &now_ms);
            tvp = &tv;
            tvp->tv_sec = shortest->when_sec - now_sec;
            if (shortest->when_ms < now_ms) {
                tvp->tv_usec = ((shortest->when_ms+1000) - now_ms)*1000;
                tvp->tv_sec --;
            } else {
                tvp->tv_usec = (shortest->when_ms - now_ms)*1000;
            }
 
            // 时间差小于 0 ，说明事件已经可以执行了，将秒和毫秒设为 0 （不阻塞）
            if (tvp->tv_sec < 0) tvp->tv_sec = 0;
            if (tvp->tv_usec < 0) tvp->tv_usec = 0;
        } else {
             
            // 执行到这一步，说明没有时间事件
            // 那么根据 AE_DONT_WAIT 是否设置来决定是否阻塞，以及阻塞的时间长度
 
            /* If we have to check for events but need to return
             * ASAP because of AE_DONT_WAIT we need to set the timeout
             * to zero */
            if (flags & AE_DONT_WAIT) {
                // 设置文件事件不阻塞
                tv.tv_sec = tv.tv_usec = 0;
                tvp = &tv;
            } else {
                /* Otherwise we can block */
                // 文件事件可以阻塞直到有事件到达为止
                tvp = NULL; /* wait forever */
            }
        }
 
        // 处理文件事件，阻塞时间由 tvp 决定
        numevents = aeApiPoll(eventLoop, tvp);
        for (j = 0; j < numevents; j++) {
            // 从已就绪数组中获取事件
            aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
 
            int mask = eventLoop->fired[j].mask;
            int fd = eventLoop->fired[j].fd;
            int rfired = 0;
 
           /* note the fe->mask & mask & ... code: maybe an already processed
             * event removed an element that fired and we still didn't
             * processed, so we check if the event is still valid. */
            // 读事件
            if (fe->mask & mask & AE_READABLE) {
                // rfired 确保读/写事件只能执行其中一个
                rfired = 1;
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
            }
            // 写事件
            if (fe->mask & mask & AE_WRITABLE) {
                if (!rfired || fe->wfileProc != fe->rfileProc)
                    fe->wfileProc(eventLoop,fd,fe->clientData,mask);
            }
 
            processed++;
        }
    }
 
    /* Check time events */
    // 执行时间事件
    if (flags & AE_TIME_EVENTS)
        processed += processTimeEvents(eventLoop);
 
    return processed; /* return the number of processed file/time events */
}
```
事件循环主要逻辑在aeProcessEvents函数中，该函数就是实现IO多路复用程序的关键逻辑，aeApiPoll就是IO多路复用的处理函数，接收eventLoop和超时时间作为参数，eventLoop维护了需要监听的所有套接字，而超时时间的用意是在没有网络事件时，系统能够响应一些定时任务，所以整个函数逻辑可以分为三个小部分：计算超时时间、IO多路复用阻塞直到有网络事件或超时时间到、处理网络事件或时间事件

**计算超时时间的逻辑**：基本上就是根据是否有时间事件来决定，如果有时间事件，就获取得最近需要执行的时间事件，然后超时时间tvp就设置成从当前时间到时间事件执行时间的间隔，如果没有时间事件就根据配置进行设置，一般会设置成阻塞直到有网络事件到达为止

**I/O多路复用**：等待网络事件或超时时间到，下面代码展示了epoll对网络时间的处理，直接调用epoll_wait函数进行处理，该函数应该是Redis封装操作系统epoll的结果，当有网络时事件时，会返回并将网络事件保存到eventLoop的fired属性中

```
static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp) {
    aeApiState *state = eventLoop->apidata;
    int retval, numevents = 0;

    // 等待时间
    retval = epoll_wait(state->epfd,state->events,eventLoop->setsize,
            tvp ? (tvp->tv_sec*1000 + tvp->tv_usec/1000) : -1);

    // 有至少一个事件就绪？
    if (retval > 0) {
        int j;

        // 为已就绪事件设置相应的模式
        // 并加入到 eventLoop 的 fired 数组中
        numevents = retval;
        for (j = 0; j < numevents; j++) {
            int mask = 0;
            struct epoll_event *e = state->events+j;

            if (e->events & EPOLLIN) mask |= AE_READABLE;
            if (e->events & EPOLLOUT) mask |= AE_WRITABLE;
            if (e->events & EPOLLERR) mask |= AE_WRITABLE;
            if (e->events & EPOLLHUP) mask |= AE_WRITABLE;

            eventLoop->fired[j].fd = e->data.fd;
            eventLoop->fired[j].mask = mask;
        }
    }
    
    // 返回已就绪事件个数
    return numevents;
}
```
**事件处理**:

网络事件处理：从eventLoop中取出可操作的套接字和对应的事件，并根据不同时的事件进行相应的处理， fe->rfileProc函数进行读请求， fe->wfileProc处理写请求

时间事件处理器：在处理完网络事件后，直接去处理时间事件processTimeEvents，我们在初始化服务器时配置的时间事件serverCron就是在这里进行处理的

然后我们继续来了解各种事件处理器的逻辑吧

### 4.4 连接应答事件处理器

### 4.5 命令请求事件处理器

### 4.6 命令回复事件处理器

### 4.7 时间事件处理器





