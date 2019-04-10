# Nginx event模块

## event的类型和功能

Nginx是以event（事件）处理模型为基础的模块，它为了支持跨平台，抽象出了event模块。它支持的event处理类型有：AIO（异步IO），/dev/poll（Solaris和Unix特有），epoll（Linux特有），eventport（Solaris 10特有），kqueue（BSD特有），poll，rtsig（实时信号），select等。

event模块的主要功能是：**监听accept后建立的连接，对读写事件进行添加删除**。

事件处理模型和Nginx的非阻塞IO模型结合在一起使用。当IO可读可写的时候，相应的读写事件就会被唤醒，此时就会去处理事件的回调函数。

特别对于Linux，Nginx大部分event采用epoll EPOLLET（边沿触发）的方法来触发事件，只有listen端口的读事件是EPOLLLT（水平触发）。对于边沿触发，如果出现了可读事件，必须及时处理，否则可能会出现读事件不再触发，连接饿死的情况。

event处理抽象出来的关键结构体如下：

```cpp
// 每个event处理模型，都需要实现部分功能
typedef struct {
        /* 添加删除事件，这两个是最基本的功能 */
        ngx_int_t  (*add)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);
        ngx_int_t  (*del)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);

        ngx_int_t  (*enable)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);
        ngx_int_t  (*disable)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);

        /* 添加删除连接，会同时监听读写事件 */
        ngx_int_t  (*add_conn)(ngx_connection_t *c);
        ngx_int_t  (*del_conn)(ngx_connection_t *c, ngx_uint_t flags);

        ngx_int_t  (*process_changes)(ngx_cycle_t *cycle, ngx_uint_t nowait);
        /* 处理事件的函数 */
        ngx_int_t  (*process_events)(ngx_cycle_t *cycle, ngx_msec_t timer,
                                   ngx_uint_t flags);

        ngx_int_t  (*init)(ngx_cycle_t *cycle, ngx_msec_t timer);
        void       (*done)(ngx_cycle_t *cycle);
} ngx_event_actions_t;
```

## accept锁

Nginx是多进程程序，80端口是各进程所共享的，多进程同时listen 80 端口，势必会产生竞争，也产生了所谓的“惊群”效用。

当内核accept一个连接时，会唤醒所有等待中的进程，但实际上只有一个进程能获取连接，其它的进程都是被无效唤醒的。所以Nginx**采用了自有的一套accept加锁机制**，避免多个进程同时调用accept。Nginx多进程的锁在底层默认是通过CPU自旋锁来实现。如果操作系统不支持**自旋锁**，就采用**文件锁**。

{% hint style="info" %}
互斥锁 mutex：

* 在访问共享资源之前对其进行加锁操作，在访问完成之后对其进行解锁操作。
* 加锁后，任何其它试图再次加锁的线程会被阻塞，直到当前线程解锁。
* 如果解锁时有一个以上的线程阻塞，那么所有该锁上的线程都会被变成就绪状态。
* 第一个变为就绪状态的线程又执行加锁操作，那么其它的线程又会进入等待。

**这种情况下，只有一个线程能够访问被互斥锁保护的资源。**
{% endhint %}

{% hint style="info" %}
自旋锁：

自旋锁的使用模式和互斥锁很类似，只是在加锁后，有线程试图再次执行加锁操作的时候，该线程**不会阻塞，而处于循环等待的忙等状态**（CPU不能够做其它事情）。

自旋锁适用的场景：**锁被持有的时间较短**，而且进程并不希望在重新调度上花费太多的成本
{% endhint %}

{% hint style="info" %}
读写锁 rwlock，也称共享互斥锁：读模式共享，写模式互斥：

读写锁有三种状态：读加锁状态，写加锁状态和不加锁状态

**一次只有一个线程可以占有写模式的读写锁，但是多个线程可以同时占有读模式的读写锁。（这也是它能够实现高并发的一种手段）**

* 当读写锁在写加锁模式下，任何试图对这个锁进行加锁的**线程都会被阻塞**，直到写进程对其解锁。
* 当读写锁在读加锁模式下，**任何线程**都可以对其进行读加锁操作，但是所有试图进行写加锁操作的线程都会被阻塞，直到所有的读线程都解锁。

读写锁适用场景：对数据结构**读的次数远远大于写**。

如果直接按照上面所说的读写锁操作进行的话，就可能导致写操作等待过长，一种解决方案是：

* 当处于读模式的读写锁接收到一个试图对其进行写加锁操作的时候，便会阻塞后面对其进行读加锁操作的进程，这样只要等当前的读加锁的线程释放锁之后，就能执行写加锁操作了。
{% endhint %}

{% hint style="info" %}
RCU锁 Read-Copy Update lock：

这是对读写锁的一种改进，同样是对读线程和写线程进行区别对待，只不过对待的方式是不同的。

读写锁中只允许多个读线程同时访问被保护的数据，但是**在RCU中允许多个读线程和写线程同时访问被保护的资源**。写线程的同步开销则取决于使用的写线程间同步机制，RCU并不对此进行支持。

RCU中:

* 读线程不需要使用锁，直接访问资源即可。
* 写线程同步开销比较大，要等到所有的读线程都访问完成了才能够对被保护的资源**进行更新**。写线程修改数据前**首先拷贝一个被修改元素的副本**，然后**在副本上进行修改**，修改完毕后它向垃圾回收器注册一个回调函数以便**在适当的时机执行真正的修改**操作。
{% endhint %}

Nginx事件处理的入口函数是`ngx_process_events_and_times`，下面是部分代码，可以看到其加锁过程：

```cpp
if (ngx_use_accept_mutex) {
        // ngx_accept_disabled = ngx_cycle->connection_n / 8 - ngx_cycle->free_connection_n;
        // 小于八分之一则禁止接受accept一段时间
        if (ngx_accept_disabled > 0) {
                ngx_accept_disabled--;

        } else {
                if (ngx_trylock_accept_mutex(cycle) == NGX_ERROR) {
                        return;
                }

                if (ngx_accept_mutex_held) {
                        flags |= NGX_POST_EVENTS;

                } else {
                        if (timer == NGX_TIMER_INFINITE
                                || timer > ngx_accept_mutex_delay)
                        {
                                timer = ngx_accept_mutex_delay;
                        }
                }
        }
}
```

在`ngx_trylock_accept_mutex`函数里面，如果拿到了锁，Nginx会把listen的端口读事件加入event处理，该进程在有新连接进来时就可以进行accept了。

{% hint style="warning" %}
accept操作是一个普通的读事件，下面的代码说明了这点：

```cpp
(void) ngx_process_events(cycle, timer, flags);

if (ngx_posted_accept_events) {
        ngx_event_process_posted(cycle, &ngx_posted_accept_events);
}

if (ngx_accept_mutex_held) {
        ngx_shmtx_unlock(&ngx_accept_mutex);
}
```
{% endhint %}

`ngx_process_events`函数是所有事件处理的入口，它会遍历所有的事件。抢到了accept锁的进程跟一般进程稍有不同的是，它被加上了`NGX_POST_EVENTS`标志，也就是说在`ngx_process_events`函数里面只接受而不处理事件，并加入`post_events`队列里。直到`ngx_accept_mutex`锁去掉以后才去处理具体的事件。这样做的目的是**减少该进程抢到锁之后，从accept开始到结束的事件，以便其它进程继续接受新的连接，提高吞吐量**。

* ngx\_posted\_accept\_events：accept延迟事件队列，是放到`ngx_accept_mutex`锁里面处理的，该队列里面处理的都是accept事件，它会一口气把内核`backlog`里等待的连接都accept进来，注册到读写事件里。
* ngx\_posted\_events：普通延迟事件队列，一般来说，那些CPU耗时比较多的都可以放进去。

{% hint style="warning" %}
Nginx事件处理都是根据触发顺序在一个大循环里面依次处理的，因为Nginx一个进程同时只能处理一个事件，所以有些耗时多的事件会把后面所有事件的处理都耽搁了。
{% endhint %}

另外，Nginx也对各进程的请求处理的均衡性做了优化，比如在负载高的时候，进程抢到的锁过多，会导致这个进程被禁止接受请求一段时间。

```cpp
// ngx_event_accept函数中，有如下代码：
// ngx_cycle->connection_n：进程可以分配的连接总数
// ngx_cycle->free_connection_n：空闲的连接数
// 当前进程空闲连接数小于1/8时，就会被禁止accept一段事件
ngx_accept_disabled = ngx_cycle->connection_n / 8 - ngx_cycle->free_connection_n;
```

## 定时器

Nginx在需要用到超时的时候，都会用到定时器机制，比如建立连接以后的那些读写超时。Nginx使用红黑树来构造定时器，红黑树是一种有序的二叉平衡树，其查找插入和删除的复杂度都为O\(logn\)，所以是一种比较理想的二叉树。

定时器的机制就是，**二叉树的值是其超时时间**，每次查找二叉树的最小值，如果最小值已经过期，就删除该节点，然后继续查找，直到所有超时节点都被删除。

