# Nginx 配置解析

## 概述

Nginx是使用一个master进程来管理多个worker进程提供服务。**master负责管理worker进程，而worker进程则提供真正的客户服务**，worker进程的数量一般跟服务器上CPU的核心数相同，worker之间通过一些进程间**通信机制**实现负载均衡等功能。

![Nginx&#x8FDB;&#x7A0B;&#x4E4B;&#x95F4;&#x7684;&#x5173;&#x7CFB;](../.gitbook/assets/image%20%2834%29.png)

Nginx服务启动时会读入配置文件，后续的行为则按照配置文件中的指令进行。Nginx的配置文件时纯文本文件，默认安装Nginx后，其配置文件均在`/usr/local/ngin/conf/`目录下。其中，`nginx.conf`为主配置文件。

Nginx配置文件是以block（块）形式组织的，每个block都是一个块名字和一对大括号”{}“表示组成，block分为几个层级，整个配置文件为main层级，即最大的层级；在main层级下可以有event、http、mail等层级，而http中又会有server block，server block中可以包含location block。即块之间是可以嵌套的，内层块继承外层块。最基本的配置项语法格式是”配置项名 配置项值1 配置项值2 配置项值3 ...“

#### 各个层级介绍如下：

* 全局块：配置影响nginx全局的指令。一般有运行nginx服务器的用户组，nginx进程pid存放路径，日志存放路径，配置文件引入，允许生成worker process数等。
* events块：配置影响nginx服务器或用户的网络连接。有每个进程的最大连接数，选取哪种事件驱动模型处理连接请求，是否允许同时接收多个网络连接，开启多个网络连接序列化等。
* http块：可以嵌套多个server，配置代理，缓存，日志定义等绝大多数功能和第三方模块的配置。如文件引入，mime-type定义，日志自定义，是否适用sendfile传输文件，连接超时时间，单连接请求数等。
* server块：配置虚拟主机的相关参数，一个http中可以由多个server
* location块：配置请求的路由，以及各种页面的处理情况

每个层级可以有自己的指令（directive），例如worker\_processes是一个main层级指令，它指令Nginx服务的worker进程数量。**有的指令只能在一个层级中配置**，如worker\_processes只能存在于main中，而**有的指令可以存在于多个层级**，在这种情况下，子block会继承父block的配置，**同时如果子block配置了与父block不同的指令，则会覆盖掉父block的配置**。指令的格式是”指令名 参数1 参数2 ... 参数N;“，注意参数间可用**任意数量空格分隔**，**最后要加分号**。

![Nginx&#x914D;&#x7F6E;&#x6587;&#x4EF6;&#x901A;&#x5E38;&#x7ED3;&#x6784;](../.gitbook/assets/image%20%2815%29.png)

## Nginx服务的基本配置项

Nginx服务运行时，需要加载几个**核心模块**和一个**事件模块**，这些模块运行时所支持的配置项称为**基本配置项**，基本配置项大概可以分为以下四类：

* 用于调试、定位的配置项
* 正常运行的必备配置项
* 优化性能的配置项
* 事件类配置项

各个配置项的具体实现如下：

```bash
### Nginx 服务基本配置项

## 1、用于调试、定位的配置项

# 功能：以守护进程Nginx运行方式
# 语法：daemon off | on;
# 默认：daemon on;

# 功能：master/worker工作方式
# 语法：master_process on | off;
# 默认：master_process on;

# 功能：error日志设置
#                路径        错误级别，从左至右级别增大，若设定一个级别后，则在输出的日志文件中只输出级别大于或等于已设定的级别      
# 语法：error_log /path/file [debug | info | notice | warn | error | crit | alert | emerg];
# 默认：errpr_log logs/error.log error;

# 功能：处理特殊调试点，用于跟踪调试Nginx的
# 语法：debug_points [stop | abort]

# 功能：仅对指定的客户端输出debug级别的日志
# 语法：debug_connection [IP | DIR]

# 功能：限制coredump核心转储文件的大小
# 语法：worker_rlimit_core size;

# 功能：指定coredump文件的生成目录
# 语法：working_directory path;

## 2、正常运行的配置项

# 功能：定义环境变量
# 语法：env VAR | VAR=VALUE; 
# 说明：VAR是变量名，VALUE是目录

# 功能：嵌入其它配置文件
# 语法：include /path/file; 
# 说明：include配置项可以将其它配置文件嵌入到Nginx的nginx.conf文件中

# 功能：pid的文件路径
# 语法：pid path/file;
# 默认：pid logs/nginx.pid;
# 说明：保存master进程ID的pid文件存放路径

# 功能：Nginx worker可运行的用户及用户组
# 语法：user username [groupname];
# 默认：user nobody nobody;

# 功能：指定Nginx worker进程可打开最大句柄个数
# 语法：worker_rlimit_nofile limit;

# 功能：限制信号队列
# 语法：worker_rlimit_sigpending limit;
# 说明：设置每个用户发送给Nginx的信号队列大小，超出则丢弃

## 3、优化性能的配置项

# 功能：Nginx worker进程的个数
# 语法：worker_process number;
# 默认：worker_process 1;

# 功能：绑定Nginx worker进程到指定的CPU内核
# 语法：worker_cpu_affinity cpumask [cpumask ...]

# 功能：SSL硬件加速
# 语法：ssl_engine device;

# 功能：系统调用gettimeofday的执行频率
# 语法：timer_resolution t;

# 功能：Nginx worker进程优先级设置
# 语法：worker_priority nice;
# 默认：worker_priority 0;

## 4、事件类配置项

# 功能：是否打开accept锁
# 语法：accept_mutex [on | off];

# 功能：lock文件的路径
# 语法：lock_file path/file;

# 功能：使用accept锁后到真正建立连接之间的延迟事件
# 语法：accept_mutex_delay Nms;

# 功能：批量建立新连接
# 语法：multi_accept [on | off];

# 功能：选择事件模型
# 语法：use [kqueue | rtisg | epoll | /dev/poll | select | poll | eventport];meig

# 功能：每个worker进程的最大连接数
# 语法：worker_connections number;
```



## HTTP核心模块的配置

 具体可以参看《[Nginx 中 HTTP 核心模块配置](http://nginx.org/en/docs/http/ngx_http_core_module.html)》

```bash
### HTTP 核心模块配置的功能

## 虚拟主机与请求分发

# 功能：监听端口
# 语法：listen address:port [default | default_server | [backlong=num | rcvbuf=size |
#      sndbuf=size | accept_filter | deferred | bind | ipv6only=[on | off] | ssl]];
# 默认：listen:80;
# 说明：
#     default或default_server：将所在的server块作为web服务的默认server块，当请求无法匹配配置文件中的所有主机时，就会选择默认的虚拟主机；
#     backlong=num：表示TCP中backlog队列存放TCP新连接请求的大小，默认是-1，表示不予设置；
#     rcvbuf=size：设置监听句柄SO_RCVBUF的参数；
#     sndbuf=size：设置监听句柄SO_SNDBUF的参数；
#     accept_filter：设置accept过滤器，只对FreeBSD操作系统有用；
#     deferred：设置该参数后，若用户发起TCP连接请求，并且完成TCP三次握手，但是若用户没有发送数据，则不会唤醒worker进程，直到发送数据；
#     bind：绑定当前端口/地址对，只有同时对一个端口监听多个地址时才会生效。
#     ssl：在当前端口建立的连接必须基于ssl协议。
# 配置范围：server

# 功能：主机名称
# 语法：server_name name[...];
# 默认：server_name "";
# 配置范围：server

# server name使用散列表存储的时候：
# 功能：1、每个散列桶占用内存大小
# 语法：server_names_hash_bucket_size siez;
# 默认：server_names_hash_bucket_size 32 | 64 | 128;
# 功能：2、散列表最大bucket数量
# 语法：server_names_hash_max_size size;
# 默认：server_names_hash_max_size 512;
#      server_name_in_redirect on;
# 配置范围：server、http、location

# 功能：处理重定向主机名
# 语法：server_name_in_redirect on | off;
# 默认：server_name_in_redirect in;
# 配置范围：server、http、location

# 功能：location尝试根据用户请求中的uri来匹配/uri表达式，若匹配成功，则执行{}里面的配置来处理用户请求
# 语法：location [= | ~ | ~* | ^~ | @] /uri/ {}
# 说明：location的一般配置项：
#      1、
#         功能：以root方式设置资源路径
#         语法：root path;
#      2、
#         功能：以alias方式设置资源路径
#         语法：alias path;
#      3、
#         功能：访问首页
#         语法：index file...;
#      4、
#         功能：根据HTTP返回码重定向页面
#         语法：error_page code [code ...] [= | =andwer-code] uri | @named_location;
#      5、
#         功能：是否允许递归使用error_page
#         语法：recursive_error_pages [on|off];
#      6、
#         功能：try_files
#         语法：try_files_path1 [path2] uri;
#      [= | ~ | ~* | ^~ | @]介绍：
#      1、完全匹配 =
#      2、大小写敏感 ~
#      3、忽略大小写 ~*
#      4、前半部分匹配 ^~
#      5、变量定义 @
# 配置范围：server

## 文件路径的定义

# 功能：root方式设置资源路径
# 语法：root path;
# 默认：root html;
# 配置范围：server、http、location、if

# 功能：以alias方式设置资源路径
# 语法：alias path;
# 配置范围：location

# 功能：访问主页
# 语法：index file ...;
# 默认：index index.html;
# 配置范围：http、server、location

# 功能：根据HTTP返回码重定向页面
# 语法：error_page code [code ...] [= | =answer-code] uri | @named_location;
# 配置范围：server、http、location、if

# 功能：是否允许递归使用error_page
# 语法：recursive_error_pages [on|off];
# 配置范围：http、server、location

# 功能：try_files
# 语法：try_files_path1 [path2] uri;
# 配置范围：server、location

## 内存及磁盘资源分配

# 功能：HTTP包体只存储在磁盘文件中
# 语法：client_body_in_file_only on | clean | off;
# 默认：client_body_in_file_only off;
# 配置范围：http、server、location

# 功能：HTTP包体尽量写入到一个内存buffer中
# 语法：client_body_single_buffer on | off;
# 默认：client_body_single_buffer off;
# 配置范围：http、server、location

# 功能：存储HTTP头部的内存buffer大小
# 语法：client_header_buffer_size size;
# 默认：client_header_buffer_size 1k;
# 配置范围：http、server

# 功能：存储超大HTTP头部的内存buffer大小
# 语法：large_client_header_buffer_size number size;
# 默认：large_client_header_buffer_size 4 8k;
# 配置范围：http、server

# 功能：存储HTTP包体的内存buffer大小
# 语法：client_body_buffer_size size;
# 默认：client_body_buffer_size 8k/16k;
# 配置范围：http、server、location

# 功能：HTTP包体的临时存放目录
# 语法：client_body_temp_path dir-path [level1 [level2[level3]]];
# 默认：client_body_temp_path client_body_temp;
# 配置范围：http、server、location

# 功能：存储TCP成功建立连接的内存大小
# 语法：connection_pool_size siez;
# 默认：connection_pool_size 256;
# 配置范围：http、server

# 功能：存储TCP请求连接的内存池大小
# 语法：request_pool_size size;
# 默认：request_pool_size 4k;
# 配置范围：http、server

## 网络连接设置

# 功能：读取HTTP头部的超时时间
# 语法：client_header_timeout time;
# 默认：client_header_timeout 60;
# 配置范围：http、server、location

# 功能：读取HTTP包体的超时时间
# 语法：client_body_timeout time;
# 默认：client_body_timeout 60;
# 配置范围：http、server、location

# 功能：发送响应的超时时间
# 语法：send_timeout time;
# 默认：send_timeout 60;
# 配置范围：http、server、location

# 功能：TCP连接的超时重置
# 语法：reset_timeout_connection on | off;
# 默认：reset_timeout_connection  off;
# 配置范围：http、server、location

# 功能：控制关闭TCP连接的方式
# 语法：lingering_close off | on | always;
# 默认：lingering_close in;
# 说明：
#      off：不处理
#      on：一般会处理
#      always：表示关闭连接之前无条件处理连接上所有用户数据
# 配置范围：http、server、location

# 功能：lingering time
# 语法：lingering_time time;
# 默认：lingering_time 30s;
# 配置范围：http、server、location

# 功能：lingering timeout
# 语法：lingering_timeout time;
# 默认：lingering_timeout 5s;
# 配置范围：http、server、location

# 功能：对某些浏览器禁止keepalive功能
# 语法：keepalive_disable [mise6 |  safari | none] ...
# 默认：keepalive_disable mise6 safari;
# 配置范围：http、server、location

# 功能：keepalive超时时间
# 语法：keepalive_timeout time;
# 默认：keepalive_timeout 75;
# 配置范围：http、server、location

# 功能：keepalive长连接上允许最大请求数
# 语法：keepalive_requests n;
# 默认：keepalive_requests 100;
# 配置范围：http、server、location

# 功能：tcp_nodelay
# 语法：tcp_nodelay on | off;
# 默认：tcp_nodelay in;
# 配置范围：http、server、location

# 功能：tcp_nopush
# 语法：tcp_nopush on | off;
# 默认：tcp_nopush off;
# 配置范围：http、server、location

## MIME 类型设置

# 功能：MIME type与文件扩展的映射
# 语法：type{...}
# 说明：多个扩展名可映射到同一个MIME type
# 配置范围：http、server、location

# 功能：MIME type
# 语法：default_type MIME-type;
# 默认：default_type text/plain;
# 配置范围：http、server、location

# 功能：type_hash_bucket_size
# 语法：type_hash_bucket_size size;
# 默认：type_hash_bucket_size 32 | 64 | 128;
# 配置范围：http、server、location

# 功能：type_hash_max_size
# 语法：type_hash_max_size size;
# 默认：type_hash_max_size 1024;
# 配置范围：http、server、location

## 限制客户端请求

# 功能：按HTTP方法名限制用户请求
# 语法：limit_except method...{...}
# 说明：method的取值：
#      GET HEAD POST PUT DELETE MKCOL COPY MOVE OPTIONS PROPFIND PROPPATCH LOCK UNLOCK PATCH
# 配置范围：location

# 功能：HTTP请求包体的最大值
# 语法：client_max_body_size size;
# 默认：client_max_body_size 1m;
# 配置范围：http、server、location

# 功能：对请求限制速度
# 语法：limit_rate speed;
# 默认：limit_rate 0;
# 说明：0表示不限速
# 配置范围：http、server、location、if

# 功能：limit_rate_after规定时间后限速
# 语法：limit_rate_after time;
# 默认：limit_rate_after 1m;
# 配置范围：http、server、location、if

## 文件操作的优化

# 功能：sendfile系统调用
# 语法：sendfile on | off;
# 默认：sendfile off;
# 配置范围：http、server、location

# 功能：AIO系统调用
# 语法：aio on | off
# 默认：aio off;
# 配置范围：http、server、location

# 功能：directio
# 语法：directio size | off;
# 默认：directio off;
# 配置范围：http、server、location

# 功能：directio_alignment
# 语法：directio_alignment size;
# 默认：directio_alignment 512;
# 配置范围：http、server、location

# 功能：打开文件缓存
# 语法：open_file_cache max=N [inactive=time] | off;
# 默认：open_file_cache  off;
# 配置范围：http、server、location

# 功能：是否混村打开文件的错误信息
# 语法：open_file_cache_errors on | off;
# 默认：open_file_cache_errors off;
# 配置范围：http、server、location

# 功能：不被淘汰的最小访问次数
# 语法：open_file_cache_min_user number;
# 默认：open_file_cache_min_user 1;
# 配置范围：http、server、location

# 功能：检验缓存中元素有效性的频率
# 语法：open_file_cache_valid time;
# 默认：open_file_cache_valid 60s;
# 配置范围：http、server、location

## 客户请求的特殊处理

# 功能：忽略不合法的HTTP头部
# 语法：ignore_invalid_headers on | off;
# 默认：ignore_invalid_headers on;
# 配置范围：http、server

# 功能：HTTP头部是否允许下划线
# 语法：underscores_in_headers on | off;
# 默认：underscores_in_headers off;
# 配置范围：http、server

# 功能：If_Modified_Since头部的处理策略
# 语法：if_modified_since [off | exact | before]
# 默认：if_modified_since exact;
# 配置范围：http、server、location

# 功能：文件未找到时是否记录到error日志
# 语法：log_not_found on | off;
# 默认：log_not_found on;
# 配置范围：http、server、location

# 功能：是否合并相邻的“/”
# 语法：merge_slashes on | off;
# 默认：merge_slashes on;
# 配置范围：http、server、location

# 功能：DNS解析地址
# 语法：resolver address...;
# 配置范围：http、server、location

# 功能：DNS解析的超时时间
# 语法：resolver_timeout time;
# 默认：resolver_timeout 30s;
# 配置范围：http、server、location

# 功能：返回错误页面是否在server中注明Nginx版本
# 语法：server_tokens on | off;
# 默认：server_tokens on;
# 配置范围：http、server、location

```

