# Nginx 的请求处理

nginx使用一个多进程模型来对外提供服务，其中一个master进程，多个worker进程，master进程负责管理nginx本身和其它的worker进程。而所有的业务逻辑处理都在worker进程。worker进程中有一个函数，执行无限循环，不断处理收到的来自客户端的请求，并进行处理，直到整个nginx服务被停止。

## 一个请求的简单处理流程

1. 操作系统提供的机制（如epoll、kqueue等）产生相关的事件 
2. 接收和处理这些事件，如是接收到数据，则产生更高层的request对象 
3. 处理request的header和body 
4. 产生响应，并发送回客户端 
5. 完成request的处理 
6. 重新初始化定时器及其它事件

## http request的处理过程

1. 初始化http request（读取来自客户端的数据，生成http request对象，该对象含有该请求所有的信息\) 
2. 处理请求头 
3. 处理请求体 
4. 如果有的话，调用与此请求（URL或Loction）关联的handler 
5. 依次调用各phase handler进行处理。

### phase handler

就是包含若干处理阶段的一些handler，在每一个阶段，包含有若干个handler，在处理到某个阶段的时候，依次调用该阶段的handler对HTTP Request进行处理。 

通常情况下，一个phase handler对这个request进行处理，并产生一些输出。 

通常phase handler是与定义在配置文件中的某个location相关联的。 

一个phase handler通常执行以下几项任务： 

* 获取location配置 
* 产生适当的响应 
* 发送response header 
* 发送response body 

当nginx读取到一个HTTP Request的header的时候，nginx首先查找与这个请求关联的虚拟主机的配置。如果找到了这个虚拟主机的配置，那么通常情况下，这个HTTP Request将回经过以下几个阶段的处理（phase handlers）： 

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

所有的filter都被组织成一条链，输出回一次穿越所有的filter，直到有一个filter模块的返回值表名已经处理完成。

### 常见的filter模块

* server-side includes 
* XSLT filtering 
* 图像缩放之类的 
* gzip压缩

{% hint style="warning" %}
几个需要注意的filter：

write：写输出到客户端，实际上是写到连接对应的socket上

postpone：这个filter是负责subrequest的，也就是子请求的。

copy：将一些需要复制的buf（文件或内容）重新复制一份然后交给剩余的body filter处理。
{% endhint %}





