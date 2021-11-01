# 
0. 预备知识
熟悉C的读者为佳，不仅仅是C的语法，更需要熟悉、并不惧怕结构、指针、函数引用和预处理技术。若你需要充电，没有比K&R更合适的了。

理解HTTP协议，毕竟你要在一个web服务器上工作。

熟悉Nginx的配置文件。如果缺乏这个经验，可以参考下列大纲。

Nginx中有4个上下文 (注：context)：main, server, upstream, location。后接一个指令，指令可以带一个或多个参数。

在main中的指令是全局的，被应用在所有适用的对象上；

在server中的指令只会被应用在特定的主机/端口上；

在upstream中的指令会被应用到一系列的后端服务器上；

在location中的指令只会被应用到特定匹配的web路径上，如("/", "/images"等)

继承方面，location继承其父亲server的配置，server继承main的配置。upstream既不向上继承，也不向下传授其配置，其独特的指令，不会被应用到其它任何地方。

在后续的内容中，会多次提到这4个上下文，千万别忘记。

那么，出发！
高度概述Nginx的模块所担当的角色
我们将介绍Nginx模块具有的三个角色：

处理器：处理请求，然后生成输出
过滤器：通过处理器维护所生成的输出
负载均衡器：当有多个后台服务器都满足条件时，为发送来的请求选择一个后台服务器。
模块做了你可能想到的与互联网服务器相关的所有”真正的工作“：当Nginx处理文件或者做为代理把请求转给一个服务器时，处理器模块就做这方面的工作；当Nginx压缩输出或者执行服务端的包含项时，它就使用过滤器模块。Nginx的“核心”实际上关心的是所有的网络和应用协议，并且建立一个能够处理请求的模块队列。这种分散式结构使你可以用一个友好的自包含单元做你想做的事情。

注意：与Apache里的模块不同，Nginx模块是非动态链接的。（换句话说，它们要完全编译到Nginx的二进制包里。）

几点人
几点人
翻译于 2013/08/16 11:00
 
怎样调用模块呢？通常在服务器启动的时候，每个处理器都有机会把自己与配置里所定义的特定位置关联起来；如果有多个处理器与一个特定的位置关联，那么只有一个获得“关联”（不过一个好的配置编写者不会让这种冲突发生。）处理器可以以三种方式返回：如果一切都正常，返回一个错误，或者处理器拒绝处理请求，转由默认的处理器处理（通常处理的是静态文件）。

如果处理器刚好是某些后台服务器的反向代理，那么这就是另一类型模块的用武之地：负载均衡器。负载均衡器考虑了请求和一组后台服务器，然后决定哪个服务器将处理请求。Nginx装载了两个负载均衡模块：轮询方法，它就像扑克游戏开始时发牌那样分发请求，还有"IP散列“方法，它确保特定客户端的多个请求都选中同一个后台服务器。

几点人
几点人
翻译于 2013/08/16 12:06
 
如果处理器没有生成错误，那么就调用了过滤器。多个过滤器可以和一个位置挂钩，这样（例如）响应就可以压缩然后再组包。它们的执行顺序是在编译的时候确定的。过滤器采用的是古典的”职责链“设计模式：调用一个过滤器，它就执行它的工作，然后调用下一个过滤器，直到调用了最后一个过滤器，这时Nginx才完成响应。

过滤器链真正酷的地方是每个过滤器都不会等待前一个过滤器结束；它可以处理前一个过滤器正在生成的输出，有点像Unix的管道。过滤器是在缓冲上运行的，缓冲器大小通常是页的大小（4K），然而你可以在ngnx.conf里更改缓冲大小。例如，这意味着模块在从后台接受到整个响应之前就可以压缩来自后端服务器的响应，然后以流的方式发送给客户端。很好！

几点人
几点人
翻译于 2013/08/16 12:32
 
简而言之，典型的处理循环是：

客户端发送 HTTP 请求 → Nginx 依据给定的 location 配置文件选择合适的handler  → (如果可用) 负载均衡器选择一个后端服务器 → Handler 处理自己的事情并将每个输出缓冲传递给第一个 filter → 第一个 filter 将输出传递给第二个 filter → 第二个传给第三个 → 第三个传给第四个 → 以此类推。 → 最后的响应被发送给客户端

我说 "典型" 是因为 Nginx 的模块调用时是有着 极致地 可定制性的。精确地定义一个模块如何运行和何时运行对模块编写者来说是个极大的负担 (我碰巧认为是个极大的负担)。调用实际上是通过一系列的回调函数执行的，而且有好多回调函数。就是说，你可以提供一个函数，让它运行在：

仅当服务器读取配置文件前
对 location 和 server 的每一个配置指令，当它出现时;
当 Nginx 初始化主配置时
当 Nginx 初始化服务器 (即 host/port) 配置时
当 Nginx 合并服务器配置和主配置时
当 Nginx 初始化 location 配置时
当 Nginx 合并 location 配置和它的父服务器配置时
当 Nginx 的主进程启动时
当一个新的 worker 进程启动时
当一个 worker 进程退出时
当主进程退出时
处理一个请求时
过滤响应头时
过滤相应体时
选择一个后端服务器时
初始化一个发往后端服务器的请求时
重新初始化一个发往后端服务器的请求时
处理来自后端服务器的响应时
完成和后端服务器的交互时
天哪！ 有点势不可挡。你已经得到了好多任由自己处置的能量，但你依然可以仅仅用其中的几个钩子和相应的函数做一些有用的事情。该是钻研一些模块的时候了。

开源中国首席投资人
开源中国首席投资人
翻译于 2013/10/04 00:11
 
2. Nginx模块组件
正如我所说的，当提及到使用Nginx模块，你有很多灵活性的选择。这一节描述是经常出现的部分。这作为一个理解模块的指南，也在你想自己写一个模块的时候充当一个参考。

2.1. 模块配置结构
模块能定义三个配置结构，一个主结构，一个服务器结构，一个位置上下文。大多数模块只需要一个位置配置。它们的命名习惯是ngx_http_<module name>_(main|srv|loc)_conf_t。这是一个从dav模块的例子：

typedef struct {
    ngx_uint_t  methods;
    ngx_flag_t  create_full_put_path;
    ngx_uint_t  access;
} ngx_http_dav_loc_conf_t;
注意Nginx有特殊的数据类型(ngx_uint_tandngx_flag_t);这是你知道和喜欢的基本数据类型的别名(如果感到好奇，参见 core/ngx_config.h ).

配置结构中的元素被模块指令填充。

徐继开
徐继开
翻译于 2013/08/28 15:51
 
2.2.模块指令
模块指令出现于静态数组ngx_command_ts。这里是从我写的一个小模块中摘取的，关于它们是如何声明的例子：

static ngx_command_t  ngx_http_circle_gif_commands[] = {
    { ngx_string("circle_gif"),
      NGX_HTTP_LOC_CONF|NGX_CONF_NOARGS,
      ngx_http_circle_gif,
      NGX_HTTP_LOC_CONF_OFFSET,
      0,
      NULL },

    { ngx_string("circle_gif_min_radius"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_num_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_circle_gif_loc_conf_t, min_radius),
      NULL },
      ...
      ngx_null_command
};
这里是ngx_command_t(我们所定义的结构)的声明, 在 core/ngx_conf_file.h 中:

struct ngx_command_t {
    ngx_str_t             name;
    ngx_uint_t            type;
    char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
    ngx_uint_t            conf;
    ngx_uint_t            offset;
    void                 *post;
};
它看起来有点多，但是每个元素都有其目的。

name是指令字符串，没有空格。它的数据类型是ngx_str_t，通常它被实例化的方式为(例如)ngx_str("proxy_pass")。注意：ngx_str_t是一个具有数据元素的数据结构，它是一个字符串，和一个len元素，这个len元素是这个字符串的长度。在大多数你需要一个字符串的地方，Nginx使用的就是这样的数据结构。
type是一些列标记组成,它用来说明指令在哪里是合法的,指令需要多少参数.合格的标识,按位是按照以下OR排列的:

NGX_HTTP_MAIN_CONF: 在主要配置里上指令是合法的
NGX_HTTP_SRV_CONF:  在服务器配置上指令是合法的
NGX_HTTP_LOC_CONF: 在本地配置上指令是合法的
NGX_HTTP_UPS_CONF: 在反向代理配置上指令是合法的
NGX_CONF_NOARGS: 指令不要参数
NGX_CONF_TAKE1: 指令需要1个精确的参数
NGX_CONF_TAKE2: 指令需要2个精确的参数
…
NGX_CONF_TAKE7: 指令需要7个精确的参数
NGX_CONF_FLAG: 指令需要一个bool值的参数
NGX_CONF_1MORE: 指令至少需要一个参数
NGX_CONF_2MORE: 指令至少需要2个参数
还有更多的配置选项,参考:core/ngx_conf_file.h.

这一系列元素都是指向一个函数的,主要为了来设置模块的部分配置项的; 很典型的就是这些函数会把你传进来的参数传给指令,然后按照合适的值保存在配置项里面去. 设置函数需要耽搁参数:

指向包含给指令传参的ngx_conf_t结构指针
指向ngx_command_t结构的指针
指向模块自定义配置的指针
旋转360
旋转360
翻译于 2013/10/12 11:16
 
在触发指令的时候会调用这些设置函数. Nginx提供了一些列的函数在自定义配置里面设置不同类型的值.这些函数是:
ngx_conf_set_flag_slot: 把"on"和"off" 转换为 1 和 0
ngx_conf_set_str_slot: 把字符串转换为ngx_str_t
ngx_conf_set_num_slot: 解析数字存为整形的值
ngx_conf_set_size_slot: 解析一组数据的大笑 ("8k", "1m",等.) 保存为size_t
不过还有更多游泳的函数(参考core/ngx_conf_file.h). 要是这些内置的函数不够好的话,模块也支持传参引入自定义函数.

这些内置的函数怎么知道要把这些数据保存到哪里去的呢?那就是下面要说的ngx_command_t中的两个参数conf和offset.conf会告知Nginx应该把值保存到模块的主配置里面,服务配置还是本地配置里面去(通过NGX_HTTP_MAIN_CONF_OFFSET,NGX_HTTP_SRV_CONF_OFFSET, 和NGX_HTTP_LOC_CONF_OFFSET).offset会确认要保存到配置结构中的哪里去.

最后,文章会指向其他的那些不重要的配置,只有在读取配置的时候才需要. 这些参数一般都是NULL.

commands数组最后是以ngnix_null_command为最后一个参数而结束.

旋转360
旋转360
翻译于 2013/10/12 11:32
 
2.3. 模块上下文
这是一个静态的ngx_http_module_t结构, 它有一些列的参数,用来创建和合并前面说的那三个配置.它的命名方式是ngx_http_<模块名>_module_ctx. 按照顺序这些参数是:

调用前时的配置
提交时的配置
创建主要的配置 (像, 内存分配和默认值)
初始化配置时 (比如,通过nginx.conf里面的配置覆盖默认值)
创建服务时的配置
跟住配置合并时的配置
创建本地配置时的配置
跟服务配置合并时的配置
要使用哪个参数主要取决于要做什么. 下面是摘自 http/ngx_http_config.h的结构定义,这样就可以很清楚每个函数的回调签名了:

typedef struct {
    ngx_int_t   (*preconfiguration)(ngx_conf_t *cf);
    ngx_int_t   (*postconfiguration)(ngx_conf_t *cf);

    void       *(*create_main_conf)(ngx_conf_t *cf);
    char       *(*init_main_conf)(ngx_conf_t *cf, void *conf);

    void       *(*create_srv_conf)(ngx_conf_t *cf);
    char       *(*merge_srv_conf)(ngx_conf_t *cf, void *prev, void *conf);

    void       *(*create_loc_conf)(ngx_conf_t *cf);
    char       *(*merge_loc_conf)(ngx_conf_t *cf, void *prev, void *conf);
} ngx_http_module_t;
你可以把不需要的函数设置为NULL, Nginx是不会解析的

旋转360
旋转360
翻译于 2013/10/12 11:45
 
大部分的操作者都会时候最后的两个: 一个是给特定位置分配内存的配置 称为ngx_http_<模块名>_create_loc_conf), 另一个是设置默认值并合并任何继承下来的配置(称为ngx_http_<模块名>_merge_loc_conf). 当然合并的函数也会检测配置是否合法,不合格的会返回错误的;终止运行服务.

下面是一个模块上下文结构的例子:

static ngx_http_module_t  ngx_http_circle_gif_module_ctx = {
    NULL,                          /* preconfiguration */
    NULL,                          /* postconfiguration */

    NULL,                          /* create main configuration */
    NULL,                          /* init main configuration */

    NULL,                          /* create server configuration */
    NULL,                          /* merge server configuration */

    ngx_http_circle_gif_create_loc_conf,  /* create location configuration */
    ngx_http_circle_gif_merge_loc_conf /* merge location configuration */
};
改深入地说明了. 这些配置项在所有的模块里面都很相似,也都使用了同样的Nginx API, 所以值得去研究下.

旋转360
旋转360
翻译于 2013/10/12 11:53
 
2.3.1. create_loc_conf
这里是一个极简单的 create_loc_conf 函数，从我写的circle_gif模块中摘取 (参看 源代码)。它具有一个指令结构 (ngx_conf_t) 并返回了一个新创建的模块配置结构(在此例中是ngx_http_circle_gif_loc_conf_t)。

static void *
ngx_http_circle_gif_create_loc_conf(ngx_conf_t *cf)
{
    ngx_http_circle_gif_loc_conf_t  *conf;

    conf = ngx_pcalloc(cf->pool, sizeof(ngx_http_circle_gif_loc_conf_t));
    if (conf == NULL) {
        return NGX_CONF_ERROR;
    }
    conf->min_radius = NGX_CONF_UNSET_UINT;
    conf->max_radius = NGX_CONF_UNSET_UINT;
    return conf;
}
第一件要注意的事是Nginx的内存分配；只要该模块使用sngx_palloc(一个malloc包装) 或ngx_pcalloc(一个calloc包装)，它将一直负责内存释放。

可能的 UNSET常量有 NGX_CONF_UNSET_UINT,NGX_CONF_UNSET_PTR,NGX_CONF_UNSET_SIZE,NGX_CONF_UNSET_MSEC, 以及全面的NGX_CONF_UNSET. UNSET告诉合并函数，哪些数值应该被覆盖。

super0555
super0555
翻译于 2013/09/04 13:35
 
2.3.2. merge_loc_conf
这里是circle_gif模块中使用的合并函数：

static char *
ngx_http_circle_gif_merge_loc_conf(ngx_conf_t *cf, void *parent, void *child)
{
    ngx_http_circle_gif_loc_conf_t *prev = parent;
    ngx_http_circle_gif_loc_conf_t *conf = child;

    ngx_conf_merge_uint_value(conf->min_radius, prev->min_radius, 10);
    ngx_conf_merge_uint_value(conf->max_radius, prev->max_radius, 20);

    if (conf->min_radius < 1) {
        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0, 
            "min_radius must be equal or more than 1");
        return NGX_CONF_ERROR;
    }
    if (conf->max_radius < conf->min_radius) {
        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0, 
            "max_radius must be equal or more than min_radius");
        return NGX_CONF_ERROR;
    }

    return NGX_CONF_OK;
}
首先要注意的是Nginx为不同的数据类型提供了很好的合并函数(ngx_conf_merge_<data type>_value);其参数为

它的位置的数值
如果#1没有设置时的继承数值
如果#1或#2都没有设置时的默认值
结果保存在第一个参数中。可用的合并函数包括ngx_conf_merge_size_value,ngx_conf_merge_msec_value, 还有一些其他的函数。请看 core/ngx_conf_file.h ，那里有完整列表。

super0555
super0555
翻译于 2013/09/04 13:08
 
小问题:既然第一个参数是作为值传递进来的，那这些函数如何写入第一个参数呢?

答:这些函数是由预处理定义的(所以他们在到达编译器前会被扩展成一些“if”语句和赋值)。

也注意一下错误是如何生产的;函数向日志文件写一些内容,返回NGX_CONF_ERROR。返回的代码中断服务器启动。(因为消息是在NGX_LOG_EMERG级别记录的,所以消息也会输出到标准输出设备上,仅供参考,core/ngx_log.h有个日志级别的列表)。

赵亮-碧海情天
赵亮-碧海情天
翻译于 2013/08/17 21:10
 
2.4.模块定义
下面我们增加一层中间层，ngx_module_t结构。变量命名为ngx_http_<module name>_module。它指向上下文和指令，以及遗留的回调（退出线程，退出进程，等等时候）。模块定义有时被用来作为查询与特定模块相关的数据的关键字。模块定义通常看起来像这样：

ngx_module_t  ngx_http_<module name>_module = {
    NGX_MODULE_V1,
    &ngx_http_<module name>_module_ctx, /* module context */
    ngx_http_<module name>_commands,   /* module directives */
    NGX_HTTP_MODULE,               /* module type */
    NULL,                          /* init master */
    NULL,                          /* init module */
    NULL,                          /* init process */
    NULL,                          /* init thread */
    NULL,                          /* exit thread */
    NULL,                          /* exit process */
    NULL,                          /* exit master */
    NGX_MODULE_V1_PADDING
};
…替代 <module name>以合适的参数。模块可以为进程/线程的创建与结束增加回调，但大多数模块都保持着简单功能。（关于传递给每个回调的参数，请看core/ngx_conf_file.h。）

super0555
super0555
翻译于 2013/09/04 13:23
 
2.5. 模块的安装
安装一个模块的适当方法取决于模块是否是一个处理程序,过滤器,或负载平衡器。所以细节留给那些各自部分进行讲解。

3. 处理程序
现在我们将把一些琐碎的模块放在显微镜下看看它们是如何工作的。

3.1. 一个处理程序的剖析(非代理)
处理程序通常做四件事:获得区域配置,生成一个适当的回应,发送数据头,发送数据体。一个处理程序有一个参数,请求结构体。一个请求结构体有很多关于客户请求的有用信息,如请求方式,URI,和headers。我们将逐步经历这些步骤。

赵亮-碧海情天
赵亮-碧海情天
翻译于 2013/08/17 21:25
 
3.1.1. 获取本地配置
这部分很简单. 你只需要传入正确的参数和模块定义调用ngx_http_get_module_loc_conf即可. 下面是我circle gif 处理的相关部分配置:

static ngx_int_t
ngx_http_circle_gif_handler(ngx_http_request_t *r)
{
    ngx_http_circle_gif_loc_conf_t  *circle_gif_config;
    circle_gif_config = ngx_http_get_module_loc_conf(r, ngx_http_circle_gif_module);
    ...
现在可以使用合并函数中我设置的任何变量了.

3.1.2. 生成相应
这部分很有趣,是模块真正意义上工作运行的部分.

下面的请求结果一目了然:

typedef struct {
...
/* the memory pool, used in the ngx_palloc functions */
    ngx_pool_t                       *pool; 
    ngx_str_t                         uri;
    ngx_str_t                         args;
    ngx_http_headers_in_t             headers_in;

...
} ngx_http_request_t;
uri是请求的地址, 例如. "/query.cgi".

args请求参数中,问号后面的(如. "name=john").

headers_in 包含很多有用的参数,比如cookies和浏览器的信息, 不过很多模块都不需要.有兴趣可以看下http/ngx_http_request.h 

有了这些参数就足够输出很多有用的信息了.全部ngx_http_request_t结构在这里可以找到 http/ngx_http_request.h.
3.1.3. 发送头信息
在结构中响应的头信息称为headers_out, 并在请求结构中引用. 处理器设置后这些头信息后调用ngx_http_send_header(r). 下面是头信息中一些有用的:

typedef stuct {
...
    ngx_uint_t                        status;
    size_t                            content_type_len;
    ngx_str_t                         content_type;
    ngx_table_elt_t                  *content_encoding;
    off_t                             content_length_n;
    time_t                            date_time;
    time_t                            last_modified_time;
..
} ngx_http_headers_out_t;
(可以在这里查看结果 http/ngx_http_request.h.)

举例来说, 假设某个模块把头信息的Content-Type 设为 "image/gif", Content-Length为 100, 返回200状态码, 那么就是这个样子:

    r->headers_out.status = NGX_HTTP_OK;
    r->headers_out.content_length_n = 100;
    r->headers_out.content_type.len = sizeof("image/gif") - 1;
    r->headers_out.content_type.data = (u_char *) "image/gif";
    ngx_http_send_header(r);
大多合法的HTTP 头信息都是有效的. 不过, 有些设置起来就不像上面那么简单了;如,含有(ngx_table_elt_t*)的content_encoding, 此时模块还需要给它分配内存.可以通过调用ngx_list_push函数来实现, 此函数需要ngx_list_t(类似数组)参数会返回一组新创建的成员列表(输入ngx_table_elt_t的成员). 下面的代码就是把Content-Encoding 设为"deflate"并发送头信息:

    r->headers_out.content_encoding = ngx_list_push(&r->headers_out.headers);
    if (r->headers_out.content_encoding == NULL) {
        return NGX_ERROR;
    }
    r->headers_out.content_encoding->hash = 1;
    r->headers_out.content_encoding->key.len = sizeof("Content-Encoding") - 1;
    r->headers_out.content_encoding->key.data = (u_char *) "Content-Encoding";
    r->headers_out.content_encoding->value.len = sizeof("deflate") - 1;
    r->headers_out.content_encoding->value.data = (u_char *) "deflate";
    ngx_http_send_header(r);
上面这种情况一般是在头部信息会同时有多个值的时候; 理论上来说对于过滤模块的添加和删除某些值保留其他值会变得容易些,因为这可以省去对他们排序为字符串的操作.

旋转360
旋转360
翻译于 2013/10/12 14:57
 
3.1.4. 发送响应体
到这里模块就已经生成好了相应并保存到内存中去了, 只需要把它分配到特定的buffer缓冲里面去就行了, 然后把这个缓冲分配到一个"链"中去,然后调用"链"中的"发送响应体"函数.

上面说的"链"有什么用呢? Nginx让处理模块(还有过滤模块的处理方式)一次性在缓冲里面生成响应信息 ; 在连表中每个元素都会指向下一个指针(最后一个指向的是NULL). 简单起见,我们每次只呈现一个缓冲.

首先, 模块需要申明缓冲和连表:

    ngx_buf_t    *b;
    ngx_chain_t   out;
接下来就是分配缓冲,并把我们的响应数据指向分配的缓冲:

    b = ngx_pcalloc(r->pool, sizeof(ngx_buf_t));
    if (b == NULL) {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0, 
            "Failed to allocate response buffer.");
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }

    b->pos = some_bytes; /* first position in memory of the data */
    b->last = some_bytes + some_bytes_length; /* last position */

    b->memory = 1; /* content is in read-only memory */
    /* (i.e., filters should copy it rather than rewrite in place) */

    b->last_buf = 1; /* there will be no more buffers in the request */
那么这样模块就把它附到链去了:

    out.buf = b;
    out.next = NULL;
最后就是, 就是发送响应体了, 返回一次性过滤链所输出的状态码:

    return ngx_http_output_filter(r, &out);
缓冲链是Nginx IO模块中很重要的部分,所以你需要注意下它们的工作方式.

细节考虑: 为什么缓冲还有一个"last_buf"变量? 通过判断"下一个"是否为空,我们怎么知道已经到达链的最后了?

回答: 链有时候可能不是完整的,比如,有很多缓冲的时候并不是所有的缓冲都在请求和响应中的.所以有的缓冲在链的末尾但并没有在请求的尾部,这些就引入了下面的...

旋转360
旋转360
翻译于 2013/10/12 16:01
 
3.2. 解剖上游(又称代理) 处理
在前面我说了有关处理(handler)生成相应的知识点. 有时候一段C语言也能帮你实现得到相应的目的, 不过你经常需要跟另一台服务器交互 (比如, 你写的是一个实现另一个网络协议的模块). 你可以自己写完所有的网络程序,但要是你收到部分相应怎么办?我想你也不愿意看到主循环还要等待自己循环接收余下响应而受阻的情况吧.幸运的是Nginx支持你对它处理后台服务的机制加上自己的钩子hook(称"上游"), 这样一来你写的模块就可以直接跟另一服务器交互而避免要接收其他响应的弊端. 这部分描述的是模块怎么跟上游(Memcached, FastCGI, 或者其他HTTP服务)交互的.

旋转360
旋转360
翻译于 2013/10/14 10:32
 
3.2.1 上游回调简介

上游模块处理函数不像其他模块的处理函数，它很少在实际中工作。它不叫做x＿http＿输出过滤器。当上游服务器准备被编写和读取时，它只是设置被当做可调用的回调函数。可实际上只有六个可用的钩子：

如果连接后端复位(正如开始创建可呼吁的第二次请求)，向上游发送的create＿requestis的请求缓冲器(或链)reinit＿requestis可以呼吁

处理＿headerprocesses上游响应的第一个位，通常保持上游有效负载的一个指针。

当客服端终止请求时，abort＿requests可以呼吁

当Nginx最后在上游读取时，finalize＿requests可以呼吁

input＿requests体过滤器被称之为响应体(例如，移去拖车)

crossmix
crossmix
翻译于 2013/09/08 20:17
 
那么这些是怎么实现相互关联的呢?下面是一个代码模块处理的简单版例子:

static ngx_int_t
ngx_http_proxy_handler(ngx_http_request_t *r)
{
    ngx_int_t                   rc;
    ngx_http_upstream_t        *u;
    ngx_http_proxy_loc_conf_t  *plcf;

    plcf = ngx_http_get_module_loc_conf(r, ngx_http_proxy_module);

/* set up our upstream struct */
    u = ngx_pcalloc(r->pool, sizeof(ngx_http_upstream_t));
    if (u == NULL) {
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }

    u->peer.log = r->connection->log;
    u->peer.log_error = NGX_ERROR_ERR;

    u->output.tag = (ngx_buf_tag_t) &ngx_http_proxy_module;

    u->conf = &plcf->upstream;

/* attach the callback functions */
    u->create_request = ngx_http_proxy_create_request;
    u->reinit_request = ngx_http_proxy_reinit_request;
    u->process_header = ngx_http_proxy_process_status_line;
    u->abort_request = ngx_http_proxy_abort_request;
    u->finalize_request = ngx_http_proxy_finalize_request;

    r->upstream = u;

    rc = ngx_http_read_client_request_body(r, ngx_http_upstream_init);

    if (rc >= NGX_HTTP_SPECIAL_RESPONSE) {
        return rc;
    }

    return NGX_DONE;
}
看起来都是很常规的功能,但最重要的部分还是回调函数. 同时还要注意gx_http_read_client_request_body这块. 这部分主要是设置当Nginx从客户端完成读取周时的回调函数.

那么每个回调函数都有什么功能呢? reinit_request,abort_request, 和finalize_request 通常是设置或重置内部某些部分的状态,自然代码就简短了. 正儿八经做大事的是create_request和process_header.

旋转360
旋转360
翻译于 2013/10/14 10:48
 
3.2.2. create_request回调函数
为了简单起见, 假设我现在有一个读一个字符而能输出二个字符的反向代理服务器. 函数怎么写?

create_request需要给单字符请求分配一个缓冲, 给这个缓冲分配一个链, 再把反向代理结构执行这个链. 就是这个样子了:

static ngx_int_t
ngx_http_character_server_create_request(ngx_http_request_t *r)
{
/* make a buffer and chain */
    ngx_buf_t *b;
    ngx_chain_t *cl;

    b = ngx_create_temp_buf(r->pool, sizeof("a") - 1);
    if (b == NULL)
        return NGX_ERROR;

    cl = ngx_alloc_chain_link(r->pool);
    if (cl == NULL)
        return NGX_ERROR;

/* hook the buffer to the chain */
    cl->buf = b;
/* chain to the upstream */
    r->upstream->request_bufs = cl;

/* now write to the buffer */
    b->pos = "a";
    b->last = b->pos + sizeof("a") - 1;

    return NGX_OK;
}
看起来不特别差吧? 当然, 实际中你可能想使用URI , ngx_str_t存在于r->uri, r->args可以获得GET参数, 不要忘记你还可以获得请求头部信息和cookie哦.

旋转360
旋转360
翻译于 2013/10/14 11:06
 
3.2.3. process_header 回调函数
现在该说说process_header了. 跟create_request一样给响应体增加指针,process_header会自动把响应的指针指向客户端将要接收的那部分去. 它还会读取反向代理的头部信息并相应的设置客户端响应的头信息.

下面有一个小例子, 主要是来读取上面的那些两子杰的响应. 假设第一个字符是有关状态的字节. 如果为问号, 我们想要给客户端返回的是一个404找不到文件状态码,那么就要忽略掉其他的字符.如果是一个空格, 我们想要给客户端返回的是包含其他字符的202状态. 好了, 这还不是最有用的协议, 只是一个很好的例子而已. 那么我们怎么来写process_header函数呢?

static ngx_int_t
ngx_http_character_server_process_header(ngx_http_request_t *r)
{
    ngx_http_upstream_t       *u;
    u = r->upstream;

    /* read the first character */
    switch(u->buffer.pos[0]) {
        case '?':
            r->header_only; /* suppress this buffer from the client */
            u->headers_in.status_n = 404;
            break;
        case ' ':
            u->buffer.pos++; /* move the buffer to point to the next character */
            u->headers_in.status_n = 200;
            break;
    }

    return NGX_OK;
}
就是这样. 操纵头部, 改变指针就ok了. 你有木有发现headers_in实际上是一个跟我们在之前说的(http/ngx_http_request.h)差不多的头部响应结构, 不过它可以用来自反向代理头部的信息填充而已. 一个真正的代理模块往往比头部处理做的事情要多的多, 不仅仅是提到那些错误处理部分,但重要的是你理解了就行.

但是.. 要是我们没有获取到来自反向代理缓冲的整个头部信息会怎么样呢?

旋转360
旋转360
翻译于 2013/10/14 11:39
 
3.2.4. 保持状态
好了,还记得我层说过的abort_request,reinit_request, 和 finalize_request是怎么重置内部状态的吗? 那是因为许多反向代理模块都有内部状态. 模块还要定义一个自定义上下文结构来跟踪它从某个反向代理所获取到的东东.这跟之前提到的"模块上下文"是不同的. 毕竟它是预定义的类型,然而自定义上下文需要有你自己需要的元素和数据 (这个要看你自己的结构来定). 自定义上下文结构需要在create_request函数中初始化, 或许会是这样的:

    ngx_http_character_server_ctx_t   *p;   /* my custom context struct */

    p = ngx_pcalloc(r->pool, sizeof(ngx_http_character_server_ctx_t));
    if (p == NULL) {
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }

    ngx_http_set_ctx(r, p, ngx_http_character_server_module);
最后一行实际上是用来注册自定义上下文结构,为了方便检索特殊请求和模块名的东西的. 在你要使用自定义上下文结构的时候 (在其他回调函数中), 只需这样调用即可:

    ngx_http_proxy_ctx_t  *p;
    p = ngx_http_get_module_ctx(r, ngx_http_proxy_module);
p将会存储当前的状态. 设置,重置,增加和减少数据...所有你想要的数据, 可以显示任何你想要的数据. 对于持久状态机大量读取反向代理数据来说是很棒的,自然也不会破坏主循环,很酷!

旋转360
旋转360
翻译于 2013/10/14 12:32
 
3.3. 安装处理
通过给回调函数增加启用模块的指令代码来安装处理.比如,我自己写的circle gif模块 ngx_command_t是这样:

    { ngx_string("circle_gif"),
      NGX_HTTP_LOC_CONF|NGX_CONF_NOARGS,
      ngx_http_circle_gif,
      0,
      0,
      NULL }
第三个参数是回调函数, 在这个例子中ngx_http_circle_gif就是处理. 指令结构(ngx_conf_t, 含有用户自己的参数),相应ngx_command_t结构和自定义配置的指针指的是本回调函数所回调的参数.对我的circle gif 模块函数就是这样:

static char *
ngx_http_circle_gif(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ngx_http_core_loc_conf_t  *clcf;

    clcf = ngx_http_conf_get_module_loc_conf(cf, ngx_http_core_module);
    clcf->handler = ngx_http_circle_gif_handler;

    return NGX_CONF_OK;
}
可以分为两步来分析: 第一, 得到本地的核心结构, 然后,把它分配给处理. 相当简单,是吧?

我把自己对模块处理所知道的都写出来了. 下面该说说存在于过滤链中的过滤模块了.

旋转360
旋转360
翻译于 2013/10/14 15:11
 
4. Filters
Filter处理生成的响应。头部filter操作HTTP头，主体filter操作响应的内容.

4.1. 剖析头部Filter
头部Filter由三个步骤组成:

决定何时操作响应
操作相应
调用下一个filter
举个例子，比如有一个简化版本的未修改的头部filter：如果客户请求头中的If-Modified-Since和响应头中的Last-Modified相符，它把响应状态设置成304。注意这个头部filter只读入一个参数：ngx_http_request_t结构体，而我们可以通过它操作到客户请求头和一会将被发送的响应消息头

static
ngx_int_t ngx_http_not_modified_header_filter(ngx_http_request_t *r)
{
    time_t  if_modified_since;

    if_modified_since = ngx_http_parse_time(r->headers_in.if_modified_since->value.data,
                              r->headers_in.if_modified_since->value.len);

/* step 1: decide whether to operate */
    if (if_modified_since != NGX_ERROR && 
        if_modified_since == r->headers_out.last_modified_time) {

/* step 2: operate on the header */
        r->headers_out.status = NGX_HTTP_NOT_MODIFIED;
        r->headers_out.content_type.len = 0;
        ngx_http_clear_content_length(r);
        ngx_http_clear_accept_ranges(r);
    }

/* step 3: call the next filter */
    return ngx_http_next_header_filter(r);
}
结构headers_out和我们在hander那一节中看到的是一样的（参考::http/ngx_http_request.h), 可以无禁止的随意操作.
4.2. 解剖主体Filter
由于主体filter一次只能处理一个缓冲,所以要用缓冲链来写主体filter就有点点麻烦. 模块需要判断是用新分配的缓冲替换和覆盖输入的缓冲,还是在我们讨论的缓冲之前或之后插入新的缓冲. 更复杂的是, 有的时候模块会接收到许多缓冲,这样就造成了有许多待完成的缓冲链. 不幸的是, Nginx 根本就没有提供高等级的API来处理这些缓冲链,造成主体filter很难识别和写入了. 但是,有些操作在实际中你还是能发现的.

一个标准的主体filter应该是这样的 (来源于Nginx代码中的"chunked" filter):

static ngx_int_t ngx_http_chunked_body_filter(ngx_http_request_t *r, ngx_chain_t *in);
第一个参数是我们熟悉的请求结构. 第二个参数是指向当前链的指针(可能包含0, 1, 或者更多的缓冲).

旋转360
旋转360
翻译于 2013/10/14 15:57
 
再来举个简单的例子. 假设我们想把 "<l!-- Served by Nginx -->" 插入到每个请求的最后面去. 首先, 我们要做的就是判断响终端缓冲是否包含在所给出的缓冲链里面,像我之前说的, 没有一个实用的API, 所以还得自己写循环:

    ngx_chain_t *chain_link;
    int chain_contains_last_buffer = 0;

    chain_link = in;
    for ( ; ; ) {
        if (chain_link->buf->last_buf)
            chain_contains_last_buffer = 1;
        if (chain_link->next == NULL)
            break;
        chain_link = chain_link->next;
    }
现在保证在没有最后缓冲的时候就这样:

    if (!chain_contains_last_buffer)
        return ngx_http_next_body_filter(r, in);
很好, 现在最后的缓冲就存到chain_link里面去了. 该分配一个新的缓冲了:

    ngx_buf_t    *b;
    b = ngx_calloc_buf(r->pool);
    if (b == NULL) {
        return NGX_ERROR;
    }
并写入数据:

    b->pos = (u_char *) "<!-- Served by Nginx -->";
    b->last = b->pos + sizeof("<!-- Served by Nginx -->") - 1;
把这个缓冲hook到一个新的链表去:

    ngx_chain_t   *added_link;

    added_link = ngx_alloc_chain_link(r->pool);
    if (added_link == NULL)
        return NGX_ERROR;

    added_link->buf = b;
    added_link->next = NULL;
最后,把刚新建的链表hook到我们之前找到的那个终端链表去:

    chain_link->next = added_link;
并根据实际情况重置"last_buf"变量:

    chain_link->buf->last_buf = 0;
    added_link->buf->last_buf = 1;
把修改后的链传给下一个过滤函数的返回值:

    return ngx_http_next_body_filter(r, in);
作为最后结果的函数往往会比像mod_perl ($response->body =~ s/$/<!-- Served by mod_perl -->/)处理起来要麻烦些.但缓冲链确实是一个很有用的结构体,它允许程序员循序渐进地处理数据,这样客户端就能及时获取到新的数据了. 不过, 在我看来, 对于缓冲链来说它极度需要一个更为简洁的接口,这样来能保证程序员不会使链出现前后不一致状态的情况. 目前, 所有的操作都由自己负责了.

旋转360
旋转360
翻译于 2013/10/15 10:11
 
4.3. Filter的安装
在post-configuration那一步的时候Filters其实就安装了.头部filters和主体filters都是在那时同步安装的.

那我们就以chunked filter模块为例来分析下. 本模块的上下文大概是这样的:

static ngx_http_module_t  ngx_http_chunked_filter_module_ctx = {
    NULL,                                  /* preconfiguration */
    ngx_http_chunked_filter_init,          /* postconfiguration */
  ...
};
下面的发生在ngx_http_chunked_filter_init中的:

static ngx_int_t
ngx_http_chunked_filter_init(ngx_conf_t *cf)
{
    ngx_http_next_header_filter = ngx_http_top_header_filter;
    ngx_http_top_header_filter = ngx_http_chunked_header_filter;

    ngx_http_next_body_filter = ngx_http_top_body_filter;
    ngx_http_top_body_filter = ngx_http_chunked_body_filter;

    return NGX_OK;
}
这里都发生了什么呢? 嗯,如果你记得的话, filters设置的时候会有一些列的功能的. 当处理生成响应的时候, 就需要调用两个函数:一个是ngx_http_output_filter,主是为了调用引用函数ngx_http_top_body_filter; 另一个是ngx_http_send_header, 主要是调用引用函数ngx_top_header_filter.

ngx_http_top_body_filter和ngx_http_top_header_filter分别是主体的头部和链表的头部. 连表上的每个链都有一个引用函数对应链表中的下一个链(引用分别是gx_http_next_body_filter和ngx_http_next_header_filter).当某个filter完成一项执行命令后,就会调用下一个filter, 直到遇到一个定义为"写入"(它包含HTPP相应)的filter的时候. 你在filter_init函数中所看到的就是模块把自己加到filter链的代码; 在"下"一个变量中它会在旧的"顶部"filter设置一个参照的东东,并把自己的函数变为新的"头部"filter. (如此, 带安装的最后一来链将会第一个被执行.ps:这句话我没想到一个比较好点的翻译,唉)

旁注: 这是怎么运行的呢?

每个filter要么会返回错误.要么使用ngx_http_next_body_filter()作为返回值:

如此, 要是filter链能运行到那个特殊定义链的最后, 就会返回"ok", 要是, 中途有错误的话链就会终止运行,Nginx会显示响应的错误信息. 它是一个单相的,易返回错误的,只有引用函数的表.很牛X的.
旋转360
旋转360
翻译于 2013/10/15 11:14
 
5. 负载均衡
负载均衡是决定哪个后端服务器将接收特殊请求的一种方式; 具体实施的是打断循环性的请求或是对请求信息进行hash. 本节会介绍负载均衡的安装和调用,举例使用upstream_hash模块(全部资源). upstream_hash会按照规定hash配置文件nginx.conf中的变量来选择后端服务器.

负载均衡模块有6部分:

授权配置指令(如hash)将要调用的注册函数
注册函数会定义规范的服务选项(比如设置高度),并实例化upstream函数
配置只要合格,初始化upstream函数就会自动调用的,初始化upstream函数将会:
把服务器名字解析到指定IP
给sockets分配空间
给对端构造函数设置回调
每次请求都会调用对端构造函数, 它主要是设置负载均衡函数能使用和操作的数据结构 ;
负载均衡函数会分配请求通道; 客户端每次(要是后端服务请求失败的话不止调用一次的)请求至少会调用一次. 有意思的事情在这里.
最后,对端析构函数在跟某个后端服务器通信(不管失败还是成功)之后会更新统计.
这些有复杂,我会分块来解说的.

旋转360
旋转360
翻译于 2013/10/16 11:21
 
5.1. 授权指令
指令的声明, 回调, 既指定了它们在哪里是合法,又确定了该调用的时候使用什么函数. 加载平衡的指令要有NGX_HTTP_UPS_CONF设置标志, 这样Nginx才会识别该指令只在反向代理代码快内部才是合法的.它应该给注册的函数分配一个指针. 下面是来自upstream_hash 模块的指令:

    { ngx_string("hash"),
      NGX_HTTP_UPS_CONF|NGX_CONF_NOARGS,
      ngx_http_upstream_hash,
      0,
      0,
      NULL },
都是很熟悉的知识点.

旋转360
旋转360
翻译于 2013/10/14 22:31
 
5.2. 注册函数
上面说到的ngx_http_upstream_hash函数就是一个注册函数, 我这样命名那是因为它是一个把周边的反向代理配置信息和自己一起注册的反向代理初始化函数. 而且, 在特定的反向代理块(高度,失败时间)里面,这个注册函数会定义哪些选项对于服务器指令是规范的. 下面是upstream_hash模块的注册函数:

ngx_http_upstream_hash(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
 {
    ngx_http_upstream_srv_conf_t  *uscf;
    ngx_http_script_compile_t      sc;
    ngx_str_t                     *value;
    ngx_array_t                   *vars_lengths, *vars_values;

    value = cf->args->elts;

    /* the following is necessary to evaluate the argument to "hash" as a $variable */
    ngx_memzero(&sc, sizeof(ngx_http_script_compile_t));

    vars_lengths = NULL;
    vars_values = NULL;

    sc.cf = cf;
    sc.source = &value[1];
    sc.lengths = &vars_lengths;
    sc.values = &vars_values;
    sc.complete_lengths = 1;
    sc.complete_values = 1;

    if (ngx_http_script_compile(&sc) != NGX_OK) {
        return NGX_CONF_ERROR;
    }
    /* end of $variable stuff */

    uscf = ngx_http_conf_get_module_srv_conf(cf, ngx_http_upstream_module);

    /* the upstream initialization function */
    uscf->peer.init_upstream = ngx_http_upstream_init_hash;

    uscf->flags = NGX_HTTP_UPSTREAM_CREATE;

    /* OK, more $variable stuff */
    uscf->values = vars_values->elts;
    uscf->lengths = vars_lengths->elts;

    /* set a default value for "hash_method" */
    if (uscf->hash_function == NULL) {
        uscf->hash_function = ngx_hash_key;
    }

    return NGX_CONF_OK;
 }
除跳过以外唯命是从外,我们还可以稍后计算$variable, 真的相当直观; 分配一个回调函数, 设置标记. 有写什么标记呢?

NGX_HTTP_UPSTREAM_CREATE: 反向代理中有服务指令. 我实在想不出什么场合你不会使用.
NGX_HTTP_UPSTREAM_WEIGHT: 让服务指令获取高度选项
NGX_HTTP_UPSTREAM_MAX_FAILS: 允许max_fail选项
NGX_HTTP_UPSTREAM_FAIL_TIMEOUT: 允许fail_timeout选项
NGX_HTTP_UPSTREAM_DOWN: 允许向下
NGX_HTTP_UPSTREAM_BACKUP:允许向上
每个模块都能访问获取这些配置的. 一切取决于模块怎么用了. 也就是说,max_fails不会自动执行; 所以的错误逻辑都是模块的作者来决定的. 更多的信息留在以后说. 现在, 我们仍还没有完成跟踪回调. 接下来, 我们来看下反向代理初始化函数 (就是之前说的init_upstream中的回调).

旋转360
旋转360
翻译于 2013/10/16 12:02
 
5.3. 反向代理初始化函数
反向代理初始化函数的主要功能是解析主机,给socket分配内存, 分配 (另一个分配功能) 回调函数. 下面是upstream_hash具体情况:

ngx_int_t
ngx_http_upstream_init_hash(ngx_conf_t *cf, ngx_http_upstream_srv_conf_t *us)
{
    ngx_uint_t                       i, j, n;
    ngx_http_upstream_server_t      *server;
    ngx_http_upstream_hash_peers_t  *peers;

    /* set the callback */
    us->peer.init = ngx_http_upstream_init_upstream_hash_peer;

    if (!us->servers) {
        return NGX_ERROR;
    }

    server = us->servers->elts;

    /* figure out how many IP addresses are in this upstream block. */
    /* remember a domain name can resolve to multiple IP addresses. */
    for (n = 0, i = 0; i < us->servers->nelts; i++) {
        n += server[i].naddrs;
    }

    /* allocate space for sockets, etc */
    peers = ngx_pcalloc(cf->pool, sizeof(ngx_http_upstream_hash_peers_t)
            + sizeof(ngx_peer_addr_t) * (n - 1));

    if (peers == NULL) {
        return NGX_ERROR;
    }

    peers->number = n;

    /* one port/IP address per peer */
    for (n = 0, i = 0; i < us->servers->nelts; i++) {
        for (j = 0; j < server[i].naddrs; j++, n++) {
            peers->peer[n].sockaddr = server[i].addrs[j].sockaddr;
            peers->peer[n].socklen = server[i].addrs[j].socklen;
            peers->peer[n].name = server[i].addrs[j].name;
        }
    }

    /* save a pointer to our peers for later */
    us->peer.data = peers;

    return NGX_OK;
}
这个函数比你期望的难些. 大多数的功能ms都有点抽象, 但实际不是这样的, 那才是我们能接收它的原因. 倒是有个简单的策略可以解决,那就是调用另一个模块的反向代理初始化函数, 让它来完成这些冗杂的功能(监听分配什么的), 然后覆盖us->peer.init回调即可.看个例子:http/modules/ngx_http_upstream_ip_hash_module.c.

在我们观点中最重要的一点就是给监听初始化函数设置一个指针, 也就像在这个例子中的http_upstream_init_upstream_hash_peer.

旋转360
旋转360
翻译于 2013/10/17 14:08
 
5.4. 对端初始化
每次请求都会调用对端初始化函数.它的功能主要是构造一组数据结构供模块在尝试找适合能服务于该请求所需要的后端服务器时使用; 这个结构会一值保存从后端服务器上通信的重复记录,所以通过它可以方便跟踪链接失败的次数, 或者是计算好的hash值. 按照惯例, 这个结构称为ngx_http_upstream_<模块名>_peer_data_t.

此外, 它还会生成两个回调函数:

get: 加载均衡方法
free: 对端析构函数 (通常是更新一些通信结束后的统计)
好像那些还不够,它还会初始化一个名叫tries的变量. 只要tries是正数, nginx就会一直重试这个平衡加载器. 当tries是0的时候, nginx就会终止. 这一切都要看get和free这两个函数是怎么设置tries的了.

下面是来自upstream_hash的对端初始化函数:

static ngx_int_t
ngx_http_upstream_init_hash_peer(ngx_http_request_t *r,
    ngx_http_upstream_srv_conf_t *us)
{
    ngx_http_upstream_hash_peer_data_t     *uhpd;
    
    ngx_str_t val;

    /* evaluate the argument to "hash" */
    if (ngx_http_script_run(r, &val, us->lengths, 0, us->values) == NULL) {
        return NGX_ERROR;
    }

    /* data persistent through the request */
    uhpd = ngx_pcalloc(r->pool, sizeof(ngx_http_upstream_hash_peer_data_t)
	    + sizeof(uintptr_t) 
	      * ((ngx_http_upstream_hash_peers_t *)us->peer.data)->number 
                  / (8 * sizeof(uintptr_t)));
    if (uhpd == NULL) {
        return NGX_ERROR;
    }

    /* save our struct for later */
    r->upstream->peer.data = uhpd;

    uhpd->peers = us->peer.data;

    /* set the callbacks and initialize "tries" to "hash_again" + 1*/
    r->upstream->peer.free = ngx_http_upstream_free_hash_peer;
    r->upstream->peer.get = ngx_http_upstream_get_hash_peer;
    r->upstream->peer.tries = us->retries + 1;

    /* do the hash and save the result */
    uhpd->hash = us->hash_function(val.data, val.len);

    return NGX_OK;
}
也没有想象的那么糟糕. 现在我们就该选择反向代理服务器了.

旋转360
旋转360
翻译于 2013/10/17 14:37
 
5.5. 负载均衡函数
该上主角了. 这才是重点. 也是模块选择负载均衡的地方.标准的函数是:

static ngx_int_t 
ngx_http_upstream_get_<module_name>_peer(ngx_peer_connection_t *pc, void *data);
上面的data就是我们关注客户交互的有用信息结构体. pc会获取我们将要链接的服务器信息. 负载均衡函数的功能是传forpc->sockaddr,pc->socklen, andpc->name等数据. 那是你有网络编程知识的话, 这些变量你可能见过; 但实际上跟我们手边的工作比起来还是没那么重要.我们并不需要知道它们代表什么; 我们只需要知道在哪里去给它们找到合适的数据就行了.

本函数需要找到一些列可用的服务, 选择其中的一个, 把值分配给pc. 看下upstream_hash 是怎么实现的.

upstream_hash会预先把服务列表存到ngx_http_upstream_hash_peer_data_t回到之前ngx_http_upstream_init_hash结果中.现在结构体就会以data的形式获取了:

    ngx_http_upstream_hash_peer_data_t *uhpd = data;
对端列表就存在uhpd->peers->peer中去了. 下面我就从对端数组中取出一个元素,按照服务数把这个计算好的hash值分割出来:

    ngx_peer_addr_t *peer = &uhpd->peers->peer[uhpd->hash % uhpd->peers->number];
看下结果:

    pc->sockaddr = peer->sockaddr;
    pc->socklen  = peer->socklen;
    pc->name     = &peer->name;

    return NGX_OK;
就这样了! 要是load-balancer返回NGX_OK的话,那就意味着, "干吧,再试". 要是返回NGX_BUSY的话,那就意味着后端服务不可用, Nginx 必须再次尝试.

但是… 怎么跟踪那些到底是什么不可用呢? 要是我们不再次尝试的话怎样呢?

旋转360
旋转360
翻译于 2013/10/17 16:11
 
5.6. 对端析构函数
当有反向代理链接发生后对端析构函数就会执行; 主要是为了跟踪失败记录下面是函数原型:

void 
ngx_http_upstream_free_<module name>_peer(ngx_peer_connection_t *pc, void *data, 
    ngx_uint_t state);
前两个函数跟我们之前看到的load-balancer函数中是一样的. 第三个参数是一个反映连接是否成功的状态变量. 它可能包含的是NGX_PEER_FAILED(连接失败的) 和NGX_PEER_NEXT(不管是连接失败还是成功应用都有返回值)两个位操作的或值. 0就是成功.

这都取决于模块开发者想怎么处理这些失败的事件. 如果真要用的话就会把结果存在data里面去,是一个指向每个自定义请求数据的结构体.

不过本函数最重要的目的是如果在请求中想终止Nginx保持尝试请求load-balancer的话就把pc->tries这只为0. 所以最简单可以是这样写:

    pc->tries = 0;
这样就能保证要是后端有错误的话就能给客户端返回一个502代理错误.
这是一个从 upstream_hash 模块中取出的更复杂的例子。如果一个后端连接失败，它在一个位向量（被称为 tried，一个由 uintptr_t 类型元素组成的数组）中标记出它，然后继续寻找直到它找到一个非失败的新后端。

#define ngx_bitvector_index(index) index / (8 * sizeof(uintptr_t))
#define ngx_bitvector_bit(index) (uintptr_t) 1 << index % (8 * sizeof(uintptr_t))

static void
ngx_http_upstream_free_hash_peer(ngx_peer_connection_t *pc, void *data,
    ngx_uint_t state)
{
    ngx_http_upstream_hash_peer_data_t  *uhpd = data;
    ngx_uint_t                           current;

    if (state & NGX_PEER_FAILED
            && --pc->tries)
    {
        /* the backend that failed */
        current = uhpd->hash % uhpd->peers->number;

       /* mark it in the bit-vector */
        uhpd->tried[ngx_bitvector_index(current)] |= ngx_bitvector_bit(current);

        do { /* rehash until we're out of retries or we find one that hasn't been tried */
            uhpd->hash = ngx_hash_key((u_char *)&uhpd->hash, sizeof(ngx_uint_t));
            current = uhpd->hash % uhpd->peers->number;
        } while ((uhpd->tried[ngx_bitvector_index(current)] & ngx_bitvector_bit(current)) && --pc->tries);
    }
}
因为负载均衡函数只会看到 uhpd->hash 的新值，所以这样可行。

许多程序不需要重试或高可用性的逻辑，但是如您所见，使用这里的数行代码提供它是可能的。

开源中国首席投资人
开源中国首席投资人
翻译于 2013/10/01 23:37
 
6. 编写并编译一个新的 Nginx 模块
现在，您应该准备好去查看一个 Nginx 模块，并尝试去理解发生了什么事(当然您知道可以在哪里寻求帮助）。在 src/http/modules/ 上查看可用的模块。挑选一个一个和您试图去完成的模块相似的模块并仔细查看它。似曾相识的东西？它应当是。参阅这份指南和模块源代码以便理解发生了什么。

但是 Emiller 并没有写一本 阅读 Nginx 模块的进球指南(Balls-In Guide to Reading Nginx Modules)。 该死的。这是一份 出球指南(Balls-Out Guide)。我们没有在阅读。我们在写作。在创造。在与世界分享。

开源中国首席投资人
开源中国首席投资人
翻译于 2013/10/01 22:47
 
首先，你需要一个放置你的模块的地方。在硬盘的任何地方为你的模块建立一个文件夹，但要和 Nginx 源代码（请确保有来自于   nginx.net 的最新副本）分开。文件夹中应当包含两个以下列字符串开头的文件：
"config"
"ngx_http_<your module>_module.c"
"config" 文件将被包含在 ./configure 中，它的内容依赖于模块的类型。

filter 模块的 "config":

ngx_addon_name=ngx_http_<your module>_module
HTTP_AUX_FILTER_MODULES="$HTTP_AUX_FILTER_MODULES ngx_http_<your module>_module"
NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ngx_http_<your module>_module.c"
其他模块的 "config" :

ngx_addon_name=ngx_http_<your module>_module
HTTP_MODULES="$HTTP_MODULES ngx_http_<your module>_module"
NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ngx_http_<your module>_module.c"
现在就到 C 文件了。我建议拷贝一个和你想做的模块要做的事类似的已有模块，但以 "ngx_http_<your module>_module.c" 重命名它。像这样让它成为你的模块，就像你改变行为适应自己的需要一样，当要理解和重塑不同的部分时，可参考这篇指南。
 
当您准备好编译时，仅需进入 Nginx 目录键入

./configure --add-module=path/to/your/new/module/directory
接着和平常一样 make make install。如果一切都好，您的模块将正好编译完成。很漂亮，对吧？没必要去弄糟 Nginx 源码，同时将您的模块加到一个新版本的 Nginx 上是非常容易的，仅需使用相同的 ./configure 命令。顺便说一下，如果您的模块需要某些动态链接库，您可以将这个添加到您的 "config" 文件：

CORE_LIBS="$CORE_LIBS -lfoo"
这里 foo 是您需要的库。如果您创建了一个酷的或者有用的模块，请确保发送一条记录到 Nginx 邮件列表 同时分享您的工作。

开源中国首席投资人
开源中国首席投资人
翻译于 2013/10/01 23:21
 
7. 进阶主题
这篇指南包含了 Nginx 模块开发的基础内容。关于编写更复杂的模块的指南，请查阅 Emiller 的 Nginx 模块开发高级主题（Emiller's Advanced Topics In Nginx Module Development）。

附录 A: 代码参考
[Nginx源码树（交叉引用）](http://lxr.evanmiller.org/http/source/)
[Nginx 模块目录（交叉引用）Nginx module directory (cross-referenced)]()
[示例插件：circle_gif Example addon: circle_gif]()
[示例插件：upstream_hash Example addon: upstream_hash]()
[示例插件：upstream_fair Example addon: upstream_fair](https://github.com/gnosek/nginx-upstream-fair/tree/master)

附录 B: 修订历史
January 16, 2013: 修正5.5中的代码例子
December 20, 2011: 修正4.2中的代码例子（再一次）
March 14, 2011: 修正4.2中的代码例子（再次）
November 11, 2009: 修正4.2中的代码例子
August 13, 2009: 结构重组，将高级主题（ Advanced Topics ）移动到另一篇独立的文章
July 23, 2009: 修正 3.5.3 中的代码例子
December 24, 2008: 修正 3.4 中的代码例子
July 14, 2008: 增加子请求的信息；细微的结构重组
July 12, 2008: 增加 Grzegorz Nosek 的指南到共享内存
July 2, 2008: 修正 filter modules 中的 "config" 文件;重写绪论；增加了 TODO 段
May 28, 2007: 将负载均衡器的例子改为更简单的 upstream_hash 模块
May 19, 2007: body filter 例子中的错误修正
May 4, 2007: 增加关于负载均衡器的信息
April 28, 2007: 初稿