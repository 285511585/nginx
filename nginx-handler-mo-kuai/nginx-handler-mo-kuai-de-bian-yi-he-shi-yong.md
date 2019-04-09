# Nginx handler 模块的编译和使用

模块的功能开发完成之后，模块的使用还需要编译才能够执行。

## config文件的编写

对于开发的一个模块，需要把这个模块的c代码组织到一个目录里，同时需要编写一个config文件。这个config文件的内容就是**告诉Nginx的编译脚本该如何进行编译**。

示例：

```cpp
ngx_addon_name=ngx_http_hello_module
// 模块
HTTP_MODULES="$HTTP_MODULES ngx_http_hello_module"
// 源文件路径，如果这个模块有多个源文件，则依次在这个变量里面加入就可以了
NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ngx_http_hello_module.c"
```

## 编译

Nginx必须到Nginx的源代码目录里，通过configure指令的参数来进行编译。

示例：

```bash
./configure --prefix=/usr/local/nginx-1.3.1 --add-module=模块的代码和config文件所在目录
```

## 使用

使用一个模块需要根据这个模块定义的配置指令来使用，需要在配置文件里面加入响应的配置。

示例中的模块需要在配置文件的http的默认server里加入如下配置：

```css
location /test {
        hello_string jizhao;
        hello_counter on;
}
```

访问时[`http://127.0.0.1/test`](http://127.0.0.1/test)，可以看到返回结果：`jizhao Visited Times:1`

