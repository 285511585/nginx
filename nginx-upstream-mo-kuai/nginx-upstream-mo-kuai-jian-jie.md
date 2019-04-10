# Nginx upstream 模块简介

Nginx模块一般分为三大类：**handler**、**filter**和**upstream**。利用**前两个模块**，可以使得Nginx轻松完成任何**单机工**作。而**upstream模块**，将使Nginx**跨越单机的限制，完成网络数据的接受、处理和转发**。

数据转发功能，为Nginx提供了跨越单机的横向处理能力，使Nginx摆脱只能为终端节点提供单一功能的限制，而使它具备了网络应用级别的拆分、封装和整合的战略功能。

## upstream 模块

从本质上说，upstream属于handler，只是它不产生自己的内容，而是通过请求后端服务器得到内容，所以称之为upstream（上游）。请求并取得响应内容的整个过程已经被封装到Nginx内部，所以upstream模块**只需要开发若干回调函数，完成构造请求和解析响应等具体的工作**。

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

### 处理函数

upstream模块的处理函数进行的操作都包含一个固定的流程：

1. 创建upstream数据结构
2. 设置模块的tag和schema。schema现在只会用于日志，tag会用于buf\_chain管理
3. 设置upstream的后端服务器列表数据结构

## memcached 模块

memcache是一款高性能的**分布式cache系统**，得到了非常广泛的应用。memcache定义了一套私有通信协议，使得不能通过HTTP请求来访问memcache。但协议本身简单高效，而且memcache使用广泛，所以大部分现代开发语言和平台都提供了memcache支持，方便开发者使用。

{% hint style="warning" %}
memcache类似redis，是一个分布式的缓存系统，将数据加载到缓存中，减少数据库的查找
{% endhint %}

Nginx提供了`ngx_http_memcached`模块，**提供**从memcache**读取数据**的功能，而**不提供**向memcahce**写数据**的功能。

