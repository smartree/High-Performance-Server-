# 面试常见问题

1. [C10K问题](https://www.oschina.net/translate/c10k)本质？

    - C10K问题本质是操作系统问题，创建线程多了，数据频繁拷贝（I/O，内核数据拷贝到用户进程空间、阻塞），进程/线程上下文切换消耗大，从而导致操作系统崩溃。

2. 为什么使用Reactor模型而不是Proactor模型？

    - Reactor为同步非阻塞模型，Proactor为异步非阻塞模型。核心区别在于**I/O具体是由产生I/O的主线程完成还是交给操作系统来完成**。

    - 异步I/O需要操作系统支持，因为当前Linux下AIO（异步I/O）还不够成熟，所以TKeed使用Reactor模型。

    - Reactor模型网络库：libevent, libev, libuv。

    - Proactor模型网络库：Boost.Asio，IOCP。


3. 为什么使用长连接，怎么实现的？

    - 长连接指在一个TCP连接上可以发送多个HTTP请求和响应，可以避免重复建立TCP连接导致效率低下。

    - 实现长连接主要需要以下两步： 

        - 首先要将默认的blocking I/O设为non-blocking I/O，此时read有数据则读取并返回读取的字节数，没数据则返回-1并设置errno为EAGAIN，表示下次有数据来时再读取，这么做的目的是防止长连接下数据未读取完被一直阻塞住（长连接下服务端read并不知道下一次请求会在什么时候到来）。

        - 设置epoll为ET模式（边缘触发模式），只有当状态发生变化时才会被epoll_wait返回，其他时候不会重复返回。所以长连接情况下不会因为数据未读完而每次都返回，这么做的目的是为了提高系统效率，避免每次重复检查大量处于就绪中但并没有数据的描述符。

4. epoll底层实现原理及核心参数？

    - 执行epoll_create时创建红黑树和就绪链表，执行epoll_ctl时，如果增加的句柄已经在红黑树中则立即返回，不存在则添加到描述符组成的红黑树中并向内核注册相应的回调函数。当事件到来时向就绪链表插入描述符及回调信息。

    - epoll有两种工作模式，分别是ET和LT，默认为LT，在LT下只要事件read/write等未完全处理完，每次epoll_wait时都会返回，ET模式则只在状态发生变化时返回。另外epoll_wait的第四个参数为超时时间，这里设为-1时，若无事件发生则会一直阻塞住，但若设置好timeout，每隔timeout(ms)就会被唤醒一次并返回0表示超时。

5. 定时器是做什么的，怎么实现的？

    - 初始化优先队列结构（小根堆），堆头为expire值最小的节点。

    - 新任务添加时设置超时时间（expire = add_time + timeout）。

    - 每次大循环先通过find_time函数得到最早超时请求的剩余时间（timer = expire - find_time）。 

    - 将timer填入epoll_wait的timeout位置，timer时间后自动唤醒并通过handle_expire_timers处理超时连接。

6. 如何实现线程池的？

    - 使用互斥锁保证对线程池的互斥访问，使用条件变量实现同步。

    - 初始化线程池，创建worker线程。

    - 各worker最外层为while循环，获得互斥锁的线程可以进入线程池，若无task队列为空则 pthread_cond_wait自动解锁互斥量，置该线程为等待状态并等待条件触发。若存在task则取出队列第一个任务，**之后立即开锁，之后再并执行具体操作**。这里若先执行后开锁则在task完成前整个线程池处于锁定状态，其他线程不能取任务，相当于串行操作！

    - 建立连接后，当客户端请求到达服务器端，创建task任务并添加到线程池task队列尾，当添加完task之后调用pthread_cond_signal唤醒因没有具体task而处于等待状态的worker线程。

7. 如何实现线程池同步互斥？

    处理线程同步互斥问题可以考虑互斥锁、条件变量、读写锁和信号量。本线程池为1:N模型，主线程负责监听并负责添加任务（建立连接 + 创建参数）到线程池中，之后worker线程负责取任务并执行，可供选择的同步策略可以是"互斥锁 + 条件变量"或信号量来完成。
    
    - 互斥锁 + 条件变量（线程同步）

    ```C++
    int pthread_mutex_lock(pthread_mutex_t *mptr);
    int pthread_mutex_unlock(pthread_mutex_t *mptr);
    int pthread_cond_wait(pthread_cond_t *cptr, pthread_mutex_t *mptr);
    int pthread_cond_signal(pthread_cond_t *cptr);
    ```

    - 信号量（进程、线程同步）

    ```C++
    int sem_init(sem_t *sem, int shared, unsigned int value);
    int sem_wait(sem_t *sem);
    int sem_post(sem_t *sem);
    int sem_destory(sem_t *sem);
    ```
    其实信号量初值设为1（二元信号量）时，可以实现互斥锁功能，信号量初值为N时可以实现条件变量功能。不过信号量主要上锁和解锁可以在不同线程，同步操作容易写错，另外，信号量必须先处理同步信号量再用互斥信号量包住临界区，这里写错会发生死锁情况。所以本线程池使用互斥锁 + 条件变量来实现。

8. 和生产者消费者问题区别？

    本质上依旧是生产者消费者问题，生产者消费者模型通常是对有界缓冲区进行同步操作，但在WebServer中，如果连接缓冲的大小固定的话，有可能导致新来的连接无法投入缓冲池中导致生产者线程（监听线程）被阻塞。所以目前Task任务通过链式队列实现，**但目前也在思考如果因为任务未来得及处理，但连接持续被投入线程池会不会造成溢出问题。**

9. 设置handle_for_sigpipe的目的是什么？

    - 在使用Webbench时，测试完服务器总是会挂掉，阅读Webbench源码后发现当到达设定的时间之后Webbench采取的措施是直接关闭socket而不会等待最后数据的到来。这样就导致服务器在向已关闭的socket写数据，系统发送SIGPIPE信号终止了服务器。handle_for_sigpipe函数的目的就是把SIGPIPE信号的handler设置为SIG_IGN（忽略）而非终止服务器运行。

10. 怎样才算是完整的请求处理过程？

    注：EAGAIN发生于非阻塞I/O中，当做read操作却没有数据时，会设errno为EAGAIN，表示当前没有数据，稍候再试。

    - 错误断开连接情况：

        - 发生非EAGAIN错误。

        - 解析请求失败。

    - 正常断开连接情况：

        - 执行结束且为非持久连接。

    - 重新返回循环：

        - 数据未读完。

    - 更新定时器、epoll注册信息：

        - 发生EAGAIN问题。

11. 如何实现平滑关闭和立即关闭的？

    - 当就绪任务队列有任务时：
    
        - 若为立即关闭，则直接打开互斥锁并退出线程，可用线程数（started减1）。
        
        - 若为平滑关闭，则还需要判断当前就绪队列中task是否为空，不为空则执行完就绪队列中任务再关闭。

12. 高CPU占用问题？

    - 问题描述：

        - 无任务时CPU占用率一直接近满负荷，压测时CPU占用率反而略微下降（至85%）。

    - 初步分析：

        - 通常CPU利用率高于90%一定是死循环引起的问题。

    - 故障排查：

        - 主循环体问题：

            - epoll_wait的timeout设为-1，且用strace跟踪系统调用发现已阻塞，但CPU占用率依旧很高。

            - 注释掉主执行体，CPU占用率依旧是90%以上，问题应该是在线程池上。

        - 线程池问题：

            - 排查线程池初始化函数，在worker线程最外层循环中有如下语句：

            ```C++
            while ((pool->queue_size == 0) && (pool->shutdown)){
                pthread_cond_wait(&(pool->cond), &(pool->lock));
            }
            ```
            - 本意是为了检查条件变量：任务队列是否为空和是否已关机。但这里忘了shutdown初始值为0，导致始终不会调用pthread_cond_wait来阻塞worker线程，导致CPU占用率高。实际上当无任务时使用"top -H -p pid"命令可见四个worker线程CPU使用率都在25%左右，就应该可以知道问题出在了线程池上（未及时阻塞，处于轮询状态）。

        - 修改后满负荷下各线程CPU占用率约10%（压测数据）
            
            ![压测数据](./datum/压测负载.png)


13. 抓包遇到的问题？

    - 问题：

        - 在本地使用tcpdump抓包时，无论是加上port还是src都不能抓到包。

    - 解决方法：

        - 使用tcpdump -i lo可以抓取本地环回数据。

14. 基本信息？

    - 端口号：3000

    - 线程数（worker线程）：4

    - 超时阀值：500(ms)

    - 监听最大等待队列：1024

