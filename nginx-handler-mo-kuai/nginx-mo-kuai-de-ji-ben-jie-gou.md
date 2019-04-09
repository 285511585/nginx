# Nginx 模块的基本结构

本节对每个模块所包含的一些常用的部分进行说明，这些部分有些事必须的，有些是非必须的，并且对于所有的模块都适用，不仅针对handler模块。

## **模块配置结构**

基本上每个模块都会提供一些配置指令，以便于用户可以通过配置来控制该模块的行为。这些信息需要定义该模块的配置结构来进行存储。

{% hint style="warning" %}
nginx的配置信息分成了几个作用域（scope，也称作上下文），即：main，server以及location。
{% endhint %}

{% hint style="warning" %}
每个模块提供的配置指令可以出现在那几个作用域里，**对于不同作用域的配置信息，需要定义不同的数据结构去存储**。视模块的实现而言，**需要几个就定义几个**。
{% endhint %}

{% hint style="warning" %}
模块配置信息的命名习惯：ngx\_http\_&lt;module\_name&gt;\_\(main\|srv\|loc\)\_conf\_t，例：

```cpp
typedef struct
{
    ngx_str_t hello_string;
    ngx_int_t hello_counter;
}ngx_http_hello_loc_conf_t;
```
{% endhint %}

## **模块配置指令**

一个模块的配置指令是定义在一个静态数组中的，数组元素的类型为`ngx_command_t`，位于`src/core/ngx_conf_file.h`中，结构如下：

```cpp
struct ngx_command_s {
    ngx_str_t             name;
    ngx_uint_t            type;
    char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
    ngx_uint_t            conf;
    ngx_uint_t            offset;
    void                 *post;
};
```

### 参数介绍

#### name

配置指令的名称

#### type

该配置的类型，其实更准确一点说，是**该配置指令属性的集合**。Nginx提供了很多预定义的属性值（一些宏定义），通过逻辑或运算符可组合在一起，形成对这个配置指令的详细的说明，下面列出可在这里使用的预定义属性值及说明：

* NGX\_CONF\_NOARGS：配置指令不接受任何参数
* NGX\_CONF\_TAKE1：配置指令接受1个参数
* NGX\_CONF\_TAKE2：配置指令接受2个参数
* NGX\_CONF\_TAKE3：配置指令接受3个参数
* NGX\_CONF\_TAKE4：配置指令接受4个参数
* NGX\_CONF\_TAKE5：配置指令接受5个参数
* NGX\_CONF\_TAKE6：配置指令接受6个参数
* NGX\_CONF\_TAKE7：配置指令接受7个参数

可以组合多个属性，比如一个指令既可以不填参数，也可以接受1个或2个参数。那么就是`NGX_CONF_NOARGS|NGX_CONF_TAKE1|NGX_CONF_TAKE2`。Nginx还提供了一些定义，用于简写，如下：

* NGX\_CONF\_TAKE12：配置指令接受1个或2个参数
* NGX\_CONF\_TAKE13：配置指令接受1个或3个参数
* NGX\_CONF\_TAKE23：配置指令接受2个或3个参数
* NGX\_CONF\_TAKE123：配置指令接受1个或2个或3个参数
* NGX\_CONF\_TAKE1234：配置指令接受1个或2个或3个或4个参数
* NGX\_CONF\_1MORE：配置指令接受至少一个参数
* NGX\_CONF\_2MORE：配置指令接受至少两个参数
* NGX\_CONF\_MULTI：配置指令可以接受多个参数，即个数不定
* NGX\_CONF\_BLOCK：配置指令可以接受的值是一个配置信息块。也就是一对大括号括起来的内容。里面可以再包括很多的配置指令。比如常见的server指令就是这个属性的。
* NGX\_CONF\_FLAG：配置指令可以接受的值是“on”或者“off”，最终会被转成bool值。
* NGX\_CONF\_ANY：配置指令可以接受的任意的参数值。一个或者多个，或者“on”或者“off”，或者是配置块。

最后要说明的是，无论如何，Nginx的配置指令的参数个数不可以超过`NGX_CONF_MAX_ARGS`个。目前这个值被定义为8，也就是不**能超过8个参数值**。

下面介绍一组说明配置指令可以出现的位置的属性：

* NGX\_DIRECT\_CONF：可以出现再配置文件中最外层。例如已经提供的配置指令daemon，master\_process等。
* NGX\_MAIN\_CONF：http、mail、events、error\_log等。
* NGX\_ANY\_CONF：该配置指令可以出现在任意配置级别上。

对于我们编写的大多数模块而言，都是再处理http相关的事情，也就是所谓的都是`NGX_HTTP_MODULE`，对于这样类型的模块，其配置可能出现的位置也是直接出现在http里面，以及其它位置：

* NGX\_HTTP\_MAIN\_CONF：可以直接出现在http配置指令里。
* NGX\_HTTP\_SRV\_CONF：可以出现在http里面的server配置指令里。
* NGX\_HTTP\_LOC\_CONF：可以出现在http server块里面的location配置指令里。
* NGX\_HTTP\_UPS\_CONF：可以出现在http里面的upstream配置指令里。
* NGX\_HTTP\_SIF\_CONF：可以出现在http里面的server配置指令里的if语句所在的block中。
* NGX\_HTTP\_LMT\_CONF：可以出现在http里面的limit\_except指令的block中。
* NGX\_HTTP\_LIF\_CONF：可以出现在http server块里面的location配置指令里的if语句的所在的block中。

#### set

这是一个**函数指针**，当Nginx在**解析配置**的时候，如果遇到这个配置指令，将会**把读取到的值传递给这个函数进行分解处理**。因为具体每个配置指令的值如何处理，只有定义这个配置指令的人是最清楚的。函数原型如下：

```cpp
char *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
```

返回值：处理成功时，返回`NGX_OK`，否则返回`NGX_CONF_ERROR`或者是一个自定义的错误信息的字符串。

参数：

* cf：该参数里面保存从配置文件读取到的原始字符串以及相关的一些信息。特别注意的是这个参数的`args`字段是一个`ngx_str_t`类型的数组，该数组的首个元素是这个配置指令本身，第二个元素是指令的第一个参数，第三个元素是第二个参数，依次类推。
* cmd：这个配置指令对应的`ngx_command_t`结构。
* conf：就是定义的存储这个配置值的结构体，比如在上面展示的那个`ngx_http_hello_loc_conf_t`。当解析这个`hello_string`变量的时候，传入的conf就指向一个`ngx_http_hello_loc_conf_t`类型的变量。用户在处理的时候可以使用类型转换，转换成自己知道的类型，再进行字段的赋值。

为了更加方便地实现对配置指令参数的读取，Nginx已经默认提供了对一些标准类型的参数进行读取的函数，可以直接赋值给set字段使用，已经实现的`set`类型函数如下：

* ngx\_conf\_set\_flag\_slot：读取`NGX_CONF_FLAG`类型的参数。
* ngx\__conf\_set\_str\_slot：读取字符串类型的参数。_
* ngx\__conf\_set_\_str\_array\_slot：读取字符串数组类型的参数。
* ngx\__conf\_set_\_keyval\_slot：读取键值对类型的参数。
* ngx\__conf\_set_\_num\_slot：读取整数类型（有符号整数`ngx_int_t`）的参数。
* ngx\__conf\_set_\_size\_slot：读取`size_t`类型的参数，也就是无符号数。
* ngx\__conf\_set_\_off\_slot：读取`off_t`类型的参数。
* ngx\__conf\_set_\_msec\_slot：读取毫秒值类型的参数。
* ngx\__conf\_set_\_sec\_slot：读取秒值类型的参数。
* ngx\__conf\_set_\_bufs\_slot：读取的参数值是2个，一个是buf的个数，一个是buf的大小。例如：`output_buffers 1 128k`。
* ngx\__conf\_set_\_enum\_slot：读取枚举类型的参数，将其转换成整数`ngx_uint_t`类型。
* ngx\__conf\_set_\_bitmask\_slot：读取参数的值，并将这些参数的值以`bit`位的形式存储。例如：`HttpDavModule`模块的`dav_methods`指令。

#### conf

该字段被`NGX_HTTP_MODULE`类型模块所用（我们编写的基本上都是`NGX_HTTP_MODULE`，只有一些Nginx核心模块是非`NGX_HTTP_MODULE`），**该字段指定当前配置项存储的内存位置**。实际上是使用哪个内存池的问题。因为http模块对所有http模块所要保存的配置信息，划分了main、server和location三个地方进行存储，每个地方都有一个内存池用来分配存储这些信息的内存。这里可能的值为`NGX_HTTP_MAIN_CONF_OFFSET`、`NGX_HTTP_SRV_CONF_OFFEST`或`NGX_HTTP_LOC_CONF_OFFSET`。当然也可以直接置为0，就是`NGX_HTTP_MAIN_CONF_OFFSET`。

#### offset

**指定该配置项值得精确存放位置**，一般指定为某一个结构体变量得字段便宜。因为对于配置信息的存储，一般我们都是定义个结构体来存储的。那么比如我们定义了一个结构体A，该项配置的值需要存储到该结构体的b字段。那么在这里就可以填写为`offsetof(A, b)`。对于有些配置项，它的值不需要保存或者是需要保存到更为复杂的结构中，这里可以设置为0.

#### post

该字段存储一个指针，**可以指向任何一个在读取配置过程中需要的数据**，以便于进行配置读取的处理。大多数时候，都不需要，所以简单地设为0即可。

### 示例

```cpp
static ngx_command_t ngx_http_hello_commands[] = {
   { 
        ngx_string("hello_string"),
        NGX_HTTP_LOC_CONF|NGX_CONF_NOARGS|NGX_CONF_TAKE1,
        ngx_http_hello_string,
        NGX_HTTP_LOC_CONF_OFFSET,
        offsetof(ngx_http_hello_loc_conf_t, hello_string),
        NULL },

    { 
        ngx_string("hello_counter"),
        NGX_HTTP_LOC_CONF|NGX_CONF_FLAG,
        ngx_http_hello_counter,
        NGX_HTTP_LOC_CONF_OFFSET,
        offsetof(ngx_http_hello_loc_conf_t, hello_counter),
        NULL },               

    ngx_null_command
};
```

可以看出`ngx_http_hello_commands`这个数组每5个元素为一组，**用来描述一个配置项的所有情况**。那么如果由多个配置项，只要按照需要增加5个对应的元素对新的配置项进行说明即可。

{% hint style="warning" %}
在`ngx_http_hello_commands`这个数组定义的最后，都要加一个`ngx_null_command`作为结尾。
{% endhint %}

## 模块上下文结构

这是一个`ngx_http_module_t`类型的静态变量。这个变量实际上是**提供一组回调函数指针**，这些函数有在创建存储配置信息的对象的函数，也有在创建前和创建后会调用的函数。这些函数都将被Nginx在合适的时间进行调用。

### 参数介绍

```cpp
typedef struct {
    // preconfiguration：在创建和读取该模块的配置信息之前被调用
    ngx_int_t   (*preconfiguration)(ngx_conf_t *cf);
    // postconfiguration：在创建和读取该模块的配置信息之后被调用
    ngx_int_t   (*postconfiguration)(ngx_conf_t *cf);
    // create_main_conf：调用该函数创建本模块位于http block的配置信息存储结构。该函数成功的时候，返回创建的配置对象。失败的话，返回NULL.
    void       *(*create_main_conf)(ngx_conf_t *cf);
    // init_main_conf：调用该函数初始化本模块位于http block的配置信息存储结构。该函数成功的时候，返回NGX_CONF_OK。失败的话，返回NGX_CONF_ERROR或错误字符串。
    char       *(*init_main_conf)(ngx_conf_t *cf, void *conf);
    // create_srv_conf：调用该函数创建本模块位于http server block的配置信息存储结构，每个server block会创建一个。该函数成功的时候，返回创建的配置对象。失败的话，返回NULL。
    void       *(*create_srv_conf)(ngx_conf_t *cf);
    /*
     * merge_srv_conf：因为有些配置指令既可以出现在http block，
     * 也可以出现在http server block中。那么遇到这种情况，
     * 每个server都会有自己存储结构来存储该server的配置，
     * 但是在这种情况下http block中的配置于server block中的配置信息发生冲突的时候，
     * 就需要调用此函数进行合并，该函数并非必须提供，当预计到绝对不会发生需要合并的情况的时候，
     * 就无需提供。当然为了安全起见还是建议提供。该函数执行成功的时候，返回NGX_CONF_OF。
     * 失败的话，返回NGX_CONF_ERROR或错误字符串。
     */ 
    char       *(*merge_srv_conf)(ngx_conf_t *cf, void *prev, void *conf);
    // create_loc_conf：调用该函数创建本模块位于location block的配置信息存储结构。每个在配置中指明的location创建一个。该函数执行成功，返回创建的配置对象。失败的话，返回NULL。
    void       *(*create_loc_conf)(ngx_conf_t *cf);
    // merge_loc_conf：与merge_srv_conf类似，这个也是进行配置值合并的函数。该函数成功的时候，返回NGX_CONF_OK。失败的话，返回NGX_CONF_ERROR或错误字符串。
    char       *(*merge_loc_conf)(ngx_conf_t *cf, void *prev, void *conf);
} ngx_http_module_t; 
```

{% hint style="warning" %}
Nginx里面的配置信息都是上下一层层地嵌套地，对于具体某个location的话，对于同一个配置，**如果当前层次没有定义，那么就使用上层的配置，否则使用当层的配置**。
{% endhint %}

### 示例

```cpp
static ngx_http_module_t ngx_http_hello_module_ctx = {
    NULL,                           /* preconfiguration */
    ngx_http_hello_init,            /* postconfiguration */

    NULL,                           /* create main configuration */
    NULL,                           /* init main configuration */

    NULL,                           /* create server configuration */
    NULL,                           /* merge server configuration */

    ngx_http_hello_create_loc_conf, /* create location configuration */
    NULL                            /* merge location configuration */
};
```

## 模块的定义

对于

