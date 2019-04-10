# Nginx core 模块

## Nginx的启动模块

启动模块从刚启动Nginx进程开始，做了一系列的初始化工作，源代码位于`src/core/nginx.c`，从main函数开始：

1. 时间、正则、错误日志、ssl等初始化
2. 读入命令行参数
3. OS相关初始化
4. 读入并解析配置
5. 核心模块初始化
6. 创建各种暂时文件和目录
7. 创建共享内存
8. 打开listen的端口
9. 所有模块初始化
10. 启动worker进程

{% hint style="warning" %}
注意：

1. 先初始化核心模块再初始化所有模块
2. worker进程最后才启动
{% endhint %}

