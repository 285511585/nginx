# 使用upstream模块时出现的问题

## 提示502 Bad Gateway错误

此时`/var/log/nginx/error.log`文件中提示：

`[error] 6685#6685: *20 recv() failed (104: Connection reset by peer) while reading response header from upstream, client: xxx, server: test, request: "POST / HTTP/1.1", upstream: xx, host: xx, referrer: xx`

error=104，这个错误表示在对一个对端socket已经关闭的连接调用write或send方法，在这种情况下，调用write或send方法后，对端socket便会向本端socket发送一个RESET信号，在此之后如果继续执行write或send操作，就会得到错误码为：104，错误信息为：Connection reset by peer的错误。

按照网上的说法，这个问题是因为upstream将连接重置了，原因可能是：

1. timeout太小，连接超时了
2. 数据长度不一致
3. nginx的buffer太小

但是，按照网上的说法，修改nginx的配置文件之后，**问题仍然没有得到解决**，之后，误打误撞，**修改了后端服务的监听端口**，这个问题就解决了。。。

但是，Linux的端口划分如下：

* 0~1023：系统/保留端口，这些端口只有系统特许的进程才能使用
* 1024~65535：用户端口，又分为：
  * 1024~5000：BSD临时端口，一般的应用程序使用1024到4999来进行通讯
  * 5001~65535：BSD服务器\(非特权\)端口，用来给用户自定义端口

又或者这样划分：

* 0~1023：公认端口
* 1024~49151：注册端口
* 49152~65535：动态/私有端口

我使用的是5000以上的端口，按理说不应该出现问题的，但莫名其妙出现502错误，修改端口到7000以上之后就没有问题了。。。

