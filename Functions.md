# 主要函数

1. util.c

    - 读配置：int read_conf(char* filename, tk_conf_t* conf);

    - 绑定监听：int socket_bind_listen(int port);

    - 处理连接：void accept_connection(int listen_fd, int epoll_fd, char* path);

2. epoll.c

    - 创建epoll：int sy_epoll_create(int flags);

    - 添加到epoll：int sy_epoll_add(int epoll_fd, int fd, sy_http_request_t* request, int events);

    - 从epoll删除：int sy_epoll_del(int epoll_fd, int fd, sy_http_request_t* request, int events);

    - 修改事件状态：int sy_epoll_mod(int epoll_fd, int fd, sy_http_request_t* request, int events);

    - 等待事件：int sy_epoll_wait(int epoll_fd, struct epoll_event* events, int max_events, int timeout);

    - 分发对应事件：void sy_handle_events(int epoll_fd, int listen_fd, struct epoll_event* events, int events_num, char* path);

- http.c

    - 处理请求总入口：void do_request(void* ptr);

    - 解析URI：void parse_uri(char* uri, int length, char* filename, char *query);

    - 获取文件类型：const char* get_file_type(const char* type);

    - 错误信息处理：void do_error(int fd, char* cause, char* err_num, char* short_msg, char* long_msg);

    - 响应静态文件：void serve_static(int fd, char* filename, size_t filesize, sy_http_out_t* out);

- http_parse.c

    - 解析请求行：int sy_http_parse_request_line(sy_http_request_t* request);

    - 解析请求体：int sy_http_parse_request_body(sy_http_request_t* request);

- http_request.c

    - 初始化请求头结构：int sy_init_request_t(sy_http_request_t* request, int fd, int epoll_fd, char* path);

    - 删除请求头结构：int sy_free_out_t(sy_http_out_t* out);

    - 初始化响应结构：int sy_init_out_t(sy_http_out_t* out, int fd);

    - 删除响应头结构：int sy_free_out_t(sy_http_out_t* out);

    - 获取状态码对应提示：const char* get_shortmsg_from_status_code(int status_code);

    - 关闭连接：int sy_http_close_conn(sy_http_request_t* request);

- timer.c

    - 刷新当前时间：void sy_time_update();
    
    - 初始化时间：int sy_timer_init();

    - 新增时间戳：void sy_add_timer(sy_http_request_t* request, size_t timeout, timer_handler_pt handler);

    - 删除时间戳：void sy_del_timer(sy_http_request_t* request);

    - 处理超时：void sy_handle_expire_timers();

- threadpool.c

    - 初始化线程池：sy_threadpool_t* threadpool_init(int thread_num);

    - 添加任务：threadpool_add(sy_threadpool_t* pool, void (* func)(void*), void* arg);

    - 释放线程池及任务：threadpool_free(sy_threadpool_t* pool);

    - 回收线程资源：int threadpool_destory(sy_threadpool_t* pool, int graceful);

    - 工作线程：void* threadpool_worker(void* arg);


---
