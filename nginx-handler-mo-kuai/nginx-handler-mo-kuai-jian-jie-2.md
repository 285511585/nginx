# Nginx handler模块简介

作为第三方开发者最可能开发的就是三种类型的模块：handler、filter和load-balancer。

{% hint style="warning" %}
handler模块就是接收来自客户端的请求并产生输出的模块。
{% endhint %}

配置文件中使用location指令可以配置content handler模块，当Nginx系统启动的时候，**每个handler模块都有一次机会把自己关联到对应的location上**。如果有多个handler模块都关联到了同一个location，那么实际上只有一个handler模块真正会起作用。当然大多数情况下，模块开发人员都会避免出现这种情况。

handler模块处理的结果通常有三种情况：**处理成功**、**处理失败**（处理的时候发生了错误）或者**拒绝去处理**。

{% hint style="warning" %}
在拒绝处理的情况下，这个location的处理就会由默认的handler模块来进行处理。
{% endhint %}

