---
layout: "@layouts/ArticleLayout.astro"
title: Nginx
description: Nginx
date: 2023-02-21 17:46:00
tags:
  - Nginx
  - Web
---

## 配置文件详解

```nginx

#user  nobody; # Nginx用户及用户组，Windows下不指定
worker_processes  1; # worker进程的数量，根据硬件调整，通常等于CPU数量或者2倍于CPU

# 错误日志路径
error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid; # pid(进程标识符)存放路径，Windows下也不需要

# worker_rlimit_nofile 204800;
# 这个指令是指当一个nginx进程打开的最多文件描述符数目，理论值应该是最多打开文件数（ulimit -n）与nginx进程数相除，但是nginx分配请求并不是那么均匀，所以最好与ulimit -n 的值保持一致。

# 现在在linux 2.6内核下开启文件打开数为65535，worker_rlimit_nofile就相应应该填写65535。

# 这是因为nginx调度时分配请求到进程并不是那么的均衡，所以假如填写10240，总并发量达到3-4万时就有进程可能超过10240了，这时会返回502错误。

events { # 事件块开始
    worker_connections  1024;
    # 每个工作进程的最大连接数量。根据硬件调整，和前面工作进程配合起来用，尽量大，但是别把cpu跑到100%就行。每个进程允许的最多连接数，理论上每台nginx服务器的最大连接数为。worker_processes*worker_connections
    use epoll;
    # 使用epoll的I/O 模型。linux建议epoll，FreeBSD建议采用kqueue，window下不指定。
    #keepalive_timeout 60;
    #client_header_buffer_size 4k;
    # 客户端请求头部的缓冲区大小。这个可以根据你的系统分页大小来设置，一般一个请求头的大小不会超过1k，不过由于一般系统分页都要大于1k，所以这里设置为分页大小。分页大小可以用命令getconf PAGESIZE 取得
    #open_file_cache max=65535 inactive=60s;
    # 这个将为打开文件指定缓存，默认是没有启用的，max指定缓存数量，建议和打开文件数一致，inactive是指经过多长时间文件没被请求后删除缓存。
    #open_file_cache_valid 80s;
 # 这个是指多长时间检查一次缓存的有效信息
    #open_file_cache_min_uses 1;
 # open_file_cache指令中的inactive参数时间内文件的最少使用次数，如果超过这个数字，文件描述符一直是在缓存中打开的，如上例，如果有一个文件在inactive时间内一次没被使用，它将被移除。
}# 事件块结束


# 设定http服务器，利用它的反向代理功能提供负载均衡支持
http { # http块开始
    include       mime.types; # nginx支持的媒体类型库文件
    default_type  application/octet-stream; # 默认的媒体类型

    # 格式化日志输出
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    # 通常web服务器放在反向代理的后面，这样就不能获取到客户的IP地址了，通过$remote_add拿到的IP地址是反向代理服务器的iP地址。反向代理服务器在转发请求的http头信息中，可以增加x_forwarded_for信息，用以记录原有客户端的IP地址和原来客户端的请求的服务器地址

    #access_log  logs/access.log  main;
    # 用了log_format指令设置了日志格式之后，需要用access_log指令指定日志文件的存放路径；
    
 #server_names_hash_bucket_size 128;
    #client_header_buffer_size 4k;
    #large_client_header_buffers 8 128k;
    #open_file_cache max=102400 inactive=20s;
    #client_max_body_size 300m;
    # 设定通过nginx上传文件的大小

    sendfile        on; # 开启高效传输模式
    # sendfile指令指定 nginx 是否调用sendfile 函数（zero copy 方式）来输出文件，对于普通应用，必须设为on。如果用来进行下载等应用磁盘IO重负载应用，可设置为off，以平衡磁盘与网络IO处理速度，降低系统uptime。
    #tcp_nopush     on;
    # 此选项允许或禁止使用socket的TCP_CORK的选项，此选项仅在使用sendfile的时候使用
    
    #proxy_connect_timeout 90;
    # 后端服务器连接的超时时间，发起握手等候响应的超时时间
    
    #proxy_read_timeout 180;
    # 连接成功后，等候后端服务器响应时间，其实已经进入后端的排队之中等候处理（也可以说是后端服务器处理请求的时间）
    
    #proxy_send_timeout 180;
    # 后端服务器数据回传时间_就是在规定时间之内后端服务器必须传完所有的数据
    
    #proxy_buffer_size 256k;
    # 设置从被代理服务器读取的第一部分应答的缓冲区大小，通常情况下这部分应答中包含一个小的应答头，默认情况下这个值的大小为指令proxy_buffers中指定的一个缓冲区的大小，不过可以将其设置为更小

    #keepalive_timeout  0;
    keepalive_timeout  65; # 连接超时

    #gzip  on;
    
    # 负载均衡配置
    upstream bakend {
        server 127.0.0.1:8027;
        server 127.0.0.1:8028;
        server 127.0.0.1:8029;
        hash $request_uri;
    }

    server { # 第一个server块开始，表示一个独立的虚拟主机站点
        listen       8000; # 提供服务的端口，默认是80
        server_name  localhost; # 提供服务的域名主机名

        #charset koi8-r;

        access_log  logs/host.access.log  main;

        location / { # 第一个location区块开始
            root   F:/xxnjh/tdtinstall; # 站点的根目录，相当于nginx的安装目录
            index  index.html index.htm; # 默认的首页文件，多个用空格分开
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html; # 出现第一个对应的http状态码时，使用50x.html返回
        location = /50x.html { # location区块开始，访问50x.html
            root   html; # 指定对应的站点目录为html
        }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        #location ~ \.php$ {
        #    root           F:/xxnjh/tdtinstall;
        #    fastcgi_pass   127.0.0.1:8000;
        #    fastcgi_index  index.php;
        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        #    include        fastcgi_params;
        #}
        
        #location / {
        #    if (!-e $request_filename) {
        #        rewrite ^/(.*)$ /index.php?q=$1 last;
        #    }
        #}


        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        location ~ /\.ht { # 以~开头表示启用正则表达式匹配uri，这里表示uri匹配精准.ht时，拒绝请求
            deny  all;
        }
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
    #    listen       443 ssl;
    #    server_name  localhost;

    #    ssl_certificate      cert.pem;
    #    ssl_certificate_key  cert.key;

    #    ssl_session_cache    shared:SSL:1m;
    #    ssl_session_timeout  5m;

    #    ssl_ciphers  HIGH:!aNULL:!MD5;
    #    ssl_prefer_server_ciphers  on;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}

} # http区块结束

```

**main**（全局设置）：main部分设置的指令将影响到其它所有部分设置；

**server**（主机设置）：server部分的指令主要用于指定虚拟主机域名、IP和端口；

**upstream**（上游服务器设置，主要为反向代理、负载均衡相关配置）：upstream的指令用于设置一系列的后端服务器，设置反向代理及后端服务器的负载均衡；

**location**（URL匹配特定位置后的设置）：location部分用于匹配网页位置（比如，根目录“/”，“/images”，等等）；

## Nginx文件结构

1、全局块：配置影响nginx全局的指令。一般有运行nginx服务器的用户组，nginx进程pid存放路径，日志存放路径，配置文件引入，允许生成worker process数等。

2、events块：配置影响nginx服务器或与用户的网络连接。有每个进程的最大连接数，选取哪种事件驱动模型处理连接请求，是否允许同时接受多个网路连接，开启多个网络连接序列化等。

3、http块：可以嵌套多个server，配置代理，缓存，日志定义等绝大多数功能和第三方模块的配置。如文件引入，mime-type定义，日志自定义，是否使用sendfile传输文件，连接超时时间，单连接请求数等。

4、server块：配置虚拟主机的相关参数，一个http中可以有多个server。

5、location块：配置请求的路由，以及各种页面的处理情况。

## Nginx常见的配置项

1.$remote_addr 与 $http_x_forwarded_for 用以记录客户端的ip地址；

2.$remote_user ：用来记录客户端用户名称；

3.$time_local ： 用来记录访问时间与时区；

4.$request ： 用来记录请求的url与http协议；

5.$status ： 用来记录请求状态；成功是200；

6.$body_bytes_s ent ：记录发送给客户端文件主体内容大小；

7.$http_referer ：用来记录从那个页面链接访问过来的；

8.$http_user_agent ：记录客户端浏览器的相关信息；

每个指令必须有分号结束
