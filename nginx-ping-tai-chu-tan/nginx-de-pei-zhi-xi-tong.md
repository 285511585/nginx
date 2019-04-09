# Nginx 的配置系统

Nginx的配置系统由一个主配置文件和其它一些辅助的配置文件构成。这些配置文件均是**纯文本**，全部位于Nginx安装目录下的**conf目录**下。

除了主配置文件nginx.conf以外的文件都是在某些情况下才使用的，**只有主配置文件时在任何情况下都被使用的**。

{% hint style="warning" %}
配置文件中以**`#`**开始的行，或者是前面有若干**空格**或**TAB**，然后再跟**`#`**的行，都被认为是注释。
{% endhint %}

## **指令概述**

配置指令是一个**字符串**，可以用**单引号**或者**双引号**括起来，也可以不括。但是如果配置指令包括空格，一定要括起来。

## **指令参数**

配置指令的参数是由一个或多个TOKEN串组成的，TOKEN串分为：

* 简单字符串 
* 复合配置块：由大括号括起来的一堆内容，一个复合配置块中可能包含若干其它的配置指令。

根据TOKEN串的不同，配置指令分为：

* 简单配置项（不包含复合配置块）：结尾使用分号结束，例：

  ```cpp
  error_page 500 502 503 504 /50x.html;
  ```

* 复杂配置项（包含复合配置块）：若包含多个TOKEN串，一般都是**简单TOKEN串放在前面**，**复合配置块一般位于后面**，而且其结尾，并**不需要再添加分号**，例：

  ```css
  location / {
      root   /home/jizhao/nginx-book/build/html;
      index  index.html index.htm;
  }
  ```

{% hint style="warning" %}
分隔：

指令的参数和指令：由一个或多个空格或TAB分隔

指令的参数和参数：由空格或TAB分隔
{% endhint %}

## 指令上下文

主配置文件`nginx.conf`中的配置信息，根据其逻辑上的意义，可以对它们进行分类，分成多个作用域，或者称之为配置指令上下文。不同的作用域含有一个或者多个配置项。

当前nginx支持的指令上下文如下：

* main：Nginx在运行时与具体业务功能（比如http服务或者email服务代理）无关的一些参数，比如工作进程数，运行的身份等。
* http：与提供http服务相关的一些配置参数，例如：是否使用keepalive，是否使用gzip进行压缩等。
* server：http服务上支持若干虚拟机。每个虚拟主机对应一个server配置项，配置项里面包含该虚拟主机相关的配置。在提供mail服务的代理时，也可以建立若干server，每个server通过监听的地址来区分。
* location：http服务中，某些特定的URL对应的一系列配置项。
* mail：实现email相关的SMTP/IMAP/POP3代理时，共享的一些配置项（因为可能实现多个代理，工作在多个监听地址上）。

{% hint style="info" %}
SMTP：Simple Mail Transfer Protocol，简单邮件传输协议，是在Internet传输电子邮件的标准，通过它来控制邮件的中转方式。
{% endhint %}

{% hint style="info" %}
IMAP：Internet Message Access Protocol，因特网信息访问协议，是一个应用层协议，用来从本地邮件客户端（如Foxmail、Microsoft Outlook等）访问远程服务器上的邮件。
{% endhint %}

{% hint style="info" %}
POP3：Post Office Protocol - Version 3，邮局协议第三版，是TCP/IP协议族中的一员，主要用于支持使用客户端远程管理在服务器上的电子邮件。
{% endhint %}

{% hint style="warning" %}
IMAP和POP3的区别：

IMAP和POP3类似，都是用于支持客户端远程访问和管理服务器上的电子邮件，但是IMAP与POP3不同，POP3收取了邮件之后，邮件会从服务器上删除，而IMAP则不会从服务器上删除邮件，而是**保持客户端和服务器的同步**，而且所有对邮件的操作也会同步到服务器上，**无论从浏览器还是客户端登陆邮箱，看到的邮件及其状态都是一样的（这个就是IMAP出现的理由吧，为了保证无论从哪里看到的邮件都是保持一致的，而不是从一处读取了文件之后，另一处就读不到这个文件了）**。
{% endhint %}

指令上下文，可能有包含的情况出现：

* 通常**http和mail上下文一定是出现在main上下文里面**的
* 在一个上下文里，**可能包含另外一种类型的上下文多次**，例如：如果http服务支持了多个虚拟主机，那么在http的上下文里，就会出现多次server的上下文。

配置文件示例：

```css
user  nobody;
worker_processes  1;
error_log  logs/error.log  info;

events {
    worker_connections  1024;
}

http {  
    server {  
        listen          80;  
        server_name     www.linuxidc.com;  
        access_log      logs/linuxidc.access.log main;  
        location / {  
            index index.html;  
            root  /var/www/linuxidc.com/htdocs;  
        }  
    }  

    server {  
        listen          80;  
        server_name     www.Androidj.com;  
        access_log      logs/androidj.access.log main;  
        location / {  
            index index.html;  
            root  /var/www/androidj.com/htdocs;  
        }  
    }  
}

mail {
    auth_http  127.0.0.1:80/auth.php;
    pop3_capabilities  "TOP"  "USER";
    imap_capabilities  "IMAP4rev1"  "UIDPLUS";

    server {
        listen     110;
        protocol   pop3;
        proxy      on;
    }
    server {
        listen      25;
        protocol    smtp;
        proxy       on;
        smtp_auth   login plain;
        xclient     off;
    }
}
```

在以上这个配置文件中，包含了上面提到的物种配置指令上下文：

1. 存在于main上下文中的配置指令如下：
   * user
   * worker\_processes
   * error\_log
   * events
   * http
   * mail
2. 存在于http上下文中的配置指令如下：
   * server
3. 存在于mail上下文中的配置指令如下：
   * server
   * auth\_http
   * imap\_capabilities
4. 存在于server上下文中的配置指令如下：
   * listen
   * server\_name
   * access\_log
   * location
   * protocol
   * proxy
   * smtp\_auth
   * xclient
5. 存在于location上下文中的配置指令如下：
   * index
   * root

