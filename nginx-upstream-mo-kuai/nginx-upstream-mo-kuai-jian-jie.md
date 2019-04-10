# Nginx upstream 模块简介

Nginx模块一般分为三大类：**handler**、**filter**和**upstream**。利用**前两个模块**，可以使得Nginx轻松完成任何**单机工**作。而**upstream模块**，将使Nginx**跨越单机的限制，完成网络数据的接受、处理和转发**。

数据转发功能，为Nginx提供了跨越单机的横向处理能力，使Nginx摆脱只能为终端节点提供单一功能的限制，而使它具备了网络应用级别的拆分、封装和整合的战略功能。

## upstream 模块

从本质上说，upstream属于handler，**指令系统和模块生效方式与handler无异**。只是它不产生自己的内容，而是通过请求后端服务器得到内容，所以称之为upstream（上游）。请求并取得响应内容的整个过程已经被封装到Nginx内部，所以upstream模块**只需要开发若干回调函数，完成构造请求和解析响应等具体的工作**。

{% hint style="warning" %}
构造请求、解析响应是upstream模块的主要工作，这个模块其实就是完成了数据的接受、处理和转发，所以其代码逻辑大部分都是针对如何处理后端服务器的响应的。
{% endhint %}

每个回调函数都在upstream的某个固定阶段执行，各司其职，大部分回调函数一般不会真正用到。

upstream最重要的回调函数是：create\_request、process\_header和input\_filter，它们共同实现了与后端服务器的协议的解析部分。

### 回调函数

| SN | 描述 |
| :--- | :--- |
| create\_request | **生成**发送到后端服务器的**请求缓冲**（缓冲链），在初始化upstream时使用 |
| reinit\_request | 在某台后端服务器出错的时候，Nginx会尝试另一台后端服务器。Nginx选定新的服务器以后，会先调用此函数，以**重新初始化upstream模块的工作状态**，然后再次进行upstream连接 |
| process\_header | **处理后端服务器返回的信息头部**。头部是由与upstream server通信的协议规定的，比如HTTP协议的header部分，或者memcached协议的响应状态部分 |
| abort\_request | 在客户端放弃请求时被调用。不需要在函数中实现关闭关闭后端服务器连接的功能，系统会自动完成关闭连接的步骤，所以**一般此函数不会进行任何具体工作** |
| finalize\_request | 正常完成与后端服务器的请求后调用此函数，与`abort_request`相同，**一般也不会进行任何具体工作** |
| input\_filter | **处理后端服务器返回的响应正文**。Nginx默认的`input_filter`会将收到的内容封装成缓冲区链`ngx_chain`。该链由upstream的`out_bufs`指针域定位，所以开发人员可以在模块以外通过该指针得到后端服务器返回的正文数据。另外，**memcached模块实现了自己的`input_filter`** |
| input\_filter\_init | 初始化`input_filter`的上下文。Nginx默认的`input_filter_init`直接返回 |

## memcached 模块

{% hint style="warning" %}
memcached模块是upstream模块的一种
{% endhint %}

memcache是一款高性能的**分布式cache系统**，得到了非常广泛的应用。memcache定义了一套私有通信协议，使得不能通过HTTP请求来访问memcache。但协议本身简单高效，而且memcache使用广泛，所以大部分现代开发语言和平台都提供了memcache支持，方便开发者使用。

{% hint style="warning" %}
memcache类似redis，是一个分布式的缓存系统，将数据加载到缓存中，减少数据库的查找
{% endhint %}

Nginx提供了`ngx_http_memcached`模块，**提供**从memcache**读取数据**的功能，而**不提供**向memcahce**写数据**的功能。

### 处理函数

**所有的upstream模块的处理函数进行的操作都包含一个固定的流程**，观察memcached的`ngx_http_memcached_handler`的代码，可以看到如下的流程：

1. 创建upstream**数据结构**

   ```cpp
       if (ngx_http_upstream_create(r) != NGX_OK) {
           return NGX_HTTP_INTERNAL_SERVER_ERROR;
       }
   ```

2. 设置模块的tag和schema。schema现在只会用于日志，tag会用于`buf_chain`管理

   ```cpp
       u = r->upstream;

       ngx_str_set(&u->schema, "memcached://");
       u->output.tag = (ngx_buf_tag_t) &ngx_http_memcached_module;
   ```

3. 设置upstream的**后端服务器列表数据结构**

   ```cpp
       mlcf = ngx_http_get_module_loc_conf(r, ngx_http_memcached_module);
       u->conf = &mlcf->upstream;
   ```

4. 设置upstream**回调函数**

   ```cpp
       u->create_request = ngx_http_memcached_create_request;
       u->reinit_request = ngx_http_memcached_reinit_request;
       u->process_header = ngx_http_memcached_process_header;
       u->abort_request = ngx_http_memcached_abort_request;
       u->finalize_request = ngx_http_memcached_finalize_request;
       u->input_filter_init = ngx_http_memcached_filter_init;
       u->input_filter = ngx_http_memcached_filter;
   ```

5. 创建并设置upstream**环境数据结构**

   ```cpp
       ctx = ngx_palloc(r->pool, sizeof(ngx_http_memcached_ctx_t));
       if (ctx == NULL) {
           return NGX_HTTP_INTERNAL_SERVER_ERROR;
       }

       ctx->rest = NGX_HTTP_MEMCACHED_END;
       ctx->request = r;

       ngx_http_set_ctx(r, ctx, ngx_http_memcached_module);

       u->input_filter_ctx = ctx;
   ```

6. 完成upstream初始化并进行收尾工作

   ```cpp
       r->main->count++;
       ngx_http_upstream_init(r);
       return NGX_DONE;
   ```

任何upstream模块，简单如memcached，复杂如proxy、fastcgi都是如此。

不同的upstream模块在这6步中的最大差别出现在第2、3、4、5上。

* 其中第2、4两步很容易理解，不同的模块设置的标志和使用的回调函数肯定不同。
* 第5步也不难理解。
* 最晦涩的是第3步，不同的模块在取得后端服务器列表时，策略的差异非常大，有如memcached这样简单明了的，也犹如proxy那样逻辑复杂的。
* 第6步是一个常态。将count加1，然后返回NGX\_DONE。Nginx遇到这种情况，虽然会认为当前请求的处理已经结束，但是不会释放请求使用的内存资源，也不会关闭与客户端的连接。之所以需要这样，是因为**Nginx建立了upstream请求和客户端请求之间一对一的关系**，在后续使用ngx\_event\_pipe将upstream响应发送回客户端时，还要使用到这些保存着客户端信息的数据结构。这个设计有好处也有坏处，好处是简化了开发，坏处是不能满足复杂的逻辑。

### 回调函数

* ngx\_http\_memcached\_create\_request：很简单的按照设置的内容生成一个key，接着生成一个“get$key”的请求，放在r-&gt;upstream-request\_bufs里面
* ngx\_http\_memcached\_reinit\_request：无需初始化
* ngx\_http\_memcached\_abort\_request：无需额外操作
* ngx\_http\_memcached\_finalize\_request：无需额外操作
* ngx\_http\_memcached\_process\_header：：模块的业务重点函数。memcahce下而已的头部信息被定义为第一行文本，可以找到这段代码证明：
  * ```cpp
        for (p = u->buffer.pos; p < u->buffer.last; p++) {
            if ( * p == LF) {
            goto found;
        }
    ```

    Nginx处理后端服务器的响应头时只会使用一块缓存，所有数据都在这块缓存中，所以解析头部信息时不需要考虑头部信息跨越多块缓存的情况，若头部信息过大无法放入缓存中，会返回给客户端，提示缓存不够大。

    `process_header`的重要职责是将后端服务器返回的状态翻译成返回给客户端的状态，例如在`ngx_http_memcached_process_header`中，有这样的代码：

    ```cpp
        // 设置返回给客户端的响应的长度
        r->headers_out.content_length_n = ngx_atoof(len, p - len - 1);
    
        // u->headers_id将被作为返回给客户端的响应返回状态码
        u->headers_in.status_n = 200;
        // u->state用于计算upstream相关的变量
        // 计算变量“upstream_status”的值
        u->state->status = 200;

        u->headers_in.status_n = 404;
        u->state->status = 404;
    
        // 要记得在处理完头部信息之后，将指针pos后移，否则这段数据也将被赋值到返回给客户端的响应的正文中， 进而导致正文内容不正确
        u->buffer.pos= p  +1;
    
        // process_header函数完成响应头的正确处理，应该返回NGX_OK，
        // 如果返回NGX_AGAIN，表示未读取完整数据，需要从后端服务器继续读取数据，
        // 如果返回NGX_DECLINED无意义，其它任何返回值都被认为是出错状态，Nginx将结束upstream请求并返回错误信息
    ```
* ngx\_http\_memcached\_filter\_init：修正从后端服务器收到的内容长度。因为在处理header时没有加上这个长度
* ngx\_http\_memcached\_filter：memcached模块是少有的带有处理正文的回调函数的模块。**需要实现自己的filter回调函数**。处理正文的实际意义是**从后端服务器收到的正文有效内容封装成`ngx_chain_t`**，并加在`u->out_bufs`末尾。Nginx并不进行数据拷贝，而是**建立`ngx_buf_t`数据结构指向这些数据内存区，然后由`ngx_chain_t`组织这些buf**。这种实现避免了内存大量搬迁，也是Nginx高效的奥秘之一。



