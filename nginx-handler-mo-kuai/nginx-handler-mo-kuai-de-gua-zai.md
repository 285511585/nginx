# Nginx handler 模块的挂载

## handler 模块的挂载

handler 模块真正的处理函数通过两种方式挂载到处理过程中，一种方式就是**按处理阶段挂载**，另外一种挂载方式就是**按需挂载**。

## 按处理阶段挂载

为了更精细地控制对于客户端请求的处理过程，Nginx 把这个处理过程划分成了 11 个阶段。他们从前到后，依次列举如下：

1. NGX\_HTTP\_POST\_READ\_PHASE：读取请求内容阶段 
2. NGX\_HTTP\_SERVER\_REWRITE\_PHASE：server请求地址重写阶段 
3. NGX\_HTTP\_FIND\_CONFIG\_PHASE：配置查找阶段 
4. NGX\_HTTP\_REWRITE\_PHASE：请求地址重写阶段 
5. NGX\_HTTP\_POST\_REWRITE\_PHASE：请求地址重写提交阶段 
6. NGX\_HTTP\_PREACCESS\_PHASE：访问权限检查准备阶段 
7. NGX\_HTTP\_ACCESS\_PHASE：访问权限检查阶段 
8. NGX\_HTTP\_POST\_ACCESS\_PHASE：访问权限检查提交阶段 
9. NGX\_HTTP\_TRY\_FILES\_PHASE：配置项try\_files处理阶段 
10. NGX\_HTTP\_CONTENT\_PHASE：内容产生阶段（需要把请求交给一个合适的content handler处理） 
11. NGX\_HTTP\_LOG\_PHASE：日志模块处理阶段 

一般情况下，我们自定义的模块，大多数是挂载在 `NGX_HTTP_CONTENT_PHASE` 阶段的。挂载的动作一般是在模块上下文调用的 `postconfiguration` 函数中。

{% hint style="warning" %}
有几个阶段是特例，它不调用挂载地任何的handler，也就是你就不用挂载到这几个阶段了：

* NGX\_HTTP\_FIND\_CONFIG\_PHASE
* NGX\_HTTP\_POST\_ACCESS\_PHASE
* NGX\_HTTP\_POST\_REWRITE\_PHASE
* NGX\_HTTP\_TRY\_FILES\_PHASE
{% endhint %}

示例：

```cpp
static ngx_int_t
ngx_http_hello_init(ngx_conf_t *cf)
{
    ngx_http_handler_pt        *h;
    ngx_http_core_main_conf_t  *cmcf;

    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);
    // 判断是否是NGX_HTTP_CONTENT_PHASE阶段
    h = ngx_array_push(&cmcf->phases[NGX_HTTP_CONTENT_PHASE].handlers);
    if (h == NULL) {
        return NGX_ERROR;
    }
    // 是则进行挂载
    *h = ngx_http_hello_handler;

    return NGX_OK;
}
```

 使用这种方式挂载的 handler 也被称为 **content phase handlers**。

## 按需挂载

 以这种方式挂载的 handler 也被称为 **content handler**。

当一个请求进来时，Nginx会从`NGX_HTTP_CONTENT_PHASE`阶段开始依次执行每个阶段的所有handler。执行到`NGX_HTTP_CONTENT_PHASE`阶段的时候，如果这个location有一个对应的content handler模块，那么就去执行这个content handler 模块真正的处理函数，并且不会再执行`NGX_HTTP_CONTENT_PHASE`中挂载的content phase handlers了，否则继续依次执行这些handler。

{% hint style="warning" %}
这种方式挂载必须再`NGX_HTTP_CONTENT_PHASE`阶段才能执行到，如果想在更早的阶段之心给，就不要使用这种挂载方式。
{% endhint %}

{% hint style="warning" %}
当某个模块对某个location进行了处理以后，发现符合自己的处理逻辑，而且也没有必要再调用`NGX_HTTP_CONTENT_PHASE`阶段的其它的handler进行处理了，就**动态挂载**上这个handler。
{% endhint %}

示例：

```cpp
static char *
ngx_http_circle_gif(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ngx_http_core_loc_conf_t  *clcf;

    clcf = ngx_http_conf_get_module_loc_conf(cf, ngx_http_core_module);
    // 进行挂载
    clcf->handler = ngx_http_circle_gif_handler;

    return NGX_CONF_OK;
}
```

