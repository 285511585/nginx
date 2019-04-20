# Nginx 过滤模块的分析

## 相关结构体

在过滤模块中，所有输出的内容都是通过一条单向链表`ngx_chain_t`组成。这种单向链表的设计，正好应和了Nginx流式的输出模式。每次Nginx都是读到一部分的内容，就放到链表，然后输出出去。这种设计的好处是见到那、非阻塞，但是相应的问题就是跨链表的内容操作非常麻烦，如果需要跨链表，很多时候都只能缓存链表的内容。

`ngx_chain_t`的结构如下：

```cpp
    typedef struct ngx_chain_s ngx_chain_t;

    struct ngx_chain_s {
        ngx_buf_t    *buf;
        ngx_chain_t  *next;
    };
```

单链表负载的就是`ngx_buf_t`，这个结构体的使用非常广泛，结构如下：

```cpp
    // 这个结构体定义了一个buffer的存储位置以及一些配置，比如是否可写，输出后是否可被回收等
    struct ngx_buf_s {
        u_char          *pos;       /* 当前buffer真实内容的起始位置 */
        u_char          *last;      /* 当前buffer真实内容的结束位置 */
        off_t            file_pos;  /* 在文件中真实内容的起始位置   */
        off_t            file_last; /* 在文件中真实内容的结束位置   */

        u_char          *start;    /* buffer内存的开始分配的位置 */
        u_char          *end;      /* buffer内存的结束分配的位置 */
        ngx_buf_tag_t    tag;      /* buffer属于哪个模块的标志 */
        ngx_file_t      *file;     /* buffer所引用的文件 */

        /* 用来引用替换过后的buffer，以便当所有buffer输出以后，
         * 这个影子buffer可以被释放。
         */
        ngx_buf_t       *shadow; 

        /* the buf's content could be changed */
        unsigned         temporary:1;

        /*
         * the buf's content is in a memory cache or in a read only memory
         * and must not be changed
         */
        unsigned         memory:1;

        /* the buf's content is mmap()ed and must not be changed */
        unsigned         mmap:1;

        unsigned         recycled:1; /* 内存可以被输出并回收 */
        unsigned         in_file:1;  /* buffer的内容在文件中 */
        /* 马上全部输出buffer的内容, gzip模块里面用得比较多 */
        unsigned         flush:1;
        /* 基本上是一段输出链的最后一个buffer带的标志，标示可以输出，
         * 有些零长度的buffer也可以置该标志
         */
        unsigned         sync:1;
        /* 所有请求里面最后一块buffer，包含子请求 */
        unsigned         last_buf:1;
        /* 当前请求输出链的最后一块buffer         */
        unsigned         last_in_chain:1;
        /* shadow链里面的最后buffer，可以释放buffer了 */
        unsigned         last_shadow:1;
        /* 是否是暂存文件 */
        unsigned         temp_file:1;

        /* 统计用，表示使用次数 */
        /* STUB */ int   num;
    };
```

一个`buf`结构体可以表示一块内存，内存的起始和结束地址分别用`start`和`end`表示，`pos`和`last`表示实际的内容。这几个指针的关系如下图：

![start&#x3001;pos&#x3001;last&#x3001;end&#x6307;&#x9488;&#x7684;&#x5173;&#x7CFB;](../.gitbook/assets/image%20%2829%29.png)

即`start`是分配的内存的开始位置，`pos`是已处理的内存的结束位置，`last`指向占用的内存的结束位置，`end`是分配的内存的结束位置

## 响应头过滤函数

响应头过滤函数主要的用处就是**处理HTTP响应的头**，可以根据实际情况对于响应头进行修改或者增删。响应头过滤函数先于响应体过滤函数，而且**只调用一次（但有多个过滤模块需要执行）**，所以一般可作过滤模块的初始化工作。

响应头过滤函数的**入口**只有一个：

```cpp
    ngx_int_t
    ngx_http_send_header(ngx_http_request_t *r)
    {
        ...

        return ngx_http_top_header_filter(r);
    }
```

该函数向客户端发送回复的时候调用，然后按如下的顺序执行。

| filter module | description |
| :--- | :--- |
| ngx\_http\_not\_modified\_filter\_module | 默认打开，如果请求的`if-modified-since`等于回复的`last-modified`的值，说明**回复没有变化**，清空所有回复的内容，**返回304** |
| ngx\_http\_range\_body\_filter\_module | 默认打开，只是响应体过滤函数，**支持range功能**，如果请求包含range请求，那就只发送range请求的一段内容 |
| ngx\_http\_copy\_filter\_module | 始终打开，只是响应体过滤函数，主要工作是**把文件中内容读到内存**中，以便进行处理 |
| ngx\_http\_headers\_filter\_module | 始终打开，可以设置`expire`和`Cache-control`头，可以**添加任意名称的头** |
| ngx\_http\_userid\_filter\_module | 默认关闭，可以**添加统计用的识别用户的cookie** |
| ngx\_http\_charset\_filter\_module | 默认关闭，可以**添加charset**，也可以将内容从一种字符集转换到另外一种字符集，不支持多字节字符集 |
| ngx\_http\_ssi\_filter\_module | 默认关闭，**过滤SSI请求，可以发起子请求**，去获取include进来的文件 |
| ngx\_http\_postpone\_filter\_module | 始终打开，用来将子请求和主请求的**输出链合并** |
| ngx\_http\_gzip\_filter\_module | 默认关闭，**支持流式的压缩内容** |
| ngx\_http\_range\_header\_filter\_module | 默认打开，只是响应头过滤函数，**用来解析range头，并产生range响应的头** |
| ngx\_http\_chunked\_filter\_module | 默认打开，对于HTTP/1.1和缺少`content-length`的回复自动打开 |
| ngx\_http\_header\_filter\_module | 始终打开，用来**将所有header组成一个完整的HTTP头** |
| ngx\_http\_writie\_filter\_module | 始终打开，将**输出链拷贝到`r->out`**中，然后**输出内容** |

该函数的返回值一般是`NGX_OK`，`NGX_ERROR`，`NGX_AGAIN`，分别表示处理成功，失败和未完成。

可以把HTTP响应头的存储方式想象成一个hash表，在Nginx内部可以很方便地查找和修改各个响应头部，`ngx_http_header_filter_module`过滤模块把所有的HTTP头组合成一个完整的buf，最终`ngx_http_write_filter_module`过滤模块把buf输出。

## 响应体过滤函数

响应体过滤函数是**过滤响应主体**的函数。`ngx_http_top_body_filter`这个函数每个请求可能会被**执行多次**，它的入口函数是`ngx_http_output_filter`，比如：

```cpp
    ngx_int_t
    ngx_http_output_filter(ngx_http_request_t *r, ngx_chain_t *in)
    {
        ngx_int_t          rc;
        ngx_connection_t  *c;

        c = r->connection;
        // 过滤响应主体
        rc = ngx_http_top_body_filter(r, in);

        if (rc == NGX_ERROR) {
            /* NGX_ERROR may be returned by any filter */
            c->error = 1;
        }

        return rc;
    }
```

`ngx_http_output_filter`可以被一般的静态处理模块调用，也有可能是在upstream模块里面被调用，对于整个请求的处理阶段来说，它们处于的用处都是一样的，就是**把响应内容过滤，然后发送给客户端**。

具体模块的响应体过滤函数的格式类似这样：

```cpp
    static int 
    ngx_http_example_body_filter(ngx_http_request_t *r, ngx_chain_t *in)
    {
        ...
        // 调用下一个响应体过滤函数
        return ngx_http_next_body_filter(r, in);
    }
```

该函数的返回值一般是`NGX_OK`，`NGX_ERROR`，`NGX_AGAIN`，即处理成功，失败和未完成（同响应头过过滤函数）。

### 主要功能介绍

* 响应的主体内容存于单链表in中，链表一般**不会太长**，可为NULL。in中存有buffer结构体，对于静态文件，这个buf的大小**默认是32K**，对于反向代理的应用，这个buf可能是**4k或者8k**。为了保持内存的低消耗，Nginx一般不会分配过大的内存，处理的原则是**收到一定的数据，就发送出去**。 （**处理数据**）
* 在响应体过滤模块中，要**注意buf结构的标志位**：（**设置buf的标志位**）
  * 如果包含last标志，说明是最后一块buf，可以直接输出并结束请求了
  * 如果有flush标志，说明这块buf需要马上输出，不能缓存 
  * 如果整块buf处理完成后，没有数据了，可以把sync标志置上，表示只是同步的用处
* 当所有过滤模块都处理完毕时，在最后的`write_filter`模块中，Nginx会将in输出链拷贝到`r->out`输出链的末尾，然后调用`sendfile`或者`writev`接口输出。由于Nginx是**非阻塞的socket接口**，写操作并不一定会成功，可能会有部分数据还残存在`r->out`。在下次的调用中，Nginx**会继续尝试发送，直至成功**。

## 发出子请求

Nginx过滤模块一大特色就是可以发出子请求，也就是**在过滤响应内容的时候，可以发送新的请求**，Nginx会根据调用的先后顺序，将**多个回复的内容拼接成正常的响应主体**。（可以参考addition模块）

### 主请求和子请求的处理

当Nginx发出子请求时，就会调用`ngx_http_subrequest`函数，将子请求插入主请求的`r->postponed`链表中。子请求会在主请求执行完毕后依次调用。子请求**同样会有一个请求所有的生存期和处理过程，也会进入过滤模块流程**。

而在postpone\_filter模块中，它会拼接主请求和子请求的响应内容。`r->postponed`**按次序保存主请求和子请求**，它是一个链表，**如果前面一个请求未完成，那后一个请求内容就不会输出**。当所有的请求都完成是，所有的响应内容也就输出完毕了。

## 一些优化措施

Nginx采用类似freelist**重复利用的原则**，在chain和buf（使用频繁）使用完毕后，放置到一个固定的空闲链表里，以待下次使用。

对于buffer结构体，还有一种**busy链表**，表示该链表中的buf都处于输出状态，如果buf输出完毕，这些buf就可以释放并重复利用了。

| 功能 | 函数名 |
| :--- | :--- |
| chain 分配 | ngx\_alloc\_chain\_link |
| chain 释放 | ngx\_free\_chain |
| buf 分配 | ngx\_chain\_get\_free\_buf |
| buf 释放 | ngx\_chain\_update\_chains |

## 过滤内容的缓存

由于Nginx设计流式的输出结构，当我们需要对响应内容作全文过滤的时候，必须缓存部分的buf内容。该类过滤模块往往比较复杂，比如sub，ssi，gzip等模块。这类模块的设计非常灵活，设计原则：

1. 因为经过缓存的过滤模块，输入输出链已经完全不一样了，需要拷贝，得通过`ngx_chain_add_cop`函数完成
2. 一般都有自己的**free和busy缓存链表池**，可以提高buf分配效率
3. 如果需要分配**大块内容**，一般**分配固定大小的内存卡**，并设置recycled标志，表示可以重复利用
4. 原有的输入buf被替换缓存时，必须将其`buf->pos`设为`buf->last`，表明原有的buf已经被输出完毕，或者在新建立的buf，将`buf->shadow`指向旧的buf，以便输出完毕时及时释放旧的buf。

