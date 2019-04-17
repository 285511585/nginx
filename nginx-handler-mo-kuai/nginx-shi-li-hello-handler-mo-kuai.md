# Nginx 示例：hello handler模块

## 示例

```cpp
#include <ngx_config.h>
#include <ngx_core.h>
#include <ngx_http.h>

// 模块的配置结构
typedef struct
{
    ngx_str_t hello_string;
    ngx_int_t hello_counter;
}ngx_http_hello_loc_conf_t;

/* 函数声明 */
// 在创建和读取模块的配置信息之后调用的函数的声明
static ngx_int_t ngx_http_hello_init(ngx_conf_t *cf);
// 用于创建本模块位于location block的配置信息存储结构的声明
static void *ngx_http_hello_create_loc_conf(ngx_conf_t *cf);
// 配置指令hello_string的解析配置的函数的声明
static char *ngx_http_hello_string(ngx_conf_t *cf, ngx_command_t *cmd,
    void *conf);
// 配置指令hello_counter的解析配置的函数的声明
static char *ngx_http_hello_counter(ngx_conf_t *cf, ngx_command_t *cmd,
    void *conf);

// 模块的配置指令
static ngx_command_t ngx_http_hello_commands[] = {
   { 
        ngx_string("hello_string"), // 配置指令的名称
        NGX_HTTP_LOC_CONF|NGX_CONF_NOARGS|NGX_CONF_TAKE1, // 设置配置出现的位置和接受参数的个数
        ngx_http_hello_string, // 解析配置的函数
        NGX_HTTP_LOC_CONF_OFFSET, // 指定当前配置项的存储位置
        offsetof(ngx_http_hello_loc_conf_t, hello_string), // 指定配置项值的精确存储位置，也即是说，这个配置指令传入的参数将会存到ngx_http_hello_loc_conf结构的变量hello_string中
        NULL // 读取配置过程宏需要的数据
    },

    { 
        ngx_string("hello_counter"),
        NGX_HTTP_LOC_CONF|NGX_CONF_FLAG,
        ngx_http_hello_counter,
        NGX_HTTP_LOC_CONF_OFFSET,
        offsetof(ngx_http_hello_loc_conf_t, hello_counter),
        NULL 
    },               
    ngx_null_command
};

/* 
static u_char ngx_hello_default_string[] = "Default String: Hello, world!";
*/
// 模块被访问的次数
static int ngx_hello_visited_times = 0; 

// 模块的上下文结构
static ngx_http_module_t ngx_http_hello_module_ctx = {
    NULL,                          /* preconfiguration */
    ngx_http_hello_init,           /* postconfiguration */

    NULL,                          /* create main configuration */
    NULL,                          /* init main configuration */

    NULL,                          /* create server configuration */
    NULL,                          /* merge server configuration */

    ngx_http_hello_create_loc_conf, /* create location configuration */
    NULL                            /* merge location configuration */
};

// 模块的定义
ngx_module_t ngx_http_hello_module = {
    NGX_MODULE_V1,
    &ngx_http_hello_module_ctx,    /* module context */
    ngx_http_hello_commands,       /* module directives */
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

// handler模块真正的处理函数，这个函数是真正用于处理来自客户端的请求的，处理之后可以产生响应结果，当然也可以拒绝处理，留给后续的filter处理
static ngx_int_t
ngx_http_hello_handler(ngx_http_request_t *r)
{
    ngx_int_t    rc;
    /*
     * 是ngx_chain_t链表的每个节点的实际数据，该数据结构实际上是一种抽象的数据结构，
     * 它代表某种具体的数据，这个数据可能是指向内存中的某个缓冲区，也可能指向一个文件的某一部分，
     * 也可能是一些纯元数据（元素局的作用在于指示这个链表的读取者对读取的数据进行不同的处理）。
     */ 
    ngx_buf_t   *b;
    ngx_chain_t  out;
    ngx_http_hello_loc_conf_t* my_conf;
    u_char ngx_hello_string[1024] = {0};
    ngx_uint_t content_length = 0;

    ngx_log_error(NGX_LOG_EMERG, r->connection->log, 0, "ngx_http_hello_handler is called!");
    
    // 获取模块的location配置，因为模块的配置指令中设置的两个配置都只出现再location中
    my_conf = ngx_http_get_module_loc_conf(r, ngx_http_hello_module);
    if (my_conf->hello_string.len == 0 )
    {
        ngx_log_error(NGX_LOG_EMERG, r->connection->log, 0, "hello_string is empty!");
        return NGX_DECLINED;
    }

    if (my_conf->hello_counter == NGX_CONF_UNSET
        || my_conf->hello_counter == 0)
    {
        // 将my_conf->hello_string.data输出到ngx_hello_string
        ngx_sprintf(ngx_hello_string, "%s", my_conf->hello_string.data);
    }
    else
    {
        // 追加请求的次数（模块被访问的次数）
        ngx_sprintf(ngx_hello_string, "%s Visited Times:%d", my_conf->hello_string.data, 
            ++ngx_hello_visited_times);
    }
    ngx_log_error(NGX_LOG_EMERG, r->connection->log, 0, "hello_string:%s", ngx_hello_string);
    // 设置数据长度
    content_length = ngx_strlen(ngx_hello_string);

    /* we response to 'GET' and 'HEAD' requests only */
    // 只处理GET和HEAD，其余直接返回
    if (!(r->method & (NGX_HTTP_GET|NGX_HTTP_HEAD))) {
        return NGX_HTTP_NOT_ALLOWED;
    }

    /* discard request body, since we don't need it here */
    // 丢掉请求体
    rc = ngx_http_discard_request_body(r);

    if (rc != NGX_OK) {
        return rc;
    }

    /* set the 'Content-type' header */
    /*
     *r->headers_out.content_type.len = sizeof("text/html") - 1;
     *r->headers_out.content_type.data = (u_char *)"text/html";
             */
    // 设置 Content-type
    ngx_str_set(&r->headers_out.content_type, "text/html");

    /* send the header only, if the request type is http 'HEAD' */
    // 如果请求类型为HEAD，则只发送header
    if (r->method == NGX_HTTP_HEAD) {
        r->headers_out.status = NGX_HTTP_OK;
        r->headers_out.content_length_n = content_length;

        return ngx_http_send_header(r);
    }

    /* allocate a buffer for your response body */
    // 为响应体分配一个buffer
    b = ngx_pcalloc(r->pool, sizeof(ngx_buf_t));
    if (b == NULL) {
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }

    /* attach this buffer to the buffer chain */
    // 将buffer加入到buffer链上
    out.buf = b;
    out.next = NULL;

    /* adjust the pointers of the buffer */
    // 调整buffer指针
    b->pos = ngx_hello_string;
    b->last = ngx_hello_string + content_length;
    b->memory = 1;    /* this buffer is in memory */
    b->last_buf = 1;  /* this is the last buffer in the buffer chain */

    /* set the status line */
    // 设置状态行
    r->headers_out.status = NGX_HTTP_OK;
    r->headers_out.content_length_n = content_length;

    /* send the headers of your response */
    // 过滤响应头
    rc = ngx_http_send_header(r);

    if (rc == NGX_ERROR || rc > NGX_OK || r->header_only) {
        return rc;
    }

    /* send the buffer chain of your response */
    // 过滤响应体，发送响应的缓冲链
    return ngx_http_output_filter(r, &out);
}

// 用于创建本模块位于location block的配置信息存储结构的实现
static void *ngx_http_hello_create_loc_conf(ngx_conf_t *cf)
{
    ngx_http_hello_loc_conf_t* local_conf = NULL;
    local_conf = ngx_pcalloc(cf->pool, sizeof(ngx_http_hello_loc_conf_t));
    if (local_conf == NULL)
    {
        return NULL;
    }

    ngx_str_null(&local_conf->hello_string);
    local_conf->hello_counter = NGX_CONF_UNSET;

    return local_conf;
} 

/*
// merge_loc_conf 函数设置到模块的上下文定义中。
// 这样有一个缺点：就是如果一个指令没有出现在配置文件中的时候，配置信息中的值，
// 将永远会保持在 create_loc_conf 中的初始化的值。那如果，在类似 create_loc_conf 这样的函数中，
// 对创建出来的配置信息的值，没有设置为合理的值的话，后面用户又没有配置，就会出现问题。
static char *ngx_http_hello_merge_loc_conf(ngx_conf_t *cf, void *parent, void *child)
{
    ngx_http_hello_loc_conf_t* prev = parent;
    ngx_http_hello_loc_conf_t* conf = child;

    ngx_conf_merge_str_value(conf->hello_string, prev->hello_string, ngx_hello_default_string);
    ngx_conf_merge_value(conf->hello_counter, prev->hello_counter, 0);

    return NGX_CONF_OK;
}*/

// 配置指令hello_string的解析配置的函数的实现
static char *
ngx_http_hello_string(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{

    ngx_http_hello_loc_conf_t* local_conf;

    local_conf = conf;
    
    // 读取字符串类型的参数
    char* rv = ngx_conf_set_str_slot(cf, cmd, conf);

    ngx_conf_log_error(NGX_LOG_EMERG, cf, 0, "hello_string:%s", local_conf->hello_string.data);

    return rv;
}

// 配置指令hello_counter的解析配置的函数的实现
static char *ngx_http_hello_counter(ngx_conf_t *cf, ngx_command_t *cmd,
    void *conf)
{
    ngx_http_hello_loc_conf_t* local_conf;

    local_conf = conf;

    char* rv = NULL;
    
    // 读取NGX_CONF_FLAG类型的参数
    rv = ngx_conf_set_flag_slot(cf, cmd, conf);
    
    ngx_conf_log_error(NGX_LOG_EMERG, cf, 0, "hello_counter:%d", local_conf->hello_counter);
    return rv;    
}

// 在创建和读取模块的配置信息之后调用的函数的实现
static ngx_int_t
ngx_http_hello_init(ngx_conf_t *cf)
{
    ngx_http_handler_pt        *h;
    ngx_http_core_main_conf_t  *cmcf;

    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);
    // 判断是否时NGX_HTTP_CONTENT_PHASE
    h = ngx_array_push(&cmcf->phases[NGX_HTTP_CONTENT_PHASE].handlers);
    if (h == NULL) {
        return NGX_ERROR;
    }
    // 是则挂载handler模块真正的处理函数
    *h = ngx_http_hello_handler;

    return NGX_OK;
}
```

## 流程图

![&#x6A21;&#x5757;&#x7684;&#x7F16;&#x5199;&#x6D41;&#x7A0B;](../.gitbook/assets/image%20%2833%29.png)

![](../.gitbook/assets/image%20%2836%29.png)

