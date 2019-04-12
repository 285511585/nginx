# Nginx 配置文件nginx.conf中文详解

```bash
# 定义Nginx运行的用户和用户组
# user  nobody;
user www www;
# Nginx进程数，建议设置为等于CPU总核心数
worker_processes  1;

# 全局错误日志定义类型，[debug | info | notice | warn | error | crit ]
# error_log  logs/error.log;
# error_log  logs/error.log  notice;
# error_log  logs/error.log  info;

# 进程pid文件
# pid        logs/nginx.pid;

# 指定进程可以打开的最大描述符：数目
# 工作模式与连接数上限
# 这个指令是指当一个nginx进程打开的最多文件描述符数目，理论值应该是最多打开文件数（ulimit -n）与nginx进程数相除，但是nginx分配请求并不是那么均匀，所以最好与ulimit -n的值保持一致
# 现在在linux 2.6内核下开启文件打开数为65535，worker_rlimit_nofile就相应应该填写65535
# 这是因为nginx调度时分配请求到进程并不是那么均衡，所以假如填写10240，总并发量达到3-4万时就有进程可能超过10240了，这时会返回502错误
worker_rlimit_nofile 65535;

events {
    # 参考事件模型，use [ kqueue | rtsig | epoll | /dev/poll | select | poll ]; 
    # epoll模型时Linux 2.6以上版本内核中的高性能网络I/O模型，Linux建议epoll，如果跑在FreeBSD上面，就用kqueue。
    # 补充说明：
    # 与apache类似，nginx针对不同的操作系统，有不同的事件模型
    # A) 标准事件模型
    # select、poll属于标准事件模型，如果当前系统不存在更有效，nginx会选择select或poll
    # B) 高效事件模型
    # kqueue：使用于FreeBSD 4.1+，OpenBSD 2.9+，NetBSD 2.0和MacOS X（使用双处理器的MacOS X系统使用kqueue可能会造成内核崩溃）
    # epoll：使用于Linux 2.6+
    # /dev/poll：使用于Solaris 7 11/99+，HP/UX 11.22+（eventport），IRIX 6.5.15+和Tru64 UNIX 5.1A+
    # eventport：使用于Solaris 10，为了防止出现内核崩溃的问题，有必要时安装安全补丁
    use epoll;
    
    # 单个进程最大连接数（最大连接数=连接数*进程数）
    # 根据硬件调整，和前面工作进程配合起来用，尽量大，但是别把cpu跑到100%就行。每个进程允许的最多连接数，理论上每台nginx服务器的最大连接数为65535.
    worker_connections  1024;
    
    # keepalive超时事件
    keepalive_timeout 60;
    
    # 客户端请求头部的缓冲区大小。这个可以根据系统分页大小来设置，一般一个请求头的大小不会超过1k，不过由于一般系统分页都要大于1k，所以这里设置为分页大小。
    # 分页大小可以用命令getconf PAGESIZE取得：
    # # getconf PAGESIZE
    # 4096
    # 当然也有client_header_buffer_size超过4k的情况，但是client_header_buffer_size的值必须设置为“系统分页大小”的整倍数
    client_header_buffer_size 4k;
    
    # 这个将为打开文件指定缓存，默认是没有启用的，max指定缓存数量，建议和打开文件数一致，inactive是指经过多长时间文件没被请求后删除缓存
    open_file_cache max=65535 inactive=60s;
    
    # 这个是指多长时间检查一次缓存的有效信息
    # 语法：open_file_cache_valid time
    # 默认值：open_file_cache_valid 60
    # 使用字段：http，server，location
    # 这个指令指定了何时需要检查open_flie_cache中缓存项目的有效信息
    open_file_cache_valid 80s;
    
    # open_file_cache指令中的inactive参数时间内文件的最少使用次数，如果超过这个数字，文件描述符是一直在缓存中打开的，如上例，如果有一个文件在inactive时间内一次都没被使用，它将被移除。
    # 语法：open_file_cache_min_uses number
    # 默认值：open_file_cache_min_uses 1
    # 使用字段：http，server，location
    # 这个指令指定了在open_file_cache指令无效的参数中一定的时间范围内文件的最小使用数，如果使用更大的值，文件描述符在cache中总是打开状态。
    open_file_cache_min_uses 1;
    
    # 语法：open_file_cache_errors on | off
    # 默认值：open_file_cache_errors off
    # 使用字段：http，server，location 
    # 这个指令指定是否在搜索一个文件时记录cache错误
    open_file_cache_errors on;
}

http {
    # 文件扩展名于文件类型映射表
    include       mime.types;
    # 默认文件类型
    default_type  application/octet-stream;
    
    # 默认编码
    charset utf-8;
    
    # 服务器名字的hash表大小
    # 保存服务器名字的hash表是由指令server_names_hash_max_size和server_names_hash_bucket_size所控制的。
    # 参数 hash bucket size总是等于hash表的大小，并且是一路处理器缓存大小的倍数。在减少了在内存中的存取次数后，
    # 使在处理器中加速查找hash表键值称为可能。如果hash bucket size等于一路处理器缓存的大小，那么在查找键的时候，
    # 最坏的情况下在内存中查找的次数为2.第一次是确定存储单元的地址，第二次是在存储单元中查找键值。
    # 因此，如果Nginx给出需要增大hash max size或hash bucket size的提示，那么首要的是增大前一个参数的大小。
    server_names_hash_bucket_size 128;
    
    # 客户端请求头部的缓冲区大小。这个可以根据系统的分页大小来设置，一般一个请求的头部大小不会超过1k，
    # 不过由于一般系统分页都要大于1k，所以这里设置为分页大小。分页大小可以用命令getconf PAGESIZE取得。
    client_header_buffer_size 32k;
    
    # 客户端请求头缓存大小。nginx默认会用client_header_buffer这个buffer来读取header值，
    # 如果header过大，它会使用large_client_header_buffer来读取。
    large_client_header_buffers 4 64k;
    
    # 设定通过nginx上传文件的大小
    client_max_body_size 8m;
    
    # 日志格式设定
    # $remote_addr与$http_x_forwarded_for用以记录客户端的ip地址
    # $remote_user：用来记录客户端用户名称
    # $time_local：用来记录访问时间与时区
    # $request：用来记录请求的url与http协议
    # $status：用来记录请求状态，成功是200
    # $body_bytes_sent：记录发送给客户端文件主体内容大小
    # $http_referer：用来记录从哪个页面链接访问过来的
    # $http_user_agent：记录客户浏览器的相关信息
    # 通常web服务器放在反向代理的后面，这样就不能获取到客户的IP地址了，通过$remote_add拿到的IP
    # 地址是反向代理服务器的IP地址。反向代理服务器在转发请求的http头信息中，可以增加x_forwarded_for信息，
    # 用以记录原有客户端的IP地址和原有客户端的请求的服务器地址。
    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;
    
    # 开启高效文件传输模式，sendfile指令指定nginx是否调用sendfile函数（zero copy方式）来输出文件。
    # 对于普通应用设为on，如果用来进行下载等应用磁盘IO重负载应用，可设置为off，以平衡磁盘于网络I/O处理速度，降低系统的负载。
    # 注意，如果图片显示不正常把这个改为off
    sendfile        on;
    
    # 开启目录列表访问，适合下载服务器，默认关闭
    autoindex on;
    
    # 此选项允许或禁止使用socket的TCP_CORK的选项，此选项仅在使用sendfile的时候使用
    #tcp_nopush     on;
    
    tcp_nodelay on;
    
    # 长连接超时时间，单位为秒
    #keepalive_timeout  0;
    keepalive_timeout  65;
    
    # FastCGI相关参数是为了改善网站的性能：减少资源占用，提高访问速度
    fastcgi_connect_timeout 300;
    fastcgi_send_timeout 300;
    fastcgi_read_timeout 300;
    fastcgi_buffer_size 64k;
    fastcgi_buffers 4 64k;
    fastcgi_busy_buffers_size 128k;
    fastcgi_temp_file_write_size 128k;
    
    # gzip模块设置
    gzip  on; # 开启gzip压缩输出
    gzip_min_length 1k; # 最小压缩文件大小
    gzip_buffers 4 16k; # 压缩缓冲区
    gzip_http_version 1.0; # 压缩版本（默认1.1，前端如果是squid2.5请使用1.0）
    gzip_comp_level 2; # 压缩等级
    # 压缩类型，默认就已经包含textml，所以下面就不用再写了，写上去也不会由问题，但是会有一个warn。
    gzip_types text/plain application/x-javascript text/css application/xml;
    gzip_vary on;
    
    # 开始限制IP连接数的时候需要使用
    limit_zone crawler $binary_remote_addr 10m;
    
    # 负载均衡配置
    upstream jh.w3cschool.cn {
        # upstream的负载均衡，weight是权重，可以根据机器配置定义权重给。
        # weight参数表示权值，权值越高被分配到的几率越大。
        # 定义了一组服务器列表，通过upstream指定的name【jh.w3cschool.cn】，将请求转发到写服务器中
        server 192.168.80.121:80 weight=3;
        server 192.168.80.122:80 weight=2;
        server 192.168.80.123:80 weight=3;
        
        # nginx的upstream目前支持4种方式的分配
        # 1、轮询（默认）
        # 每个请求按时间顺序逐一分配到不同的后端服务器，如果后端服务器down掉，能自动剔除。
        # weight：指定轮询纪律，weight和访问比率成正比，用于后端服务器性能不均的情况
        # 例如：
        # upstream bakend {
        #     server 192.168.80.121:80 weight=10;
        #     server 192.168.80.122:80 weight=1;
        # }
        # 2、ip_hash
        # 每个请求按访问ip的hash结构分配，这样每个访客固定访问一个后端服务器，可以解决session的问题
        # 例如：
        # upstream bakend {
        #     ip_hash;
        #     server 192.168.80.121:80;
        #     server 192.168.80.122:80;
        # }
        # 3、fair（第三方）
        # 按后端服务器的响应时间来分配请求，响应时间端的优先分配
        # upstream bakend {
        #     server server1;
        #     server server2;
        #     fair;
        # }
        # 4、url_hash（第三方）
        # 按访问url的hash结构来分配请求，使每个url定向到同一个后端服务器，后端服务器为缓存时比较有效
        # 例：在upstream中加入hash语句，server语句中不能写入weight等其它的参数，hash_method是使用的hash算法
        # upstream bakend {
        #     server squid1::3128;
        #     server squid2::3128;
        #     hash $request_url;
        #     hash_method crc32;
        # }
        
        # tips：定义负载均衡设备的IP及设备状态
        # upstream bakend {
        #     ip_hash;
        #     server 192.168.80.121:80 down;
        #     server 192.168.80.122:80 weight=2;
        #     server 192.168.80.123:80;
        #     server 192.168.80.124:80 backup;
        # }
        # 在需要使用负载均衡的server中增加proxy_pass http://bakend/;
        # 每个设备的状态可设置为：
        # 1、down：表示当前的server暂时不参与负载
        # 2、weight：越大，负载的权重就越大
        # 3、max_fails：允许请求失败的次数默认为1，当超过最大次数时，返回proxy_next_upstream模块定义的错误
        # 4、fail_timeout：max_fails次失败后，暂停的时间
        # 5、backup：其它所有的非backup机器down或者忙的时候，请求backup机器，所以这台机器压力会最轻。
        
        # nginx支持同时设置多组的负载均衡，用来给不用的server来使用。
        # client_body_in_file_only设置为On 可以将client post过来的数据记录到文件中用来做debug
        # client_body_temp_path设置记录文件的目录 可以设置最多3层目录
        # location对URL进行匹配，可以进行重定向或者进行新的代理均衡
    }
    
    # 虚拟主机的配置
    server {
        # 监听端口
        listen       80;
        # 域名，可以有多个，用空格隔开
        server_name  localhost;
        index index.html index.htm index.php;
        root /data/www/w3cschool;

        #charset koi8-r;
        
        # 定义本虚拟主机的访问日志
        #access_log  logs/host.access.log  main;
        
        # 功能：location尝试根据用户请求中的URI来匹配/uri表达式，成功则执行{}里面的配置
        # 语法：location [ = | ~ | ~* | ^~ | @ ] /uri/ {}
        # 范围：server
        # 一般配置项
        # 1、以root方式设置资源路径
        # 语法格式：root path;
        # 2、以alias方式设置资源路径
        # 语法格式：alias path;
        # 3、访问首页
        # 语法格式：index file...;
        # 4、根据HTTP反悔吗重定向页面
        # 语法格式：error_page code [code ...] [= | =answer-code] uri | @named_location;
        # 5、是否允许递归使用error_page
        # 语法格式：recursive_error_pages [on|off];
        # 6、try_files
        # 语法格式：try_files_path1 [path2] uri;
        location / {
            root   html;
            index  index.html index.htm;
            # 反向代理配置
            # 不设upstream，直接指定ip和端口也是可以的，又或者proxy_pass http://jh.w3cschool.cn
            proxy_pass http://127.0.0.1:88;
            proxy_redirect off;
            proxy_set_header X-Real-IP $remote_addr;
            
            # 后端的web服务器可以通过X-Forwarded-For获取用户真实IP
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            
            # 以下是一些反向代理的配置，可选
            proxy_set_header Host $host;
            
            # 允许客户端请求的最大单文件字节数
            client_max_body_size 10m;
            
            # 缓冲区代理缓冲用户端请求的最大字节数
            # 如果把它设置为比较大的数值，例如256k，那么，无论使用firefox还是IE浏览器，来提交任意小于256k的图片，
            # 都很正常。如果注释该指令，使用默认的设置的话，也就是操作系统页面大小的两倍，8k或者16k，问题就出现了：
            # 无论使用firefox还是IE，提交一个比较大的200k左右的图片，都会返回500错误
            client_body_buffer_size 128k;
            
            # 表示使nginx阻止HTTP应答代码为400或者更高的应答
            proxy_intercept_errors on;
            
            # nginx跟后端服务器连接超时时间：发起握手等候响应超时时间（代理连接超时）
            proxy_connect_timeout 90;
            
            # 后端服务器数据回传时间：在规定时间内后端服务器必须传完所有的数据（代理发送超时）
            proxy_send_timeout 90;
            
            # 连接成功后，后端服务器响应时间：这个时候其实已经进入后端的排队之中等候处理了，所以也可以说是后端服务器处理请求的时间（代理接收超时）
            proxy_read_timeout 90;
            
            # 设置代理服务器（nginx）保存用户头信息的缓冲区大小
            # 设置从被代理服务器读取的第一部分应答的缓冲区大小，通常情况下这部分应答中包含一个小小的应答头，默认情况下
            # 这个值的大小为指令proxy_buffers中指定的第一个缓冲区的大小，不过可以将其设置为更小
            proxy_buffer_size 4k;
            
            # proxy_buffers缓冲区，网页平均在32k以下的设置
            # 设置用于读取应答（来自被代理服务器）的缓冲区数目和大小，默认情况也分为分页大小，根据操作系统的不同可能是4k或者8k
            proxy_buffers 4 32k;
            
            # 高负荷下缓冲大小（proxy_buffers*2）
            proxy_busy_buffers_size 64k;
            
            # 设置在写入proxy_temp_path时数据的大小，预防一个工作进程在传递文件时阻塞太长
            # 设定缓存文件夹大小，大于这个值，将从upstream服务器传
            proxy_temp_file_write_siez 64k;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        # 将php脚本代理到127.0.0.1:80监听的apache
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        # 将php脚本传递给在127.0.0.1:9000监听的FastCFI服务
        #location ~ \.php$ {
        #    root           html;
        #    fastcgi_pass   127.0.0.1:9000; # 监听地址
        #    fastcgi_index  index.php; # index
        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        #    include        fastcgi_params;
        #}

        # 设置图片缓存时间
        #location ~ .*.(gif|jpg|jpeg|png|bmp|swf)?$ {
        #    expires 10d;
        #}
        
        # 设置js和css缓存时间
        #location ~ .*.(js|css)?$ {
        #    expires 1h;
        #}
        
        # deny access to .htaccess files, if Apache's document roo concurs with nginx's one
        # 拒绝访问.ht文件
        #location ~ /\.ht {
        #    deny  all;
        #}
    }


    # another virtual host using mix of IP-, name-, and port-based configuration
    #
    #server {
    #    listen       8000;
    #    listen       somename:8080;
    #    server_name  somename  alias  another.alias;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}


    # HTTPS server
    #
    #server {
    #    listen       443;
    #    server_name  localhost;

    #    ssl                  on;
    #    ssl_certificate      cert.pem;
    #    ssl_certificate_key  cert.key;

    #    ssl_session_timeout  5m;

    #    ssl_protocols  SSLv2 SSLv3 TLSv1;
    #    ssl_ciphers  HIGH:!aNULL:!MD5;
    #    ssl_prefer_server_ciphers   on;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}

}
```

