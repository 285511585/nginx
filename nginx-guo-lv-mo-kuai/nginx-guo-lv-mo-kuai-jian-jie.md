# Nginx 过滤模块简介

## 执行时间和内容

过滤（filter）模块是过滤响应头和内容的模块，可以对回复的头和内容进行处理。它的处理时间在获取回复内容之后，向用户发送响应之前。它的处理过程分为两个阶段，过滤HTTP回复的头部和主体，在这两个阶段可以分别对头部和主体进行修改。

相应的函数：

```cpp
ngx_http_top_header_filter(r);
ngx_http_top_body_filter(r, in);
```

这两个函数分别对头部和主体进行过滤，所有模块的响应内容要返回给客户端，都必须调用这两个接口。

## 执行顺序

过滤模块的调用是有顺序的，它的顺序在编译的时候就决定了。控制编译的脚本位于`auto/moudles`中，当编译完Nginx以后，可以在objs目录下面看到一个`ngx_moudles.c`文件。打开这个文件，有类似的代码：

```cpp
    ngx_module_t *ngx_modules[] = {
        // ...
        // 注意，执行顺序是反向的，即最早执行的是ngx_http_not_modified_filter_module
        &ngx_http_write_filter_module,
        &ngx_http_header_filter_module,
        &ngx_http_chunked_filter_module,
        &ngx_http_range_header_filter_module,
        &ngx_http_gzip_filter_module,
        &ngx_http_postpone_filter_module,
        &ngx_http_ssi_filter_module,
        &ngx_http_charset_filter_module,
        &ngx_http_userid_filter_module,
        &ngx_http_headers_filter_module,
        &ngx_http_copy_filter_module,
        &ngx_http_range_body_filter_module,
        &ngx_http_not_modified_filter_module,
        NULL
    };
```

{% hint style="warning" %}
一般情况下，第三方过滤模块的config文件会将模块名追加到变量`HTTP_AUX_FILTER_MOUDULES`中，此时该模块只能加到`copy_filter`和`headers_filter`模块之间执行。
{% endhint %}

{% hint style="warning" %}
实现依次执行过滤模块的方式：

通过全局变量`ngx_http_top_header_filter`保存当前filter模块的处理函数

通过局部全局变量`ngx_http_next_header_filter`保存编译前上一个filter模块的处理函数

用这样的方法，使得全局变量组成一条单向链表。
{% endhint %}

![&#x54CD;&#x5E94;&#x5934;&#x548C;&#x54CD;&#x5E94;&#x4F53;&#x8FC7;&#x6EE4;&#x51FD;&#x6570;&#x7684;&#x6267;&#x884C;&#x987A;&#x5E8F;](../.gitbook/assets/image%20%288%29.png)

## 模块编译

Nginx可以很方便地加入第三方地过滤模块，在过滤模块的目录里，首先需要加入config文件，文件的内容如下：

```cpp
ngx_addon_name=ngx_http_example_filter_module 
// 把名为ngx_http_example_filter_module的模块加入
HTTP_AUX_FILTER_MODULES="$HTTP_AUX_FILTER_MODULES ngx_http_example_filter_module" 
// ngx_http_example_filter_module.c是该模块的源代码
NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ngx_http_example_filter_module.c"
```

{% hint style="warning" %}
`HTTP_AUX_FILTER_MODULES` 这个变量与一般的内容处理模块不同。
{% endhint %}

