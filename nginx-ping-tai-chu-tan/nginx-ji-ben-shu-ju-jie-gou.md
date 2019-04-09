# Nginx基本数据结构

nginx的作者为了追求极致的高效，自己实现了很多颇具特色的nginx风格的数据结构以及公共函数。书写nginx模块时，应该尽量调用nginx提高的api，尽管有些api只是对glibc的宏定义。

**ngx\_str\_t**

nginx源码目录的src/core下面的ngx\_string.h\|c里面，包含了字符串的封装以及字符串相关操作的api。ngx\_str\_t时一个带长度的字符串结构，它的原型如下：

> ```c
> typedef struct {
>     size_t      len;
>     u_char     *data;
> } ngx_str_t;
> ```

**ngx\_pool\_t**

提供了一种机制，帮助管理一系列的资源（如内存，文件等），使得对这些资源的使用和释放统一进行，免除了使用过程中考虑到对各种各样资源的释放时间，是否遗漏了释放的担心

**ngx\_array\_t**

是nginx内部使用的数组结构

**ngx\_hash\_t**

是nginx自己的hash表的实现，但是不能增删元素，实际上也不是一张链表，而是一段连续空间的“数组”。

**ngx\_hash\_wildcard\_t**

为了解决带有通配符的域名的匹配问题

**ngx\_hash\_combined\_t**

组合类型hash表

**ngx\_hash\_keya\_array\_t**

为了方便hash表的构建所提供的辅助类型

**ngx\_chain\_t**

nginx的filter模块在处理从别的filter模块或者handler模块传递过来的数据（实际上就是需要发送给客户端的http response）。这个传递过来的数据是以一个链表的形式（ngx\_chain\_t）存在的。而且数据可能被分多次传递过来，也就是多次调用filter的处理函数。

**ngx\_buf\_t**

是ngx\_chain\_t链表的每个节点的实际数据，该数据结构实际上是一种抽象的数据结构，它代表某种具体的数据，这个数据可能是指向内存中的某个缓冲区，也可能指向一个文件的某一部分，也可能是一些纯元数据（元素局的作用在于指示这个链表的读取者对读取的数据进行不同的处理）。

**ngx\_list\_t**

类似list的数据结构，但是每个元素却是一个固定大小的数组。

**ngx\_queue\_t**

是nginx中的双向链表

