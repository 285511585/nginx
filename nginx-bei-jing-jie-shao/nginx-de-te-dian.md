# Nginx的特点

#### Nginx作为HTTP服务器，有以下几项基本特性：

处理静态文件，索引文件以及自动索引，打开文件描述符缓冲

{% hint style="info" %}
文件描述符：

* 在Linux系统中打开文件就会获得文件描述符，它是一个很小的非负数。
* 每个进程在PCB（process control block）中保存着一份文件描述符表，文件描述符就是这个表的索引，每个表项都有一个指向已经打开文件的指针。
* 在c语言中，可以通过open或create获得文件描述符，或通过父进程继承。
* 文件描述符也称句柄，Windows即称文件描述符为句柄。
* 标准输入，标注输出，标准错误输出也对应了三个文件描述符，分别为0，1，2 。
* 每个进程能持有的文件描述符数有限的，一般为65536
{% endhint %}

{% hint style="warning" %}
文件描述符的好处与坏处：

* **好处**

1. 基于文件描述符的I/O操作兼容POSIX（portable operating system interface of UNIX）标准
2. 在UNIX，LINUX的系统调用中，大量依赖文件描述符

* **坏处**

1. 非UNIX/LINUX的OS无法基于这个概念进行编程
2. 文件描述符时整数，不具备可读性
{% endhint %}

{% hint style="info" %}
文件指针：

c语言中使用文件指针作为I/O的句柄。文件指针指向进程用户区中的一个被称为FILE结构的数据结构。FILE结构包括一个缓冲区和一个文件描述符。而文件描述符时文件描述符表的一个索引，因此从某种意义上说，文件指针就是句柄的句柄。
{% endhint %}

{% hint style="info" %}
句柄：

指一种特殊的智能指针，当一个应用程序要引用其他系统，如数据库、操作系统所管理的内存块或对象时，就要使用句柄
{% endhint %}

{% hint style="info" %}
文件描述符缓冲：

Nginx缓存会将最近使用的文件描述符和相关元数据（如修改事件、大小等）存储在缓存中。

配置：

`open_file_cache #  存储缓存`

`open_file_cache_valid # 每隔一段时间验证open_file_cache中的元素`

`open_file_cache_min_uses # 最小访问次数，以将元素标记为活动使用`

`open_file_cache_errors # 启用缓存错误，访问资源错误Nginx也会报告错误`
{% endhint %}

{% hint style="warning" %}
Nginx缓存并不会缓存请求文件的内容
{% endhint %}

无缓存的后向代理加速，简单的负载均衡和容错

{% hint style="info" %}
前向代理：

前向代理作为客户端的代理，将从互联网上获取的资源返回给一个或多个的客户端，服务端（如web服务器）只知道代理的IP地址而不知道客户端的IP地址。
{% endhint %}

{% hint style="info" %}
反向代理：

服务器根据客户端的请求，从其关系的一组或多组后端服务器（如web服务器）上获取资源，然后再将这些资源返回给客户端，客户端只会得知反向代理的IP地址，而不知道再代理服务器后面的服务器簇的存在。
{% endhint %}

{% hint style="info" %}
负载均衡：

负载均衡是一种计算机技术，用来在多个计算机（计算机集群）、网络连接、CPU、磁盘驱动或其它资源中分配负载，以达到最优化资源使用、最大化吞吐率、最小化响应事件，同时避免过载的目的。使用带有负载平衡的多个服务器组件，取代单一的组件，可以通过冗余提高可靠性。

负载平衡服务通常是由专门软件或硬件来完成。**主要作用是将大量作业合理地分摊到多个操作单元上进行执行，用以解决互联网架构中的高并发和高可用问题**。

* 负载均衡的最重要的一个应用时利用多台服务器提供单一服务
* 负载均衡器有各种各样的**工作调度算法**：最简单的时随机选择轮询，更高级的算法会考虑更多的相关元素。
* 高性能系统通常会使用多层负载均衡，另外专用的硬件负载均衡器以及纯软件负载均衡器
* 分发策略有两种：bowties（每层多条路径可选）、stovepipes（从上到下）
* 添加了负载均衡之后，需要注意用户会话的保存
{% endhint %}

{% hint style="warning" %}
负载均衡中用户会话的保存方案

1. 负载均衡器缓存会话——会导致负载问题
2. 发往同一个后台服务器——无法容错（故障转移）
3. 依据用户名、IP来分配服务器——但IP可能有变
4. 会话信息保存到数据库中——会增加数据库负载
5. 客户端自行保持会话信息——但会话数据可能过大。且存在安全问题
{% endhint %}

{% hint style="info" %}
容错：

是指软件检测应用程序所运行的软件或硬件中发生的错误并从错误中恢复的能力，通常可以从系统的可靠性，可用性，可测性等几个方面来衡量。

fault tolerance，确切来说时容故障（fault），而并非错误（error）。例如在双机容错系统中，一台机器出现问题时，另一台机器可以取而代之，从而保证系统的正常运行。

其实是进行了**故障转移**。
{% endhint %}

FastCGI，简单的负载均衡和容错

{% hint style="info" %}
快速通用网关接口（Fast Common Gateway Interface）：

* 是一种让交互程序与web服务器通信的协议，FastCGI是早期的通用网关接口CGI的增强版
* 致力于减少web服务器与CGI程序之间交互的开销，从而使服务器可以同时处理更多的请求
* CGI为每个请求创建一个进程，而FastCGI则用持续进程处理一连串的请求
{% endhint %}

{% hint style="info" %}
通用网关接口（Common Gateway Interface）：

CGI是外部应用程序（CGI程序）与web服务器之间的接口标准，是在CGI程序与web服务器之间传递信息的过程。

CGI规定了web服务器以什么格式传什么数据给CGI程序。
{% endhint %}

{% hint style="info" %}
CGI程序：

就有点类似php，jsp这些服务端脚本

在python的CGI编程中，可以直接在python文件中print html代码，生成相应的页面。
{% endhint %}

![](../.gitbook/assets/cgi-gong-zuo-yuan-li.png)

模块化的结构，包括gzipping，byte ranges，chunked responses，以及SSI-filter等filter。如果由FastCGI或其它代理服务器处理但也存在的多个SSI，则这项处理可以并行运行，而不需要互相等待。

{% hint style="info" %}
SSI（Server Side Include）：

服务端包含，是一种类似于ASP的基于服务器的网页制作技术，将文本，图形或应用程序信息包含到网页。
{% endhint %}

{% hint style="info" %}
gzip：

是一种流行的数据压缩程序，减少浏览器与web服务器之间的数据传输，可以让数据量更小，速度更快，但对于一些图像文件，压缩效果就不那么明显。
{% endhint %}

{% hint style="info" %}
byte ranges：

html5提供了一个range标签来实现文件的分段下载，nginx也要设置才能允许range标签返回。
{% endhint %}

{% hint style="info" %}
chunked responses：

chunked transfer encoding是超文本传输协议（HTTP）中的一种数据传输机制，允许HTTP由服务器发送给客户端应用（通常是网页浏览器）的数据可以分成多个部分。分块传输编码只在HTTP协议1.1版本提供。此时content-length的长度就不固定了。

Nginx也要进行相应的设置才能允许此功能。
{% endhint %}

支持SSL和TLS SNI

{% hint style="info" %}
SSL（Secure Socket Layer，安全套接层）和TLS（Transport Layer Security，传输层安全性协议）：

TLS是SSL的延续，两者都是一种安全协议，目的是为互联网通信提供安全及数据完整性保障。

SSL包含记录层（Record Layer）和传输层，记录层确定传输层数据的封装格式。传输层安全协议使用X.509认证，之后利用非对称加密演算来对通信方做身份认证，之后交换对称密钥作为会话密钥（session key）。这个会话密钥是用来将通信两方交换的数据做加密的，保证两个应用间通信的保密性和可靠性，使得客户与服务器应用之间的通信不会被攻击者窃听。
{% endhint %}

![SSL&#x7ED3;&#x6784;](../.gitbook/assets/image%20%282%29.png)

{% hint style="info" %}
SNI（Server Name Indication，服务器名称指示）：

是为了解决一个服务器使用了多个域名和证书的SSL/TLS扩展。它允许客户端在发起SLL握手请求时提交请求中的**主机名**，使得服务器能够**切换到正确的域并返回相应的证书**。
{% endhint %}

{% hint style="info" %}
证书：

又称数字证书（digital certificate）、公钥证书，公开密钥证书，公开密钥认证（public key certificate），电子证书或安全证书，是用于公开密钥基础建设的电子文件，用来证明公开密钥持有者的身份。（服务端、客户端都可以由自己的证书）
{% endhint %}

