# Nginx 负载均衡模块

负载均衡模块用于从`upstream`指令定义的**后端主机列表中选取一台主机**。Nginx**先**使用**负载均衡模块找到一台主机**，**再**使用**upstream模块实现与这台主机的交互**。

## 配置

负载均衡模块与之前提到的模块差别较大，先从它的配置入手比较容易理解该模块。

在配置文件中，我们如果需要使用ip hash的负载均衡算法，需要写一个类似下面的配置：

```bash
    upstream test {
        ip_hash;

        server 192.168.0.1;
        server 192.168.0.2;
    }
```

从配置中可以看出负载均衡模块的使用场景：

* 核心指令`ip_hash`只能在upstream{}中使用。这条指令用于通知Nginx使用ip hash负载均衡算法。如果没加这条指令，Nginx会使用默认的round robin负载均衡算法。
* upstream{}中的指令可能出现在server指令前、后、中间，`ip_hash`指令会影响到配置的解析的。

## 指令

配置决定指令系统，`ip_hash`的指令定义如下：

```cpp
static ngx_command_t  ngx_http_upstream_ip_hash_commands[] = {

    { ngx_string("ip_hash"),
      NGX_HTTP_UPS_CONF|NGX_CONF_NOARGS,
      ngx_http_upstream_ip_hash,
      0,
      0,
      NULL },

    ngx_null_command
};
```

和一般的模块的指令定义没有太大差别，`NGX_HTTP_UPS_CONF`这个属性代表该指令的使用范围是upstream{}

## 钩子

{% hint style="info" %}
钩子编程：

也称作挂钩，是计算机程序设计术语，指**通过拦截**软件模块间的函数调用、消息传递、事件传递**来修改或扩展**操作系统、应用程序或其它软件组件的**行为的各种技术**。
{% endhint %}

钩子是模块的切入点，负载均衡模块的钩子代码都是由规律的，这里通过`ip_hash`模块来分析这个规律。

```cpp
static char *
ngx_http_upstream_ip_hash(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ngx_http_upstream_srv_conf_t  *uscf;

    uscf = ngx_http_conf_get_module_srv_conf(cf, ngx_http_upstream_module);
    
    // 设置init_upstream回调
    uscf->peer.init_upstream = ngx_http_upstream_init_ip_hash;
    
    // 设置uscf->flags
    uscf->flags = NGX_HTTP_UPSTREAM_CREATE
                |NGX_HTTP_UPSTREAM_MAX_FAILS
                |NGX_HTTP_UPSTREAM_FAIL_TIMEOUT
                |NGX_HTTP_UPSTREAM_DOWN;

    return NGX_CONF_OK;
}
```

其中需要关注的是以下两点：

### 设置uscf-&gt;flags

这里可取值如下：

* NGX\_HTTP\_UPSTREAM\_CREATE：创建标志，如果含有创建标志的话，Nginx会检查重复创建，以及必要参数是否填写
* NGX\_HTTP\_UPSTREAM\_MAX\_FAILS：可以在server中使用max\_fails属性
* NGX\_HTTP\_UPSTREAM\_FAIL\_TIMEOUT：可以在server中使用fail\_timeout属性
* NGX\_HTTP\_UPSTREAM\_DOWN：可以在server中使用down属性
* NGX\_HTTP\_UPSTREAM\_WEIGHT：可以在server中使用weight属性
* NGX\_HTTP\_UPSTREAM\_BACKUP：可以在server中使用backup属性

可以看到`uscf->flags`的设置中，是可以对`server`指令支持的属性进行设置的，因为不同的负载均衡模块对各种属性的支持情况都是不一样的，那么就需要在解析配置文件的时候检测出**是否使用了不支持的负载均衡属性并给出错误提示**，这对提升系统维护性是很有意义的。

### 设置init\_upstream回调

Nginx**初始化upstream时**，会在`ngx_http_upstream_init_main_conf`函数中调用设置的回调函数**初始化负载均衡模块**。

#### upstream负载均衡模块的配置的内存布局

![&#x8D1F;&#x8F7D;&#x5747;&#x8861;&#x6A21;&#x5757;&#x7684;&#x914D;&#x7F6E;&#x7684;&#x5185;&#x5B58;&#x5E03;&#x5C40;](../.gitbook/assets/image%20%2840%29.png)

可以看到，`main_conf`中有一个指针数组`upstreams`，每个元素对应就是配置文件中每个`upstreams{}`的信息

#### 初始化配置

`init_upstream`回调函数执行时需要**初始化负载均衡模块的配置**，还要设置一个新钩子，这个钩子函数会在Nginx处理每个请求时作为初始化函数调用。

ip hash模块初始化配置的代码：

```cpp
// 先调用另一个负载均衡模块round robin的初始化桉树
ngx_http_upstream_init_round_robin(cf, us);
// 设置处理请求阶段初始化钩子
us->peer.init = ngx_http_upstream_init_ip_hash_peer;
```

实际上，几个负载均衡模块可以组成一条链表，每次都是从链首的模块开始进行处理。如果模块决定不处理，可以将处理权交给链表中的下一个模块。这里，ip hash模块指定了round robin模块作为自己的后继负载均衡模块，所以在自己的初始化配置函数中也对round robin模块进行初始化。

## 初始化请求

Nginx收到一个请求后，如果发现需要访问upstream，就会执行对应的`peer.init`函数。这是在初始化配置时设置的回调函数。这个函数最重要的作用是**构造一张表**，当前请求可以使用的upstream服务器被依次添加到这张表中。

之所以需要这张表，最重要的原因时如果upstream服务器出现异常，不能提供服务时，可以**从这张表中取得其它服务器进行重试操作**。此外，这张表也可以**用于负载均衡的计算**。

之所以构造这张表的行为放在这里而不是在前面初始化配置的阶段，是因为upstream**需要为每一个请求提供独立隔离的环境**。

为了讨论peer.init的核心，可以看ip hash模块的实现：

```cpp
// 设置数组指针，这个指针即指向那张表
r->upstream->peer.data = &iphp->rrp;
// 调用round robin模块的回调函数对该模块进行请求初始化。
ngx_http_upstream_init_round_robin_peer(r, us);
// 设置一个新的回调函数get。该函数负责从表中取出某个服务器。
r->upstream->peer.get = ngx_http_upstream_get_ip_hash_peer;
// 除了get回调函数，还有一个回调函数r->upstream->peer.free，这个函数在请求完成后调用，负责做一些善后工作
```

### peer.get和peer.free回调函数

这两个函数是负载均衡模块最底层的函数，负责实际获取一个**连接**和**回收**一个连接的预备操作。之所以说是**预备操作**，是因为在这两个函数中，并**不实际进行**建立连接或者释放连接的动作，而只是执行**获取连接的地址**或**维护连接状态**的操作。

通过peer.get获取连接的地址信息，但并不代表连接一定没有被建立，Nginx可以通过get函数的返回值去了解连接的情况，返回值的总结如下：

| 返回值 | 说明 | Nginx后续动作 |
| :--- | :--- | :--- |
| NGX\_DONE | 得到了连接地址信息，并且连接已经建立 | 直接使用连接，发送数据 |
| NGX\_OK | 得到了连接地址信息，但连接并未建立 | 建立连接，如连接不能立即建立，设置事件，暂停执行本请求，执行别的请求 |
| NGX\_BUSY | 所有连接均不可用 | 返回502错误给客户端 |

{% hint style="warning" %}
几个问题：

1. 什么时候连接时已经建立的？

   使用后端keepalive连接的时候，连接在使用完以后并不关闭，而是存放在一个队列中，新的请求只需要从队列中取出连接，这些连接都是已经准备好的。

2. 什么叫所有连接均不可用？

   初始化请求的过程中，建立了一张表，get函数负责每次从这张表中不重复地取出一个连接，当无法从表中取得一个新的连接时，即所有连接均不可用。

3. 对于一个请求，peer.get函数可能被调用多次么？

   会，当某次peer.get函数得到的连接地址连接不上，或者请求对应的服务器得到异常响应，Nginx会执行`ngx_http_upstream_next`，然后可能再次调用peer.get函数尝试别的连接。
{% endhint %}

upstream整体流程如下：

![upstream&#x6574;&#x4F53;&#x6D41;&#x7A0B;](../.gitbook/assets/image%20%286%29.png)

{% hint style="warning" %}
总结：

负载均衡模块的回调函数是以`init_upstream`为起点，经过`peer.init`，到达`peer.get`和`peer.free`的。

其中，各个回调函数的功能介绍如下：

* init\_upstream：初始化配置，设置peer.init等回调函数
* peer.init：负责建立每个请求使用的server列表
* peer.get：负责从server列表中选择某个server（一般是不重复选择）
* peer.free：负责server释放前的资源释放工作
{% endhint %}

