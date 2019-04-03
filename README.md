---
description: 前言
---

# nginx入门指南

## nginx基本特点

#### 处理静态文件，索引文件以及自助索引，打开文件描述符

{% hint style="info" %}
文件描述符：

在Linux系统中打开文件就会获得文件描述符，它是一个很小的非负数。

每个进程在PCB（process control block）中保存着一份文件描述符表，文件描述符就是这个表的索引，每个表项都有一个指向已经打开文件的指针。

在c语言中，可以通过open或create获得文件描述符，或通过父进程继承。

文件描述符也称句柄
{% endhint %}

{% hint style="info" %}
文件指针：

c语言中使用文件指针作为I/O的句柄。文件指针指向进程用户区中的一个被称为FILE结构的数据结构。FILE结构包括一个缓冲区和一个文件描述符。而文件描述符时文件描述符表的一个索引，因此从某种意义上说，文件指针就是句柄的句柄。
{% endhint %}

{% hint style="info" %}
句柄：

指一种特殊的智能指针，当一个应用程序要引用其他系统，如数据库、操作系统所管理的内存块或对象时，就要使用句柄
{% endhint %}

{% hint style="warning" %}
每个进程能持有的文件描述符数有限的，一般为65536
{% endhint %}

#### 无缓存的后向代理加速，简单的负载均衡和容错

#### FastCGI，简单的负载均衡和容错

#### 模块化的结构，包括gzipping，byte ranges，chunked responses，以及SSI-filter等filter。如果由FastCGI或其它代理服务器处理但也存在的多个SSI，则这项处理可以并行运行，而不需要互相等待。

#### 支持SSL和TLS SNI

## Nginx的适用人群

