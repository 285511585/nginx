# Nginx handler 模块的基本结构

除了上一节介绍的模块的基本结构之外，handler模块必须提供一个真正的处理函数，这个函数负责对来自客户端请求的真正处理。这个函数的处理，既可以选择自己直接生成内容，也可以选择拒绝处理，由后续的handler去进行处理，或者是选择丢给后续的filter进行处理。

原型声明如下：

```cpp
typedef ngx_int_t (*ngx_http_handler_pt)(ngx_http_request_t *r);
```

该函数处理成功返回`NGX_OK`，处理发生错误返回`NGX_ERROR`，拒绝处理（留给后续的handler进行处理）返回`NGX_DECLINE`。返回`NGX_OK`也就代表给客户端的响应已经生成好了，否则返回`NGX_ERROR`即表示发生错误了。

