# 初探Nginx架构

nginx在unix系统中通常以守护进程的方式在后台运行，后台进程包含一个master进程和多个worker进程，可以手动关掉后台模式，让Ngix在前台运行，方便调试，也可以通过配置让Nginx取消master进程，从而可以使Nginx以单进程方式运行，这也只是方便调试，在正式的生产环境下，Nginx是以后台方式、多进程运行的。

另外，nginx**也支持多线程**，但是**主流还是多进程**。

## **模型进程**

nginx在启动后，会有一个master进程和多个worker进程。master进程主要用来管理worker进程，包括：接收来自外界的信号，向各worker进程发送信号，监控worker进程的运行状态，当worker进程退出后（异常情况下），会自动重新启动新的worker进程。

基本的网络事件，是放在worker进程中处理的，多个worker进程之间是对等的，他们同等竞争来自客户端的请求，各进程互相之间是独立的。一个请求完全，也只能被一个worker进程处理。

{% hint style="warning" %}
worker进程的个数和CPU的核数保持一致，更多的worker也只会导致进程来竞争CPU资源而已。
{% endhint %}

![Nginx&#x7684;&#x8FDB;&#x7A0B;&#x6A21;&#x578B;](../.gitbook/assets/image%20%287%29.png)

在nginx启动后，要操作nginx，只需要操作master就行了，master进程会接收来自外界发来的信息，再根据信号做不同的事情。

{% hint style="warning" %}
Nginx的命令：

 `./nginx -s reload：`重启 Nginx

`./nginx -s stop：`停止 Nginx 的运行
{% endhint %}

### Nginx的重启

1. master接收到信号后，会先重新加载配置文件
2. 然后再启动新的worker进程
3. 并向所有老的worker进程发送信号，告诉它们可以光荣退休了。
4. 新的worker在启动后，就开始接收新的请求
5. 老的worker在接收到来自master的信号后，就不再接收新的请求，并且在当前进程中的所有未处理完的请求处理完成后，再退出。

### worker处理请求的过程

1. 每个worker都是从master进程fork过来的，在master进程里面，先建立好需要listen的socket（listenfd）之后，然后再fork出多个worker进程。
2. 所有的worker进程在注册listenfd读事件前抢accept\_mutex，抢到互斥锁的那个进程注册listenfd读事件，在读事件里调用accept接收该连接。
3. 在一个worker进程accept了连接之后，就开始读取请求，解析请求，处理请求，产生数据后，再返回给客户端，最后才断开连接。

{% hint style="warning" %}
进程模型的好处：

1. 无需加锁
2. 互不影响
{% endhint %}

## **事件**

对于一个基本的web服务器来说，事件通常有三种类型：

1. 网络事件：**通过异步非阻塞来解决**
2. 信号：若nginx在`epoll_wait`时，程序收到信号，在信号处理函数处理完成后，`epoll_wait`会返回错误，然后程序可再次进入`epoll_wait`调用
3. 定时器：nginx借助超时时间来实现定时器，定时器事件放在一棵维护定时器的**红黑树**里面，**每次进入epoll\_wait前**，先从该红黑树里面拿到所有定时器事件的最小时间，在计算出`epoll_wait`的超时时间后进入`epoll_wait`。所以，当没有事件产生，也没有中断信号时，`epoll_wait`会超时，也就是说，定时事件到了。这时，nginx会**检查所有的超时事件**，将它们的状态设置为超时，然后**再处理网络事件**。伪代码：

   ```c
   // 每次进入epoll_wait的操作
   while (true) {
       for t in run_tasks:
           t.handler();
       update_time(&now);
       // 检查所有的超时事件，并设置超时
       timeout = ETERNITY;
       for t in wait_tasks: /* sorted already */
           if (t.time <= now) {
               t.timeout_handler();
           } else {
               timeout = t.time - now;
               break;
           }
       // 再处理网络事件
       nevents = poll_function(events, timeout);
       for i in nevents:
           task t;
           if (events[i].type == READ) {
               t.handler = read_handler;
           } else { /* events[i].type == WRITE */
               t.handler = write_handler;
           }
           run_tasks_add(t);
   }
   ```

{% hint style="warning" %}
nginx采用异步非阻塞的方式来处理请求，利用epoll模型，实现在请求之间的互相切换（切换的原因是因为异步事件还没有处理好），而不需要为每个请求创建线程，减少资源的浪费，提高并发数。

但是要注意，这里的**并发数，是指未处理完的请求**，而**不是同时处理的请求**，因为真正用于处理请求的线程可能只有一个。（当然也可以使用多线程来处理，以充分利用资源）
{% endhint %}

