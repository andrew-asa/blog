# 连接相关

## 数据结构
```

struct ngx_connection_s {
	/* 连接未使用时，data成员相当于链表的next指针。当连接被使用时，data由使用该连接的Nginx的模块定义，比如HTTP模块data指向请求 */
    void               *data;
    
    ngx_event_t        *read;   // 连接对应的读事件
    ngx_event_t        *write;  // 连接对应的写事件
 
    ngx_socket_t        fd;     // 套接字句柄
 
    ngx_recv_pt         recv;   // 直接接收网络字节流的方法
    ngx_send_pt         send;   // 直接发送网络字节流的方法
    ngx_recv_chain_pt   recv_chain;  // 以ngx_chain_t链表为参数来接收网络字节流的方法
    ngx_send_chain_pt   send_chain;  // 以ngx_chain_t链表为参数来发送网络字节流的方法
 
    ngx_listening_t    *listening;   // 该连接对应ngx_listening_t监听对象，此连接由listening监听端口的事件建立
 
    off_t               sent;   // 该连接上已发送出去的字节数 
 
    ngx_log_t          *log;    // 日志
 
	/* 内存池。一般在accept一个新连接时，会创建一个内存池，而在这个连接结束时将其销毁 */
    ngx_pool_t         *pool;
 
	/* 连接客户端的socket信息 */
    struct sockaddr    *sockaddr;
    socklen_t           socklen;
    ngx_str_t           addr_text;
 
    struct sockaddr    *local_sockaddr;  // 本地socket结构
 
    ngx_buf_t          *buffer;  // 用于节后、缓存客户端发送过来的字节流
 
    ngx_queue_t         queue;   // 将当前连接添加到ngx_cycle_t核心结构中的reuseable_connections_queue双向链表中，表示可重用连接
 
    ngx_atomic_uint_t   number;  // 连接使用次数
 
    ngx_uint_t          requests;  // 处理请求次数
 
    unsigned            buffered:8;  // 缓存中的业务类型，至多可以同时标识8个不同业务
 
    unsigned            log_error:3;     /* ngx_connection_log_error_e */
 
    unsigned            unexpected_eof:1;  // 不期待字节流结束，目前无意义
    unsigned            timedout:1;        // 连接超时
    unsigned            error:1;           // 连接处理过程中出现错误
    unsigned            destroyed:1;       // 连接已销毁
 
    unsigned            idle:1;            // 连接处于空闲状态
    unsigned            reusable:1;        // 连接可重用
    unsigned            close:1;           // 连接关闭
 
    unsigned            sendfile:1;        // 正在发送文件
    unsigned            sndlowat:1;        // 表示只有在连接套接字对应的发送缓冲区必须满足最低设置的大小阈值时，事件驱动模块才分发该事件
    unsigned            tcp_nodelay:2;   /* ngx_connection_tcp_nodelay_e */
    unsigned            tcp_nopush:2;    /* ngx_connection_tcp_nopush_e */
 
#if (NGX_HAVE_AIO_SENDFILE)
    unsigned            aio_sendfile:1;  // 使用异步I/O方式发送磁盘文件
    ngx_buf_t          *busy_sendfile;   // 该缓冲区保存待发送的文件信息
#endif

};
```


ngx_module_s
```
/* Nginx 模块的数据结构 */
#define NGX_MODULE_V1          0, 0, 0, 0, 0, 0, 1
#define NGX_MODULE_V1_PADDING  0, 0, 0, 0, 0, 0, 0, 0
 
struct ngx_module_s {
    /* 模块类别由type成员决定，ctx_index表示当前模块在type类模块中的序号 */
    ngx_uint_t            ctx_index;
    /* index 区别与ctx_index，index表示当前模块在所有模块中的序号 */
    ngx_uint_t            index;
 
    /* spare 序列保留变量，暂时不被使用 */
    ngx_uint_t            spare0;
    ngx_uint_t            spare1;
    ngx_uint_t            spare2;
    ngx_uint_t            spare3;
 
    /* 当前模块的版本 */
    ngx_uint_t            version;
 
    /* ctx指向特定类型模块的公共接口，例如在HTTP模块中，ctx指向ngx_http_module_t结构体 */
    void                 *ctx;
    /* 处理nginx.conf中的配置项 */
    ngx_command_t        *commands;
    /* type表示当前模块的类型 */
    ngx_uint_t            type;
 
    /* 下面的7个函数指针是在Nginx启动或停止时，分别调用的7中方法 */
    /* 在master进程中回调init_master */
    ngx_int_t           (*init_master)(ngx_log_t *log);
 
    /* 初始化所有模块时回调init_module */
    ngx_int_t           (*init_module)(ngx_cycle_t *cycle);
 
    /* 在worker进程提供正常服务之前回调init_process初始化进程 */
    ngx_int_t           (*init_process)(ngx_cycle_t *cycle);
    /* 初始化多线程 */
    ngx_int_t           (*init_thread)(ngx_cycle_t *cycle);
    /* 退出多线程 */
    void                (*exit_thread)(ngx_cycle_t *cycle);
    /* 在worker进程停止服务之前回调exit_process */
    void                (*exit_process)(ngx_cycle_t *cycle);
 
    /* 在master进程退出之前回调exit_master */
    void                (*exit_master)(ngx_cycle_t *cycle);
 
    /* 保留字段，未被使用 */
    uintptr_t             spare_hook0;
    uintptr_t             spare_hook1;
    uintptr_t             spare_hook2;
    uintptr_t             spare_hook3;
    uintptr_t             spare_hook4;
    uintptr_t             spare_hook5;
    uintptr_t             spare_hook6;
    uintptr_t             spare_hook7;
};

```

ngx_http_upstream_server_t 结构体

```
/* 服务器结构体 */
typedef struct {
    /* 指向存储 IP 地址的数组，因为同一个域名可能会有多个 IP 地址 */
    ngx_addr_t                      *addrs;
    /* IP 地址数组中元素个数 */
    ngx_uint_t                       naddrs;
    /* 权重 */
    ngx_uint_t                       weight;
    /* 最大失败次数 */
    ngx_uint_t                       max_fails;
    /* 失败时间阈值 */
    time_t                           fail_timeout;

    /* 标志位，若为 1，表示不参与策略选择 */
    unsigned                         down:1;
    /* 标志位，若为 1，表示为备用服务器 */
    unsigned                         backup:1;
} ngx_http_upstream_server_t;
```

ngx_peer_connection_s